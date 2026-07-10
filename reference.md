---
title: Reference
---

## Glossary

Benchmark
: A script that runs detector simulations and analyzes the resulting data to extract quantities
  related to detector performance.

Snakemake
: A workflow management system used to define and run data-analysis pipelines. See the
  [Snakemake documentation](https://snakemake.readthedocs.io).

Pipeline
: The sequence of continuous-integration (CI) jobs (compile, simulate, collect, finish) that GitLab
  runs for a benchmark.

Status flag
: A binary pass/fail indicator produced by a benchmark and summarized at the end of a pipeline to
  alert developers to detrimental changes in software or detector design.

Artifact
: Any file produced by a CI job (for example a figure or JSON file) that is kept and made available
  for download after the pipeline runs.
