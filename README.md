# RMS Trigonometry Example
An example of implementing trigonometric functions in Aoe2 random map scripts.

## Version History

[Update 141935](https://www.ageofempires.com/news/a-sneak-peek-at-new-content-coming-to-age-of-empires-ii-definitive-edition/) for Age of Empires II: Definitive Edition adds the ability to perform basic arithmetic in random map scripts.
The [July 2025 Public Update Preview](https://steamcommunity.com/app/813780/discussions/34/597406481786246211/?snr=2___) introduces breaking changes to how arithmetic expressions are evaluated.
These notes contain an up-to-date implementation that works on the current version of the game.

- [April 2025](https://github.com/twestura/RMS-Trigonometry-Example/blob/main/previous-versions/Notes%20April%202025.md): the original implementation.
- [July 2025](https://github.com/twestura/RMS-Trigonometry-Example/blob/main/previous-versions/Notes%20July%202025.md): updated for the breaking changes in the July 2025 Public Update Preview.
- July 2026 (this page): simplifies the July 2025 implementation.

## RMS Math Notes

Math operations are still in their infancy, but nevertheless are extremely useful.
Here are some of their basic details:

- There are five operators: `+`, `-`, `*`, `/`, and `%`.
- All operations have the same precedence and are left-associative.
- Parentheses cannot be used to nest subexpressions.
- Floating point values of `inf`, `-inf`, and `-0` are supported.
- The four operations of `+`, `-`, `*`, and `/` do not round.
- The remainder operator `%` casts both operands to ints before computing the remainder.
- There are no arrays.
- There are no comparison operators.
- If statements cannot interact with constants (only with RMS labels).

## Trigonometry

### The Approximation Formula

To compute the sine function we use [Bhāskara I's sine approximation formula](https://en.wikipedia.org/wiki/Bh%C4%81skara_I%27s_sine_approximation_formula), valid for degrees $x$ where $0 \le x \le 180$:

$$\sin{x} \approx \frac{4x(180 - x)}{40500 - x(180 - x)}.$$

The value of cosine is computed with the trigonometric identity

$$\cos{x} = \sin(90 + x).$$

The following code snippet computes the sine and cosine approximations:

```text
#const DEGREES rnd(-10000,10000)
#const R (DEGREES % 360 + 360 % 360 * -1 + 180)
#const S (R * 2 + 1 % 2)
#const P (180 * S - R * R)
#const D (40500 - P)
#const SIN (S * 4 * P / D)
#const CR (270 - R % 360 * -1 + 180)
#const CS (CR * 2 + 1 % 2)
#const CP (180 * CS - CR * CR)
#const CD (40500 - CP)
#const COS (CS * 4 * CP / CD)
```

The constant $\mathtt{DEGREES}$ is the input to the trigonometric functions.
Here it is generated randomly for demonstration purposes.
In the example map script, set this constant to a specific number of degrees to compute the sine and cosine of the respective angle.

### Code Overview

The main difficulty involves the lack of conditional branching for RMS math.
We rely on various tricks to avoid the lack of if statements.

The approximation formula computes $\sin(x)$ for an integer $x$ in the range ${0 \le x \le 180}$.
In order to compute $\sin(\mathtt{DEGREES})$ for any integer number of $\mathtt{DEGREES}$, even integers outside of $\mathtt{[0, 180]}$, we calculate two values $R$ and $S$.
Here, $R$ is a number in the range ${{-179} \le R \le 180}$ such that ${\sin(\mathtt{DEGREES}) = \sin(R)}$.
And $S$ is a number similar to the signum of $R$ that allows us to use the trigonometric identity

$$\sin(x) = {-\sin({-x})}$$

to compute $\sin(R)$ using arithmetic operations, without any conditional branching.
This number is defined by

$$S = \begin{cases}1 & R \ge 0,\\\\{-1} & R < 0.\end{cases}$$

In total, the algorithm evaluates ${S \sin(SR)}$, where the following equalities apply:

$$\sin(\mathtt{DEGREES}) = \sin(R) = S \sin(SR).$$

### Almost the Remainder

First we compute the <q>remainder</q> $R$ constant.

```text
#const R (DEGREES % 360 + 360 % 360 * -1 + 180)
```

Since sine has a period of $360$ degrees, there is an integer $r$ such that $0 \le r \le 359$ and ${\sin(\mathtt{DEGREES}) = \sin(r)}$.
Here $r$ is the remainder of $\mathtt{DEGREES}$ when divided by $360$.

We can use the RMS remainder operator to compute `DEGREES % 360`.
However, the RMS `%` operator behaves analogously to C's `%` operator (and differently than Python's `%` operator), leaving us with two cases:

$$\mathtt{DEGREES}\\,\\%\\, 360 = \begin{cases}r & \mathtt{DEGREES} \ge 0,\\\\r - 360 & \mathtt{DEGREES} < 0.\end{cases}$$

In order to support negative numbers, we add $360$ to this value:

$$\mathtt{DEGREES} \\,\\%\\, 360 + 360 = \begin{cases}r + 360 & \mathtt{DEGREES} \ge 0,\\\\r & \mathtt{DEGREES} < 0.\end{cases}$$

Now, we take the remainder once more to handle both cases at once:

$$(\mathtt{DEGREES} \\,\\%\\, 360 + 360) \\,\\%\\, 360 = \begin{cases}r & \mathtt{DEGREES} \ge 0,\\\\r & \mathtt{DEGREES} < 0.\end{cases}$$

Rather than use the interval of $[0, 359]$, it'll be more convenient to work around the lack of if statements by using the interval $[{-179}, 180]$.
Thankfully we can use the trigonometric identity

$$\sin{x} = \sin(180 - x).$$

At the end of our line of code, we multiply by ${-1}$ and add $180$ in order to use this identity.
That leaves us with a number $R$ that is in $[{-179}, 180]$ and satisfies $\sin(\mathtt{DEGREES}) = \sin(R)$.

Note that this line of code is structured to advantage the fact that RMS math expressions treat every operator as left-associative with the same precedence.

### Almost the Signum

For values of $R$ in $[0, 180]$, we can plug them directly into our approximation formula.
But what about $R$ in $[{-179}, {-1}]$?
For these values we use the trigonometric identity:

$$\sin{x} = {-\sin({-x})}.$$

In order to use this identity, we need to:

- Detect if ${R < 0}$.
- Compute $\sin({-R})$.
- Negate the computed sine value.

And all of that must be done without if statements!

For our implementation, we compute a value $S$ as follows:

$$S = \begin{cases}
    1 & R \ge 0,\\\\
    {-1} & R < 0.
\end{cases}$$

This value is similar to the signum of $R$, except that when $R = 0$, we set $S = 1$.

It turns out that the trigonometric identity can be rewritten, using $S$ in place of the negative signs.
We have two cases where we show

$$S \sin(S R) = \sin(R).$$

- ${R \ge 0}$, then ${S = 1}$ and the $S$ values are just multiplications by $1$.
- ${R < 0}$, where ${S = {-1}}$ and by the aforementioned trigonometric identity, ${-\sin({-R}) = \sin(R)}$.

Let's examine the line where we compute $S$:

```text
#const S (R * 2 + 1 % 2)
```

Here we first double $R$ to obtain an even number, then add $1$ to obtain an odd number.
Calculating the remainder with $2$ then returns $1$ for positive numbers and ${-1}$ for negative numbers:

$$S = (2R + 1) \\,\\%\\, 2 = \begin{cases}1 & R \ge 0,\\\\{-1} & R < 0.\end{cases}$$

This definition of $S$ ensures that ${0 \le SR \le 180}$, satisfying the bounds required by the approximation formula to compute $\sin(SR)$.

### Finishing the Sine Computation

The rest of the code consists of using the approximation formula to compute ${S \sin(SR)}$.

First let's focus on the line

```text
#const P (180 * S - R * R)
```

Both the numerator and the denominator of the approximation formula use the product of the angle and its supplementary angle:

$$P = x(180 - x).$$

Plugging in ${x = SR}$, we can rewrite this product as:

$$x(180 - x) = (180 - x)x = (180 - S R) S R  = (180 \cdot S - S^2 R)R = (180 \cdot S - R)R.$$

The rightmost equality follows because ${S^2 = 1}$.

Rearranging this to account for the left-associative nature of the math expressions yields the line of code used to determine $P$.

The next line is a straightforward calculation of the approximation formula's denominator:

```text
#const D (40500 - P)
```

Finally, we calculate the sine value.

```text
#const SIN (S * 4 * P / D)
```

Putting everything together, this line calculates ${S \sin(SR)}$, which equals $\sin(\mathtt{DEGREES})$.

### Calculating the Cosine

The rest of the computation repeats the code used for the sine function, reusing the same algorithm with a different input value to determine the cosine.
The only difference is in the line used to determine the <q>remainder</q> value for the cosine computation.

```text
#const CR (270 - R % 360 * -1 + 180)
```

Here we use the property:

$$\cos(\mathtt{DEGREES}) = \sin(90 + \mathtt{DEGREES}).$$

Recall that we can write

$$R = 180 - r,$$

where $r$ is the unique integer such that $0 \le r \le 359$ and $r \equiv \mathtt{DEGREES} \pmod{360}$.
Hence we have

$$\cos(\mathtt{DEGREES}) = \sin(90 + \mathtt{DEGREES}) =\sin(90 + r) =\sin(90 + (180 - R)) =\sin(270 - R).$$

Since ${{-179} \le R \le 180}$, we have ${90 \le 270 - R \le 449}$.
This value is strictly positive, so we don't need separate reasoning for a negative number like we did for the original remainder.
We simply take the remainder with respect to $360$, multiply by ${-1}$, and add ${180}$.

The rest of the cosine computation proceeds analogously to the sine computation.

### Conclusion

The output of the example can be seen in the player resource Food and Gold values when launching the accompanying [example map script](https://github.com/twestura/RMS-Trigonometry-Example/blob/main/Math%20Trig%20July%202026.rms) in Single Player.
The example multiplies the `COS` and `SIN` values by `10000` to display more decimal digits.
