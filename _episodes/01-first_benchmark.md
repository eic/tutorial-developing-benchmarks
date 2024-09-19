---
title: "Exercise 1: Setting up your first benchmark"
teaching: 20
exercises: 10
questions: How do we create a new pipeline with GitLab CI?
objectives:
- "Go through the process of contributing benchmarks on GitHub"
- "Learn basics of running on eicweb GitLab CI"
keypoints:
---

## Setting up a repository

Benchmarks are currently organized into two repositories:

- [https://github.com/eic/detector_benchmarks](https://github.com/eic/detector_benchmarks)
- [https://github.com/eic/physics_benchmarks](https://github.com/eic/physics_benchmarks)

Let's make a physics benchmark. We start by cloning the git repository

```bash
git clone https://github.com/eic/physics_benchmarks.git
cd physics_benchmarks
```

(If you get an error here, you might need to set up your SSH keys.)

Please create a feature branch in your local repository:

```bash
git checkout -b pr/your_benchmark_<mylastname>
```

*(Replace `<mylastname>` with your last name or any other nickname.)*

## Defining GitLab Continuous Integration jobs

Let's see what kind of bechmarks are available:

```output
# ls benchmarks
backgrounds  benchmarks.json  demp  diffractive_vm  dis  dvcs  dvmp  lambda  neutron  sigma  single  tcs  u_omega
```

Now, create a new directory for your benchmark

```bash
mkdir benchmarks/your_benchmark
```

The Continuous Integration system needs to know what steps it has to execute. This is specified using YAML files. Create a file `benchmarks/your_benchmark/config.yml`. For a detector benchmark, start with [this `config.yml`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/config.yml).

For a physics benchmark and to follow along with this tutorial, create a `config.yml` with the following contents:
~~~
your_benchmark:compile:
  extends: .phy_benchmark 
  stage: compile
  script:
    - echo "You can compile your code here!"

your_benchmark:generate:
  extends: .phy_benchmark 
  stage: generate
  script:
    - echo "I will generate events here!"
    - echo "Add event-generator code to do this"

your_benchmark:simulate:
  extends: .phy_benchmark
  stage: simulate
  script:
    - echo "I will simulate detector response here!"

your_benchmark:reconstruct:
  extends: .phy_benchmark
  stage: reconstruct
  script:
    - echo "Event reconstruction here!"

your_benchmark:analyze:
  extends: .phy_benchmark
  stage: analyze
  needs:
    - ["your_benchmark:reconstruct"]
  script:
    - echo "I will analyze events here!"
    - echo "This step requires that the reconstruct step be completed"

your_benchmark:results:
  extends: .phy_benchmark
  stage: collect
  needs:
    - ["your_benchmark:analyze"]
  script:
    - echo "I will collect results here!"
    - echo "This step requires that the analyze step be completed"

~~~
{: .language-yaml }

The basic idea here is that we are defining the rules for each step of the [pipeline](https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines/102530). 

A few things to note about the `config.yml`:
- The rules take basic bash script as input. Anything you would write in a bash script you can put in the script section of a rule in the `config.yml` file.
- Each rule does not need to do something. In the example `config.yml` given here, each rule is just printing a statement.
- Each rule corresponds to a stage in GitLab's pipelines. So the collect rule in your `config.yml` tells the pipeline what to do when it gets to the collect stage of the [pipeline](https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines/102530).

Since we've just created a new file, we need to let git know about it by staging it:

```shell
git add benchmarks/your_benchmark/config.yml
```

We also need to let the CI system know that we want it to execute steps that we've just defined. For that, it has to be included from the `.gitlab-ci.yml` file. Open it in your text editor of choice and locate lines that look like:

```yaml
include:
  - local: 'benchmarks/diffractive_vm/config.yml'
  - local: 'benchmarks/dis/config.yml'
  - local: 'benchmarks/dvmp/config.yml'
  - local: 'benchmarks/dvcs/config.yml'
  - local: 'benchmarks/tcs/config.yml'
  - local: 'benchmarks/u_omega/config.yml'
  - local: 'benchmarks/single/config.yml'
  - local: 'benchmarks/backgrounds/config.yml'
```

Insert an appropriate line for your newly created `benchmarks/your_benchmark/config.yml`. We will be doing a lot of testing using GitLab's pipelines. We don't need GitLab to simulate every other benchmark while we're still testing ours. To speed things up, you can comment out most other benchmarks. Consider leaving a few uncommented to make sure everything is working right:
```yaml
include:
  #- local: 'benchmarks/diffractive_vm/config.yml'
  - local: 'benchmarks/dis/config.yml'
    #- local: 'benchmarks/dvmp/config.yml'
  #- local: 'benchmarks/dvcs/config.yml'
  #- local: 'benchmarks/tcs/config.yml'
  #- local: 'benchmarks/u_omega/config.yml'
  - local: 'benchmarks/single/config.yml'
  #- local: 'benchmarks/backgrounds/config.yml'
  - local: 'benchmarks/your_benchmark/config.yml'
```

In order to make your benchmark produce artifacts, also add your benchmark to this section, and comment out any benchmarks you commented out above: 
```yaml
summary:
  stage: finish
  needs:
    #- "diffractive_vm:results"
    - "dis:results"
    #- "dvcs:results"
    #- "tcs:results"
    #- "u_omega:results"
    - "single:results"
    #- "backgrounds:results"
    - "your_benchmark:results"
```
Save and close the file.


The change that you've just made needs to be also staged. We will now learn a cool git trick. Run this:

```shell
git add -p
```

Here `-p` stands for `--patch`. This will display unstaged changes to the local files and let you review and optionally stage them. There will be only one change for you to check, so just type <kbd>y</kbd> and press <kbd>Enter</kbd>.

## Submit a GitHub Pull Request

Even though our benchmark doesn't do anything yet, let's submit it to the CI and see it run and do nothing useful. The way to do it is to submit a pull request. We first commit the staged changes to the current branch:

```shell
git commit -m "add benchmarks/your_benchmark"
```

And push that branch from the local repository to the shared repository on GitHub (referenced to as `origin`):

```shell
git push origin pr/your_benchmark_<mylastname>
```
*(Replace `<mylastname>` with your last name.)*

1. This should instruct you to go to `https://github.com/eic/detector_benchmarks/compare/master...pr/your_benchmark_<mylastname>` to create a PR. Follow that link.
3. ![Add a title]({{ page.root }}/fig/github_add_a_title.png) Provide a title like "Adding benchmark for ...".
4. ![Create draft pull request]({{ page.root }}/fig/github_switch_to_draft.png) Since this work is not yet complete, open dropdown menu of the "Create pull requst" button and select "Create draft pull request"
5. ![Draft pull request]({{ page.root }}/fig/github_draft_pull_request.png) Click "Draft pull request"

Your newly created Pull Request will show up.

## Examine CI output on eicweb GitLab

You can now scroll to the bottom of the page and see what checks are running. You may need to wait a bit and/or refresh the page to see a `eicweb/physics_benchmarks (epic_craterlake)` check running.

![Check running on GitHub]({{ page.root }}/fig/github_running_check.png)

Click "Details", it will take you to eicweb GitLab instance. The pipeline will show all the existing jobs. You should be able to see your new jobs. Each stage of the pipeline shown here corresponds to a rule in the `config.yml`:

![Check running on eicweb]({{ page.root }}/fig/configToPipelines.png)

- This example pipeline is viewable [here](https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines/102686). 
- All physics benchmark pipelines are here: [https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines](https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines)
- All detector benchmark pipelines are here: [https://eicweb.phy.anl.gov/EIC/benchmarks/detector_benchmarks/-/pipelines](https://eicweb.phy.anl.gov/EIC/benchmarks/detector_benchmarks/-/pipelines)

You can click on individual jobs and see output they produce during running. Our newly created jobs should produce messages in the output. Real scripts could return errors and those would appear as CI failures.

![Job output on eicweb]({{ page.root }}/fig/eicweb_job_output.png)

There is another important feature that jobs can produce artifacts. They can be any file. Take a look at [this](https://eicweb.phy.anl.gov/EIC/benchmarks/detector_benchmarks/-/jobs/2284270) "collect_results:barrel_ecal" job, click "Browse" button in the right column, then navigate to "results", there will be some plots produced by the benchmars related to performance of the barrel electromagnetic calorimeter.

## Conclusion

We've practiced contributing code that runs within eicweb Continuous Integration system.

You can view these pipelines here: 
- [physics benchmarks](https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines)
- [detector benchmarks](https://eicweb.phy.anl.gov/EIC/benchmarks/detector_benchmarks/-/pipelines)

{% include links.md %}

