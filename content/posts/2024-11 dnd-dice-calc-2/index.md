---
title: "D&D Dice Calculator - Math & Malice"
author: me
date: 2024-11-19T15:09:18-07:00
draft: true
toc: false
images:
tags: 
  - d&d
  - coding
  - python
  - math
  - statistics
  - probability
  - imnotbitter
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

...but *how* unlucky is this, exactly?

In the last post I talked about the coding side of building a D&D dice probability calculator / visualization tool. In this post I want to dip into the (imo easier) math behind calculating these distributions.

# some setup

Before getting into the math, I want to make sure readers understand some fundamental background about probability and statistics.

- an [*experiment*](https://en.wikipedia.org/wiki/Experiment_(probability_theory)) is a real-world process, that can be repeated, that has a well-defined set of [*outcomes*](https://en.wikipedia.org/wiki/Outcome_(probability)). E.g., rolling a `d6` is an experiment, and the outcomes are the numbers 1, 2, 3, 4, 5, and 6.
- an [*event*](https://en.wikipedia.org/wiki/Event_(probability_theory)) is a collection of outcomes. E.g., rolling an even number on a `d6` is an event, which is a collection of the outcomes \({2, 4, 6}\).
- a [*probability distribution*](https://en.wikipedia.org/wiki/Probability_distribution) (or in the case of dice rolls specifically, a [*probability mass function (PMF)*](https://en.wikipedia.org/wiki/Probability_mass_function)) is a mathematical function that takes as input a possible outcome / event, and outputs how probable it is for that result to be realized.

  For example, if a coin had a 55% change to land on heads when flipped (and a 45% of the same for tails), the PMF \(f\) would be defined as \(f(\text{heads}) := 0.55\),  \(f(\text{tails}) := 0.45\), \(f(\text{anything else}) := 0\).
- [convolution](https://en.wikipedia.org/wiki/Convolution) is a mathematical operation that (in this case) takes as input two distributions, and as output generates a third distribution. In this case, we'll use convolution to [calculate the distribution for the sum of two experiments](https://en.wikipedia.org/wiki/Convolution_of_probability_distributions) (e.g., `2d6` = `1d6` + `1d6`).

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
        index_high = max(self._index_end(), other._index_end())

        seq = np.zeros(
            index_high - index_low,
            dtype=(self.seq[0] + other.seq[0]).dtype,
        )
        seq[self.offset - index_low : self._index_end() - index_low] = self.seq
        seq[other.offset - index_low : other._index_end() - index_low] += other.seq

        return SequenceWithOffset(seq=seq, offset=index_low)
```

Re: runtime complexity - unfortunately the docs aren't super clear, so I'm left to either read the source code (this is for fun I don't wanna) or guess.

- From some external research it seems that a naive algorithm would run in \(O(n_1 n_2)\) time.
- There are other algorithms ([Karatsuba @ \(O(n^{1.58})\)](https://en.wikipedia.org/wiki/Karatsuba_algorithm), [Tom-Cook @ \(O(n^{1.46})\)](https://en.wikipedia.org/wiki/Toom–Cook_multiplication)) that have large coefficients that might be impractical for typical NumPy users.
- There's also [a method for convolution that involves FFT's](https://docs.scipy.org/doc/scipy-1.14.1/reference/generated/scipy.signal.fftconvolve.html), but since that only works on floating-point data I elected not to use it.

-> Moving forward I'll assume \(O(n^2)\) time complexity for this function.

## probabilities vs. outcomes

One small change in perspective I want to implement here: instead of thinking in terms of probability distributions, I instead want to think in terms of *outcome-count distributions*.

So for example, let's consider the `2d6` case. There are \(6 \times 6 = 36\) different possible *outcomes*. The *events* we want to consider are the possible sums resulting from any of these 36 different outcomes; for `2d6` there are \(11\) such different events, which include all the integers from \(2\) (realized by rolling two \(1\)'s) to \(12\) (realized by rolling two \(6\)'s).

Now let's say we're interested in how we might roll the value \(5\). There are 4 ways to roll a \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\).

- in terms of *probability*, there is a \(\frac{4}{36}\) chance of rolling a \(5\) (assuming fair dice).
- in terms of *outcome counts*, there are 4 outcomes that result in a roll of \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\) are those outcomes.
- as long as the dice are fair, then one can convert between the two using the equation:
  $$
  (\text{probability that value} = x) = \frac{(\text{# of outcomes where value} = x)}{(\text{total # of outcomes})}
  $$

So when calculating probability distributions, I want to **first calculate an outcome-count function (OCF), and then convert that into a probability mass function (PMF) using the above equation on each event value**.

There are a couple of advantages to doing this:

1. Intermediate computations are all performed on integers instead of floating-point numbers. This means
    - there's no rounding errors that can accumulate through the computation,
    - each computer-stored number can store more information (64-bit floating-point numbers only have 52 bits of decimal-point precision), and
    - any resulting values that are too large to store will generally trigger [a hardware-implemented error flag](https://en.wikipedia.org/wiki/Integer_overflow#Flags) that [is easier to respond to](https://doc.rust-lang.org/std/primitive.i64.html#method.checked_add).
1. It makes recursive calculations simpler. I'll discuss what I mean by this later on.

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

Visualized, the plot for e.g. `1d6` looks like this:

![Plot for OCF of 1d6](plot-1d6.png)

Not super exciting, but definitions aren't really supposed to be.

## rolling `kdn`

For multiple dice, we can use the convolution operation mentioned above. Specifically, we can compose the OCF for `kdn` by convolving together the OCF of `1dn` with itself `k` times.

As a matter of fact, convolution is such a commonly-used operation that NumPy has a builtin convolution function. 

## rolling `kdn` and dropping the lowest die

## rolling `kdn` and dropping the lowest `m` dice