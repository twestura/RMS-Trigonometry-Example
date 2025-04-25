# RMS-Trigonometry-Example
An example of implementing trigonometric functions in Aoe2 random map scripts.

## RMS Math Notes

Update 141935 for Age of Empires II: Definitive Edition adds the ability to perform basic arithmetic in random map scripts: [https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/](https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/)

With this update various tasks that required a preprocessor and static code generation now can be computed dynamically.
Building tools with these operationas can be quite tricky, as they are subject to the following limitations:

- There are only four operations: `+`, `-`, `*`, and `/`.
- There is no remainder or modulus operator.
- All operators have the same precedence and are left-associative.
- Parentheses cannot be used to nest subexpressions.
- All divisions round, even as internal steps of a computation.
- There are no arrays.
- There are no comparison operators.
- If statements cannot interact with constants (only with RMS labels).

For example, `2 + 3 * 5` evaluates not to `17` but to `25`.
And `1 / 2 / 2` evaluates not to `0` (rounding `0.25`) but to `1` (rounding `0.5` twice).

## Trigonometry

### The Approximation Formula

Despite these limitations, it nevertheless is possible to compute more advanced functions.
This repository is an example of computing the sine and cosine functions.
We use [Bhāskara I's sine approximation](https://en.wikipedia.org/wiki/Bh%C4%81skara_I%27s_sine_approximation_formula), valid for degrees $x$ where $0 \le x \le 180$:

$$\sin{x} = \frac{4x(180 - x)}{40500 - x(180 - x)}.$$

The value of cosing can be computed using this formula as well as the trigonometric identity

$$\cos{x} = \sin{(90 - x)}.$$

The following code snipped computes the sine approximation:

```text
#const PADDING 100000
#const DEGREES rnd(-9999,9999)
#const REMAINDER' (DEGREES / 360 * -360 + DEGREES)
#const FLIP (REMAINDER' / 2 / 2 / 2 / 2 / 2 / 2 / 2 / 2 * 2 + 1 / 2 / 2)
#const ARG (FLIP * REMAINDER')
#const ARG_SUPP (180 - ARG * ARG)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (FLIP * 4 * ARG_SUPP * PADDING / DENOMINATOR)
```

First we define a `PADDING` value.
Since every division rounds, the `PADDING` is a power of ten multiplier for shifting the decimal point.
The values of sine are real numbers falling in `[-1, 1]`, and it would be very boring if our approximation only rounded to `-1`, `0`, and `1` due to the fraction in the formula.

Next, an input in `DEGREES` is generated randomly for demonstration purposes.
Set this constant to any number of degrees to compute the sine of the respective angle if you want to test a specific value.

### Oh For a Remainder Operator

Now comes the `REMAINDER'` constant.

```text
#const REMAINDER' (DEGREES / 360 * -360 + DEGREES)
```

Since sine is periodic, we might desire the integer $r$ such that $0 \le r \le 359$ and ${\sin(r) = \sin(\mathtt{DEGREES})}$, which occurs when $r$ is the remainder of `DEGREES` when divided by $360$.
That is, ${r = \mathrm{DEGREES} \bmod 360}$.
RMS expressions do not present a remainder operator, but we can compute the remainder $r$ of an integer $n$ divided by $360$ using the formula

$$r = n - {\bigg\lfloor\frac{n}{360}\bigg\rfloor} \cdot 360,$$

where $0 \le r \le 359$.

While some programming languages provide a form of integer division equivalent to taking the floor, RMS affords no such luxury.
Rounding is our only recourse, and it produces either the floor or the ceiling of $n / 360$.
The floor and ceiling are equal only when $n$ is a multiple of $360$, in which case the remainder is $0$.
When the floor and ceiling are different, we have:

$$n - {\bigg\lceil\frac{n}{360}\bigg\rceil} \cdot 360 = n - \Bigg({\bigg\lfloor\frac{n}{360}\bigg\rfloor} + 1\Bigg) \cdot 360 = n - {\bigg\lfloor\frac{n}{360}\bigg\rfloor} \cdot 360 - 360 = r - 360.$$

When we take the ceiling, we're just subtracting an extra $360$ and shifting $r$ one period to the left.
And because the ceiling only is taken when rounding up, this shift only occurs for values of $n$ where $180 \le n \bmod 360 \le 360$.
Thus we set

$$r' = n - 360 \cdot \mathrm{round}\left(\frac{n}{360}\right) = \mathrm{round}(n / 360) \cdot ({-360}) + n,$$

with ${{-180} \le r' \le 180}$ and ${\sin(r')  = \sin(\mathtt{DEGREES})}$.
Implmenting this function in RMS code, we use the rightmost expression.
It is rearranged to read from left-to-right, accounting for the lack of different operator precedence levels.

### The Hack

Oh boi, it's time to discuss our good friend `FLIP`.

```text
#const FLIP (REMAINDER' / 2 / 2 / 2 / 2 / 2 / 2 / 2 / 2 * 2 + 1 / 2 / 2)
```

We've so far computed a value $r'$ such that ${-180 \le r' \le 180}$, and now we need to compute $\sin{} r'$.
Recall that our approximation formula is valid only for ${0 \le r' \le 180}$.
That's fine for the upper half of the period, but what about the lower half?
Well we're in luck, as we can use the handy dandy trigonometric identity:

$$\sin{({-x})} = -\sin{} x.$$

For values $r'$ where ${-180 \le r' \le 0}$, we negate $r'$, compute the sine, and negate the output.
For this process, we define a constant `FLIP` such that:

- `FLIP` equals $1$ when ${r' \ge 0}$,
- `FLIP` equals $-1$ when ${r' < 0}$.

We'll multiply by `FLIP`, and doing so will perform the negations only where necessary.

To compute `FLIP`, we'll use three properties:
- `1 / 2` evaluates to `1`.
- `-1 / 2` evaluates to `-1`.
- ${180 < 256 = 2^8}$.

The first two properties arise from the rounding behavior of division, and lead to an interesting property.
If we take any nonzero number and divide by $2$ repeatedly, eventually we reach $1$ if the number is positive and ${-1}$ if the number is negative.
Imagine what Zeno would think if he saw this!
Now, since ${-2^8 < -180 \le r' \le 180 < 2^8}$, we simply divide $r'$ by $2$ eight times!

```text
REMAINDER' / 2 / 2 / 2 / 2 / 2 / 2 / 2 / 2
```

We're left with a value that is either $1$, $0$, or ${-1}$ for ${r' > 0}$, ${r' = 0}$, and ${r' < 0}$, respectively.
Because we're relying upon the rounding, we must perform the divisions separately—we cannot simply divide by $256$.

Now that we've divided by $2$ eight times, what comes next?
Multiplying by $2$ of course!
Recall that our goal is for `FLIP` to be $1$ or ${-1}$ depending on whether ${r' \ge 0}$ or ${r' < 0}$.
We need to handle the case where ${r' = 0}$.
We performing the following, keeping in mind that all operators have the same precedence:

- Multiply by $2$.
- Add $1$.
- Divide by $2$ twice.

These operations are summarized in the following table.

| Initial $r'$ Value | `/ 2` x8 | `* 2`  | `+ 1`  | `/ 2`  | `/ 2`  |
| -----------------: | -------: | -----: | -----: | -----: | -----: |
| $r' > 0$           | $1$      | $2$    | $3$    | $2$    | $1$    |
| $r' = 0$           | $0$      | $0$    | $1$    | $1$    | $1$    |
| $r' < 0$           | ${-1}$   | ${-2}$ | ${-1}$ | ${-1}$ | ${-1}$ |

And that concludes our calcuation of `FLIP`.

### Finishing the Computation

The rest of the code consists of plugging the value into the approximation formula.

```text
#const ARG (FLIP * REMAINDER')
#const ARG_SUPP (180 - ARG * ARG)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (FLIP * 4 * ARG_SUPP * PADDING / DENOMINATOR)
```

Notable here are the two usages of `FLIP` to handle the identity ${\sin{({-x})} = -\sin{(x)}}$.
We also apply the `PADDING` to the numerator of the fraction.
The padding preserves decimal precision, and it's usage can be seen in the player resource Food and Gold values when launching the map script in Single Player.
