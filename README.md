# RMS-Trigonometry-Example
An example of implementing trigonometric functions in Aoe2 random map scripts.

Update 141935 for Age of Empires II: Definitive Edition adds the ability to perform basic arithmetic in random map scripts: [https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/](https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/)

With this update various tasks that required a preprocessor and static code generation now can be computed dynamically within the map scripts themselves.
Building tools with these operationas can be quite tricky, as they are subject to the following limitations:

- There are only four operations: `+`, `-`, `*`, and `/`.
- There is no remainder or modulus operator.
- All operators have the same precedence and are left-associative.
- Parentheses cannot be used to nest subexpressions.
- All divisions round, even as internal steps of a computation.
- There are no arrays.
- There are no comparison operators.
- If statements cannot interact with constants (only with RMS labels).

For example, `2 + 3 * 5` evaluates not to `17` but to `25`, and `1 / 2 / 2` evaluates not to `0` (rounding `0.25`) but to `1` (rounding `0.5` twice).

Nevertheless it is possible to compute more advanced functions using just these basic operations.
This repository is an example of computing the sine and cosine functions.
We use [Bhāskara I's sine approximation](https://en.wikipedia.org/wiki/Bh%C4%81skara_I%27s_sine_approximation_formula), valid for degrees `x` in the interval `[0, 180]`:

$$\sin{x} = \frac{4x(180 - x)}{40500 - x(180 - x)}.$$

The value for cosine is calculated using the trigonometric identity $\cos{x} = \sin(90 - x)$.

The computation proceeds in several steps:

1. The "modulus" of the input angle is computed to bring the input to an equivalent angle in the range `[0, 360]`.
2. The location of the new angle, within `[0, 179]` or `[180, 360]`, is determined, and the angle is translated to be in `[0, 180]`, if necessary.
3. The sine approximation formula is applied, with padding multiplied to the numerator to preserve digits.
4. If necessary, the identity $-\sin{x} = \sin(-x)$ is applied if the modified angle was in the "upper half" range `[180, 360]` after the modulus was taken.

Note the output values must be multiplied by the desired radius and then divided by the padding.
This map script displays the calculation with Wood, Food, Gold, and Stone representing the input angle, the angle modded by 360, the cosine, and the sine, respectively.
Flags in the middle of the map are positioned based on the calculated values.

The following techniques are used:

- By multiplying by a power of $10$, the digits in the output may be preserved, despite the rounding.
- By repeatedly dividing by `2` and taking advantage of the rounding, we can determine whether or not a number is positive, zero, or negative.
- If statements may be replicated by using a "boolean" value of `0` or `1` in a multiplication. More specifically, to replicate `if b then x else y`, we can use `b * x + (1 - b) * y` (and, of course, this expression must be rearranged to account for the precedence and the lack of parentheses).
