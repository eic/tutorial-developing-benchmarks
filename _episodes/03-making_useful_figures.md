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

- beam settings if any (ep, eAu, ... and \(5 \times 41\), \(10 \times 100\) GeV, ...)
- event generator used
- kinematic cuts ($x$, $Q^2$, $W$)
- EPIC label
- legible axis labels

Ideally, benchmark designers should aim to design paper-ready figures labeled with large clear text and a [perceptual color palette](https://root.cern/blog/rainbow-color-map/). Remember that we want to create figures that can be used as-is in the TDR, in conference presentations, and in evaluating detector and software performance.

An example figure from the u-channel rho benchmark is shown here:

<img src="{{ page.root }}/fig/example_benchfig.png" alt="Example benchmark figure" width="800" style="box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.3);">

In this example, the plot is labeled with the collision system and other details:
```c++
TLatex* beamLabel       = new TLatex(0.18, 0.91, "ep 10#times100 GeV");
TLatex* epicLabel       = new TLatex(0.9,0.91, "#bf{EPIC}");
TLatex* kinematicsLabel = new TLatex(0.53, 0.78, "10^{-3}<Q^{2}<10 GeV^{2}, W>2 GeV");
TLatex* generatorLabel  = new TLatex(0.5, 0.83, ""+vm_label+" #rightarrow "+daug_label+" eSTARlight");
```
The axis labels and titles were scaled for clarity:
```c++
histogram->GetYaxis()->SetTitleSize(histogram->GetYaxis()->GetTitleSize()*1.5);
histogram->GetYaxis()->SetLabelSize(histogram->GetYaxis()->GetLabelSize()*1.5);
```
The number of axis divisions was decreased to reduce visual clutter:
```c++
histogram->GetXaxis()->SetNdivisions(5);
```
And margins were increased to allow for more space for larger axis labels:
```c++
virtualPad->SetBottomMargin(0.2);
```

Even a well-labeled figure will still be ambiguous as to what is being plotted. For example, how you define efficiency may differ from another analyzer. It will be useful to include with your benchmark an explainer of each of the various plots that are produced:

<img src="{{ page.root }}/fig/benchmarkplots_explained.png" alt="Benchmark explainer" width="800" style="box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.3);">

This document may be written in LaTeX, exported as a PDF, and then uploaded to [GitHub under your benchmark](https://github.com/eic/physics_benchmarks/blob/pr/u_channel_sweger/benchmarks/u_rho/BenchmarkPlotsExplained.pdf).

The way most users will interact with your benchmark is at the level of the artifacts it produces. When others are looking through the plots produced by your benchmark, this description of your figures should be readily available as an artifact itself. To achieve this, you can use this template tex document:

```tex
%====================================================================%
%                  BENCH.TEX                                       %
%           Written by YOUR NAME                                   %
%====================================================================%
\documentclass{bench}
%
% A useful Journal macro
\def\Journal#1#2#3#4{{#1} {\bf #2}, #3 (#4)}
```



