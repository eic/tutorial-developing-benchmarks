# Migration: Software Carpentry (styles) → Carpentries Workbench

This branch (`sw-carpentry-workbench-format`) ports the lesson from the legacy
**Software Carpentry / `carpentries/styles`** (Jekyll) format to the current
**[Carpentries Workbench](https://carpentries.github.io/sandpaper-docs/)** (`sandpaper`/`varnish`)
format, matching the reference lessons
[`tutorial-setting-up-environment`](https://github.com/eic/tutorial-setting-up-environment) and
[`tutorial-analysis`](https://github.com/eic/tutorial-analysis).

## Why

The old `styles` toolchain (Ruby/Jekyll, `_episodes`, `_includes`, `_layouts`) is deprecated and no
longer receives updates. The Workbench (R/`sandpaper`, pandoc fenced-divs) is the supported path and
is what the newer EIC lessons already use.

## Structural changes

| Old (styles/Jekyll)                        | New (Workbench/sandpaper)                          |
| ------------------------------------------ | -------------------------------------------------- |
| `_config.yml`                              | `config.yaml` (new schema; `carpentry: incubator`) |
| `_episodes/*.md`                           | `episodes/*.md`                                    |
| `setup.md`, `reference.md` (root)          | `learners/setup.md`, `learners/reference.md`       |
| `_extras/discuss.md`                       | `learners/discuss.md`                              |
| `_extras/guide.md`                         | `instructors/instructor-notes.md`                  |
| `_extras/about.md`, `_extras/figures.md`   | removed (provided automatically by the Workbench)  |
| `files/*` (analysis script, macros, Snakefile, configs) | `episodes/files/*` (and a subset in `learners/files/*` for the Setup page) |
| `fig/*`                                    | `episodes/fig/*`                                   |
| `_includes/`, `_layouts/`, `assets/`, `code/`, `bin/`, `data/`, `_episodes_rmd/`, `Gemfile`, `aio.md`, `404.md`, `AUTHORS`, `CITATION` | removed (Workbench provides these) |
| —                                          | added `CITATION.cff`, `Makefile`, `links.md`, `profiles/learner-profiles.md`, and the `.github/workflows/` sandpaper CI |

## Syntax changes (every episode)

* Episode front matter no longer carries `questions:`/`objectives:`/`keypoints:`. These became
  fenced-div blocks in the body:
  * `::: questions` and `::: objectives` at the top,
  * `::: keypoints` at the bottom.
* Blockquote callouts `> ... {: .callout}` → `::: callout` fenced divs.
* Exercises `> ... {: .challenge}` → `::: challenge` fenced divs.
* `> ... {: .prereq}` (index) → `::: prereq`.
* Removed all Jekyll constructs: `{% include links.md %}`, `{% link _extras/... %}`,
  `{{ page.root }}`, `{{ site.* }}`. Internal `{% link %}` cross-references were rewritten as plain
  relative Markdown links. External `https://` links were kept as-is.
* Fenced-div fences use 45 colons for episode-level blocks and a nested 15-colon `solution` block
  inside each challenge, matching the reference lessons.

### Important: challenges need a solution

With `ghcr.io/carpentries/workbench-docker:latest`, a `::: challenge` that contains **no nested
`::: solution`** renders as a generic grey "Discussion" box instead of a blue "Challenge" box. The
one exercise in this lesson (episode 04, "Adding a Status Flag") was therefore given a `::: solution`
block that describes where the status-flag information is kept.

Challenge/solution counts after conversion (verified in the built `site/docs/*.html`):

| Episode                              | Challenges | Solutions |
| ------------------------------------ | ---------- | --------- |
| 01-analysis_scripts_and_snakemake    | 0          | 0         |
| 02-first_benchmark                   | 0          | 0         |
| 03-filling_out_your_benchmark        | 0          | 0         |
| 04-status_flags                      | 1          | 1         |
| 05-making_useful_figures             | 0          | 0         |

## Link and code-block corrections

* Fenced code blocks were relabelled to their actual languages (`bash`, `python`, `yaml`, `json`,
  `c++`, `snakemake`, `output`). URLs inside code blocks and inline-code spans were left as plain
  text.
* **Uninformative link text fixed.** In `02-first_benchmark.md` three `[here]` links were rewritten
  with descriptive text naming the destination: two combined into
  `[physics benchmark pipelines]` / `[detector benchmark pipelines]`, and the third into
  `[this example pipeline]`.
* **ePIC uses Mattermost, not Slack** — the Discussion page (`learners/discuss.md`) points help
  requests at the
  [software-tutorials channel on Mattermost](https://chat.epic-eic.org/main/channels/software-tutorials).
* The former `https://eic.github.io/documentation/` landing page (404) is referenced through its
  working sub-pages instead: `https://eic.github.io/documentation/getstarted` and
  `https://eic.github.io/documentation/tutorials.html` (both 200).
* The lesson's cross-links to the sibling EIC tutorials (Setting Up Your Environment, DD4hep
  geometry, npsim/Geant4 simulations, JANA2, Analysis) already point at the canonical
  `https://eic.github.io/tutorial-*` sites on the `index.md` prerequisites list; no additional
  cross-links were needed.

## Link verification

All `http(s)` URLs in the edited Markdown were extracted (excluding code blocks/spans) and
curl-checked. All resolve (200) **except**:

* `https://eicweb.phy.anl.gov/EIC/benchmarks/physics_benchmarks/-/pipelines/{102530,102686,103909,103955}`
  — individual pipeline pages on the eicweb GitLab return **404** to anonymous requests (a fake
  pipeline id 404s too), i.e. individual runs are auth-gated/historical. These are illustrative
  "example pipeline" references and cannot be repointed to a public URL. The public pipeline
  *listing* pages (`.../-/pipelines`) do return 200 and are left as-is.
* `https://dtn01.sdcc.bnl.gov:9001/...` (the S3 file browser used in episode 01) is not reachable
  from an automated curl (connection times out / bot-gated) but is expected to work in a browser;
  left unchanged.

`xrootd`/`root://` data paths were not curl-checked.

## Building locally

```bash
make preview   # build the site into site/docs/ using the Workbench Docker image
make serve     # serve at http://localhost:4321
make clean     # clear the build cache
```

Verified: `sandpaper::build_lesson` completes with **no errors** (only the benign
`! No valid citation information available.` line), all episodes render with correct
Callout/Challenge/Key-Points boxes, and the one challenge shows a `Challenge` header (not
"Discussion").

## Needs data verification

The following data references, versions, and IDs were **not changed** and should be re-checked
against the current environment before relying on them:

* **Simulation campaign `24.07.0`.** Episodes 01, 03 and the Setup page download and analyse
  `.../EPIC/RECO/24.07.0/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.*.eicrecon.tree.edm4eic.root`
  from `root://dtn-eic.jlab.org`. Confirm the campaign, path and file indices (`0000`–`0048`) are
  still current; newer campaigns may have superseded `24.07.0`.
* **EVGEN input file** used in episode 03's `setup.config`
  (`.../EVGEN/EXCLUSIVE/UCHANNEL_RHO/10x100/rho_10x100_uChannel_Q2of0to10_hiDiv.hepmc3.tree.root`)
  was not stat-checked.
* **Example pipeline IDs** (`102530`, `102686`, `103909`, `103955`) on eicweb point to specific
  historical CI runs that are no longer publicly reachable — consider replacing with current
  example pipelines.
* **Benchmark repository layout.** The `benchmarks/` contents listed in episode 02
  (`backgrounds  benchmarks.json  demp  diffractive_vm  dis  dvcs  dvmp  tcs  u_omega`) and the
  `.gitlab-ci.yml`/top-level `Snakefile` include lists may have changed in
  `eic/physics_benchmarks` and `eic/detector_benchmarks`.
* **Reconstruction commands / options.** `eicrecon -Ppodio:output_file=...`, the `juggler`
  fallback (`gaudirun.py options/reconstruction.py`), `ddsim` flags, `collect_tests.py`,
  `common_bench::Test`/`write_test`, and `strict-mode.sh` were not executed; verify they match the
  current tooling.
* **tectonic 0.15.0** download URL in episode 05 is version-pinned and will drift as new releases
  appear.
* **The `u_channel_sweger` PR branch link** in episode 05
  (`.../physics_benchmarks/blob/pr/u_channel_sweger/...BenchmarkPlotsExplained.pdf`) currently
  resolves (200) but points at a personal PR branch that could be deleted or merged away.
* **Efficiency figures** quoted in episode 04 ("roughly 95%", "below 90%", the April campaign) are
  illustrative numbers that were not re-measured.

## 2026-07-07 — Data repoint + analysis-script fixes (verified in eic-shell)

The `24.07.0` simulation campaign flagged above as "needs verification" was **purged** (the
`RECO/24.07.0/.../UCHANNEL_RHO/10x100/` path is empty). The analysis script also failed to compile
and had stale schema assumptions. Fixed and verified against the live container.

### Data references repointed to `25.10.2`

* Confirmed the live dataset with
  `xrdfs root://dtn-eic.jlab.org:1094 ls /volatile/eic/EPIC/RECO/25.10.2/epic_craterlake/EXCLUSIVE/UCHANNEL_RHO/10x100/`
  — **117 files**, indices `0000`–`0116`. Newer campaigns dropped the `.tree` filename segment:
  the files are now `rho_10x100_uChannel_Q2of0to10_hiDiv.NNNN.eicrecon.edm4eic.root`
  (was `…eicrecon.tree.edm4eic.root`).
* Applied two global substitutions — `24.07.0` → `25.10.2` and `.eicrecon.tree.edm4eic` →
  `.eicrecon.edm4eic` — across `episodes/01-analysis_scripts_and_snakemake.md`,
  `episodes/03-filling_out_your_benchmark.md`, `learners/setup.md`, and
  `episodes/files/Snakefile`. This updates the standalone `xrdcp`, all Snakefile
  input/output patterns, the download/analysis file names, and the derived
  `campaign_25.10.2_*` plot names so they stay mutually consistent.
* Episode 01: updated the S3 file-browser link (base64 path now encodes
  `EPIC/RECO/25.10.2/…`) and the quoted index range (`0000` up to `0116`).

### `episodes/files/uchannelrho.cxx` — compile + schema fixes

* **Missing includes added:** `TProfile2D.h`, `TChain.h`, `TTreeReader.h`,
  `TTreeReaderArray.h` (the script uses all four but relied on transitive includes that no longer
  resolve; it would not compile with ACLiC).
* **`MCParticles.momentum.{x,y,z}` now read as `TTreeReaderArray<double>`** (was `float`). In the
  current file these leaves are `Double_t` (edm4hep `Vector3d`); the float readers produced
  `TBranchProxy` type-mismatch read errors and empty MC arrays.
* **Association access ported to the current podio layout.** The old
  `ReconstructedChargedParticleAssociations.simID/.recID` `TTreeReaderArray<unsigned int>` readers
  now fail with `Unable to initialize … .simID`. Switched to the relation branches
  `_ReconstructedChargedParticleAssociations_{sim,rec}.index` (`TTreeReaderArray<int>`), which hold
  the MCParticles / ReconstructedChargedParticles indices respectively.
* **Fixed the track↔association matching.** The old loop indexed the association arrays with
  `itrk - failed`, assuming the association collection was aligned with the reconstructed-track
  collection (and even indexed the *charge* array with that offset). The association collection is
  separate, so this is replaced with an explicit `rec_id`→`sim_id` lookup table
  (`simForRec[rec_id[ia]] = sim_id[ia]`), then each track uses `simForRec[itrk]`.

### Verification (eic-shell, container `eic_xl-nightly.sif`, ROOT via `root -n`)

* `xrdcp`'d `…hiDiv.0000.eicrecon.edm4eic.root` (60 MB) to `/tmp` and ran the lesson's exact
  invocation `root -n -l -b -q 'uchannelrho.cxx+("/tmp/rho_0000.root","/tmp/out.root")'`.
  It **compiles** (ACLiC builds `uchannelrho_cxx.so`) and **runs to completion** over all
  **854 events**, producing a non-empty `out.root`.
* Histogram counts: `h_VM_mass_MC` 854 entries (mean 0.821 GeV — the ρ peak), `h_VM_mass_MC_etacut`
  49, `h_VM_mass_REC` 20, efficiency profiles `h_effEtaPtPi` 1708 / `h_effPhiEtaPi` 1419 entries.
* **Known residual (physics, not a bug):** `h_VM_mass_REC_etacut` and `h_VM_mass_REC_justpions` are
  empty in a *single* file. Across the whole file only **21** reconstructed tracks associate to the
  signal pions (MC indices 4/5) vs 100 to the proton and 37 to the electron — the very-forward
  pions (η≈4–6) mostly fall outside tracking acceptance, and only 146/854 events reconstruct any
  charged track. This is exactly why the tutorial hadd's 10–45 files before plotting; the proper
  `rec_id`→`sim_id` matching gives the same (correct) counts as the old heuristic here, confirming
  the sparsity is physical.
* **Lesson rebuild:** `make clean && make preview` (workbench-docker) completes with only the benign
  `! No valid citation information available.` warning. All five episodes render; the built HTML
  contains 25.10.2 references and no `24.07.0`/`.tree.edm4eic` strings outside this MIGRATION page.

### Accessibility

* Replaced the placeholder `![Add a title]` alt-text on three figures with descriptive text
  (`github_add_a_title.png`, `your_bench_dir_new.png`, `gitstatus_example.png`); the other seven
  figures in episodes 01–03 already carried descriptive alt-text.
