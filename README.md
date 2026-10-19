# Schema-Valid Is Not Complete: Strict Output Answered 8 of 30

![Schema-Valid Is Not Complete: Strict Output Answered 8 of 30](assets/poster.jpg)

My translation job told me it had translated 8 product names out of 30. The response was valid JSON, it matched my strict schema, and nothing threw.

Strict mode guarantees the shape of the answer, not that every input got one.

## What's Inside

- Why an OpenAI `strict: true` schema with a free-length array accepts a short answer
- The fix: build the schema per request, one required key per input row
- A parser that maps answers back to rows by key
- `minItems` in strict mode today: supported, but it pins only the count
- Before/after numbers from a 2000-name bench: 22 lost names per run down to 0

## 📎 Read Full

[Schema-Valid Is Not Complete: Strict Output Answered 8 of 30](https://dev.to/tegos/PLACEHOLDER)
