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

We are giving the Snakefile the S3 access key so it can download the files we request. 

We also defined a new rule: `your_benchmark_campaign_reco_get`. This rule defines how to download a single file from S3 to the location `sim_output`.

After saving the edited file, let's try running it. 

The important thing to remember about Snakemake is that Snakemake commands behave like requests. So if I want Snakemake to produce a file called `output.root`, I would type `snakemake --cores 2 output.root`. If there is a rule for producing `output.root`, then Snakemake will find that rule and execute it. We've defined a rule to produce a file called `../../sim_output/campaign_24.07.0_rho_10x100_uChannel_Q2of0to10_hiDiv_{INDEX}_eicrecon.edm4eic.root`, but really we can see from the construction of our rule that the `{INDEX}` is a wildcard, so we should put a number there instead. Checking out the [files on S3](https://dtn01.sdcc.bnl.gov:9001/buckets/eictest/browse/RVBJQy9SRUNPLzI0LjA3LjAvZXBpY19jcmF0ZXJsYWtlL0VYQ0xVU0lWRS9VQ0hBTk5FTF9SSE8vMTB4MTAwLw==), we see files with indices from `0000` up to `0048`. Let's request the Snakemake download the file `campaign_24.07.0_rho_10x100_uChannel_Q2of0to10_hiDiv_0005_eicrecon.edm4eic.root`:
```bash
snakemake --cores 2 ../../sim_output/campaign_24.07.0_rho_10x100_uChannel_Q2of0to10_hiDiv_0005_eicrecon.edm4eic.root
```

Snakemake now looks for the rule it needs to produce that file. It finds the rule we wrote, and it downloads the file. Check for the file:
```bash
ls ../../sim_output/
    campaign_24.07.0_rho_10x100_uChannel_Q2of0to10_hiDiv_0005_eicrecon.edm4eic.root
```


## Conclusion

In this exercise we've built an analysis workflow using Snakemake. That required us to think about the flow of the data and come up with a file naming scheme to reflect it. This approach can be scaled between local testing with handful of files and largely parallel analyses on full datasets.

{% include links.md %}
