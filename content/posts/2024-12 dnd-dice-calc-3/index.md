---
title: "D&D Dice Calculator pt. 3 - poly splines make math rock guesser go brrrr?"
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
