---
title: "Exercise 3: Filling out your benchmark"
teaching: 20
exercises: 10
questions: How do we fill in each stage of the benchmark pipeline?
objectives:
- "Fill out the many steps of your benchmark"
- "Collect templates for the benchmark stages"
keypoints:
- "Create `setup.config` to switch between using the simulation campaign and re-simulating events"
- "Each stage of the benchmark pipeline is defined in `config.yml`"
- "`config.yml` takes normal bash scripts as input"
- "Copy resulting figures over to the `results` directory to turn them into artifacts"
---

In this lesson we will be beefing up our benchmark by filling out several of the pipeline stages.

## Setting up

Before filling out the stages for GitLab's CI and pipelines, we want to first create a file that contains some settings used by our benchmark.

Create a new file: [`benchmarks/your_benchmark/setup.config`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/setup.config) with the following contents
```bash
#!/bin/bash
source strict-mode.sh

export ENV_MODE=eicweb

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

Also create a new file [`benchmarks/your_benchmark/simulate.sh`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/simulate.sh) with the following contents:
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

Create a script named [`benchmarks/your_benchmark/reconstruct.sh`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/reconstruct.sh) to manage the reconstruction:
```bash
#!/bin/bash
source strict-mode.sh
source benchmarks/your_benchmark/setup.config $*

# Reconstruct
if [ ${RECO} == "eicrecon" ] ; then
  eicrecon ${OUTPUT_FILE} -Ppodio:output_file=${REC_FILE}
  if [[ "$?" -ne "0" ]] ; then
    echo "ERROR running eicrecon"
    exit 1
  fi
fi

if [[ ${RECO} == "juggler" ]] ; then
  gaudirun.py options/reconstruction.py || [ $? -eq 4 ]
  if [ "$?" -ne "0" ] ; then
    echo "ERROR running juggler"
    exit 1
  fi
fi

if [ -f jana.dot ] ; then cp jana.dot ${REC_FILE_BASE}.dot ; fi

#rootls -t ${REC_FILE_BASE}.tree.edm4eic.root
rootls -t ${REC_FILE}
```

Finally create a file called [`benchmarks/your_benchmark/analyze.sh`](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/analyze.sh) which will run the analysis and plotting scripts:
```bash
#!/bin/bash
source strict-mode.sh
source benchmarks/your_benchmark/setup.config $*

OUTPUT_PLOTS_DIR=sim_output/nocampaign
mkdir -p ${OUTPUT_PLOTS_DIR}
# Analyze
command time -v \
root -l -b -q "benchmarks/your_benchmark/analysis/uchannelrho.cxx+(\"${REC_FILE}\",\"${OUTPUT_PLOTS_DIR}/plots.root\")"
if [[ "$?" -ne "0" ]] ; then
  echo "ERROR analysis failed"
  exit 1
fi

if [ ! -d "${OUTPUT_PLOTS_DIR}/plots_figures" ]; then
    mkdir "${OUTPUT_PLOTS_DIR}/plots_figures"
    echo "${OUTPUT_PLOTS_DIR}/plots_figures directory created successfully."
else
    echo "${OUTPUT_PLOTS_DIR}/plots_figures directory already exists."
fi
root -l -b -q "benchmarks/your_benchmark/macros/plot_rho_physics_benchmark.C(\"${OUTPUT_PLOTS_DIR}/plots.root\")"
cat benchmark_output/*.json
```


## The "simulate" rule
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
    - config_file=benchmarks/your_benchmark/setup.config
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

## The "reconstruct" rule

Now the `reconstruct` rule should look like this:
```yaml
your_benchmark:reconstruct:
  extends: .phy_benchmark
  stage: reconstruct
  script:
    - echo "Event reconstruction here!"
```

We need the `simulate` rule to finish before starting the reconstruction so add the line `needs: ["your_benchmark:simulate"]` below `extends: .phy_benchmark`.

In a new line below that, add a timeout: `timeout: 10 hour`.

Again in the script section, source the `setup.config` file:
```yaml
    - config_file=benchmarks/your_benchmark/setup.config
    - source $config_file
```

Now add instructions to do the reconstruction if not using the simulation campaign:
```yaml
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign so skipping this step!"
    - else
    -     echo "Running eicrecon!"
    -     bash benchmarks/your_benchmark/reconstruct.sh
    - fi
    - echo "Finished reconstruction"
```

Finally add an instruction to retry if this fails:
```yaml
  retry:
    max: 2
    when:
      - runner_system_failure
```

The `reconstruct` rule should now look like this:
```yaml
your_benchmark:reconstruct:
  stage: reconstruct
  extends: .phy_benchmark
  needs: ["your_benchmark:simulate"]
  timeout: 10 hour
  script:
    - echo "Event reconstruction here!"
    - config_file=benchmarks/your_benchmark/setup.config
    - source $config_file
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign so skipping this step!"
    - else
    -     echo "Running eicrecon!"
    -     bash benchmarks/your_benchmark/reconstruct.sh
    - fi
    - echo "Finished reconstruction"
  retry:
    max: 2
    when:
      - runner_system_failure
```

## The "analyze" rule

The `analyze` rule in `config.yml` is right now just this:
```yaml
your_benchmark:analyze:
  extends: .phy_benchmark
  stage: analyze
  needs:
    - ["your_benchmark:reconstruct"]
  script:
    - echo "I will analyze events here!"
    - echo "This step requires that the reconstruct step be completed"
```

First make two directories to contain output from the benchmark analysis and source `setup.config` again:
```yaml
    - mkdir -p results/your_benchmark
    - mkdir -p benchmark_output
    - config_file=benchmarks/your_benchmark/setup.config
    - source $config_file
```

If using the simulation campaign, we can request the rho mass benchmark with snakemake. Once snakemake has finished creating the benchmark figures, we copy them over to `results/your_benchmark/` in order to make them into artifacts:
```yaml
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign!"
    -     snakemake --cores 2 ../../sim_output/campaign_24.07.0_combined_45files_eicrecon.edm4eic.plots_figures/benchmark_rho_mass.pdf
    -     cp ../../sim_output/campaign_24.07.0_combined_45files_eicrecon.edm4eic.plots_figures/*.pdf results/your_benchmark/
```

If not using the simulation campaign, we can just run the `analyze.sh` script and copy the results into `results/your_benchmark/` in order to make them into artifacts:
```yaml
    - else
    -     echo "Not using simulation campaign!"
    -     bash benchmarks/your_benchmark/analyze.sh
    -     cp sim_output/nocampaign/plots_figures/*.pdf results/your_benchmark/
    - fi
```

The `analyze` rule should now look like this:
```yaml
your_benchmark:analyze:
  extends: .phy_benchmark
  stage: analyze
  needs:
    - ["your_benchmark:reconstruct"]
  script:
    - echo "I will be analyzing events here!"
    - mkdir -p results/your_benchmark
    - mkdir -p benchmark_output
    - config_file=benchmarks/your_benchmark/setup.config
    - source $config_file
    - if [ "$USE_SIMULATION_CAMPAIGN" = true ] ; then
    -     echo "Using simulation campaign!"
    -     snakemake --cores 2 ../../sim_output/campaign_24.07.0_combined_45files_eicrecon.edm4eic.plots_figures/benchmark_rho_mass.pdf
    -     cp ../../sim_output/campaign_24.07.0_combined_45files_eicrecon.edm4eic.plots_figures/*.pdf results/your_benchmark/
    - else
    -     echo "Not using simulation campaign!"
    -     bash benchmarks/your_benchmark/analyze.sh
    -     cp sim_output/nocampaign/plots_figures/*.pdf results/your_benchmark/
    - fi
    - echo "Finished copying!" 
```
