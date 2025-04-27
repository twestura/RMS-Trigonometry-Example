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

The value of cosine can be computed using this formula along with the trigonometric identity

$$\cos{x} = \sin{(90 - x)}.$$

The following code snipped computes the sine approximation:

```text
#const PADDING 100000
#const DEGREES rnd(-9999,9999)
#const R (DEGREES / 360 * -360 + DEGREES)
#const SGN (R / 2 / 2 / 2 / 2 / 2 / 2 / 2 / 2)
#const ARG_SUPP (180 * SGN - R * R)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (SGN * 4 * ARG_SUPP * PADDING / DENOMINATOR)
```

First we define a `PADDING` value.
Since every division rounds, the `PADDING` is a power of ten multiplier for shifting the decimal point.
The values of sine are real numbers falling in `[-1, 1]`, and it would be very boring if our approximation only rounded to `-1`, `0`, and `1` due to the fraction in the formula.

Next, an input in `DEGREES` is generated randomly for demonstration purposes.
Set this constant to any number of degrees to compute the sine of the respective angle if you want to test a specific value.

### Oh For a Remainder Operator

Now comes the `R` constant.

```text
#const R' (DEGREES / 360 * -360 + DEGREES)
```

Since sine is periodic, we might desire the integer $r$ such that $0 \le r \le 359$ and ${\sin(r) = \sin(\mathtt{DEGREES})}$, which occurs when $r$ is the remainder of `DEGREES` when divided by $360$.
That is, ${r = \mathtt{DEGREES} \bmod 360}$.
RMS expressions do not present a remainder operator, but we can compute the remainder $r$ of an integer $n$ divided by $360$ using the formula

$$r = n - {\bigg\lfloor\frac{n}{360}\bigg\rfloor} \cdot 360,$$

where $0 \le r \le 359$.

But here we have a problem.
While some programming languages provide a form of integer division equivalent to taking the floor, RMS affords no such luxury.
Rounding ${n / 360}$ to the nearest integer is our only recourse.
Such rounding produces either the floor or the ceiling.

The floor and ceiling are equal only when $n$ is a multiple of $360$, in which case the remainder is $0$.
When the floor and ceiling are different, we have:

$$n - {\bigg\lceil\frac{n}{360}\bigg\rceil} \cdot 360 = n - \Bigg({\bigg\lfloor\frac{n}{360}\bigg\rfloor} + 1\Bigg) \cdot 360 = n - {\bigg\lfloor\frac{n}{360}\bigg\rfloor} \cdot 360 - 360 = r - 360.$$

Hence taking the ceiling subtracts an extra $360$ and shifts $r$ one period to the left.
And because the ceiling only is taken when rounding up, this shift only occurs for values of $n$ where $180 \le n \bmod 360 \le 360$.
Thus we set

$$r = n - 360 \cdot \mathrm{round}\left(\frac{n}{360}\right) = \mathrm{round}(n / 360) \cdot ({-360}) + n,$$

with ${{-180} \le r' \le 180}$ and ${\sin(r')  = \sin(\mathtt{DEGREES})}$.
Implmenting this function in RMS code, we use the rightmost expression.
It is rearranged to read from left-to-right, accounting for the lack of different operator precedence levels.

Now we might look at the interval of $[-180, 180]$ and lament that it's not our original target of $[0, 360]$.
But the interval still emcompasses one complete period of sine.
And, as we'll see in the next section, actually is more convenient.

### Zeno's Signum

For values of $r$ in $[0, 180]$, we can plug them directly into our approximation formula.
But what about $r$ in $[-180, -1]$?
For these values we use the trig identity:

$$\sin{} x = -\sin{({-x})}.$$

In order to use this identity, we need to:

- Detect if ${r < 0}$.
- Compute $\sin{(-r)}$.
- Negate the computed sine value.

And all of that must be done without if statements!

For our implementation, we'll use the signum function, defined by:

$$\mathrm{sgn}(x) = \begin{cases}
    1 & x > 0,\\
    0 & x = 0,\\
    -1 & x < 0.
\end{cases}$$

This function is also called the "sign" function, but we'll use the term "signum" to avoid phoenic ambiguity between sign and sine.
And we'll name our constant `SGN` becuase `SIGN` already is the constant for the signpost object in Aoe2.

Let's examine the line where we compute `SGN`:

```text
#const SGN (R / 2 / 2 / 2 / 2 / 2 / 2 / 2 / 2)
```

Here we're using four properties:
- `1 / 2` evaluates to `1`.
- `0 / 2` evaluates to `0`.
- `-1 / 2` evaluates to `-1`.
- ${180 < 256 = 2^8}$.

The rounding behavior of RMS division leads to an interesting property.
If we take any nonzero number and divide by $2$ repeatedly, eventually we reach $1$ if the number is positive and ${-1}$ if the number is negative.
Now, since ${|r| \le 180 < 2^8}$, we simply divide $r$ by $2$ eight times!

We're left with a value that is either $1$, $0$, or ${-1}$ for ${r > 0}$, ${r = 0}$, and ${r < 0}$, respectively.
And that's exactly the value we want for the signum of $r$.
Because we're relying upon the rounding of each division, we must perform the divisions separately—we cannot simply divide by $256$.

### Finishing the Computation

For brevity, let $s$ denote the `SGN` value.
We compute

$$s \sin{(sr)}.$$

We have three cases, and in each case we have that ${s \sin{(sr)} = \sin{} r}$:

- ${r > 0}$, then ${s = 1}$ and the $s$ values are just multiplications by $1$.
- ${r = 0}$, in which case we have ${0 \cdot \sin{(0 \cdot 0)} = 0}$.
- ${r < 0}$, where ${s = -1}$ and by the aforementioned trig identity, we have ${-\sin{({-r})} = \sin{} r}$.

The rest of the code consists using the approximation formula.

```text
#const ARG_SUPP (180 * SGN - R * R)
#const DENOMINATOR (40500 - ARG_SUPP)
#const SIN (SGN * 4 * ARG_SUPP * PADDING / DENOMINATOR)
```

Both the numerator and the denominator of the approximation formula use the product of the angle and its supplementary angle:

$$x(180 - x).$$

Pluggin in $x = sr$, we can rewrite this product as:

$$x(180 - x) = (180 - sr)sr = (180s - s^2r)r = (180s - r)r.$$

The rightmost equality follows because ${s^2 = 1}$ when $r$ is nonzero.

We also apply the `PADDING` to the numerator of the fraction.
The padding preserves decimal precision, and it's usage can be seen in the player resource Food and Gold values when launching the map script in Single Player.
