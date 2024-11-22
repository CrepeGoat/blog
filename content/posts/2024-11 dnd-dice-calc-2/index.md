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

- a **[probability distribution](https://en.wikipedia.org/wiki/Probability_distribution)** (in the case of dice rolls specifically, a *[probability mass function (PMF)](https://en.wikipedia.org/wiki/Probability_mass_function)*) is a mathematical function that takes as input a possible result, and outputs how probable it is for that result to be realized.

  For example, if a coin had a 55% change to land on heads when flipped (and a 45% of the same for tails), the PMF \(f\) would be defined as \(f(\text{heads}) := 0.55\),  \(f(\text{tails}) := 0.45\).
- a 

TODO

- 
- probability distribution - a mapping of specific rolled sums to a probability
- convolution

# probabilities vs. outcomes

One small change in perspective I want to implement here: instead of thinking in terms of probability distributions, I instead want to think in terms of *outcome distributions*.

So for example, let's consider the `2d6` case, where there are \(6 \times 6 = 36\) different possible *outcomes*. I also want to distinguish an outcome from a *value*: some outcomes have the same value, e.g. \((2, 4)\) and \((3, 3)\) both have the value \(6\), but they are still considered unique and different outcomes. (I don't think my above definitions for "outcomes" and "values" are widely used, but I need to establish some language to talk about this effectively.)

Now let's say we're interested in how we might roll the value \(5\). There are 4 ways to roll a \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\).

- in terms of *probability*, there is a \(\frac{4}{36}\) chance of rolling a \(5\) (assuming fair dice).
- in terms of *outcome counts*, there are 4 outcomes that result in a roll of \(5\): \({(1, 4), (2, 3), (3, 2), (4, 1)}\) are those outcomes.
- as long as the dice are fair, one can convert between the two using the 

There are a couple reasons 

# calculations

## `kdn`

## `kdn drop lowest 1`

## `kdn drop lowest m`