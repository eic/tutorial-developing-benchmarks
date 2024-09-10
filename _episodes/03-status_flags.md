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
