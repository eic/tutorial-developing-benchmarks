---
title: "Exercise 1: Setting up your first benchmark"
teaching: 20
exercises: 10
questions:
objectives:
- "Go through the process of contributing benchmarks on GitHub"
- "Learn basics of running on eicweb GitLab CI"
keypoints:
---

## Setting up a repository

Benchmarks are currently organized into two repositories:

- [https://github.com/eic/detector_benchmarks](https://github.com/eic/detector_benchmarks)
- [https://github.com/eic/physics_benchmarks](https://github.com/eic/physics_benchmarks)

Let's make a detector benchmark. We start by cloning the git repository

```bash
git clone git@github.com:eic/detector_benchmarks.git
cd detector_benchmarks
```

(If you get an error here, you might need to set up your SSH keys.)

Please create a feature branch in your local repository:

```bash
git checkout -b pr/backwards_ecal_<mylastname>
```

*(Replace `<mylastname>` with your last name or any other nickname.)*

## Defining GitLab Continuous Integration jobs

Let's see what kind of bechmarks are available:

```output
# ls benchmarks
b0_tracker  backgrounds  barrel_ecal  barrel_hcal  material_maps  others  pid  timing  tracking_detectors  zdc
```

Now, create a new directory for your benchmark

```bash
mkdir benchmarks/backwards_ecal
```

The Continuous Integration system needs to know what steps it has to execute. This is specified using YAML files. Create a file `benchmarks/backwards_ecal/config.yml` with the following contents:

~~~
sim:ecal_backwards_electron:
  extends: .det_benchmark 
  stage: simulate
  script:
    - echo "I will be simulating some electron events here!"
    - echo "0.511 MeV" >> $LOCAL_DATA_PATH/electron_mass

sim:ecal_backwards_photon:
  extends: .det_benchmark 
  stage: simulate
  script:
    - echo "I will be simulating some photon events here!"

bench:ecal_backwards_resolution:
  extends: .det_benchmark
  stage: benchmarks
  needs:
    - ["sim:ecal_backwards_electron", "sim:ecal_backwards_photon"]
  script:
    - echo "I will be analyzing events here!"
    - echo "Our intelligence reports that the electron mass is $(cat $LOCAL_DATA_PATH/electron_mass)"
~~~
{: .language-yaml }

The basic idea here is that we define three jobs to be run in two steps. The reason for the separation is to allow parallel execution and for them to be presented nicely in the user interface.

Since we've just created a new file, we need to let git know about it by staging it:

```shell
git add benchmarks/backwards_ecal/config.yml
```

We also need to let the CI system know that we want it to execute steps that we've just defined. For that, it has to be included from the `.gitlab-ci.yml` file. Open it in your text editor of choice and locate lines that look like:

```yaml
include:
  - local: 'benchmarks/backgrounds/config.yml'
  - local: 'benchmarks/tracking_detectors/config.yml'
  - local: 'benchmarks/barrel_ecal/config.yml'
  - local: 'benchmarks/barrel_hcal/config.yml'
  - local: 'benchmarks/zdc/config.yml'
  - local: 'benchmarks/material_maps/config.yml'
  - local: 'benchmarks/pid/config.yml'
  - local: 'benchmarks/timing/config.yml'
  - local: 'benchmarks/b0_tracker/config.yml'
  - local: 'benchmarks/others/config.yml'
```

Insert an appropriate line for your newly created `benchmarks/backwards_ecal/config.yml`. Save and close.

The change that you've just made needs to be also staged. We will now learn a cool git trick. Run this:

```shell
git add -p
```

Here `-p` stands for `--patch`. This will display unstaged changes to the local files and let you review and optionally stage them. There will be only one change for you to check, so just type <kbd>y</kbd> and press <kbd>Enter</kbd>.

## Submit a GitHub Pull Request

Even though our benchmark doesn't do anything yet, let's submit it to the CI and see it run and do nothing useful. The way to do it is to submit a pull request. We first commit the staged changes to the current branch:

```shell
git commit -m "add benchmarks/ecal_backwards"
```

And push that branch from the local repository to the shared repository on GitHub (referenced to as `origin`):

```shell
git push origin pr/backwards_ecal_<mylastname>
```
*(Replace `<mylastname>` with your last name.)*

1. This should instruct you to go to `https://github.com/eic/detector_benchmarks/compare/master...pr/backwards_ecal_<mylastname>` to create a PR. Follow that link.
3. ![Add a title]({{ page.root }}/fig/github_add_a_title.png) Provide a title like "Adding benchmark for backwards ecal".
4. ![Create draft pull request]({{ page.root }}/fig/github_switch_to_draft.png) Since this work is not yet complete, open dropdown menu of the "Create pull requst" button and select "Create draft pull request"
5. ![Draft pull request]({{ page.root }}/fig/github_draft_pull_request.png) Click "Draft pull request"

Your newly created Pull Request will show up.

## Examine CI output on eicweb GitLab

You can now scroll to the bottom of the page and see what checks are running. You may need to wait a bit and/or refresh the page to see a `eicweb/detector_benchmarks (epic_craterlake)` check running.

![Check running on GitHub]({{ page.root }}/fig/github_running_check.png)

Click "Details", it will take you to eicweb GitLab instance. The pipeline will show all the existing jobs. You should be able to see your new jobs in the "simulate" and "benchmarks" columns.

![Check running on eicweb]({{ page.root }}/fig/eicweb_pipeline_view.png)

You can click on individual jobs and see output they produce during running. Our newly created jobs should produce messages in the output. Real scripts could return errors and those would appear as CI failures.

![Job output on eicweb]({{ page.root }}/fig/eicweb_job_output.png)

There is another important feature that jobs can produce artifacts. They can be any file. Take a look at [this](https://eicweb.phy.anl.gov/EIC/benchmarks/detector_benchmarks/-/jobs/2284270) "collect_results:barrel_ecal" job, click "Browse" button in the right column, then navigate to "results", there will be some plots produced by the benchmars related to performance of the barrel electromagnetic calorimeter.

## Conclusion

We've practiced contributing code that runs within eicweb Continuous Integration system.

{% include links.md %}

