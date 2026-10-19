My translation job told me it had translated 8 product names out of 30. The response was valid JSON, it matched my strict schema, and nothing threw. That was the problem.

Structured Outputs promises that the model always follows your JSON Schema. Turns out "follows the schema" and "answered every question" are two different things.

## The Problem

In a project I work on, product names come from an ERP feed in Ukrainian, and a queued job translates them with OpenAI. The job sends names in chunks of 30, numbered in the prompt, and asks `gpt-4o-mini` for a structured answer: `response_format` with a JSON Schema and `strict: true`. With strict on, the model can only produce JSON that matches the schema.

This was the response format:

```php
// BEFORE
private function responseFormat(): array
{
    return [
        'type' => 'json_schema',
        'json_schema' => [
            'name' => 'product_translations',
            'strict' => true,
            'schema' => [
                'type' => 'object',
                'additionalProperties' => false,
                'required' => ['translations'],
                'properties' => [
                    'translations' => [
                        'type' => 'array',
                        'items' => [
                            'type' => 'object',
                            'additionalProperties' => false,
                            'required' => ['n', 'en'],
                            'properties' => [
                                'n' => ['type' => 'integer'],
                                'en' => ['type' => 'string'],
                            ],
                        ],
                    ],
                ],
            ],
        ],
    ];
}
```

Looks safe. Strict mode is on, and every item needs a number and a translation. The prompt also said, in plain words, "return exactly as many items as there are texts".

Then I ran a local bench on 2000 real names before rollout. One chunk of 30 long names came back with **8 translations**. I sent that chunk four times: 8 of 30 every time, with the same 267 completion tokens. In a later probe it did come back complete, but only once in 5 attempts.

And the job was happy with it. It wrote 8 rows and logged the other 22 as rejected. No exception, no retry, nothing red. Over the whole bench, 22 names were lost per run this way.

Could a count check with a retry save it? Not really. The same chunk gave the same short answer almost every time, so a retry loop just pays for the same 8 rows again. A count check is still worth having, but as a backstop, not as the fix.

## Why Strict Mode Let It Through

Let me be real: strict mode did exactly what it promised. The schema says `translations` is an array of `{n, en}` objects. It says nothing about how long that array is. An array of 8 items is valid. So is an empty one.

Strict mode guarantees the **shape** of the answer. It does not guarantee that every input got an answer. The "return exactly 30 items" rule lived only in the prompt, and the model can ignore the prompt.

Strict mode also can't help when the answer hits the token limit (`finish_reason: length`) or the model refuses. Then you get cut-off JSON or no JSON at all, so check `finish_reason` before you parse.

## The Fix: One Required Key per Input

The fix is simple: make the schema depend on the input. Instead of a free-length array, the answer is an object with **one property per row number**. Strict mode makes every property required, so the number of properties is now the number of answers.

```php
// AFTER
private function responseFormat(int $count): array
{
    $keys = array_map(strval(...), range(1, $count));

    return [
        'type' => 'json_schema',
        'json_schema' => [
            'name' => 'product_translations',
            'strict' => true,
            'schema' => [
                'type' => 'object',
                'additionalProperties' => false,
                'required' => $keys,
                'properties' => array_combine(
                    $keys,
                    array_fill(0, $count, ['type' => 'string']),
                ),
            ],
        ],
    ];
}
```

For a chunk of 3, the schema becomes:

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["1", "2", "3"],
  "properties": {
    "1": {"type": "string"},
    "2": {"type": "string"},
    "3": {"type": "string"}
  }
}
```

Now a short answer is not a valid answer. The model can't skip row 17, because `"17"` is required, and it can't answer row 17 twice, because an object has only one `"17"` key.

The call builds the schema from the chunk it sends:

```php
$response = $client->chat()->create([
    'model' => 'gpt-4o-mini',
    'messages' => $messages,
    'response_format' => $this->responseFormat(count($ukTexts)),
]);
```

Heads up: return early on an empty chunk. `range(1, 0)` is `[1, 0]` in PHP, and `array_combine()` throws. One limit to know: the docs cap a schema at 5000 object properties in total. Fine for chunks of 30, but this pattern doesn't scale to a chunk of 10,000.

The parser reads the row number straight from the key:

```php
public function parse(string $content): array
{
    $decoded = json_decode($content, true);

    if (!is_array($decoded)) {
        return [];
    }

    $result = [];

    foreach ($decoded as $number => $en) {
        if (!is_string($en) || !is_numeric($number) || (int) $number < 1) {
            continue;
        }

        // The prompt numbers from 1, the caller indexes from 0.
        $result[(int) $number - 1] = trim($en);
    }

    return $result;
}
```

`json_decode()` turns `"1"`, `"2"` into integer array keys, so `$number` arrives as an `int`. The `is_numeric()` check only drops non-numeric keys, which is enough when strict mode already rejects any key you didn't list.

## What About minItems?

Why not keep the array and add `minItems: 30`?

When I made this fix, I believed OpenAI rejects `minItems` in strict mode. That was true in 2024 (there's a community thread about it). **It's not true today.** The current docs list `minItems` and `maxItems` as supported in strict mode, except for fine-tuned models. I also checked it with a live call: `gpt-4o-mini`, strict, `minItems: 30, maxItems: 30`. The API accepted it and returned 30 items.

So `minItems` is a real option now. But it pins only the **count**. Thirty items with `n` values `1, 2, 2, 4...` are still valid: row 2 has two answers and row 3 has none. Your parser overwrites row 2, loses row 3, and a count check on the response still says 30 of 30.

With a key per row, that answer can't pass: row 17 has exactly one slot, and it is required. What the model puts in that slot is still up to the model. Row 18's translation under `"17"`, or an empty string, passes either schema. That's the job of the quality guard, not the schema.

## Results

Same 2000-name sample, same model, same prompt, before and after:

| | Free-length array | Key per row |
|---|---|---|
| Problem chunk (30 long names) | 8/30 on 4 of 4 attempts | 30/30 on 4 of 4 attempts |
| Names lost by the model, per run | 22 | **0** |
| Names translated and accepted | 1974 | **1997** |
| Rejected by the quality guard | 4 | 3 |

The rejected ones are a different story: those are bad translations caught by a separate guard, and that's fine. The lost ones were never there to check.

## TL;DR

- `strict: true` guarantees the shape of the JSON, not that every input got an answer.
- A free-length array lets the model answer short and still pass the schema.
- Build the schema per request: one required property per input row.
- `minItems` is supported in strict mode today, but it pins only the count: duplicate and missing row numbers still pass.
- Keys stop skipped and doubled rows, not wrong content. Keep a `finish_reason` check and a quality guard behind them.
- If a rule lives only in the prompt, assume the model will break it on some chunk.

💡 If your structured output has a list that should match your input, ask yourself: can a shorter answer pass this schema? If yes, assume it will.

---

- [OpenAI: Structured model outputs, supported schemas](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI community: "min- & maxItems are not supported in Structured Output" (2024)](https://community.openai.com/t/min-maxitems-are-not-supported-in-structured-output/958567)
- [OpenAI: Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/)
- [openai-php/client](https://github.com/openai-php/client)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
