# Galaxy workflows for PyEuk

Read-level amplicon haplotype typing, end to end, with no curated haplotype catalogue.
The only inputs are paired FASTQ per specimen and the panel FASTA. Everything else is
derived from the cohort's own data.

These live here because PyEuk 0.5.0 moved the amplicon front end into the library. Every
computational step below except read alignment is a `pyeuk` subcommand.

## The four files

| file | inputs | windows |
|---|---|---|
| `haplotype_window_typing.gxwf.yml` | reads | derived from the cohort. **Start here.** |
| `haplotype_window_typing_bed.gxwf.yml` | reads | supplied as a BED |
| `haplotype_window_typing_frombam.gxwf.yml` | aligned BAMs | derived from the cohort |
| `haplotype_window_typing_frombam_bed.gxwf.yml` | aligned BAMs | supplied as a BED |

All four are gxformat2. Import with `POST /api/workflows {"from_path": ...}`, or through
Workflows → Import → Upload file.

**Do not round-trip these through `yaml.dump`.** It silently rewrites the `build_sheet`
step to `in: None, state: None`. Edit them as text.

## The graph

Paired FASTQ → bwa-mem2 against the panel → MAPQ ≥ 20 and proper pairs → windows sized
from the cohort's own fragment-length distribution → per-specimen haplotypes read off
reads that span each window end to end → one sheet → PyEuk distance and clustering.

| step | tool | provided by |
|---|---|---|
| 1 | `bwa_mem2_idx` | IUC, ToolShed |
| 2 | `bwa_mem2` | IUC, ToolShed |
| 3 | `samtools_view` | IUC, ToolShed |
| 4 | `haplotype_define_windows` | `pyeuk define-windows` |
| 5 | `haplotype_window_caller` | `pyeuk call-haplotypes` |
| 6 | `haplotype_window_sheet` | `pyeuk build-sheet` |
| 7 | `haplotype_pyeuk` | `pyeuk eukaryotyping` + `pyeuk cluster` |

The Galaxy tool wrappers are not in this repository. They are ToolShed-shaped and depend
on the Bioconda `pyeuk` package, which is not published yet — PR bioconda/bioconda-recipes
#68487 is pending. Until it lands, the wrappers resolve against a container built from a
pinned commit.

## Three properties that are deliberate

`define_windows` is a **reduction over the whole BAM collection**, not a per-specimen
step. Windows must be identical for every specimen or the sheet's columns do not
correspond across rows. The collection enters whole and the resulting BED fans back out.

A haplotype is read off a read that spans the window **end to end**. It is an observation
on one molecule, not an inference across molecules. Per-site variant calling records
mutations independently of the molecule carrying them, so it cannot distinguish one
strain carrying N mutations from a mixture in which a second strain contributes them.
Measured on a *Cryptosporidium* titration with known proportions, this graph detects and
correctly identifies every component of every mixture, including a three-way mix.

`min_freq` is a **haplotype** frequency, not a per-site allele frequency. Per-site
frequency is a sum over every haplotype carrying that site, so thresholding it can accept
some sites of a minor haplotype and reject others, assembling a genotype no molecule
carries. Thresholding haplotype frequency cannot do that.

## Parameters exposed to the user

| input | default | what it controls |
|---|---|---|
| `reads` | — | `list:paired`. **The element identifier is the specimen id**, so it must match `[A-Za-z0-9_.-]+` |
| `panel` | — | Amplicon panel FASTA. Used as the bwa reference *and* as the sequence haplotype names are described against |
| `min_span` | 30 | Minimum total fully-spanning reads before a window is called at all |
| `min_freq` | 0.05 | Minimum haplotype frequency |
| `min_reads` | 10 | Minimum supporting reads per haplotype |
| `min_completeness` | 0.1 | Minimum fraction of loci called for a specimen to enter the distance matrix |
| `cut` | `count` | `count` or `distance` |
| `linkage_threshold` | unset | Dissimilarity for `cut=distance`. Unset calibrates from the data |
| `project_psd` | `false` | Project the distance matrix onto the PSD cone |

Pinned in the tool state rather than exposed: `define_windows` samples 20 BAMs, requires
`min_spanning 0.30`, searches from `window_min 40` in `width_step 10`;
`call_haplotypes` denoises at 1 edit with ratio 8.0; `build_sheet` runs with
`min_maf 0`; `pyeuk` searches `k` in 2..50 and reports excluded specimens.

`build_sheet`'s `min_maf` is **0, i.e. off**, and should stay that way. It was 0.05. Under
the heterozygosity weighting that became the default in 0.4.0 the filter is redundant, and
removing it took a 153-specimen *Cyclospora* cohort from ARI -0.0057 to 0.9734.

## What does not transfer to a new cohort

**Sweep `min_span`.** 30 is a *Cyclospora* result, not a constant. On that cohort 24-42 is
a flat plateau at ARI 0.9737 with one specimen misassigned; between 42 and 44 it falls to
0.7536 with ten. A sweep needs no re-mapping — one permissive caller pass serves the whole
grid. The gate is harder on **mixed** specimens: k co-infecting genotypes split the
spanning reads k ways, so a mixture needs k times the depth of a clonal sample.

**Check window length against your amplicon.** It trades linkage against sequencing error.
On a *Cryptosporidium* titration: at 250 bp a pure control matched its own string in 62%
of reads and both 75:25 mixtures collapsed to a single haplotype; at 100 bp the control
reached 79% and every mixture resolved.

**Lower `min_freq` for low-frequency components.** 0.05 came from a titration whose minor
components were 25-75%, so nothing below the gate was exercised. On a cohort with 1%
components, 0.05 makes them undetectable by construction. Set 0.005 or lower and let
`min_reads` suppress noise.

**Choose the cut mode from the expected structure, not the score.** `count` splits into k
groups and suits a closed investigation. `distance` cuts at a fixed dissimilarity and
returns unrelated specimens as singletons, which suits surveillance. A cluster count
cannot represent a structure that is mostly singletons: on a CDC cohort whose published
truth is 93 groups with 79 singletons, the count rule rejects every k that could reproduce
it and returns 1.

**Raise `min_completeness` if there is a low-completeness tail.** Those specimens produce
distances pinned at the no-shared-data ceiling. On one cohort, five specimens at 0.46-0.48
completeness against a median of 0.945 generated six pairs at distance 1.0 where the real
maximum was 0.2888.

Keep `project_psd` false with `cut=count`; set it true together with `cut=distance`, where
the regularised geometry helps.

## Outputs

| output | what it is |
|---|---|
| `filtered_bam` | Per-specimen BAM, MAPQ ≥ 20 and proper pairs, coordinate sorted |
| `windows` | The windows this cohort was typed on. **Part of the result** — haplotype names only mean anything relative to these intervals |
| `calls` | Per-specimen window haplotype calls, with read counts and frequencies |
| `sheet` | Rows = specimens, columns = observed haplotypes, `X` = present |
| `haplotype_map` | Column name → window, interval, content-derived haplotype string |
| `calls_long` | Long-format calls with read frequency. **This is where mixtures live** |
| `distance_matrix` | wIBS distances |
| `clusters` | Cluster assignment; completeness failures appear as cluster `-1` |

An empty locus block in the sheet means **not called**. "Amplified and
reference-identical" is the ordinary haplotype `=`, so the two are distinguishable.

## Provenance

These files were developed and validated outside this repository, against four cohorts
(*Cyclospora*, *P. vivax* AmpliSeq, CDC AmpliSeq, *Cryptosporidium*). The reasoning behind
every default, the measurements that set them, and the options that were tried and
rejected are recorded in `brc-tools: tools/cyclospora/DEVELOPMENT-TIMELINE.md`.

A superseded Galaxy-native export exists in that history and **must not be imported**: it
carries `spanning_target: 0.7`, forcing a deprecated read-length percentile rule, and
`min_maf: 0.05`.
