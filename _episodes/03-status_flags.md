---
title: "Excercise 3: Adding a Status Flag"
teaching: 10
exercises: 10
questions:
- "How can your benchmark indicate that there were detrimental changes to software or detector design?"
objectives:
- "Learn what a status flag is and how to add one to your benchmark"
keypoints:
- "Status flags are used to indicate detrimental changes to software/detector design"
- "Add a status flag to your benchmark to alert developers to changes in performance"
---

We've created a benchmark and tested it with GitLab's CI tools. Now let's explore one of the tools available to us to alert fellow developers when there has been a detrimental change in performance for your benchmark.

To add a status flag, first define a function to set the benchmark status. In this example, the following function was added to the plotting macro `plot_benchmark.C`:

```c++
///////////// Set benchmark status!
int setbenchstatus(double eff){
        // create our test definition
        common_bench::Test rho_reco_eff_test{
          {
            {"name", "rho_reconstruction_efficiency"},
            {"title", "rho Reconstruction Efficiency for rho -> pi+pi- in the B0"},
            {"description", "u-channel rho->pi+pi- reconstruction efficiency "},
            {"quantity", "efficiency"},
            {"target", "0.9"}
          }
        };
        //this need to be consistent with the target above
        double eff_target = 0.9;

        if(eff<0 || eff>1){
          rho_reco_eff_test.error(-1);
        }else if(eff > eff_target){
          rho_reco_eff_test.pass(eff);
        }else{
          rho_reco_eff_test.fail(eff);
        }

        // write out our test data
        common_bench::write_test(rho_reco_eff_test, "./benchmark_output/u_rho_eff.json");
	return 0;
}
```

In the main plotting function, the reconstruction efficiency is calculated, then compared against the target:
```c++
int minbineff = h_VM_mass_MC_etacut->FindBin(0.6);
int maxbineff = h_VM_mass_MC_etacut->FindBin(1.0);
double reconstuctionEfficiency = (1.0*h_VM_mass_REC_etacut->Integral(minbineff,maxbineff))/(1.0*h_VM_mass_MC_etacut->Integral(minbineff,maxbineff));
//set the benchmark status:
setbenchstatus(reconstuctionEfficiency);
```


In your benchmark directory, create a file titled `benchmark.json`, or copy [this one](https://github.com/eic/tutorial-developing-benchmarks/blob/gh-pages/files/benchmark.json). The file should contain a name and title for your benchmark, as well as a description:
```json
{
  "name": "YOUR BENCHMARK NAME",
  "title": "YOUR BENCHMARK TITLE",
  "description": "Benchmark for ...",
  "target": "0.9"
}
```
For the `target`, specify a threshold number that the benchmark should achieve every time it runs.
