---
title: "D&D Dice Calculator pt. 3 - splines make math rock guesser go brrrr?"
author: me
date: 2024-12-09T23:34:00-07:00
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
  - gottagofast
  - kindacool
---

In the last post, I explained a convolution-based method for calculating dice roll probabilities for various situations, including rolling arbitrary-sided dice, rolling multiple times, dropping the lowest or highest values rolled, and nesting these operations together to calculate probabilities for more complex situations.

Here I want to investigate another method for calculating dice rolls, i.e. polynomial splines, and show how it provides a performance improvement over the convolution-based methods discussed before for specific problems.

# outline

- `kdn`
  - notice that distributions are polynomial splines
  - investigate derivatives -> see pascal's triangle
  - how to calculate integral
  - write function
  - time = \(O(k^2 n) < O(k^2 n^2)\)
- `kdn dh (k-1)`
  - notice is also spline, w/ only one segment
  - investigate derivatives -> see foreign numbers with clear pattern -> eulerian numbers
  - write function
  - time = \(O(k n) < O(k^2 n^3)\)
- `kdn dh (k-2)`
  - investigate derivatives -> see alternating pattern, abs & re-derive -> see eulerian numbers
  - write function
  - time = \(O(4k n) < O(2k^2 n^3)\)
    - pattern? - time = \(O((k-m)^2 k n)\)
- `kdn dh m`
  - investigate derivatives -> ?
    - is there no pattern? like how 6+th-root polynomials have no general solution, it works for the first few but not all of them?
  - leads:
    - more sensible under different spline equations, e.g. b-splines?