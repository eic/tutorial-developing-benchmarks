---
title: "Exercise 1: Analysis Scripts and Snakemake"
teaching: 20
exercises: 10
questions:
- "How does one set up data analysis workflows?"
objectives:
- "Learn basics of creating Snakemake workflows"
keypoints:
- "Snakemake allows one to run their data analyses and share them with others"
---

In this exercise we start with a ready-made analysis script that we're running locally. We'll practice using [Snakemake](https://snakemake.github.io/) workflow management system to define data analysis pipelines.

Snakemake comes with a great [documentation](https://snakemake.readthedocs.io), you are encouraged to read it. For now, let's cover its suggested use for needs of defining ePIC benchmarks.

## Starting from an Analysis Script

We're going to go all the way from an analysis script to a fully-integragrated benchmark with GitLab's continuous integration (CI).

First launch eic-shell 
```bash
./eic-shell
```
then create a working directory
```bash
mkdir tutorial_directory
mkdir tutorial_directory/starting_script
cd tutorial_directory/starting_script
```

Copy the following files to this working directory:
- this analysis script: [`uchannelrho.cxx`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/uchannelrho.cxx)
- this plotting macro: [`plot_rho_physics_benchmark.C`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/prefinal/plot_rho_physics_benchmark.C)
- this style header: [`RiceStyle.h`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/RiceStyle.h)

We will also start by running over a file from the simulation campaign. Download it to your workspace:
```bash
xrdcp root://dtn-eic.jlab.org//work/eic2/EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.0020.eicrecon.tree.edm4eic.root ./
```

Organize files into `analysis` and `macros` directories:
```bash
mkdir analysis
mv uchannelrho.cxx analysis/
mkdir macros
mv plot_rho_physics_benchmark.C macros/
mv RiceStyle.h macros/
```

Run the analysis script over the simulation campaign output:
```bash
root -l -b -q 'analysis/uchannelrho.cxx+("rho_10x100_uChannel_Q2of0to10_hiDiv.0020.eicrecon.tree.edm4eic.root","output.root")'
```
Now make a directory to contain the benchmark figures, and run the plotting macro:
```bash
mkdir output_figures/
root -l -b -q 'macros/plot_rho_physics_benchmark.C("output.root")'
```

You should see some errors like
~~~
Error in <TTF::SetTextSize>: error in FT_Set_Char_Size
~~~
{: .error}
but the figures should be produced just fine. If everything's run correctly, then we have a working analysis! 


With this analysis as a starting point, we'll next explore using Snakemake to define an analysis workflow.


## Getting started with Snakemake

We'll now use a tool called Snakemake to define an analysis workflow that will come in handy when building analysis pipelines.

In order to demonstrate the advantages of using snakefiles, let's start using them for our analysis.
First let's use snakemake to grab some simulation campaign files from the online storage space. In your `tutorial_directory/starting_script/` directory make a new file called `Snakefile`.
Open the file and add these lines:
```python
import os

# Set environment mode (local or eicweb)
ENV_MODE = os.getenv("ENV_MODE", "local")  # Defaults to "local" if not set
# Output directory based on environment
OUTPUT_DIR = "../../sim_output/" if ENV_MODE == "eicweb" else "sim_output/"
# Benchmark directory based on environment
BENCH_DIR = "benchmarks/your_benchmark/" if ENV_MODE == "eicweb" else "./"

rule your_benchmark_campaign_reco_get:
    output:
        f"{OUTPUT_DIR}rho_10x100_uChannel_Q2of0to10_hiDiv.{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic.root",
    shell: """
xrdcp root://dtn-eic.jlab.org//work/eic2/EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.{wildcards.INDEX}.eicrecon.tree.edm4eic.root {output}
"""
```

If you're having trouble copying and pasting, you can also copy from [here](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/Snakefile).

Thinking ahead to when we want to put our benchmark on eicweb, we add this `ENV_MODE` variable which allows us to specify paths differently based on whether we're running locally or in GitLab's pipelines.

We also defined a new rule: `your_benchmark_campaign_reco_get`. This rule defines how to download a single file from the JLab servers to the location `sim_output`.

After saving the Snakefile, let's try running it. 

The important thing to remember about Snakemake is that Snakemake commands behave like requests. So if I want Snakemake to produce a file called `output.root`, I would type `snakemake --cores 2 output.root`. If there is a rule for producing `output.root`, then Snakemake will find that rule and execute it. We've defined a rule to produce a file called `../../sim_output/rho_10x100_uChannel_Q2of0to10_hiDiv.{INDEX}.eicrecon.tree.edm4eic.root`, but really we can see from the construction of our rule that the `{INDEX}` is a wildcard, so we should put a number there instead. Checking out the [files on S3](https://dtn01.sdcc.bnl.gov:9001/buckets/eictest/browse/RVBJQy9SRUNPLzI0LjA3LjAvZXBpY19jcmF0ZXJsYWtlL0VYQ0xVU0lWRS9VQ0hBTk5FTF9SSE8vMTB4MTAwLw==), we see files with indices from `0000` up to `0048`. Let's request that Snakemake download the file `rho_10x100_uChannel_Q2of0to10_hiDiv.0005.eicrecon.tree.edm4eic.root`:
```bash
snakemake --cores 2 sim_output/rho_10x100_uChannel_Q2of0to10_hiDiv.0000.eicrecon.tree.edm4eic.root
```

Snakemake now looks for the rule it needs to produce that file. It finds the rule we wrote, and it downloads the file. Check for the file:
```bash
ls sim_output/
    rho_10x100_uChannel_Q2of0to10_hiDiv.0000.eicrecon.tree.edm4eic.root
```

Okay whatever... so we download a file. It doesn't look like Snakemake really adds anything at this point.

But the benefits from using Snakemake become more apparent as the number of tasks we want to do grows! Let's now add a new rule below the last one:

```python
rule your_benchmark_analysis:
    input:
        script=f"{BENCH_DIR}analysis/uchannelrho.cxx",
        data=f"{OUTPUT_DIR}rho_10x100_uChannel_Q2of0to10_hiDiv.{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic.root",
    output:
        plots=f"{OUTPUT_DIR}campaign_24.07.0_{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic/plots.root",
    shell:
        """
mkdir -p $(dirname "{output.plots}")
root -l -b -q '{input.script}+("{input.data}","{output.plots}")'
"""
```

This rule runs an analysis script to create ROOT files containing plots. The rule uses the simulation campaign file downloaded from JLab as input data, and it runs the analysis script `uchannelrho.cxx`.

Now let's request the output file `"sim_output/campaign_24.07.0_0005.eicrecon.tree.edm4eic/plots.root"`. When we request this, Snakemake will identify that it needs to run the new `your_benchmark_analysis` rule. But in order to do this, it now needs a file we don't have: `sim_output/rho_10x100_uChannel_Q2of0to10_hiDiv.0005.eicrecon.tree.edm4eic.root` because we only downloaded the file with index `0000` already. What Snakemake will do automatically is recognize that in order to get that file, it first needs to run the `your_benchmark_campaign_reco_get` rule. It will do this first, and then circle back to the `your_benchmark_analysis` rule. 

Let's try it out:
```bash
snakemake --cores 2 sim_output/campaign_24.07.0_0005.eicrecon.tree.edm4eic/plots.root
```

You should see something like this: 
![Snakemake output second rule]({{ page.root }}/fig/snakemake_output_rule2_new.png)
Check for the output file:
```bash
ls sim_output/campaign_24.07.0_0005.eicrecon.tree.edm4eic/
```
You should see `plots.root`.

That's still not very impressive. Snakemake gets more useful when we want to run the analysis code over a lot of files. Let's add a rule to do this:

```python
rule your_benchmark_combine:
    input:
        lambda wildcards: expand(
           f"{OUTPUT_DIR}campaign_24.07.0_{% raw %}{{INDEX:04d}}{% endraw %}.eicrecon.tree.edm4eic/plots.root",
           INDEX=range(int(wildcards.N)),
        ),	
    wildcard_constraints:
        N="\d+",
    output:
        f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots.root",
    shell:
        """
hadd {output} {input}
"""
```

On its face, this rule just adds root files using the `hadd` command. But by specifying the number of files you want to add, Snakemake will realize those files don't exist, and will go back to the `your_benchmark_campaign_reco_get` rule and the `your_benchmark_analysis` rule to create them.

Let's test it out by requesting it combine 10 files:
```bash
snakemake --cores 2 sim_output/campaign_24.07.0_combined_10files.eicrecon.tree.edm4eic.plots.root
```
It will spend some time downloading files and running the analysis code. Then it should hadd the files:
![Snakemake output third rule]({{ page.root }}/fig/snakemake_output_rule3_new.png)

Once it's done running, check that the file was produced:
```bash
ls sim_output/campaign_24.07.0_combined_10files*
```

Now let's add one more rule to create benchmark plots:
```python
rule your_benchmark_plots:
    input:
        script=f"{BENCH_DIR}macros/plot_rho_physics_benchmark.C",
        plots=f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots.root",
    output:
        f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots_figures/benchmark_rho_mass.pdf",
    shell:
        """
if [ ! -d "{input.plots}_figures" ]; then
    mkdir "{input.plots}_figures"
    echo "{input.plots}_figures directory created successfully."
else
    echo "{input.plots}_figures directory already exists."
fi
root -l -b -q '{input.script}("{input.plots}")'
"""
```

Now run the new rule by requesting a benchmark figure made from 10 simulation campaign files:
```bash
snakemake --cores 2 sim_output/campaign_24.07.0_combined_10files.eicrecon.tree.edm4eic.plots_figures/benchmark_rho_mass.pdf
```

Now check that the three benchmark figures were created: 
```
ls sim_output/campaign_24.07.0_combined_10files.eicrecon.tree.edm4eic.plots_figures/*.pdf
```
You should see three pdfs. 
We did it!

Now that our Snakefile is totally set up, the big advantage of Snakemake is how it manages your workflow. 
If you edit the plotting macro and then rerun:
```bash
snakemake --cores 2 sim_output/campaign_24.07.0_combined_10files.eicrecon.tree.edm4eic.plots_figures/benchmark_rho_mass.pdf
```
Snakemake will recognize that simulation campaign files have already been downloaded, that the analysis scripts have already run, and the files have already been combined. It will only run the last step, the plotting macro, if that's the only thing that needs to be re-run.

If the analysis script changes, Snakemake will only re-run the analysis script and everything after.

If we want to scale up the plots to include 15 simulation campaign files instead of just 10, then for those 5 extra files only Snakemake will rerun all the steps, and combine with the existing 10 files.


The final Snakefile should look like this:
```python
import os

# Set environment mode (local or eicweb)
ENV_MODE = os.getenv("ENV_MODE", "local")  # Defaults to "local" if not set
# Output directory based on environment
OUTPUT_DIR = "../../sim_output/" if ENV_MODE == "eicweb" else "sim_output/"
# Benchmark directory based on environment
BENCH_DIR = "benchmarks/your_benchmark/" if ENV_MODE == "eicweb" else "./"

rule your_benchmark_campaign_reco_get:
    output:
        f"{OUTPUT_DIR}rho_10x100_uChannel_Q2of0to10_hiDiv.{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic.root",
    shell: """
xrdcp root://dtn-eic.jlab.org//work/eic2/EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.{wildcards.INDEX}.eicrecon.tree.edm4eic.root {output}
"""

rule your_benchmark_analysis:
    input:
        script=f"{BENCH_DIR}analysis/uchannelrho.cxx",
        data=f"{OUTPUT_DIR}rho_10x100_uChannel_Q2of0to10_hiDiv.{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic.root",
    output:
        plots=f"{OUTPUT_DIR}campaign_24.07.0_{% raw %}{{INDEX}}{% endraw %}.eicrecon.tree.edm4eic/plots.root",
    shell:
        """
mkdir -p $(dirname "{output.plots}")
root -l -b -q '{input.script}+("{input.data}","{output.plots}")'
"""

rule your_benchmark_combine:
    input:
        lambda wildcards: expand(
           f"{OUTPUT_DIR}campaign_24.07.0_{% raw %}{{INDEX:04d}}{% endraw %}.eicrecon.tree.edm4eic/plots.root",
           INDEX=range(int(wildcards.N)),
        ),	
    wildcard_constraints:
        N="\d+",
    output:
        f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots.root",
    shell:
        """
hadd {output} {input}
"""

rule your_benchmark_plots:
    input:
        script=f"{BENCH_DIR}macros/plot_rho_physics_benchmark.C",
        plots=f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots.root",
    output:
        f"{OUTPUT_DIR}campaign_24.07.0_combined_{% raw %}{{N}}{% endraw %}files.eicrecon.tree.edm4eic.plots_figures/benchmark_rho_mass.pdf",
    shell:
        """
if [ ! -d "{input.plots}_figures" ]; then
    mkdir "{input.plots}_figures"
    echo "{input.plots}_figures directory created successfully."
else
    echo "{input.plots}_figures directory already exists."
fi
root -l -b -q '{input.script}("{input.plots}")'
"""

```



## Conclusion

In this exercise we've built an analysis workflow using Snakemake. That required us to think about the flow of the data and come up with a file naming scheme to reflect it. This approach can be scaled between local testing with handful of files and largely parallel analyses on full datasets.

{% include links.md %}
