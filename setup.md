---
title: Setup
---

Tutorial participants will need

- a working [eic-shell installation](https://eic.github.io/tutorial-setting-up-environment/)
- if you'll be following along on a remote cluster through ssh, you may also need to [set up ssh-keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent). This will allow you to perform GitHub operations while using ssh.

This tutorial shows analyzers how to turn their analysis script into a benchmark. To do this, we will start with a ready-made bare-bones analysis. This is modeled after the u-channel rho benchmark, but is stripped down to just a few crucial parts.

To start, download the following

- this analysis script: [`uchannelrho.cxx`](files/uchannelrho.cxx)
- this plotting macro: [`plot_rho_physics_benchmark.C`](files/prefinal/plot_rho_physics_benchmark.C)
- this style header: [`RiceStyle.h`](files/RiceStyle.h)

We will also start by running over a file from the simulation campaign. Download it to your workspace:

```bash
xrdcp root://dtn-eic.jlab.org//volatile/eic/EPIC/RECO/25.10.2/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.0020.eicrecon.edm4eic.root ./
```
