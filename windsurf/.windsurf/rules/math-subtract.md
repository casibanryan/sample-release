---
name: math-subtract
description: Subtract numbers and report the difference, stating which operand came first.
category: core
---

# Subtract

Subtract numbers and make the direction of the subtraction explicit.

## When to use

- The request asks for a difference, remainder, decrease, or change between two values.
- A total has to be reduced by a known amount.

## Instructions

1. Identify which value is being subtracted **from** which before computing anything.
2. When the wording is ambiguous, state the reading used.
3. Keep the sign. A result below zero is a valid answer, not an error to be flipped.
4. Report the expression and the result together.

## Examples

- `10 - 4` = 6
- `4 - 10` = -6
- `2.5 - 0.5` = 2

## Pitfalls

- **Order.** This is the first of the basic operations that is not commutative. "The
  difference between 4 and 10" has two defensible readings and only one is what the asker
  meant — pick one and say which.
- **Silent sign flip.** Reporting 6 for `4 - 10` because a negative looked like a mistake
  is worse than reporting -6, which is simply the answer.
