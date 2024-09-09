---
title: "Excercise 3: Making Useful Figures"
teaching: 10
exercises: 10
questions:
- "How does one make useful benchmark figures?"
objectives:
- "Learn what information is useful to ePIC in developing benchmark artifacts and figures"
keypoints:
- "Clearly-labeled figures showing both physics results and detector performance plots will be helpful during detector and software development."
---

We've discussed how to plug your analysis script into GitLab's CI, and monitor it using pipelines. We'll now briefly discuss how to make figures for your benchmark that are useful to both yourself and to others. 

## Making benchmark plots

The plots created in a benchmark should be designed to be legible not only to yourself, but to those working on detector and software development. This means benchmark plots should be clearly labeled with

- beam settings if any (ep, eAu, ... and 5$\times$41, 10$\times$100 GeV, ...)
- event generator used
- kinematic cuts ($x$, $Q^2$, $W$)
- EPIC label
- legible axis labels

Ideally, benchmark designers should aim to design paper-ready figures labeled with large clear text and a [perceptual color palette](https://root.cern/blog/rainbow-color-map/). Remember that we want to create figures that can be used as-is in the TDR, in conference presentations, and in evaluating detector and software performance.
