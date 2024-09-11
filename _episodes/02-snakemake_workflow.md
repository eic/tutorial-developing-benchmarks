---
title: "Exercise 2: Workflow management with Snakemake"
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

## Describing a data analysis pipeline

Consider a following ROOT macro:
```c++
// bench.C

#include <string>

#include <ROOT/RDataFrame.hxx>

void bench(const std::string& input_file, const std::string& output_file) {
  ROOT::EnableImplicitMT();
  ROOT::RDataFrame rdf("events", input_file);

  TFile output(output_file.c_str(), "RECREATE");

  auto df = rdf
    .Define("Q2_sim", "InclusiveKinematicsTruth.Q2")
    .Define("x_sim", "InclusiveKinematicsTruth.x")
    ;
  auto h_x_Q2 = df.Histo2D({"h_x_Q2", "x,Q^2 distribution", 100, 0., 1., 100, 0., 100.}, "x_sim", "Q2_sim");
  h_x_Q2->SetOption("COLZ");
  h_x_Q2->Clone();

  output.Write();
}
```
This implements a very simple analysis to project our input events onto $x$ - $Q^2$ plane. To run it, we need to get an input file:

```shell
mc cp S3/eictest/EPIC/RECO/23.12.0/epic_craterlake/DIS/NC/10x100/minQ2=10/pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root .
```

And then run the analysis macro itself (please, do create `bench.C` with the contents above first):

```shell
root -l -b -q 'bench.C("pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root", "plots.root")'
```

This may produce warnings like:

```
Warning in <TClass::Init>: no dictionary for class podio::version::Version is available
```

It is safe to ignore those in this exercise.

Another macro could then be used to render the histograms as plots:

```c++
// render_plots.C

#include <string>

#include <TCanvas.h>
#include <TFile.h>
#include <TH2D.h>

void render_plots(const std::string& input_file, const std::string& output_prefix) {
  TFile input(input_file.c_str());

  TCanvas c;
  
  dynamic_cast<TH2D*>(input.Get("h_x_Q2"))->Draw();

  c.SaveAs((output_prefix + "h_x_Q2.png").c_str());
  c.SaveAs((output_prefix + "h_x_Q2.pdf").c_str());
}
```

Which can be invoked like so:

```shell
root -l -b -q 'render_plots.C("plots.root", "")'
```

You should be able to see the distributions on the image files `h_x_Q2.png` and `h_x_Q2.pdf`.

![x,Q^2 distribution]({{ page.root }}/fig/initial_h_x_Q2.png)

Now, how would we share this benchmark a collaborator or with future self? One way would be to write those commands down in a readme file or in the shell history. In the real world, the analyses tend to get big, and it becomes ever more tedious to run those by hand. One could approach this by writing a script:

```bash
#!/bin/sh

set -xue

mc cp S3/eictest/EPIC/RECO/23.12.0/epic_craterlake/DIS/NC/10x100/minQ2=10/pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root .
root -l -b -q 'bench.C("pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root", "plots.root")'
root -l -b -q 'render_plots.C("plots.root", "")'
```

An obvious issue with that approach is that we don't want to re-download the input files on each iteration. And perhaps, we are looking to focus on making changes to `render.C`, then, if we don't modify `bench.C` there isn't a need to re-run it. On the other hand, if we do modify `bench.C`, we don't want forget to re-run it. "Make" systems do the job of tracking those questions for us. Let's use Snakemake instead of shell scipts. Our analysis steps will be described in a file with name `Snakefile` and following contents:

```snakemake
# Snakefile

import shutil

from snakemake.remote.S3 import RemoteProvider as S3RemoteProvider


S3 = S3RemoteProvider(
    endpoint_url="https://eics3.sdcc.bnl.gov:9000",
    access_key_id=os.environ["S3_ACCESS_KEY"],
    secret_access_key=os.environ["S3_SECRET_KEY"],
)


rule download_input:
    input:
        S3.remote("eictest/EPIC/RECO/23.12.0/epic_craterlake/DIS/NC/10x100/minQ2=10/pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root"),
    output:
        "pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root",
    run:
        shutil.move(input[0], output[0])

rule benchmark:
    input:
        script=workflow.source_path("./bench.C"),
        events="pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root",
    output:
        "plots.root",
    shell: """
root -l -b -q '{input.script}("{input.events}", "{output}")'
"""

rule render_plots:
    input:
        script=workflow.source_path("./render_plots.C"),
        plots="plots.root",
    output:
        "h_x_Q2.png",
        "h_x_Q2.pdf",
    default_target: True
    shell: """
root -l -b -q '{input.script}("{input.plots}", "")'
"""
```

This is much more verbose, but it also clearly specifies which steps have to produce which files and what they need for input.

Remove root files that we may have produced previously:

```shell
rm *.root # or snakemake --cores 1 --delete-all-output
```

Let's ask Snakemake to run our analysis:

```shell
snakemake --cores 1
```

This delivers the same output files. Now, let's see what happens if we edit `bench.C`, now add `h_x_Q2->SetXTitle("x");` and `h_x_Q2->SetYTitle("Q^2");` in an appropriate place and re-run the `snakemake --cores 1` command. You should see that it will redo the analysis and rerender the plot. Now do a different change to `render_plots.C`, add `c.SetLogx();` where appropriate and rerun snakemake. What do you see happening?

## Scaling the analysis

When working on a large dataset it is often benefitial to have it split into mutltiple files to be analyzed in with independent processes, possibly running on a computing cluster. The results are then aggregated (e.g. histograms are added). This is known as the MapReduce pattern, and it is a quite powerful technique. In this section we will see how it can be implemented in Snakemake.

First, we will focus on the "Map" part. The idea is to run independently on different input files. Looking at our original `Snakefile`, the file name is hardcoded, which is fine, except we need to let the index part of it (the `.0000.`) to vary. This is when we need to use the pattern matching feature in Snakemake. Change your `download_input` rule to use 

```
"pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{INDEX}.eicrecon.tree.edm4eic.root"
```

instead of

```
"pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.eicrecon.tree.edm4eic.root"
```

The part in curly braces is called wildcard, it needs to be named and present in the input as well as output (so make sure to update both!). You should be able to now request files from S3 with arbitrary indices:

```shell
snakemake --cores 1 pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0123.eicrecon.tree.edm4eic.root
```

Type `ls` to confirm that you now have two different files available.

Make a similar edit to the `benchmark` rule, and try requesting `plots.root`

```shell
snakemake --cores 1 plots.root
```

This should fail in the following way:

~~~
WildcardError in rule benchmark in file Snakefile, line 19:
Wildcards in input files cannot be determined from output files:
'INDEX'
~~~
{: .error}

The problem is that we can't unambiguously determine value of the `{INDEX}` wildcard from `plots.root`. The Snakemake way is to have organize the output files to be named differently and according to how they are processed. For that, change `"plots.root"` to

```
"pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{INDEX}.plots.root"
```

The rule will now look like:

```snakemake
rule benchmark:
    input:
        script=workflow.source_path("./bench.C"),
        events="pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{INDEX}.eicrecon.tree.edm4eic.root",
    output:
        "pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{INDEX}.plots.root",
    shell: """
root -l -b -q '{input.script}("{input.events}", "{output}")'
"""
```

You can now request processing of several files like so:

```shell
snakemake --cores 2 pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000.plots.root pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0123.plots.root
```

This is it for the "Map"! Let's set up "Reduce" now.

Here, we could modify the `render_plots.C` macro to be able to read mutliple `.plots.root` files and make our figures out of those, but that would add a second responsibility to it. A separate macro to merge mutiple `.plots.root` files into one is preferrable. It could probably look like a [tutorials/io/hadd.C](https://root.cern.ch/doc/master/hadd_8C.html) example for ROOT. And, for our purposes, we don't need any custom logic, so we can use the built-in "hadd" utility from the standard ROOT installation. All we need is to write a new Snakemake rule for it.

So what do we need? The rule needs to take several processed files, let's say we use ones with indices from $0$ to $N - 1$ for some user-specified $N$. This kind of logic has to leverage that `Snakefile` accepts use of Python code:

```snakemake
rule aggregate:
    input:
        lambda wildcards: expand(
           "pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{INDEX:04d}.plots.root",
           INDEX=range(int(wildcards.N)),
        ),
    # ...
```

Here, `lambda wildcards: ` just lets Snakemake know that we are inlining a Python function that uses wildcard values. The `expand()` function is provided by Snakemake and is similar to Python's [list comprehensions](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions), it expands a given template based on values provided. We use `{INDEX:04d}` tells to substitute the `INDEX` variable with integer formatting to pad index with zeros up to 4 digits. The `range(int(wildcards.N))` expression creates a list of integers from `0` to `wildcards.N - 1` (the `wildcards.N` is a string that needs to be converted to an integer first).

For `wildcards.N` to be defined, we need to define an output with it. Since the merged files are similar to output files, let's stick to a similar naming, including the `.plots.root` suffix, but add an `aggregate_` prefix to it:

```snakemake
    # ...
    output:
        "aggregate_pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.{N}.plots.root",
    # ...
```

The "hadd" utility is to be invoked with the output file as the first argument and inputs as the following arguments (run `hadd -h` for extra options). We use do it by specifying the `shell` field:

```snakemake
    shell: """
    hadd {input} {output}
"""
```

(Snakemake expands `{input}` from list as its space-separated values, which works out for for providing multiple arguments here.)

You can fire off processing of 2 files:

```
snakemake --cores 2 aggregate_pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0002.plots.root
```

Or, first, update the `render_plots` rule to process arbitrary plot files

```snakemake
rule render_plots:
    input:
        script=workflow.source_path("./render_plots.C"),
        plots="{PREFIX}.plots.root",
    output:
        "{PREFIX}/h_x_Q2.png",
        "{PREFIX}/h_x_Q2.pdf",
    default_target: True
    shell: """
root -l -b -q '{input.script}("{input.plots}", "{wildcards.PREFIX}/")'
"""
```

Notice how we now supply the value of a wildcard to the ROOT macro. This can be useful to adjust plotting labels or global analysis parameters based on what is being processed.

The full analysis can now be run as:

```shell
snakemake --cores 2 aggregate_pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0002/h_x_Q2.png
```

If you are happy with the result from 2 files, you can increase $N$:

```shell
snakemake --cores 2 aggregate_pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0004/h_x_Q2.png
```

Notice how the already processed files are not be reprocessed, since Snakemake will figure out that they are up to date and only run what it needs to.

We did not lose our ability to analyze single files at a time:

```shell
snakemake --cores 2 pythia8NCDIS_10x100_minQ2=10_beamEffects_xAngle=-0.025_hiDiv_5.0000/h_x_Q2.png
```

> You are now challenged to extend our workflow to support analyzing data with $Q^2_{min}$ of 1, 10 and 1000. How would you go about implementing that?
{: .challenge}

## Conclusion

In this exercise we've built an MapReduce analysis workflow using Snakemake. That required us to think about the flow of the data and come up with a file naming scheme to reflect it. This approach can be scaled between local testing with handful of files and largely parallel analyses on full datasets.

{% include links.md %}
