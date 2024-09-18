---
title: "Exercise 5: Filling out your benchmark"
teaching: 20
exercises: 10
questions:
objectives:
- "Fill out the many steps of your benchmark"
- "Learn basics of running on eicweb GitLab CI"
keypoints:
---

In this lesson we will be beefing up our benchmark by filling out several of the pipeline stages.

## Setting up

Before filling out the stages for GitLab's CI and pipelines, we want to first create a file that contains some settings used by our benchmark.

Create a new file: [`benchmarks/your_benchmark/setup.config`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/setup.config) with the following contents
```bash
#!/bin/bash
source strict-mode.sh

USE_SIMULATION_CAMPAIGN=true

N_EVENTS=100

FILE_BASE=sim_output/rho_10x100_uChannel_Q2of0to10_hiDiv.hepmc3.tree
INPUT_FILE=root://dtn-eic.jlab.org//work/eic2/EPIC/EVGEN/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.hepmc3.tree.root
OUTPUT_FILE=${FILE_BASE}.detectorsim.root

REC_FILE_BASE=${FILE_BASE}.detectorsim.edm4eic
REC_FILE=${REC_FILE_BASE}.root
```
Here we've defined a switch `USE_SIMULATION_CAMPAIGN` which will allow us to alternate between using output from the simulation campaign, and dynamically simulating new events.
When not using the simulation campaign, the `N_EVENTS` variable defines how many events the benchmark should run.
The rest of these variables define file names to be used in the benchmark.

Also create a new file [benchmarks/your_benchmark/simulate.sh](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/simulate.sh) with the following contents:
```bash
#!/bin/bash
source strict-mode.sh

source benchmarks/your_benchmark/setup.config $*

if [ -f ${INPUT_FILE} ]; then
  echo "ERROR: Input simulation file does ${INPUT_FILE} not exist."
else
  echo "GOOD: Input simulation file ${INPUT_FILE} exists!"
fi

# Simulate
ddsim --runType batch \
      -v WARNING \
      --numberOfEvents ${N_EVENTS} \
      --part.minimalKineticEnergy 100*GeV  \
      --filter.tracker edep0 \
      --compactFile ${DETECTOR_PATH}/${DETECTOR_CONFIG}.xml \
      --inputFiles ${INPUT_FILE} \
      --outputFile  ${OUTPUT_FILE}
if [[ "$?" -ne "0" ]] ; then
  echo "ERROR running ddsim"
  exit 1
fi
```

This script uses ddsim to simulate the detector response to your benchmark events.

## Simulating Events
We now fill out the `simulate` rule in GitLab's pipelines. Currently the instructions for this rule should be contained in `benchmarks/your_benchmark/config.yml` as: 
```yaml
your_benchmark:simulate:
  extends: .phy_benchmark
  stage: simulate
  script:
    - echo "I will simulate detector response here!"
```

This step can take a long time if you simulate too many events. So let's add an upper limit on the allowed run time of 10 hours:
In a new line below `stage: simulate`, add this: `timeout: 10 hour`.

Now in the `script` section of the rule, add two new lines to source the `setup.config` file:
```yaml
    - config_file=benchmarks/u_rho/setup.config
    - source $config_file
```

Add instructions that if using the simulation campaign you can skip detector simulations. Otherwise simulate
```yaml
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign so skipping this step!"
    - else
    -     echo "Grabbing raw events from S3 and running Geant4"
    -     bash benchmarks/your_benchmark/simulate.sh
    - fi
    - echo "Finished simulating detector response"
```

Finally, add an instruction to retry the simulation if it fails:
```yaml
  retry:
    max: 2
    when:
      - runner_system_failure
```
The final `simulate` rule should look like this:
```yaml
your_benchmark:simulate:
  stage: simulate
  extends: .phy_benchmark
  needs: ["common:detector", "your_benchmark:generate"]
  timeout: 10 hour
  script:
    - echo "Simulating detector response here!"
    - config_file=benchmarks/your_benchmark/setup.config
    - source $config_file
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign so skipping this step!"
    - else
    -     echo "Grabbing raw events from S3 and running Geant4"
    -     bash benchmarks/your_benchmark/simulate.sh
    - fi
    - echo "Finished simulating detector response"
  retry:
    max: 2
    when:
      - runner_system_failure
```

Benchmarks are currently organized into two repositories:
