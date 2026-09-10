---
name: math-divide
description: Divide numbers and report the quotient, refusing division by zero.
category: core
---

# Divide

Divide numbers, and refuse the one case that has no answer.

## When to use

- The request asks for a quotient, a rate, a per-unit value, or an average.
- A total has to be split across a number of parts.

## Instructions

1. Refuse a zero divisor and say so. Do not report `Infinity` or `NaN` as an answer.
2. Identify the dividend and the divisor before computing. Order matters.
3. Say whether the quotient is exact or rounded, and to how many decimal places.
4. For a remainder-based question, give the whole part and the remainder separately.

## Examples

- `7 / 2` = 3.5
- `1 / 3` = 0.333 (rounded to 3 decimal places)
- `7 / 2` = 3 remainder 1 (whole-number division)
- `5 / 0` is undefined — report the error rather than a value

## Pitfalls

- **Division by zero.** In JavaScript this yields `Infinity`, or `NaN` for `0 / 0`, instead
  of throwing. The bad value then propagates through every later step and surfaces
  somewhere with no visible connection to its cause.
- **Averages of averages.** Dividing a set of averages does not give the average of the
  underlying data unless every group is the same size.
