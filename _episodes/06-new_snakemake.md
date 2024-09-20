---
title: "Exercise 6: New workflow management with Snakemake"
teaching: 10
exercises: 10
questions:
- "How does one share data analysis workflows?"
objectives:
- "Learn basics of creating Snakemake workflows"
keypoints:
- "Snakemake allows one to run their data analyses and share them with others"
---

In the previous exercise we've exercised GitLab CI, which is not intended for a large-scale data analysis. Workflows running on GitLab CI are harder to test locally or at a computing farm. In this exercise we will practice using [Snakemake](https://snakemake.github.io/) workflow management system to define data analysis pipelines.

Snakemake comes with a great [documentation](https://snakemake.readthedocs.io), you are encouraged to read it. For now, let's cover its suggested use for needs of defining ePIC benchmarks.

## Getting started with Snakemake
In order to demonstrate the advantages of using snakefiles, let's start using them for our analysis.
First let's use snakemake to grab some simulation campaign files from the S3 storage space. In your `benchmarks/your_benchmark/` directory make a new file called `Snakefile`.
Open the file and add these lines:
```snakemake
import os
from snakemake.remote.S3 import RemoteProvider as S3RemoteProvider
from snakemake.remote.HTTP import RemoteProvider as HTTPRemoteProvider

S3 = S3RemoteProvider(
    endpoint_url="https://dtn01.sdcc.bnl.gov:9000",
    access_key_id=os.environ["S3_ACCESS_KEY"],
    secret_access_key=os.environ["S3_SECRET_KEY"],
)

rule your_benchmark_campaign_reco_get:
    input:
        lambda wildcards: S3.remote(f"eictest/EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.{wildcards.INDEX}.eicrecon.tree.edm4eic.root"),
    output:
        "../../sim_output/campaign_24.07.0_rho_10x100_uChannel_Q2of0to10_hiDiv_{INDEX}_eicrecon.edm4eic.root",
    shell:
        """
echo "Getting file for INDEX {wildcards.INDEX}"
ln {input} {output}
"""
```

## Conclusion

In this exercise we've built an analysis workflow using Snakemake. That required us to think about the flow of the data and come up with a file naming scheme to reflect it. This approach can be scaled between local testing with handful of files and largely parallel analyses on full datasets.

{% include links.md %}
