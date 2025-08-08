# RMS Trigonometry Example
An example of implementing trigonometric functions in Aoe2 random map scripts.
[Update 141935](https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/) for Age of Empires II: Definitive Edition adds the ability to perform basic arithmetic in random map scripts.
Originally this repository housed an example of implementing [trigonometric functions on that patch](https://github.com/twestura/RMS-Trigonometry-Example/blob/main/old-notes.md).
However, the [July 2025 Public Update Preview](https://steamcommunity.com/app/813780/discussions/34/597406481786246211/?snr=2___) introduces breaking changes to how arithmetic expressions are evaluated.
The implementation is updated to be compatible with this patch.

## RMS Math Notes

Math operations are still in their infancy, but nevertheless are extremely useful.
Here are some of their basic details:

- There are five operators: `+`, `-`, `*`, `/`, and `%`.
- All operations have the same precedence and are left-associative.
- Parentheses cannot be used to nest subexpressions.
- Floating point values of `inf`, `-inf`, and `-0` are supported.
- The four operations of `+`, `-`, `*`, and `/` do not round.
- The remainder operator casts both operands to ints before computing the remainder.
- There are no arrays.
- There are no comparison operators.
- If statements cannot interact with constants (only with RMS labels).

## Trigonometry

### The Approximation Formula

To compute the sine function we use [Bhāskara I's sine approximation formula](https://en.wikipedia.org/wiki/Bh%C4%81skara_I%27s_sine_approximation_formula), valid for degrees $x$ where $0 \le x \le 180$:

$$\sin{x} \approx \frac{4x(180 - x)}{40500 - x(180 - x)}.$$

The value of cosine is computed with the trigonometric identity

$$\cos{x} = \sin(90 - x).$$

The following code snippet computes the sine approximation:

```text
#const DEGREES rnd(-10000,10000)
#const R (DEGREES + 360000 % 360 * -1 + 180)
#const S (R + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf * 2 - 1)
#const ARG_SUPP (180 * S - R * R)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (S * 4 * ARG_SUPP / DENOMINATOR)
```

The constant `DEGREES` is the input to the trigonometric functions; here it generated randomly for demonstration purposes.
Set this constant to a specific number of degrees to compute the sine of the respective angle.

### Oh for a Proper Modulus Operator

First we compute a "remainder" `R` constant.

```text
#const R (DEGREES + 360000 % 360 * -1 + 180)
```

Since sine is periodic, we might desire the integer $r$ such that $0 \le r \le 359$ and ${\sin(r) = \sin(\mathtt{DEGREES})}$, which occurs when $r$ is the remainder of `DEGREES` when divided by $360$.
That is, ${r = \mathtt{DEGREES} \bmod 360}$.
We can use the RMS remainder operator to compute `DEGREES % 360`.
However, this expression evaluates to the modulus only when `DEGREES` is nonnegative.
To support negative numbers, we add a large multiple of `360` before taking the remainder.

Rather than use the interval of $[0, 359]$, it'll be more convenient to work around the lack of if statements by using the interval $[{-180}, 180]$.
Thankfully we can use the trigonometric identity

$$\sin x = \sin(180 - x).$$

At the end of our line of code, we multiply by `-1` and add `180` in order to use this identity.
That leaves us with a number $R$ that is in $[{-179}, 180]$ and satisfies $\sin R = \sin(180 - \mathtt{DEGREES})$.

### Zeno's Signum

For values of $R$ in $[0, 180]$, we can plug them directly into our approximation formula.
But what about $R$ in $[{-179}, {-1}]$?
For these values we use the trigonometric identity:

$$\sin{x} = -\sin({-x}).$$

In order to use this identity, we need to:

- Detect if ${R < 0}$.
- Compute $\sin({-R})$.
- Negate the computed sine value.

And all of that must be done without if statements!

For our implementation, we compute a value $S$ as follows:

$$S = \begin{cases}
    1 & x > 0,\\
    {-1} & x \le 0.
\end{cases}$$

This value is similar to the signum of $R$, except that when $R = 0$, we set $S = {-1}$.

It turns out that the trigonometric identity can be rewritten, using $S$ in place of the negative signs.
We have three cases where we show

$$S \cdot \sin(S R) = \sin R.$$

- ${R > 0}$, then ${S = 1}$ and the $S$ values are just multiplications by $1$.
- ${R = 0}$, in which case we have ${{-1} \cdot \sin({-1} \cdot 0) = 0 = \sin 0}$.
- ${R < 0}$, where ${S = -1}$ and by the aforementioned trigonometric identity, we have ${-\sin({-R}) = \sin R}$.

Let's examine the line where we compute `S`:

```text
#const S (R + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf + 1 / 2 % -inf * 2 - 1)
```

Here we're using three properties:
- `1 + 1 / 2 % -inf` evaluates to `1`.
- `0 + 1 / 2 % -inf` evaluates to `0`.
- ${181 < 256 = 2^8}$.

We abuse the semantics of the `%` operator to round towards zero.
The expression `X % -inf` casts both of its operands to ints and evaulates to `trunc(X) % MIN_INT`.
The sign of the returned value is determined by the sign of the left operand.
By using this cast in combination with the division by $2$, we repeatedly halve the value while keeping it gated to $1$ for postiive vales and $0$ for nonpositive values.
Since ${{{|{R}|} + 1} \le 181 < 2^8}$, we apply this sequence of operations eight times.

### Finishing the Computation

The rest of the code consists of using the approximation formula.

```text
#const ARG_SUPP (180 * S - R * R)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (S * 4 * ARG_SUPP / DENOMINATOR)
```

Both the numerator and the denominator of the approximation formula use the product of the angle and its supplementary angle:

$$x(180 - x).$$

Plugging in ${x = SR}$, we can rewrite this product as:

$$x(180 - x) = (180 - x)x = (180 - S R) S R  = (180 \cdot S - S^2 R)R = (180 \cdot S - R)R.$$

The rightmost equality follows because ${S^2 = 1}$.

The output of the example can be seen in the player resource Food and Gold values when launching the accompanying [example map script](https://github.com/twestura/RMS-Trigonometry-Example/blob/main/Math%20Trig%20July%202025.rms) in Single Player.
The example multiplies the `SIN` and `COS` values by `10000` to display more decimal digits.
