---
title: "D&D Dice Calculator - Math & Misfortune"
author: me
date: 2024-12-07T21:24:00-07:00
draft: false
toc: false
images:
mathjax: true
tags: 
  - d&d
  - coding
  - python
  - math
  - statistics
  - probability
  - imnotbitter
  - doesgodexist
---

> Is God willing to prevent evil, but not able? Then he is not omnipotent.
> 
> Is he able, but not willing? Then he is malevolent.
> 
> Is he both able and willing? Then whence cometh evil?
> 
> Is he neither able nor willing? Then why call him God?
> 
> ― Epicurus

So I'm rolling stats for a lv. 1 warlock character. Right off the bat I didn't like this; I've played enough Fire Emblem to know that randomly-assigned stats are just a vehicle for demise. But this is what DM decreed, so I did what I was told and rolled for stats.

The scheme that DM implemented for stat rolling was this:

- choose a *specific* stat (one of STR, DEX, CON, INT, WIS, CHA)
- roll `4d6`, drop the lowest -> that's the value for that chosen stat
- do that for each stat once -> all of those assigned values form a *stat group*
- do ^ that whole process again to make a second stat group
- choose one of the stat groups to be your stats

I rolled the following two stat groups:

| STR | DEX   | CON   | INT | WIS | CHA |
|-----|-------|-------|-----|-----|-----|
| 17  | **5** | 14    | 14  | 15  | 9   |
| 12  | 15    | **5** | 15  | 10  | 15  |

Both stat groups had a 5. DM said "oof, that's pretty unlucky. sorry dawg." Choosing the second group gave my warlock a decent CHA so that spells would be effective, but the 5 CON meant that at lv. 1 the character had 5 HP. That means 10 damage would be an instant perma-death.

Obviously this was not super lucky.

...but *how* unlucky is this, exactly? 🧐

In the last post I talked about the coding side of building a D&D dice probability calculator / visualization tool. In this post I want to dip into the (imo easier) math behind calculating these distributions.

# some setup

Before getting into the math, I want to make sure readers understand some fundamental background about probability and statistics.

- an [*experiment*](https://en.wikipedia.org/wiki/Experiment_(probability_theory)) is a real-world process, that can be repeated, that has a well-defined set of [*outcomes*](https://en.wikipedia.org/wiki/Outcome_(probability)). E.g., rolling a `d6` is an experiment, and the outcomes are the numbers 1, 2, 3, 4, 5, and 6.
- an [*event*](https://en.wikipedia.org/wiki/Event_(probability_theory)) is a collection of outcomes. E.g., rolling an even number on a `d6` is an event, which is a collection of the outcomes \({2, 4, 6}\).
- a [*probability distribution*](https://en.wikipedia.org/wiki/Probability_distribution) (or in the case of dice rolls specifically, a [*probability mass function (PMF)*](https://en.wikipedia.org/wiki/Probability_mass_function)) is a mathematical function that takes as input a possible outcome / event, and outputs how probable it is for that result to be realized.

  For example, if a coin had a 55% change to land on heads when flipped (and a 45% of the same for tails), the PMF \(f\) would be defined as:
  $$
  \begin{array}{l}
    f(\text{heads}) := 0.55 \\
    f(\text{tails}) := 0.45 \\
    f(\text{anything else}) := 0 \\
  \end{array}
  $$
- [convolution](https://en.wikipedia.org/wiki/Convolution) is a mathematical operation that takes as input two N-dimensional arrays of numbers, and as output generates a third N-dimensional array. In this case, we'll use 1D-convolution to [calculate the distribution for the sum of two experiments](https://en.wikipedia.org/wiki/Convolution_of_probability_distributions) (e.g., `2d6` = `1d6` + `1d6`).

## implementing convolution

I mentioned in the last post that I was implementing this visualization tool in Python. One of the upsides of doing so is being able to use NumPy, which has a builtin convolution function, [`np.convolve`](https://numpy.org/doc/stable/reference/generated/numpy.convolve.html).

The caveat here is that `np.convolve` doesn't calculate the index shifting that occurs when convolving pure mathematical functions. Because `np.convolve` operates on arrays which don't store this offset data, I had to write this mathematical property in myself. My solution was to write a Python class that bundles array data and offset data, includes a `convolve` method that calculates both of these together, and includes a `consolidate` method that behaves like function addition (truncated below):

```python
from dataclasses import dataclass

import numpy as np

@dataclass(slots=True, kw_only=True)
class SequenceWithOffset:
    seq: np.array
    offset: int

    def convolve(self, other: SequenceWithOffset) -> SequenceWithOffset:
        if len(self.seq) == 0 or len(other.seq) == 0:
            # make a zero-length array with the correct dtype
            seq = self.seq[:0] + other.seq[:0]
        else:
            seq = np.convolve(self.seq, other.seq)
        return SequenceWithOffset(seq=seq, offset=self.offset + other.offset)

    def consolidate(self, other: SequenceWithOffset) -> SequenceWithOffset:
        if len(self.seq) == 0:
            return other.copy()
        elif len(other.seq) == 0:
            return self.copy()
        index_low = min(self.offset, other.offset)
        index_high = max(
            self.offset + len(self.seq),
            other.offset + len(other.seq),
        )

        seq = np.zeros(
            index_high - index_low,
            dtype=(self.seq[:0] + other.seq[:0]).dtype,
        )
        seq[
            self.offset - index_low : self.offset + len(self.seq) - index_low
        ] = self.seq
        seq[
            other.offset - index_low : other.offset + len(other.seq) - index_low
        ] += other.seq

        return SequenceWithOffset(seq=seq, offset=index_low)
```

Re: runtime complexity - unfortunately the docs aren't super clear, so I'm left to either read the source code (this is for fun I don't wanna read source code 😭), or guess. [Some research](https://fse.studenttheses.ub.rug.nl/25184/1/bCS_2021_GhidirimschiN.pdf.pdf) (and [some more](https://wangwei1237.github.io/shares/Algorithms_for_Efficient_Computation_of_Convolution.pdf)) makes guessing a little easier:

- A naive algorithm would run in \(O(n_1 n_2)\) time.
- The [Karatsuba algorithm](https://en.wikipedia.org/wiki/Karatsuba_algorithm) runs in \(O(n^{1.58})\) time, and doesn't seem to have other obvious implementation constraints.
- There are other algorithms with better asymptotic performance (e.g. [Toom-Cook 3 @ \(O(n^{1.46})\)](https://en.wikipedia.org/wiki/Toom–Cook_multiplication)), but they also have large coefficients that might be impractical for typical NumPy users / use cases.
- There's also [a NumPy implementation for a method of convolution that utilizes FFT's](https://docs.scipy.org/doc/scipy-1.14.1/reference/generated/scipy.signal.fftconvolve.html) which runs in \(O(n\log n)\) time, but since that only works on floating-point data I elected not to use it.
    - It seems like there's an analogous method of doing the FFT-enabled convolution for integers in the same \(O(n log(n))\) time, but instead using [the Number-theoretic Transform (NTT)](https://en.wikipedia.org/wiki/Discrete_Fourier_transform_over_a_ring#Number-theoretic_transform).
        - However, I am under the impression that NumPy devs didn't implement this integer-specific algorithm for convolution. Mainly because the corresponding floating-point specific algorithm using the FFT has its own function, and if they *had* implemented an integer-specific NTT, I imagine it would get its own function too. And I don't see one in the docs.
        - Also, from the paper cited above it seems like there are a number of complications required in the implementation to make it feasible. And it's unclear to me whether the NumPy developers would develop a complicated special-case algorithm just for integers, which I imagine is a pretty niche use case since you can just cast ints to floats.

-> Moving forward I'll assume a conservative \(O(n^2)\) time complexity for this function, but know that this may not be the true complexity depending on implementation specifics.

## probabilities vs. outcomes

One small change in perspective I want to implement here: instead of thinking in terms of probability distributions, I instead want to think in terms of *outcome-count distributions*.

So for example, let's consider the `2d6` case. There are \(6 \times 6 = 36\) different possible *outcomes*. The *events* we want to consider are the possible sums of dice values resulting from any of these 36 different outcomes; for `2d6` there are \(11\) such different events, which include all the integers from \(2\) (realized by rolling two \(1\)'s) to \(12\) (realized by rolling two \(6\)'s). I'll refer to these event values as *scores*, or *score values*.

Now let's say we're interested in how we might roll the score value \(5\). There are 4 outcomes that result in a \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\).

- in terms of *probability*, there is a \(\frac{4}{36}\) chance of rolling a \(5\) (assuming fair dice).
- in terms of *outcome counts*, there are 4 outcomes that result in a roll of \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\) are those outcomes.
- as long as the dice are fair, then one can convert between the two using the equation:
  $$
  (\text{probability that value} = x) = \frac{(\text{# of outcomes where value} = x)}{(\text{total # of outcomes})}
  $$

So when calculating probability distributions, I want to **first calculate an outcome-count function (OCF), and then convert that into a probability mass function (PMF) using the above equation on each event value**.

There are a handful of advantages to doing this:

1. Intermediate computations are all performed on integers instead of floating-point numbers. This means
    - there's no rounding errors that can accumulate through the computation,
    - each computer-stored number will contain more information (64-bit floating-point numbers only [have 53 bits of storage for digits](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64) -> I would get more precision with a 64-bit integer), and
    - any resulting values that are too large to store will generally trigger [a hardware-implemented error flag](https://en.wikipedia.org/wiki/Integer_overflow#Flags) that [is easier to respond to](https://doc.rust-lang.org/std/primitive.i64.html#method.checked_add).
1. It allows us to make some simple correctness checks. Specifically, rolling \(k\) \(n\)-sided dice always yields \(n^k\) outcomes, so any outcome-count distribution generated from rolling \(k\) \(n\)-sided dice must sum to \(n^k\) too. And if it doesn't, then the calculation must be wrong in some way.
1. It makes recursive calculations simpler. I'll discuss what I mean by this later on.

The downside is that it can't be used to represent unfair dice, but I'm not concerned with unfair dice here, so that drawback is moot.

## rolling `0dn`

This is effectively the distribution for when we roll "0 dice". While this isn't practical in the real world, mathematically this gets used a lot in the upcoming calculations for recursive or iterative definitions and algorithms, so it makes sense to define it here upfront.

The rationale is that rolling "0 dice" yields a single outcome \(()\), where the score value is the sum of "all the individual values", of which there are none, so this sum is \(0\).

Implementing this in Python might look like this:

```python
def roll_0dn() -> SequenceWithOffset:
    return SequenceWithOffset(seq=np.ones(1, dtype=np.uint64), offset=0)
```

... and the time complexity for generating this trivial data is constant time, \(O(1)\).

# calculations

And now, finally, some math 😈

For each of the cases below, I want to figure out:

1. a method for computing the experiment's OCF (which can be then turned into a PMF), written in Python, and
1. the time complexity for that computation method, in [big-O notation](https://en.wikipedia.org/wiki/Big_O_notation).

## rolling `1dn`

By definition, rolling one die with \(n\) sides will produce the OCF:

$$
f(x) = \begin{cases}
1 & \text{if } x \in [1, n] \\
0 & \text{otherwise} 
\end{cases}
$$

which could be implemented in Python in linear time \(O(n)\) like so:

```python
def roll_1dn(n: int) -> SequenceWithOffset:
    return SequenceWithOffset(seq=np.ones(n, dtype=np.uint64), offset=1)
```

Visualized, the plot for e.g. `1d6` looks like this:

![Plot for OCF of 1d6](plot-1d6.png)

Not super exciting, but definitions aren't really supposed to be.

## rolling `kdn`

For multiple dice, we can use the convolution operation mentioned above. Specifically, we can compose the OCF for `kdn` by convolving together the OCF of `1dn` with itself `k` times.

A simple Python function for calculating `kdn` might look like this:

```python
def roll_kdn(k: int, n: int) -> SequenceWithOffset:
    _1dn = roll_1dn(n)
    result = roll_0dn()

    for _ in range(k):
        result = result.convolve(_1dn)

    return result
```

The time complexity for this algorithm depends on how long each convolution takes, which depends on the lengths of the intermediate resulting arrays:
- the length of the original array is \(n\)
- the length of each array convolved with itself \(i\) times is \(i(n-1) + 1\)
- the time complexity for convolving a length-\(n\) array with a length-\((i-1)(n-1) + 1\) array is \(O(i n^2)\)
- the time complexity for calculating all auto-convolutions is:
  $$
  \begin{array}{l}
  O(\sum_i^k i n^2) \\
  = O(n^2 \sum_i^k i) \\
  = O(k^2 n^2) \\
  \end{array}
  $$

Below are some examples of what this looks like for a couple different \(k\) values:

![Plot for OCF of 2d6](plot-2d6.png)

![Plot for OCF of 3d6](plot-3d6.png)

And again, to convert this OCF to a PMF we need to know the total number of outcomes possible. Since there are \(k\) dice involved and each die has \(n\) outcomes of its own, the number of total outcomes is the product of those individual totals, \(n^k\).

### ❌ fewer convolutions doesn't actually make it faster

A more sophisticated algorithm can leverage the binary representation of `k` to perform fewer convolutions. Specifically, the previous algorithm computed `1dn` auto-convolved `k` times, which required \(k-1\) convolutions. Instead, we can compute auto-convolutions of `1dn` for all powers of two up to `k`, then convolve the subset of those results whose powers of two sum to `k`. This involves only \(\log_2(k)\) convolutions to compute the auto-convolutions, and another at most \(\log_2(k)\) convolutions to get the final result.

*(Ftr I didn't invent this myself; I've seen this trick used for other types of numerical calculations in other contexts.)*

Such an algorithm in Python might look like this:

```python
import numpy as np

def roll_kdn(k: int, n: int) -> SequenceWithOffset:
    # Calculate auto-convolutions at powers of two
    auto_convs = [roll_1dn(n)]
    for _ in range(1, k.bit_length()):
        prev = auto_convs[-1]
        auto_convs.append(prev.convolve(prev))

    # Calculate result from auto-convolutions
    result = roll_0dn()
    for i, auto_conv in enumerate(k.bit_length):
        ith_bit = k & (1 << i)
        if ith_bit == 0:
            continue
        result = result.convolve(auto_conv)

    return result
```

Re: time complexity of this algorithm:
- the length of each auto-convolution array, for a given number of convolutions \(2^i \leq k\), is:
  $$
  \begin{array}{l}
  2^i \cdot (n-1) + 1 \\
  = O(kn)
  \end{array}
  $$
- calculating the auto-convolutions takes time:
  $$
  \begin{array}{l}
  O \left( \sum_i (2^i n)^2 \right) \\
  = O \left(n^2 \sum_i 4^i \right) \\
  = O \left(n^2 k^2 \left(1 + \frac14 + \frac1{16} + ...\right)\right) \\
  = O(n^2 k^2)
  \end{array}
  $$
- the cumulative result after the \(k\)th step is (at most) the first \(k\) auto-convolutions convolved together.
- the length of the cumulative result array after the \(j\)th step, where \(2^j \leq k\), is (at most) the sum of the lengths minus \(1\) for each convolution operation:
  $$
  \begin{array}{l}
  \left( \sum_{i \leq j} 2^i \cdot (n-1) \right) + 1 \\
  = (n-1) \sum_{i \leq j} 2^i + 1 \\
  = (n-1) (2^{j+1} - 1) + 1 \\
  = O(nk)
  \end{array}
  $$
- calculating all cumulative results, and thus the final result, takes time:
  $$
  \begin{array}{l}
  O \left( \sum_j (kn)^2 \right) \\
  = O \left(k^2 n^2 \left(1 + \frac14 + \frac1{16} + ...\right)\right) \\
  = O(k^2 n^2)
  \end{array}
  $$

This happens to have the same complexity as the simpler algorithm above, even though this algorithm uses fewer convolution operations.

## rolling `kdn drop highest`

Ahhhh. Finally. We get to the content that made me want to do this in the first place ☺️

To calculate this, I need to be able to map all of the outcomes to specific score events. But I can't just use the result that our previous function `roll_kdn` would generate, because not all of the outcomes in a specific `kdn` event will map to the same `kdn drop highest` event.

For example, if I'm calculating `2d6 drop highest`, it would be tempting to try to call `roll_kdn(k=2, n=6)`, and use those outcome counts to make the new outcome counts. But in `2d6`, e.g. both of the outcomes \((1, 4)\) and \((2, 3)\) would be in the \(5\) score event, but in `2d6 drop highest` they would both count towards different score events, specifically \(1\) and \(2\), respectively. And since I'm not storing the actual outcomes themselves, just the number of outcomes for each score event, how would one know how to reassign counts from `2d6` to `2d6 drop highest`?

So afaik, calculating `kdn drop highest` (and `drop lowest`) with convolutions only works by breaking the problem down into a bunch of smaller sub-problem distributions, calculating those OCF's, and element-wise summing all of the smaller OCF's together. The trick is how to come up with the sub-problems.

To make this a little more tangible, I'm going to work through a specific example.

### rolling `3d4 drop highest`

For our tangible example, let's specifically consider \(k=3\) and \(n=4\).

For our sub-problems, I want to individually consider the different events for how the largest dice value, e.g. \(4\), does or does not get rolled, and what score distributions correspond to those events. From those sub-distributions, I can re-map the scores to drop the highest value (i.e., \(4\)) and re-collect them into the final distribution.

Specifically, I want to separately consider the following partition of events:

1. all three dice roll a \(4\):
    - *(there is exactly one outcome for this event - \((4, 4, 4)\))*
    - one of the \(4\)'s gets dropped from the score
    - the other two \(4\)'s are kept in the score
    - -> the result is the single-event distribution `8`
1. exactly two of the dice (i.e., the first and second, the second and third, or the first and third) roll a \(4\), and the remaining one die rolls a lower number between \(1\) and \(3\):
    - one of the \(4\)'s gets dropped from the score
    - one of the \(4\)'s is kept in the score
    - the other remaining die kept in the score rolls less than a \(4\), so it is effectively a `d3` 
    - -> the distribution is effectively that of `1d3 + 4`
1. exactly one die (i.e., the first, second, or third) rolled a \(4\), and the others rolled any other lower number between \(1\) and \(3\):
    - the \(4\) gets dropped from the score
    - the remaining die roll less than \(4\), so they are both effectively `d3`'s
    - -> the scored result is effectively `2d3`
1. the remaining possibilities occur when *none* of the dice are \(4\)'s
    - all dice roll less than \(4\), so they are effectively each a `d3`
    - no dice has been designated to be dropped
    - -> this is effectively the same as `3d3 drop highest`

Note that the 1-die and 2-dice cases each have three unique orderings, while the 0-die and 3-dice cases each have one ordering. More generally, the number of orderings for a given number of fixed high dice \(i\) is [\(k\)-choose-\(i\)](https://en.wikipedia.org/wiki/Binomial_coefficient), or \(k \choose i\).

Altogether, if we write the OCF of a distribution \(d\) as \(f_d\), then we can recursively express the OCF of `3d4 drop highest` as follows:

$$
\begin{align}
f_{3d4 \text{ drop highest}} (x)
&= {3 \choose 0} f_{8} (x) \\
    &+ {3 \choose 1} f_{1d3 + 4} (x) \\
    &+ {3 \choose 2} f_{2d3} (x) \\
    &+ {3 \choose 3} f_{3d3 \text{ drop highest}} (x)
\end{align}
$$

### back to the general case, `kdn drop highest`

For the general case of dropping the highest die (specifically for the non-trivial cases, i.e. when there is more than one die \(k > 1\), and the die has multiple outcomes \(n > 1\)), we can formulate an equation similar to the `3d4 drop highest` case:

$$
f_{kdn \text{ drop highest}} (x) \\
= {k \choose k} f_{kd(n-1) \text{ drop highest}} (x) \\
    + \sum_{i \in [0, k-1]} {k \choose i} f_{id(n-1) + n \cdot (k-1 -i)}
$$

However, since this is a recursive definition, we have to define the end conditions for each of the decreasing parameters, \(k\) and \(n\):
- when \(k = 1\) - if we roll one die and drop "the highest" (a.k.a. "the only"), it's the same as rolling no dice: 
  $$
  f_{1dn \text{ drop highest}} (x) \\
  = f_{0} (x) \\
  = \begin{cases}
      1 & \text{if } x = 0 \\
      0 & \text{otherwise} \\
  \end{cases}
  $$
- and when \(n = 1\) - if all dice can only roll a \(1\), then rolling \(k\) dice and dropping "the highest" should always result in the same value, i.e. \(k-1\):
  $$
  f_{kd1 \text{ drop highest}} (x) \\
  = f_{k-1} (x) \\
  = \begin{cases}
      1 & \text{if } x = k-1 \\
      0 & \text{otherwise} \\
  \end{cases}
  $$

In both end cases, the OCF reduces to \(f_{k-1}\).

With the equation established, we can now implement it in Python:

```python
import math

def roll_kdn_drop_highest(k: int, n: int):
    if k == 1 or n == 1:
        return SequenceWithOffset(seq=np.ones(1, dtype=np.uint64), offset=k-1)

    result = roll_kdn_drop_highest(k=k, n=n-1)
    for i in range(k):
        sub_result = roll_kdn(k=i, n=n-1)
        sub_result.seq *= math.comb(k, i)
        sub_result.offset += n * (k - 1 - i)
        result = result.consolidate(sub_result)
    return result
```

Some graphs of this are shown below:

![Plot for OCF of 2d6 drop highest](plot-2d6-drop-high-1.png)

![Plot for OCF of 3d6 drop highest](plot-3d6-drop-high-1.png)


Like the function itself, the complexity can also be expressed recursively. The non-recursive parts are:
- the base case at `kd1` or `1dn`, which are constant time: \(O(1)\)
- a loop from \(i=0\) to \(k-1\): \(O( \sum_{i=0}^{k-1} ... )\):
    - calculating `kdn`: \(O(i^2 n^2)\)
    - element-wise multiplication for \(i(n-1) + 1\) elements: \(O(in)\)
    - `consolidate`, i.e. initialization and element-wise addition for at most \(k(n-1) + 1\) elements: \(O(kn)\)

Let \(O(C(k, n))\) be the time complexity of this algorithm. Then altogether the complexity would be expressed as follows:

$$
\begin{align}
O(C(k, n)) :&= O\left( C(k, n-1) + \sum_{i=1}^{k-1} (i^2 n^2 + \cancel{in} + \cancel{kn}) \right) \\
    &= O\left( C(k, n-1) + n^2 \sum_{i=1}^{k-1} i^2 \right) \\
    &= O\left( C(k, n-1) + n^2 k^3 \right) \\
    \textit{apply def. of } C(k, n-1) \rightarrow &= O\left( \left( C(k, n-2) + k^3 {(n-1)}^2 \right) + k^3 n^2 \right) \\
    \textit{repeatedly apply def. of } C(k, n-j) \rightarrow &= O\left( ... + k^3 {(n-2)}^2 + k^3 {(n-1)}^2 + k^3 n^2 \right) \\
    &= O\left( k^3 \sum_{j=1}^n {(n-j)}^2 \right) \\
    &= O\left( k^3 n^3 \right) \\
\end{align}
$$

### calculate all sub-distributions in advance -> better performance & complexity

In #kdn, we showed that it takes just as much time complexity to calculate an OCF for `kdn`, as it does to calculate all OCF's of `idn` for \(i \in [1, k]\). Doing the latter explicitly in this function is significant enough that it actually improves the overall time complexity.

The revised Python code might look like this:

```python
def iter_autoconv(dist: SequenceWithOffset):
    result = roll_0dn()
    while True:
        yield result
        result = result.convolve(dist)

def roll_kdn_drop_highest(k: int, n: int):
    if k == 1 or n == 1:
        return SequenceWithOffset(seq=np.ones(1, dtype=np.uint64), offset=k-1)

    result = roll_kdn_drop_highest(k=k, n=n-1)
    for i, kdnm1 in zip(range(k), iter_autoconv(roll_1dn(n-1))):
        sub_result = kdnm1.copy()  # reallocates a new object & numpy array
        sub_result.seq *= math.comb(k, i)
        sub_result.offset += n * (k - 1 - i)
        result = result.consolidate(sub_result)
    return result
```

... for which the new time complexity would look like this:

$$
\begin{align}
O(C(k, n)) :&= O\left( C(k, n-1) + k^2 n^2 + \sum_{i=1}^{k-1} (in) \right) \\
    &= O\left( C(k, n-1) + k^2 n^2 + n \sum_{i=1}^{k-1} i \right) \\
    &= O\left( C(k, n-1) + k^2 n^2 + \cancel{k^2 n} \right) \\
    \textit{repeatedly apply def. of } C(k, n-j) \rightarrow &= O\left( ... + k^2 {(n-2)}^2 + k^2 {(n-1)}^2 + k^2 n^2 \right) \\
    &= O\left( k^2 \sum_{j=1}^n {(n-j)}^2 \right) \\
    &= O\left( k^2 n^3 \right) \\
\end{align}
$$

So explicitly calculating the auto-convolutions of `1d(n-1)` reduces the time complexity from \(O(k^3 n^3)\) to \(O(k^2 n^3)\), which is a reduction by a factor of \(k\).

### note re: outcome counts vs. probabilities

So when I said "[using outcome counts] makes recursive calculations simpler", this is what I meant.

Specifically, I wouldn't be able to use the nested recursive `kdnm1` calculations as sub-functions to calculate `roll_kdn_drop_highest`, unless I either 1) first converted each probability distribution to an outcome-count distribution before doing any aggregation, or 2) tried to use them as conditional probabilities. Doing both of these things would be annoyingly complicated.

*(In fact, when I was first writing this code I started out using probability distributions with workaround 1), and when I got to doing the "drop highest/lowest die" functions I ran into bugs that I couldn't figure out how to fix. For me in this case, it turned out to be easier to refactor everything into using outcome counts and rewriting the function s.t. the bugs were obviated, rather than fixing whatever bugs I had.)*

## rolling `kdn drop high m`

The above can be further generalized to dropping the \(m\) highest dice from the score. We can reuse most the previous algorithm, i.e. do the same splitting on the different outcomes for rolling the highest die value; the main differences here will be in how we recurse and how we assign score values to sub-distributions.

Similar to before, we have a partition of sub-distributions based on how many high values are rolled. Sub-distributions have a piecewise definition:

- When the number of high values \(i\) is greater than (or equal to) the amount of dice to drop \(m\), then those high values are deducted from the drop count, and the remaining \(i - m\) high values contribute to the overall score:
  $$
  f_{d_i \ | \ i \geq m} (x) := {k \choose i} f_{(k-i)d(n-1) + n \cdot (i-m)} (x)
  $$
- When the number of high values \(i\) is *less than* (or equal to) the amount of dice to drop \(m\), then those high values are dropped, and the remaining drop count \(m - i\) is passed on to the sub-distribution:
  $$
  f_{d_i \ | \ i \leq m} (x) := {k \choose i} f_{(k-i)d(n-1) \text{ drop } (m-i)} (x)
  $$

In both cases, the sub-problems always have a smaller \(n\) value, and sometimes have smaller \(k\) and \(m\) values. Again as in the `kdn drop highest` problem, since this is a recursive function we need to provide base case definitions so that it doesn't recurse indefinitely. Now that there is another decreasing input value \(m\) in addition to the other previous ones \(k\) and \(n\), we need *three* base cases:

- when \(m = 0\) - this reduces to `kdn`:
  $$
  f_{kdn \text{ drop high } 0} (x) \\
  = f_{kdn} (x) \\
  $$
- when \(n = 1\) - basically the same as in `kdn drop highest`:
  $$
  f_{kd1 \text{ drop high } m} (x) \\
  = f_{k-m} (x) \\
  = \begin{cases}
      1 & \text{if } x = k-m \\
      0 & \text{otherwise} \\
  \end{cases}
  $$
- and when \(k = m\) - if we roll \(k\) dice and drop \(m\) of them, we've rolled \(0\) dice:
  $$
  f_{kdn \text{ drop high } k} (x) \\
  = f_{0} (x) \\
  = \begin{cases}
      1 & \text{if } x = 0 \\
      0 & \text{otherwise} \\
  \end{cases}
  $$

Altogether, we can mathematically express the distribution as follows:

$$
\begin{align}
f_{kdn \text{ drop high } 0} (x) &= f_{kdn} (x) \\
f_{kd1 \text{ drop high } m} (x) &= f_{k-m} (x) \\
f_{kdn \text{ drop high } k} (x) &= f_{0} (x) \\
f_{kdn \text{ drop high } m} (x) &=
  \sum_{i=0}^{m-1} {k \choose i} f_{(k-i)d(n-1) \text{ drop } (m-i)} (x) \\
  &+ \sum_{i=m}^k {k \choose i} f_{(k-i)d(n-1) + n \cdot (i-m)} (x)
\end{align}
$$

A Python implementation might look like this:

```python
def roll_kdn_drop_high(k: int, n: int, m: int) -> SequenceWithOffset:
    if k == m or n == 1:
        return SequenceWithOffset(seq=np.array([1], dtype=np.uint64), offset=k-m)
    if m == 0:
        return roll_kdn(k=k, n=n)

    result = SequenceWithOffset(seq=np.array([], dtype=np.uint64), offset=0)
    for i in range(m):
        sub_dist = roll_kdn_drop_high(k=k-i, n=n-1, m=m-i)
        sub_dist.seq *= math.comb(k, i)
        result = result.consolidate(sub_dist)

    for i in range(m, k):
        sub_dist = roll_kdn(k=k-i, n=n-1)
        sub_dist.seq *= math.comb(k, i)
        sub_dist.offset += n * (i - m)
        result = result.consolidate(sub_dist)

    return result
```

### caching calculated sub-distribution

Initially I thought the above implementation was pretty good. But after implementing and playing around with different inputs, I noticed that there was a sizeable delay when calculating large drop counts. For example, running `roll_kdn_drop_high(n=20, k=7, m=6)` took a couple seconds to calculate, and increasing the input values to `k=10, m=9` took over 30 seconds. I done borked something.

This whole issue was actually giving me flashbacks of when I was first learning to code on my own, back when I was doing Hacker Rank (effectively a LeetCode clone/ancestor) problems for fun. There were a handful of occasions where I'd write a recursive function and it would hang once the inputs got up to ~10-20 in size or magnitude. The solution: either rewrite use a non-recursive algorithm, or add in caching to the recursive definition.

#### debugging

But first, let's verify the problem. I took the above Python function definition and added in a print statement at the very top of the function, something like `print(f"roll {k}d{n} drop high {m}")`; this will tell me how many times the function gets re-called during recursion. Rerunning the print-augmented function call `roll_kdn_drop_high(n=20, k=7, m=6)` generated a *wall* of terminal text. I took that, dropped it into a text file, reformatted it into a Python list called `log`, where every entry in `log` was a separate `print` call, and then loaded `log` into a Python terminal.

With the `print` calls in a Python terminal, I could do some analysis. Firstly, how many `print` calls were there?

```pycon
>>> len(log)
407330
```

That's a *lot* of calls. Potentially the recursive function was having issues terminating, but 1) if it didn't terminate, it would've kept going indefinitely, and 2) I have my intuition that the function is getting called with the same inputs repeatedly.

What do the function call frequencies look like?

```pycon
>>> from collections import defaultdict
>>> counts = defaultdict(int)
>>> for l in log:
...     counts[l] += 1
... 
>>> counts
defaultdict(<class 'int'>, { [A BUNCH OF ENTRIES...] })
```

Ooooookay that's too much data. How many unique `print` calls were there?

```pycon
>>> len(counts)
141
```

*There*'s an issue: the function gets called \(407,330\) times, but only \(141\) of the calls are unique. So \(407,330 - 141 = 407,189\) of the function calls are redundant.

At this point we've already identified the problem, but now I'm curious: what does the frequency distribution look like? (E.g., if there were a few specific calls that was responsible for most of the repetition, then maybe I wouldn't have to cache everything?)

```pycon
>>> list(reversed(sorted(counts.values())))
[42504, 42504, 33649, 33649, 26334, 26334, 20349, 20349, 15504, 15504, 11628, 11628, 8855, 8568, 8568, 7315, 6188, 6188, 5985, 4845, 4368, 4368, 3876, 3060, 3003, 3003, 2380, 2002, 2002, 1820, 1540, 1365, 1330, 1287, 1287, 1140, 1001, 969, 816, 792, 792, 715, 680, 560, 495, 462, 462, 455, 364, 330, 286, 252, 252, 220, 210, 210, 190, 171, 165, 153, 136, 126, 126, 126, 120, 120, 105, 91, 84, 78, 70, 66, 56, 56, 56, 55, 45, 36, 35, 35, 28, 21, 21, 21, 20, 20, 19, 18, 17, 16, 15, 15, 15, 14, 13, 12, 11, 10, 10, 10, 9, 8, 7, 6, 6, 6, 6, 5, 5, 4, 4, 3, 3, 2, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
```

Okay, yeah the redundancy is *not* evenly distributed; the majority of redundancy belongs to a subset of the function calls.

Actually, there's a simple metric to characterize this majority responsibility phenomenon, by generalizing the concept behind [the 80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) to arbitrary percentages. Specifically, I now want to ask this question: what percent \(x\) of the most-frequent unique function calls contribute to \((1-x)\)% of the total function calls?

```python
import numpy as np

def sparsity_factor(values) -> float:
    values = sorted(values)
    array_acc = np.add.accumulate([0] + values)
    array_acc_percent = array_acc / array_acc[-1]
    equal_acc_percent = np.linspace(0, 1, len(array_acc_percent))[::-1]
    
    index = np.diff(array_acc_percent >= equal_acc_percent).nonzero()
    return equal_acc_percent[::-1][index]
```

```pycon
>>> sparsity_factor(counts.values())
array([0.85815603])
```

Okay, so ~14.2% of the most-frequent unique function calls account for ~85.8% of the total function calls. In this case it's probably still worth it to cache all of them, given the amount of effort it'd take to single-out the most intensive calls from the others. But this metric gives me a good high-level understanding of the data.

#### the fix

Another one of the upsides of using Python is that there are a lot of standard-library-provided solutions to common problems, and caching function calls happens to be one of those problems. For this, Python provides [`functools.lru_cache`](https://docs.python.org/3/library/functools.html#functools.lru_cache), which we can use in our function. However, to do so there are some refactors we need to make:
- I don't want the cache to persist after the function has finished. One way to achieve this is to make an inner function `inner`, that:
  1. is created inside the main function `roll_kdn_drop_high`
  1. is decorated with `lru_cache`
  1. is called initially by `roll_kdn_drop_high` with the provided parameters
  1. is called recursively by itself with the recursing parameters
  1. goes out of scope and gets garbage-collected (along with its `lru_cache`) on exiting `roll_kdn_drop_high`
- the \(m = 0\) case should define \(kdn\) recursively as well, so that we get the benefit of caching and reusing each step in the \(kdn\) calculation.

Overall the refactor might look like this:

```python
def roll_kdn_drop_high(k: int, n: int, m: int) -> SequenceWithOffset:
    # make sure I don't mix up parameters from inner and outer functions
    _k, _n, _m = k, n, m
    del k, n, m

    # Refactor 1: move logic to inner function
    @functools.lru_cache(max_size=None)
    def inner(k: int, n: int, m: int):
        if k == m or n == 1:
            return SequenceWithOffset(seq=np.array([1], dtype=np.uint64), offset=k-m)
        if m == 0:
            # Refactor 2: calculate kdn with explicit recursion
            if k == 0:
                return roll_0dn()
            if k == 1:
                return roll_1dn(n)
            return inner(k=1, n=n, m=0).convolve(inner(k=k-1, n=n, m=0))

        result = SequenceWithOffset(seq=np.array([], dtype=np.uint64), offset=0)
        for i in range(m):
            sub_dist = inner(k=k-i, n=n-1, m=m-i).copy()
            sub_dist.seq *= math.comb(k, i)
            result = result.consolidate(sub_dist)

        for i in range(m, k):
            sub_dist = inner(k=k-i, n=n-1, m=0).copy()
            sub_dist.seq *= math.comb(k, i)
            sub_dist.offset += n * (i - m)
            result = result.consolidate(sub_dist)

        return result

    return inner(k=_k, n=_n, m=_m)
```

#### time complexity

The time complexity for this function is a little tricky to calculate, since it's not explicitly clear as to when calls to `inner` actually invoke the defined function, or just pull from the LRU cache. To assess this, we'll have to figure out which calls are made recursively from a given set of starting input parameters \(k, n, m\), evaluate the complexity for each of those first function calls (assuming all sub-calls are cached & accounted for), and aggregate the result into an overall complexity.

*TODO I need to do a better job of explaining how to calculate the complexity here.*

Let's track some of the function calls:
- \(k\text{d}n \text{ dh}m\) calls:
  - \(k\text{d}(n-1) \text{ dh}m\)
  - \((k-1)\text{d}(n-1) \text{ dh}(m-1)\)
  - ...
  - \((k-m)\text{d}(n-1) \text{ dh}0 = (k-m)\text{d}(n-1)\)
  - \((k-m-1)\text{d}(n-1)\)
  - \((k-m-2)\text{d}(n-1)\)
  - ...
  - \(1\text{d}(n-1)\)
- \(k\text{d}(n-1) \text{ dh}m\) calls all the same as above, but with \((n-2)\)
- ...
- \(k\text{d}(n-i) \text{ dh}m\) calls the same as above, but with \((n-i-1)\)

So from this, it seems like the function calls span two specific ranges of parameters:
1. calling \(k_\text{r} \text{d} n_\text{r} \text{ dh} 0\) for the parameter ranges \(n_\text{r} \in [1, n-1]\) and \(k_\text{r} \in [0, k-m]\)
    - from the previous section #kdn, we know that calculating \(\{i\text{d}n \ |\  i \in [0, k]\}\) takes \(O(k^2 n^2)\) time
    - -> calculating \(\{i\text{d}j \ |\  i \in [0, k-m], j \in [0, n-1]\}\) would thus take time:
      $$
      \begin{align}
        O \left( \sum_{j=0}^{n-1} (k-m)^2 j^2 \right)
        &= O((k-m)^2 n^3) \\
      \end{align}
      $$
1. calling \((k-i) \text{d} n_\text{r} \text{ dh} (m-i)\) for the parameter ranges \(n_\text{r} \in [1, n-1]\) and \(i \in [0, m]\)
    - we assume that when calculating these, all the sub-distributions have been pre-calculated and cached; technically the cached values would get copied s.t. the cached values aren't altered, and this would take linear time on the array lengths, but they're not getting computed from scratch.
    - the function loops through \(k-i\) different sub-distributions:
        - the first \(m-i\) sub-distributions have lengths \(((k-i) - (m-i))(n-2) + 1 = O((k-m) n)\); the function does initialization and element-wise multiplication and addition, so the time to do this work should be linear on the array length, \(O((k-m) n)\)
        - the total time complexity of that loop should be:
          $$
          O \left( \sum_{j=0}^{m-i} (k-m) n \right)
          = O((m-i) (k-m) n)
          $$
        - the last \((k-i) - (m-i) = k-m\) sub-distributions have lengths \((k-i-j)(n-2) + 1 = O((k-i-j)n)\); similarly the function does only linear work here, \(O((k-i-j)n)\)
        - the total time complexity of that loop should be:
          $$
          O \left( \sum_{j=m-i+1}^{k-i} (k-i-j) n \right)
          = O \left( \sum_{j=0}^{k-m-1} j n \right)
          = O((k-m)^2 n)
          $$
    - doing this for all \(n_\text{r} \in [1, n-1], i \in [0, m]\) would take time:
      $$
      \begin{align}
        &O \left( \sum_{n_r=0}^{n-1} \sum_{i=0}^{m} \left( (m-i) (k-m) n_r + (k-m)^2 n_r \right) \right) \\
        &= O \left( \left( \sum_{n_r=0}^{n-1} n_r \right) \sum_{i=0}^{m} \left( (m-i) (k-m) + (k-m)^2 \right) \right) \\
        &= O \left( n^2 \sum_{i=0}^{m} \left( (m-i) (k-m) + (k-m)^2 \right) \right) \\
        &= O \left( n^2 (k-m) \left( \sum_{i=0}^{m} (m-i) + \sum_{i=0}^{m} (k-m) \right) \right) \\
        &= O \left( n^2 (k-m) \left( m^2 + m (k-m) \right) \right) \\
        &= O \left( n^2 (k-m) m \left( \cancel{m} + (k \cancel{-m}) \right) \right) \\
        &= O \left( n^2 k (k-m) m \right) \\
      \end{align}
      $$

Totaling all of this up, the complete time complexity should be:

$$
\begin{align}
T_{k\text{d}n \text{ dh}m}
& = O((k-m)^2 n^3 + k m (k-m) n^2) \\
m \in [0, k], \text{ max at } m=\frac{k}{2} \rightarrow \ 
& = O(k^2 n^3 + k^3 n^2) \\
\end{align}
$$

## rolling "..." and dropping the *lowest* dice

While we could repeat the thinking and problem formulation used before for the "drop highest \(m\) dice" case in order to solve the "drop lowest \(m\) dice" case, there is a simpler solution:

- reverse the distribution array of the input
  - *(technically this is needed for calculations on generic distributions, but since dice roll distributions are symmetric it's not needed here)*
- calculate `roll_kdn_drop_high(k, d, m)`
- reverse the distribution array

And we would get the correct result! But why? It's because there's actually a lot of symmetry in this calculation that we can take advantage of.

So imagine if we took the *negative inversion* of our distribution (i.e., \(f_{1d(-n)}(x) = f_{1dn}(-x)\)), where all outcomes instead are comprised of negative numbers (i.e., `2d(-6)` has outcomes like \((-3, -5)\)). It's easy enough to map outcomes & events from `1d(-n)` to `1dn` and vice versa.

Now if we used this inverted distribution to do the same calculation of dropping the "highest" values, it would instead drop the *highest negative* values in `kd(-n)`, which would correspond to the *lowest positive* values in `kdn`.

Mathematically, we can express the above concept as follows:

$$
f_{k\text{d}n \text{ drop low }m} (x) = f_{k\text{d}(-n) \text{ drop high }m} (-x)
$$

In this way, we can reuse `roll_kdn_drop_high(k, d, m)` to calculate `roll_kdn_drop_low(k, d, m)`, without having to reformulate the previous algorithm at all:

```python
def roll_kdn_drop_low(k: int, d: int, m: int):
    result = roll_kdn_drop_high(k, d, m)
    result.seq = result.seq[::-1]
    return result
```

Note that the above Python function also takes advantage of some simplifications:
- for the case of dropping highs/lows from `kdn` specifically, since `1dn` always has a symmetric distribution, it doesn't have to be reversed in the first step.
- the offset doesn't have to change since the range of valid score events (i.e., the offset and the array length) is the same for `kdn drop high m` and `kdn drop low m`, and doing two reversals leaves the original offset unchanged.

## refactors for working with generic distributions

After writing Python functions like the ones described in the prior sections, I realized that they could be made even more generic. The functions I wrote that calculated distributions for repeated rolls (i.e., `roll_kdn`, `roll_kdn_drop_lowest`, etc.) didn't actually rely on any specific features of dice-roll distributions; in fact, they could be reworked to accept a generic distribution parameter and operate directly on that with few changes to the original algorithm.

For example, `roll_kdn` was refactored into `roll_k`:

```python
def roll_k(dist: SequenceWithOffset, k: int) -> SequenceWithOffset:
    result = roll_0dn()
    for _ in range(k):
        result = result.convolve(dist)
    return result
```

TODO - hopefully this communicate the gist. For more specifics, check out [the real source code](https://github.com/CrepeGoat/heart-of-the-dice/blob/v0.1.1/dice/calc.py).

These modifications mean that we can calculate arbitrary nestings of repeated distributions with dropped high or low values. For example, calculating the distribution for the sum of rolled stats could be done by running:

```python
roll_k(roll_k_drop_low(roll_1dn(n=6), k=4, drop=1), k=6)
```

which wouldn't have been possible to calculate with the code before the refactor.

# hath god forsaken me?

And with the above change, the calculation functions are effectively feature-complete to calculate any probability result I could want. So now, it was time for answers.

After writing out this math & code, I opened up a Python interpreter, loaded in the code, and started running some numbers:

```pycon
>>> from dice import calc as dc
>>> import numpy as np
```

- what are the odds of getting the stat sums that I got for my two stat groups?
  ```pycon
  >>> sum([17, 5, 14, 14, 15, 9])
  74
  >>> sum([12, 15, 5, 15, 10, 15])
  72
  >>>
  >>> stat_sum = dc.roll_k(dc.roll_k_droplow(dc.roll_1dn(6), k=4, drop=1), k=6)
  >>> stat_cdist = np.add.accumulate(stat_sum.seq)
  >>>
  >>> stat_cdist[74 - stat_sum.offset] / stat_cdist[-1]
  np.float64(0.5507508601943684)
  >>> stat_cdist[72 - stat_sum.offset] / stat_cdist[-1]
  np.float64(0.4377304041567714)
  ```
  -> these are the 55th and 43rd percentiles; both percentiles are within 10% of the median, the center of the distribution, which is quite close. Not lucky, not unlucky.

- what are the chances of rolling a \(5\) or lower for a single stat?
  ```pycon
  >>> dc.roll_k_droplow(dc.roll_1dn(6), 4, 1)
  SequenceWithOffset(seq=array([  1,   4,  10,  21,  38,  62,  91, 122, 148, 167, 172, 160, 131,
          94,  54,  21], dtype=uint64), offset=3)
  >>> (1 + 4 + 10) / (6 ** 4)
  0.011574074074074073
  ```
  -> ~1.16%, or about \(\frac{5}{432}\) *(note that this event has \(1 + 4 + 10 = 15\) outcomes; this is less likely than rolling the highest number, an 18, which has \(21\) outcomes.)*
- what are the chances of rolling a \(5\) or lower for *any* of the six stats in a stat group?
  ```pycon
  >>> p = 15 / (6 ** 4)
  >>> 1 - (1 - p) ** 6
  0.06746579772408667
  ```
  -> ~6.75%, about \(\frac{5}{74}\)
- what are the chances of rolling a \(5\) or lower *in both stat groups*?
  ```pycon
  >>> p_g = 1 - (1 - p) ** 6
  >>> p_g ** 2
  0.004551633862547378
  ```
  **-> ~0.455%**, about \(\frac{1}{219}\)
- how unlucky is that compared to rolling a 1 with advantage?
  ```pycon
  >>> d20_adv = dc.roll_k_droplow(dc.roll_1dn(20), k=2, drop=1)
  >>> d20_adv.seq / (20 ** 2)
  array([0.0025, 0.0075, 0.0125, 0.0175, 0.0225, 0.0275, 0.0325, 0.0375,
         0.0425, 0.0475, 0.0525, 0.0575, 0.0625, 0.0675, 0.0725, 0.0775,
         0.0825, 0.0875, 0.0925, 0.0975])
  ```
  -> rolling a 5 for both stat blocks is not as bad as crit-failing with advantage (0.25%, \(\frac{1}{400}\)), but is worse than rolling a 2 with advantage (0.75%, \(\frac{3}{400}\)), which is pretty bad.

...god dammit.
