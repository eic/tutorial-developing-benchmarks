---
title: Setup
---

Tutorial participants will need 
- a working eic-shell installation

This tutorial shows analyzers how to turn their analysis script into a benchmark. To do this, we will start with a ready-made bare-bones analysis. This is modeled after the u-channel rho benchmark, but is stripped down to just a few crucial parts.

To start, download the following 
- this analysis script: [`uchannelrho.cxx`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/uchannelrho.cxx)
- this plotting macro: [`plot_rho_physics_benchmark.C`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/prefinal/plot_rho_physics_benchmark.C)
- this style header: [`RiceStyle.h`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/RiceStyle.h)

We will also start by running over a file from the simulation campaign. Download it to your workspace:
```bash
xrdcp root://dtn-eic.jlab.org//work/eic2/EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.0020.eicrecon.tree.edm4eic.root ./
```


{% include links.md %}
