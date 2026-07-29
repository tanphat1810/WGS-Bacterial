# Bacillus Whole-Genome Sequencing Analysis Workflow

## Introduction

This repository describes a complete **whole-genome sequencing (WGS)** workflow for bacterial isolates belonging to the genus *Bacillus* using Illumina paired-end sequencing data.

The workflow includes:

- raw-read quality control;
- adapter removal and quality filtering;
- post-trimming quality assessment;
- de novo genome assembly;
- assembly quality evaluation;
- ribosomal RNA prediction and 16S rRNA extraction;
- whole-genome Average Nucleotide Identity analysis;
- alternative assembly with Unicycler;
- reference-guided scaffolding with RagTag;
- bacterial genome annotation with Bakta.

This README was converted from the original `WGSBacillus.ipynb` notebook. Sample-specific and machine-specific paths have been replaced with reusable placeholder paths so that the workflow can be used on other systems and shared through GitHub.

---

## Workflow objectives

The workflow is designed to:

1. evaluate the quality of raw Illumina reads;
2. remove adapters, low-quality bases, poor-quality reads, and short reads;
3. construct a draft bacterial genome assembly;
4. assess assembly size, continuity, fragmentation, and GC content;
5. support species identification using 16S rRNA and whole-genome ANI;
6. compare assemblies produced by SPAdes and Unicycler;
7. generate reference-guided scaffolds when a sufficiently close reference genome is available;
8. produce standardized bacterial genome annotations for downstream analyses.

---

## Workflow overview

| Step | Tool | Main input | Purpose | Main output |
|---:|---|---|---|---|
| 1 | FastQC | Raw paired-end FASTQ files | Evaluate raw read quality, including base quality, GC content, adapter contamination, duplication, and overrepresented sequences | Individual HTML and ZIP QC reports |
| 2 | MultiQC | FastQC output directory | Combine individual QC reports into one interactive summary | `multiqc_report.html` and `multiqc_data/` |
| 3 | fastp | Raw R1 and R2 FASTQ files | Detect adapters, trim low-quality regions, filter poor-quality reads, and correct bases in overlapping paired reads | Clean paired-end FASTQ files, HTML report, and JSON report |
| 4 | FastQC, MultiQC, SeqKit | Clean FASTQ files | Confirm quality improvement and summarize the processed reads | Post-trimming QC reports and sequence statistics |
| 5 | SPAdes | Clean paired-end FASTQ files | Perform de novo assembly of a bacterial isolate genome | `contigs.fasta`, `scaffolds.fasta`, assembly graph, and log files |
| 6 | QUAST | Assembled contigs or scaffolds | Evaluate assembly statistics such as total length, contig count, N50, L50, GC content, gaps, and misassemblies | HTML, PDF, TSV, and text reports |
| 7 | Barrnap | Assembled genome FASTA | Detect bacterial ribosomal RNA genes, including 5S, 16S, and 23S rRNA | GFF3 annotation file |
| 8 | BEDTools | Assembly FASTA and filtered 16S GFF file | Extract predicted 16S rRNA sequences according to genomic coordinates and strand | 16S rRNA FASTA file |
| 9 | FastANI | Query assembly and reference genome | Estimate whole-genome average nucleotide identity for species-level comparison | Tab-delimited ANI result file |
| 10 | Unicycler | Clean paired-end FASTQ files | Generate an alternative bacterial assembly for comparison with SPAdes | Assembly FASTA, graph, and log files |
| 11 | RagTag | Reference genome and de novo contigs | Order and orient contigs according to a closely related reference genome | Reference-guided scaffold FASTA and AGP files |
| 12 | QUAST and SeqKit | RagTag scaffold FASTA | Re-evaluate scaffold quality, sequence length, GC content, and inserted gap characters | Scaffold QC reports and sequence statistics |
| 13 | Bakta | Selected final assembly | Perform rapid and standardized bacterial genome annotation | GFF3, GenBank, EMBL, FAA, FFN, TSV, JSON, and summary files |

---

## Workflow diagram

```text
Raw paired-end FASTQ files
            │
            ├── FastQC
            │
            └── MultiQC
            │
            ▼
          fastp
            │
            ├── Clean R1 FASTQ
            ├── Clean R2 FASTQ
            ├── HTML report
            └── JSON report
            │
            ├── FastQC
            ├── MultiQC
            └── SeqKit
            │
            ▼
          SPAdes
            │
            ├── QUAST
            ├── Barrnap ── BEDTools ── 16S rRNA FASTA
            ├── FastANI
            ├── Unicycler
            └── RagTag ── QUAST/SeqKit
                            │
                            ▼
                          Bakta
```

---

## Input data

### Required input

The workflow requires one pair of Illumina paired-end FASTQ files:

```text
<SAMPLE_ID>_R1.fastq.gz
<SAMPLE_ID>_R2.fastq.gz
```

Uncompressed FASTQ files can also be used:

```text
<SAMPLE_ID>_R1.fastq
<SAMPLE_ID>_R2.fastq
```

The R1 and R2 files must belong to the same sample and must contain synchronized paired reads.

### Optional reference genome

A closely related bacterial reference genome may be provided:

```text
<REFERENCE_GENOME>.fasta
```

The reference genome is used for:

- FastANI comparison;
- reference-guided scaffolding with RagTag;
- reference-based assembly evaluation with QUAST.

The reference should preferably be:

- a high-quality complete genome;
- from the same species or a very closely related taxonomic group;
- free from contamination;
- obtained from a curated or reliable public database.

---

## Recommended project structure

```text
WGS_Bacillus/
├── README.md
├── data/
│   ├── raw/
│   │   ├── <SAMPLE_ID>_R1.fastq.gz
│   │   └── <SAMPLE_ID>_R2.fastq.gz
│   └── reference/
│       └── <REFERENCE_GENOME>.fasta
├── envs/
│   ├── wgs_bacillus.yml
│   └── bakta.yml
├── results/
│   ├── qc/
│   │   ├── raw/
│   │   ├── raw_multiqc/
│   │   ├── clean/
│   │   └── clean_multiqc/
│   ├── trimmed/
│   │   └── <SAMPLE_ID>/
│   ├── assembly/
│   │   ├── spades/
│   │   │   └── <SAMPLE_ID>/
│   │   ├── spades_quast/
│   │   │   └── <SAMPLE_ID>/
│   │   ├── unicycler/
│   │   │   └── <SAMPLE_ID>/
│   │   └── ragtag/
│   │       └── <SAMPLE_ID>/
│   ├── rrna/
│   │   └── <SAMPLE_ID>/
│   ├── ani/
│   │   └── <SAMPLE_ID>/
│   └── annotation/
│       └── bakta/
│           └── <SAMPLE_ID>/
└── scripts/
    └── run_wgs_bacillus.sh
```

### Why outputs should be separated

The clean reads and SPAdes output should not be stored in the same directory.

SPAdes generates multiple files and subdirectories, including:

```text
contigs.fasta
scaffolds.fasta
assembly_graph.gfa
assembly_graph_with_scaffolds.gfa
spades.log
params.txt
corrected/
K21/
K33/
K55/
```

Separating outputs by tool makes it easier to:

- prevent accidental overwriting;
- identify failed steps;
- rerun individual analyses;
- compare different assemblers;
- organize multiple samples;
- exclude large result directories from Git.

---

## System requirements

### Operating system

Recommended:

- 64-bit Linux;
- Ubuntu, Debian, Rocky Linux, AlmaLinux, or another modern Linux distribution.

### Suggested computing resources

| Resource | Minimum | Recommended |
|---|---:|---:|
| CPU | 4 threads | 8–16 threads |
| RAM | 8 GB | 16–32 GB |
| Free storage | 20 GB | 50 GB or more |
| Operating system | 64-bit Linux | 64-bit Linux server |

Actual resource usage depends on:

- number of reads;
- read length;
- genome coverage;
- genome complexity;
- number of samples processed simultaneously;
- assembly parameters.

---

## Required software

| Tool | Main function |
|---|---|
| FastQC | FASTQ quality assessment |
| MultiQC | Aggregation of QC reports |
| fastp | Adapter trimming, quality filtering, and read correction |
| SeqKit | FASTA/FASTQ statistics and manipulation |
| SPAdes | De novo bacterial genome assembly |
| QUAST | Assembly quality evaluation |
| Barrnap | Ribosomal RNA prediction |
| BEDTools | Sequence extraction using genomic coordinates |
| FastANI | Whole-genome Average Nucleotide Identity calculation |
| Unicycler | Bacterial genome assembly |
| RagTag | Reference-guided scaffolding |
| Bakta | Bacterial genome annotation |

---

## Installing Conda and configuring Bioconda

Miniconda, Miniforge, Mamba, or Micromamba can be used.

Configure the required Conda channels:

```bash
conda config --add channels conda-forge
conda config --add channels bioconda
conda config --set channel_priority strict
```

Install Mamba in the base environment:

```bash
conda install -n base -c conda-forge mamba -y
```

Check the installation:

```bash
conda --version
mamba --version
```

---

## Creating the main analysis environment

Create the following file:

```text
envs/wgs_bacillus.yml
```

Use this content:

```yaml
name: wgs_bacillus

channels:
  - conda-forge
  - bioconda

dependencies:
  - fastqc
  - multiqc
  - fastp
  - seqkit
  - spades
  - quast
  - barrnap
  - bedtools
  - fastani
  - unicycler
  - ragtag
```

Create the environment:

```bash
mamba env create -f envs/wgs_bacillus.yml
```

Activate it:

```bash
conda activate wgs_bacillus
```

Check the installed tools:

```bash
fastqc --version
multiqc --version
fastp --version
seqkit version
spades.py --version
quast.py --version
barrnap --version
bedtools --version
fastANI --version
unicycler --version
ragtag.py --version
```

Some programs may print their version to standard error or may use a slightly different version command. When `--version` does not work, use:

```bash
<tool_name> --help
```

---

## Creating the Bakta environment

Create the following file:

```text
envs/bakta.yml
```

Use this content:

```yaml
name: bakta_env

channels:
  - conda-forge
  - bioconda

dependencies:
  - bakta
```

Create the environment:

```bash
mamba env create -f envs/bakta.yml
```

Activate it:

```bash
conda activate bakta_env
```

Check the installation:

```bash
bakta --version
bakta_db --help
```

---

## Downloading the Bakta database

Bakta requires a separate annotation database.

Example command for downloading the full database:

```bash
bakta_db download \
  --output /path/to/database_directory \
  --type full
```

After the download is complete, identify the exact database directory and define it as an environment variable:

```bash
export BAKTA_DB="/path/to/database_directory/db"
```

Check the directory:

```bash
ls -lh "${BAKTA_DB}"
```

The Bakta database should not normally be uploaded to GitHub because it is large.

---

## Defining common variables

Replace the placeholder values with paths and sample names from the local system.

```bash
export PROJECT_DIR="/path/to/WGS_Bacillus"
export SAMPLE="<SAMPLE_ID>"

export THREADS=16
export TRIM_THREADS=8
export ASSEMBLY_THREADS=8
export RAM_GB=16

export RAW_R1="${PROJECT_DIR}/data/raw/${SAMPLE}_R1.fastq.gz"
export RAW_R2="${PROJECT_DIR}/data/raw/${SAMPLE}_R2.fastq.gz"

export REFERENCE="${PROJECT_DIR}/data/reference/<REFERENCE_GENOME>.fasta"
export BAKTA_DB="/path/to/bakta_database"
```

### Variable descriptions

| Variable | Description |
|---|---|
| `PROJECT_DIR` | Root directory of the project |
| `SAMPLE` | Sample identifier |
| `THREADS` | Number of threads used for quality-control steps |
| `TRIM_THREADS` | Number of threads used by fastp |
| `ASSEMBLY_THREADS` | Number of threads used for assembly and related analyses |
| `RAM_GB` | Maximum memory allowed for SPAdes, in gigabytes |
| `RAW_R1` | Forward-read FASTQ file |
| `RAW_R2` | Reverse-read FASTQ file |
| `REFERENCE` | Reference genome FASTA file |
| `BAKTA_DB` | Bakta database directory |

---

## Creating output directories

```bash
mkdir -p \
  "${PROJECT_DIR}/results/qc/raw" \
  "${PROJECT_DIR}/results/qc/raw_multiqc" \
  "${PROJECT_DIR}/results/qc/clean" \
  "${PROJECT_DIR}/results/qc/clean_multiqc" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}" \
  "${PROJECT_DIR}/results/ani/${SAMPLE}" \
  "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}"
```

### Command explanation

- `mkdir` creates directories.
- `-p` creates missing parent directories and does not report an error when a directory already exists.
- `\` allows one Bash command to continue across multiple lines.

---

# 1. Checking the input data

## 1.1. Confirming that the files exist

```bash
ls -lh "${RAW_R1}" "${RAW_R2}"
```

### Purpose

This command is used to:

- verify that the paths are correct;
- inspect the file sizes;
- detect incorrect R1 or R2 names before starting the workflow.

## 1.2. Checking FASTQ statistics with SeqKit

```bash
seqkit stats "${RAW_R1}" "${RAW_R2}"
```

### Typical output fields

SeqKit usually reports:

- file format;
- sequence type;
- number of reads;
- total number of bases;
- minimum read length;
- average read length;
- maximum read length.

Paired-end files should normally contain the same number of reads. A major difference between R1 and R2 may indicate that the files are not synchronized or do not belong to the same preprocessing stage.

---

# 2. Raw-read quality control

## 2.1. Running FastQC

```bash
fastqc \
  "${RAW_R1}" \
  "${RAW_R2}" \
  --outdir "${PROJECT_DIR}/results/qc/raw" \
  --threads "${THREADS}"
```

The short options are equivalent:

```bash
fastqc \
  "${RAW_R1}" \
  "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/qc/raw" \
  -t "${THREADS}"
```

## Purpose

FastQC evaluates sequencing-read quality before trimming or filtering.

Important FastQC modules include:

- Per base sequence quality;
- Per sequence quality scores;
- Per base sequence content;
- Per sequence GC content;
- Sequence duplication levels;
- Overrepresented sequences;
- Adapter content;
- Sequence length distribution.

## Parameter explanation

| Parameter | Function |
|---|---|
| `"${RAW_R1}"` | Forward-read FASTQ file |
| `"${RAW_R2}"` | Reverse-read FASTQ file |
| `-o` or `--outdir` | Output directory |
| `-t` or `--threads` | Number of files processed in parallel |

## Main output files

Each FASTQ file usually produces:

```text
<SAMPLE_ID>_R1_fastqc.html
<SAMPLE_ID>_R1_fastqc.zip
<SAMPLE_ID>_R2_fastqc.html
<SAMPLE_ID>_R2_fastqc.zip
```

The HTML files can be opened in a web browser.

---

## 2.2. Summarizing FastQC results with MultiQC

```bash
multiqc \
  "${PROJECT_DIR}/results/qc/raw" \
  --outdir "${PROJECT_DIR}/results/qc/raw_multiqc"
```

The short option is equivalent:

```bash
multiqc \
  "${PROJECT_DIR}/results/qc/raw" \
  -o "${PROJECT_DIR}/results/qc/raw_multiqc"
```

## Purpose

MultiQC scans the FastQC output directory and combines the individual reports into one summary report.

This is useful when:

- multiple samples are analyzed;
- R1 and R2 need to be compared;
- a rapid overview of the complete dataset is required.

## Main output files

```text
multiqc_report.html
multiqc_data/
```

Open the report on a Linux desktop:

```bash
xdg-open "${PROJECT_DIR}/results/qc/raw_multiqc/multiqc_report.html"
```

On a remote server without a graphical interface, download the HTML file to a local computer.

---

# 3. Read preprocessing with fastp

## Command

```bash
fastp \
  -i "${RAW_R1}" \
  -I "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz" \
  -O "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz" \
  --detect_adapter_for_pe \
  --cut_right \
  --cut_right_window_size 4 \
  --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 \
  --unqualified_percent_limit 20 \
  --length_required 50 \
  --correction \
  --thread "${TRIM_THREADS}" \
  --html "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.html" \
  --json "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.json"
```

## Purpose

fastp performs several preprocessing operations in one step:

- adapter detection;
- adapter trimming;
- quality trimming;
- filtering of reads with too many low-quality bases;
- removal of short reads;
- correction of mismatches in overlapping paired-end reads;
- generation of HTML and JSON reports.

## Input and output parameters

| Parameter | Function |
|---|---|
| `-i` | Input R1 file |
| `-I` | Input R2 file |
| `-o` | Cleaned R1 output file |
| `-O` | Cleaned R2 output file |
| `--html` | Human-readable HTML report |
| `--json` | Machine-readable JSON report |

## Trimming and filtering parameters

### `--detect_adapter_for_pe`

Automatically detects adapters in paired-end data.

### `--cut_right`

Uses a sliding-window approach from the left side of the read toward the right side. Once the mean quality of a window falls below the threshold, the remaining bases on the right side are removed.

### `--cut_right_window_size 4`

Uses a sliding window of four nucleotides.

### `--cut_right_mean_quality 20`

Trims the read when the mean window quality is below Q20.

A Phred score of Q20 corresponds to an estimated base-call error probability of approximately 1%.

### `--qualified_quality_phred 20`

Bases with a Phred score of at least 20 are considered qualified bases.

### `--unqualified_percent_limit 20`

Removes a read when more than 20% of its bases have quality scores below the qualified quality threshold.

### `--length_required 50`

Removes a read when its length after trimming is less than 50 bp.

### `--correction`

Uses the overlap between R1 and R2 to correct some mismatched bases.

### `--thread`

Sets the number of processing threads.

## Main output files

```text
<SAMPLE_ID>_R1.clean.fastq.gz
<SAMPLE_ID>_R2.clean.fastq.gz
<SAMPLE_ID>_fastp.html
<SAMPLE_ID>_fastp.json
```

## Important metrics in the fastp report

Review:

- total reads;
- total bases;
- Q20 rate;
- Q30 rate;
- GC content;
- reads passing filters;
- low-quality reads;
- reads that became too short;
- adapter-trimmed reads;
- duplication rate;
- insert-size distribution.

---

# 4. Post-trimming quality control

## Defining the clean-read files

```bash
export CLEAN_R1="${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz"
export CLEAN_R2="${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz"
```

## 4.1. Running FastQC on clean reads

```bash
fastqc \
  "${CLEAN_R1}" \
  "${CLEAN_R2}" \
  -o "${PROJECT_DIR}/results/qc/clean" \
  -t "${THREADS}"
```

## 4.2. Running MultiQC on clean reads

```bash
multiqc \
  "${PROJECT_DIR}/results/qc/clean" \
  -o "${PROJECT_DIR}/results/qc/clean_multiqc"
```

## 4.3. Summarizing clean reads with SeqKit

```bash
seqkit stats "${CLEAN_R1}" "${CLEAN_R2}"
```

## Purpose

The raw and clean reports should be compared to confirm that:

- adapter contamination has decreased;
- read-end quality has improved;
- an acceptable proportion of reads was retained;
- read lengths remain suitable for genome assembly;
- R1 and R2 still contain the same number of reads.

## Do not rely only on PASS, WARN, and FAIL labels

FastQC uses general thresholds that are not specific to every library type or sequencing platform.

Some warnings may be expected. The plots should always be interpreted in the context of the sequencing method and biological sample.

---

# 5. De novo assembly with SPAdes

## Command

```bash
spades.py \
  --isolate \
  -1 "${CLEAN_R1}" \
  -2 "${CLEAN_R2}" \
  -o "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  -t "${ASSEMBLY_THREADS}" \
  -m "${RAM_GB}"
```

## Purpose

SPAdes assembles short reads into:

- contigs;
- scaffolds;
- an assembly graph.

The `--isolate` mode is intended for high-coverage bacterial isolate data.

## Parameter explanation

| Parameter | Function |
|---|---|
| `--isolate` | Uses settings designed for a high-coverage isolate genome |
| `-1` | Clean forward reads |
| `-2` | Clean reverse reads |
| `-o` | Output directory |
| `-t` | Number of threads |
| `-m` | Maximum memory usage in gigabytes |

## Inspecting the output

```bash
ls -lh "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}"
```

## Important SPAdes files

### `contigs.fasta`

Contains the assembled contigs.

### `scaffolds.fasta`

Contains scaffolds produced by SPAdes and may include gap regions represented by `N`.

### `assembly_graph.gfa`

Contains the assembly graph in GFA format.

### `spades.log`

Contains detailed execution messages.

### `params.txt`

Records the SPAdes parameters used in the run.

## Choosing between `contigs.fasta` and `scaffolds.fasta`

The original notebook used `contigs.fasta`.

Both files can be evaluated with QUAST before selecting the final assembly:

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/scaffolds.fasta" \
  -o "${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}/contigs_vs_scaffolds" \
  -t "${ASSEMBLY_THREADS}"
```

---

# 6. Assembly quality assessment with QUAST

## Basic command

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}" \
  -t "${ASSEMBLY_THREADS}"
```

## Purpose

QUAST evaluates:

- number of contigs;
- total assembly length;
- largest contig;
- N50;
- L50;
- GC content;
- number of ambiguous bases;
- reference-based metrics when a reference is provided.

## Important metrics

### Number of contigs

The number of contigs remaining after the minimum length threshold used by QUAST.

A lower contig count often indicates a more continuous assembly, but contig count alone is not sufficient to determine assembly quality.

### Largest contig

The length of the longest contig in the assembly.

### Total length

The sum of all contig lengths.

The observed total length should be compared with the expected genome size of the target species or closely related organisms.

### N50

Contigs are sorted from longest to shortest. N50 is the length of the contig at which the cumulative assembly length reaches at least 50% of the total assembly length.

A larger N50 generally indicates greater assembly continuity.

### L50

The minimum number of longest contigs required to cover at least 50% of the total assembly length.

A smaller L50 generally indicates greater continuity.

### GC content

The percentage of guanine and cytosine bases in the assembly.

An unusual GC value may indicate:

- contamination;
- incorrect taxonomic assignment;
- foreign contigs;
- incomplete or biased assembly.

## Viewing the report

```bash
cat "${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}/report.txt"
```

Other output files usually include:

```text
report.html
report.pdf
report.tsv
transposed_report.tsv
```

---

# 7. Ribosomal RNA prediction with Barrnap

## Command

```bash
barrnap \
  --kingdom bac \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"
```

## Purpose

Barrnap predicts ribosomal RNA genes in bacterial assemblies, including:

- 5S rRNA;
- 16S rRNA;
- 23S rRNA.

## Command explanation

### `--kingdom bac`

Selects the bacterial rRNA models.

### `>`

Redirects the standard output from Barrnap into a GFF file.

## Inspecting the result

```bash
cat "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"
```

## Counting predicted rRNA features

```bash
grep -c "5S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"

grep -c "16S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"

grep -c "23S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff"
```

The number of rRNA copies observed in a short-read draft assembly may be lower than the true biological copy number because repeated rRNA operons are difficult to resolve.

---

# 8. Filtering and extracting 16S rRNA sequences

## 8.1. Filtering 16S rRNA features

A simple method:

```bash
grep "16S_rRNA" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"
```

A more specific method using `awk`:

```bash
awk -F '\t' \
  '$3 == "rRNA" && $9 ~ /Name=16S_rRNA/' \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_rrna.gff" \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"
```

## Inspecting the filtered file

```bash
cat "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff"
```

Check whether the file is non-empty:

```bash
test -s "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff" \
  && echo "16S rRNA feature detected." \
  || echo "No 16S rRNA feature detected."
```

## 8.2. Extracting sequences with BEDTools

```bash
bedtools getfasta \
  -fi "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -bed "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.gff" \
  -s \
  -name \
  > "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"
```

## Parameter explanation

| Parameter | Function |
|---|---|
| `-fi` | Input genome FASTA file |
| `-bed` | BED, GFF, or VCF file containing genomic coordinates |
| `-s` | Extracts the sequence according to strand orientation |
| `-name` | Uses the feature name in the FASTA header |
| `>` | Writes extracted sequences to an output FASTA file |

## 8.3. Checking the extracted 16S sequence

```bash
seqkit stats \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"
```

Display the sequence:

```bash
seqkit seq \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}/${SAMPLE}_16S.fasta"
```

## Interpretation

A nearly complete bacterial 16S rRNA gene is usually approximately 1.5 kb long.

A substantially shorter sequence may indicate that:

- the gene is located near a contig boundary;
- the assembly is fragmented in the rRNA region;
- Barrnap identified a partial feature;
- the sequence is incomplete.

---

# 9. Whole-genome ANI analysis with FastANI

## Command

```bash
fastANI \
  -q "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -r "${REFERENCE}" \
  -o "${PROJECT_DIR}/results/ani/${SAMPLE}/${SAMPLE}_vs_reference.ani.txt"
```

## Purpose

FastANI estimates Average Nucleotide Identity between:

- the query genome assembly;
- the selected reference genome.

ANI provides a genome-wide similarity estimate and can support species-level identification.

## Parameter explanation

| Parameter | Function |
|---|---|
| `-q` | Query genome |
| `-r` | Reference genome |
| `-o` | Output file |

## Viewing the result

```bash
cat "${PROJECT_DIR}/results/ani/${SAMPLE}/${SAMPLE}_vs_reference.ani.txt"
```

## Output structure

FastANI usually reports:

```text
query_path
reference_path
ANI_value
matched_fragments
total_query_fragments
```

Example:

```text
query.fasta    reference.fasta    98.75    950    1000
```

## Interpretation

- ANI values of approximately 95–96% or higher often support assignment to the same bacterial species.
- A lower value does not automatically prove that the genomes belong to different species if the selected reference is not appropriate.
- The number and proportion of matching fragments should also be reviewed.
- Comparison against several representative reference genomes is preferred over comparison against only one genome.

The approximate matching-fragment proportion is:

```text
matched_fragments / total_query_fragments
```

If FastANI produces no result line, the genomes may be too divergent or the assembly may be insufficient for comparison.

---

# 10. Alternative assembly with Unicycler

## Command

```bash
unicycler \
  -1 "${CLEAN_R1}" \
  -2 "${CLEAN_R2}" \
  -o "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  -t "${ASSEMBLY_THREADS}"
```

## Purpose

Unicycler produces an alternative bacterial genome assembly that can be compared with the SPAdes result.

For Illumina-only data, Unicycler uses SPAdes internally and applies additional assembly-graph processing.

## Parameter explanation

| Parameter | Function |
|---|---|
| `-1` | Forward reads |
| `-2` | Reverse reads |
| `-o` | Output directory |
| `-t` | Number of threads |

## Important output files

```text
assembly.fasta
assembly.gfa
unicycler.log
```

## Evaluating the Unicycler assembly with QUAST

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}/assembly.fasta" \
  -o "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}/quast_comparison" \
  -t "${ASSEMBLY_THREADS}"
```

## Comparison criteria

Compare:

- total assembly length;
- number of contigs;
- N50 and L50;
- number of ambiguous bases;
- reference-based misassemblies;
- genome completeness;
- contamination;
- plasmid representation;
- read-mapping support.

The final assembly should not be selected only because it has the largest N50.

---

# 11. Reference-guided scaffolding with RagTag

## Command

```bash
ragtag.py scaffold \
  "${REFERENCE}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  -t "${ASSEMBLY_THREADS}"
```

## Purpose

RagTag uses a reference genome to:

- order contigs;
- orient contigs;
- join contigs into scaffolds;
- keep unresolved contigs as unplaced sequences when necessary.

## Positional arguments

```text
ragtag.py scaffold <REFERENCE_FASTA> <QUERY_CONTIGS>
```

## Parameter explanation

| Parameter | Function |
|---|---|
| `scaffold` | Activates reference-guided scaffolding mode |
| `"${REFERENCE}"` | Reference genome |
| `contigs.fasta` | Query assembly |
| `-o` | Output directory |
| `-t` | Number of threads |

## Important output files

```text
ragtag.scaffold.fasta
ragtag.scaffold.agp
ragtag.scaffold.stats
ragtag.scaffold.confidence.txt
```

### `ragtag.scaffold.fasta`

The final scaffolded assembly.

### `ragtag.scaffold.agp`

Describes how contigs were ordered, oriented, and connected.

## Biological caution

RagTag does not create new sequencing evidence.

The scaffold may appear more continuous, but its structure is influenced by the reference genome.

A RagTag scaffold should not be described as a complete genome when it was generated only from short-read data.

When the reference is too distant, RagTag may:

- order contigs incorrectly;
- hide real genome rearrangements;
- force the query structure to resemble the reference;
- incorrectly position plasmid or horizontally acquired regions.

---

# 12. Evaluating the RagTag scaffold

## 12.1. Running QUAST with a reference genome

```bash
quast.py \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  -r "${REFERENCE}" \
  -o "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/quast" \
  -t "${ASSEMBLY_THREADS}"
```

## Viewing the report

```bash
cat \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/quast/report.txt"
```

## Important reference-based metrics

Review:

- genome fraction;
- number of misassemblies;
- duplication ratio;
- number of mismatches;
- number of indels;
- number of ambiguous bases;
- total aligned length;
- unaligned length.

## 12.2. Scaffold statistics with SeqKit

```bash
seqkit fx2tab \
  -n \
  -l \
  -g \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
```

## Parameter explanation

| Parameter | Function |
|---|---|
| `fx2tab` | Converts FASTA or FASTQ records into tabular output |
| `-n` | Prints sequence names |
| `-l` | Prints sequence lengths |
| `-g` | Prints GC content |

## 12.3. Counting ambiguous `N` characters

Method equivalent to the original notebook:

```bash
grep -o "N" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  | wc -l
```

Case-insensitive method:

```bash
grep -o -i "N" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta" \
  | wc -l
```

SeqKit method:

```bash
seqkit fx2tab \
  -n \
  -l \
  -g \
  -B n \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
```

## Meaning of `N`

The character `N` represents an unknown nucleotide.

In scaffolded assemblies, stretches of `N` are commonly inserted as gap placeholders between ordered contigs.

---

# 13. Genome annotation with Bakta

## Selecting the final assembly

Possible inputs include:

- SPAdes `contigs.fasta`;
- SPAdes `scaffolds.fasta`;
- Unicycler `assembly.fasta`;
- RagTag `ragtag.scaffold.fasta`.

The final assembly should be selected based on multiple quality metrics rather than continuity alone.

Example using SPAdes contigs:

```bash
export FINAL_ASSEMBLY="${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta"
```

Example using a RagTag scaffold:

```bash
export FINAL_ASSEMBLY="${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}/ragtag.scaffold.fasta"
```

## Activating the environment

```bash
conda activate bakta_env
```

## Bakta command

```bash
bakta \
  --db "${BAKTA_DB}" \
  --threads "${ASSEMBLY_THREADS}" \
  --output "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}" \
  --prefix "${SAMPLE}" \
  "${FINAL_ASSEMBLY}"
```

## Purpose

Bakta predicts and annotates features such as:

- protein-coding sequences;
- tRNA genes;
- rRNA genes;
- non-coding RNAs;
- tmRNA;
- CRISPR arrays;
- signal peptides;
- pseudogenes;
- hypothetical proteins;
- proteins with supported functional assignments.

## Parameter explanation

| Parameter | Function |
|---|---|
| `--db` | Bakta database directory |
| `--threads` | Number of threads |
| `--output` | Output directory |
| `--prefix` | Output-file prefix |
| `"${FINAL_ASSEMBLY}"` | Input assembly |

## Typical Bakta output files

```text
<SAMPLE>.gff3
<SAMPLE>.gbff
<SAMPLE>.embl
<SAMPLE>.fna
<SAMPLE>.ffn
<SAMPLE>.faa
<SAMPLE>.tsv
<SAMPLE>.json
<SAMPLE>.txt
```

## Output descriptions

| File | Content |
|---|---|
| `.gff3` | Genomic coordinates and feature annotations |
| `.gbff` | GenBank flat file |
| `.embl` | EMBL-format annotation |
| `.fna` | Genome nucleotide sequence |
| `.ffn` | Nucleotide sequences of annotated features |
| `.faa` | Predicted protein sequences |
| `.tsv` | Tabular annotation |
| `.json` | Machine-readable annotation |
| `.txt` | Summary report |

---

# 14. Checking important output files

```bash
ls -lh \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}" \
  "${PROJECT_DIR}/results/rrna/${SAMPLE}" \
  "${PROJECT_DIR}/results/ani/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}" \
  "${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}" \
  "${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}"
```

Check that the SPAdes contig file exists and is non-empty:

```bash
test -s "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta" \
  && echo "SPAdes contigs: OK" \
  || echo "SPAdes contigs: MISSING OR EMPTY"
```

Check FASTA statistics:

```bash
seqkit stats \
  "${PROJECT_DIR}/results/assembly/spades/${SAMPLE}/contigs.fasta"
```

---

# 15. Complete workflow script

Create:

```text
scripts/run_wgs_bacillus.sh
```

Use the following content:

```bash
#!/usr/bin/env bash

set -euo pipefail

if [[ "$#" -lt 3 ]]; then
  echo "Usage:"
  echo "  bash run_wgs_bacillus.sh <PROJECT_DIR> <SAMPLE_ID> <REFERENCE_FASTA> [BAKTA_DB]"
  exit 1
fi

PROJECT_DIR="$(realpath "$1")"
SAMPLE="$2"
REFERENCE="$(realpath "$3")"
BAKTA_DB="${4:-}"

THREADS="${THREADS:-16}"
TRIM_THREADS="${TRIM_THREADS:-8}"
ASSEMBLY_THREADS="${ASSEMBLY_THREADS:-8}"
RAM_GB="${RAM_GB:-16}"

RAW_R1="${PROJECT_DIR}/data/raw/${SAMPLE}_R1.fastq.gz"
RAW_R2="${PROJECT_DIR}/data/raw/${SAMPLE}_R2.fastq.gz"

CLEAN_R1="${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R1.clean.fastq.gz"
CLEAN_R2="${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_R2.clean.fastq.gz"

SPADES_DIR="${PROJECT_DIR}/results/assembly/spades/${SAMPLE}"
SPADES_CONTIGS="${SPADES_DIR}/contigs.fasta"

QUAST_DIR="${PROJECT_DIR}/results/assembly/spades_quast/${SAMPLE}"
RRNA_DIR="${PROJECT_DIR}/results/rrna/${SAMPLE}"
ANI_DIR="${PROJECT_DIR}/results/ani/${SAMPLE}"
UNICYCLER_DIR="${PROJECT_DIR}/results/assembly/unicycler/${SAMPLE}"
RAGTAG_DIR="${PROJECT_DIR}/results/assembly/ragtag/${SAMPLE}"
BAKTA_OUT="${PROJECT_DIR}/results/annotation/bakta/${SAMPLE}"

for file in "${RAW_R1}" "${RAW_R2}" "${REFERENCE}"; do
  if [[ ! -s "${file}" ]]; then
    echo "ERROR: Input file is missing or empty: ${file}" >&2
    exit 1
  fi
done

mkdir -p \
  "${PROJECT_DIR}/results/qc/raw" \
  "${PROJECT_DIR}/results/qc/raw_multiqc" \
  "${PROJECT_DIR}/results/qc/clean" \
  "${PROJECT_DIR}/results/qc/clean_multiqc" \
  "${PROJECT_DIR}/results/trimmed/${SAMPLE}" \
  "${SPADES_DIR}" \
  "${QUAST_DIR}" \
  "${UNICYCLER_DIR}" \
  "${RAGTAG_DIR}" \
  "${RRNA_DIR}" \
  "${ANI_DIR}"

echo "[1/12] Raw-read quality control"

fastqc \
  "${RAW_R1}" \
  "${RAW_R2}" \
  -o "${PROJECT_DIR}/results/qc/raw" \
  -t "${THREADS}"

multiqc \
  "${PROJECT_DIR}/results/qc/raw" \
  -o "${PROJECT_DIR}/results/qc/raw_multiqc" \
  --force

echo "[2/12] Read preprocessing with fastp"

fastp \
  -i "${RAW_R1}" \
  -I "${RAW_R2}" \
  -o "${CLEAN_R1}" \
  -O "${CLEAN_R2}" \
  --detect_adapter_for_pe \
  --cut_right \
  --cut_right_window_size 4 \
  --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 \
  --unqualified_percent_limit 20 \
  --length_required 50 \
  --correction \
  --thread "${TRIM_THREADS}" \
  --html "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.html" \
  --json "${PROJECT_DIR}/results/trimmed/${SAMPLE}/${SAMPLE}_fastp.json"

echo "[3/12] Post-trimming quality control"

fastqc \
  "${CLEAN_R1}" \
  "${CLEAN_R2}" \
  -o "${PROJECT_DIR}/results/qc/clean" \
  -t "${THREADS}"

multiqc \
  "${PROJECT_DIR}/results/qc/clean" \
  -o "${PROJECT_DIR}/results/qc/clean_multiqc" \
  --force

seqkit stats "${CLEAN_R1}" "${CLEAN_R2}"

echo "[4/12] SPAdes assembly"

if [[ -d "${SPADES_DIR}" ]] && [[ -n "$(ls -A "${SPADES_DIR}" 2>/dev/null)" ]]; then
  echo "ERROR: SPAdes output directory is not empty: ${SPADES_DIR}" >&2
  echo "Remove it or choose a new output directory before rerunning." >&2
  exit 1
fi

spades.py \
  --isolate \
  -1 "${CLEAN_R1}" \
  -2 "${CLEAN_R2}" \
  -o "${SPADES_DIR}" \
  -t "${ASSEMBLY_THREADS}" \
  -m "${RAM_GB}"

if [[ ! -s "${SPADES_CONTIGS}" ]]; then
  echo "ERROR: SPAdes contigs file is missing or empty." >&2
  exit 1
fi

echo "[5/12] SPAdes assembly quality assessment"

quast.py \
  "${SPADES_CONTIGS}" \
  -o "${QUAST_DIR}" \
  -t "${ASSEMBLY_THREADS}"

echo "[6/12] Ribosomal RNA prediction"

barrnap \
  --kingdom bac \
  "${SPADES_CONTIGS}" \
  > "${RRNA_DIR}/${SAMPLE}_rrna.gff"

awk -F '\t' \
  '$3 == "rRNA" && $9 ~ /Name=16S_rRNA/' \
  "${RRNA_DIR}/${SAMPLE}_rrna.gff" \
  > "${RRNA_DIR}/${SAMPLE}_16S.gff"

if [[ -s "${RRNA_DIR}/${SAMPLE}_16S.gff" ]]; then
  bedtools getfasta \
    -fi "${SPADES_CONTIGS}" \
    -bed "${RRNA_DIR}/${SAMPLE}_16S.gff" \
    -s \
    -name \
    > "${RRNA_DIR}/${SAMPLE}_16S.fasta"

  seqkit stats "${RRNA_DIR}/${SAMPLE}_16S.fasta"
else
  echo "WARNING: No 16S rRNA feature was detected."
fi

echo "[7/12] FastANI comparison"

fastANI \
  -q "${SPADES_CONTIGS}" \
  -r "${REFERENCE}" \
  -o "${ANI_DIR}/${SAMPLE}_vs_reference.ani.txt"

echo "[8/12] Unicycler assembly"

unicycler \
  -1 "${CLEAN_R1}" \
  -2 "${CLEAN_R2}" \
  -o "${UNICYCLER_DIR}" \
  -t "${ASSEMBLY_THREADS}"

echo "[9/12] RagTag reference-guided scaffolding"

ragtag.py scaffold \
  "${REFERENCE}" \
  "${SPADES_CONTIGS}" \
  -o "${RAGTAG_DIR}" \
  -t "${ASSEMBLY_THREADS}"

echo "[10/12] RagTag scaffold quality assessment"

quast.py \
  "${RAGTAG_DIR}/ragtag.scaffold.fasta" \
  -r "${REFERENCE}" \
  -o "${RAGTAG_DIR}/quast" \
  -t "${ASSEMBLY_THREADS}"

seqkit fx2tab \
  -n \
  -l \
  -g \
  "${RAGTAG_DIR}/ragtag.scaffold.fasta" \
  > "${RAGTAG_DIR}/${SAMPLE}_scaffold_stats.tsv"

grep -o -i "N" \
  "${RAGTAG_DIR}/ragtag.scaffold.fasta" \
  | wc -l \
  > "${RAGTAG_DIR}/${SAMPLE}_number_of_N.txt"

echo "[11/12] Assembly comparison"

quast.py \
  "${SPADES_CONTIGS}" \
  "${UNICYCLER_DIR}/assembly.fasta" \
  "${RAGTAG_DIR}/ragtag.scaffold.fasta" \
  -r "${REFERENCE}" \
  -o "${PROJECT_DIR}/results/assembly/${SAMPLE}_assembly_comparison" \
  -t "${ASSEMBLY_THREADS}"

echo "[12/12] Optional Bakta annotation"

if [[ -n "${BAKTA_DB}" ]]; then
  if [[ ! -d "${BAKTA_DB}" ]]; then
    echo "ERROR: Bakta database directory does not exist: ${BAKTA_DB}" >&2
    exit 1
  fi

  mkdir -p "${BAKTA_OUT}"

  bakta \
    --db "${BAKTA_DB}" \
    --threads "${ASSEMBLY_THREADS}" \
    --output "${BAKTA_OUT}" \
    --prefix "${SAMPLE}" \
    "${SPADES_CONTIGS}"
else
  echo "Bakta annotation skipped because no database path was provided."
fi

echo "Workflow completed successfully."
```

---

## Explanation of the safety options in the script

### `set -euo pipefail`

- `-e` stops the script when a command returns an error.
- `-u` reports an error when an undefined variable is used.
- `-o pipefail` treats a pipeline as failed when any command in the pipeline fails.

### `realpath`

Converts a relative path into an absolute path.

### `test -s` or `[[ -s FILE ]]`

Checks that a file exists and has a size greater than zero.

### `"${VARIABLE}"`

Paths and variables are enclosed in quotation marks so that paths containing spaces are handled safely.

---

## Making the script executable

```bash
chmod +x scripts/run_wgs_bacillus.sh
```

---

## Running the script

Activate the main environment:

```bash
conda activate wgs_bacillus
```

Run without Bakta:

```bash
bash scripts/run_wgs_bacillus.sh \
  /path/to/WGS_Bacillus \
  <SAMPLE_ID> \
  /path/to/reference.fasta
```

Run with Bakta:

```bash
bash scripts/run_wgs_bacillus.sh \
  /path/to/WGS_Bacillus \
  <SAMPLE_ID> \
  /path/to/reference.fasta \
  /path/to/bakta_database
```

Set custom resources before running:

```bash
export THREADS=16
export TRIM_THREADS=8
export ASSEMBLY_THREADS=8
export RAM_GB=32
```

---

# 16. Interpreting the complete results

## Sequencing reads

Review:

- number of reads before and after fastp;
- proportion of reads retained;
- Q20 rate;
- Q30 rate;
- adapter content;
- duplication;
- read-length distribution.

## Genome assembly

Review:

- total assembly length;
- number of contigs;
- largest contig;
- N50;
- L50;
- GC content;
- number of ambiguous bases;
- misassemblies;
- genome fraction.

## 16S rRNA

Review:

- whether a 16S feature was detected;
- number of assembled copies;
- sequence length;
- whether the sequence appears complete or partial.

## FastANI

Review:

- ANI value;
- matched fragments;
- total query fragments;
- suitability of the selected reference.

## Genome annotation

Review:

- number of CDS features;
- number of tRNA genes;
- number of rRNA genes;
- number of pseudogenes;
- number of hypothetical proteins;
- number of proteins with functional annotations.

---

# 17. Workflow limitations

## Short-read assembly

Illumina short reads often cannot resolve:

- long repetitive regions;
- repeated rRNA operons;
- insertion sequences;
- repeated prophage regions;
- plasmids with shared homologous regions;
- large structural rearrangements.

A draft genome may therefore remain fragmented even when the sequencing reads are high quality.

## 16S rRNA from a draft assembly

The 16S rRNA sequence may:

- be fragmented;
- be incomplete;
- collapse multiple copies into one assembled copy;
- occur at the edge of a contig.

## FastANI

ANI is most informative when:

- both genomes are of sufficient quality;
- the selected reference belongs to a sufficiently close taxonomic group;
- a large enough proportion of the query genome can be compared.

## RagTag

Reference-guided scaffolding can improve apparent continuity but does not replace:

- long-read sequencing;
- optical mapping;
- Hi-C;
- validation using read mapping or synteny analysis.

## Bakta

Annotation quality depends on:

- assembly quality;
- database version;
- Bakta version;
- biological novelty of the organism;
- ability to distinguish intact genes from pseudogenes.

---

# 18. Possible workflow extensions

Depending on the research objective, the workflow can be expanded with:

- contamination screening using Kraken2 or Centrifuge;
- genome completeness assessment using BUSCO or CheckM2;
- read mapping back to the assembly using Bowtie2 or BWA;
- coverage calculation using SAMtools or Mosdepth;
- plasmid detection using PlasmidFinder or MOB-suite;
- antimicrobial resistance detection using AMRFinderPlus or Abricate;
- multilocus sequence typing;
- prophage prediction;
- FastANI comparison against multiple references;
- phylogenomic analysis;
- workflow automation using Snakemake or Nextflow.

---

# 19. Suggested `.gitignore`

Create a `.gitignore` file:

```gitignore
# Raw sequencing data
data/raw/*.fastq
data/raw/*.fq
data/raw/*.fastq.gz
data/raw/*.fq.gz

# Large reference files
data/reference/*.fasta
data/reference/*.fa
data/reference/*.fna

# Analysis results
results/

# Bakta database
bakta_db/
db/
db-light/
db-full/

# Conda environments
.conda/
env/
venv/

# Temporary files
*.tmp
*.log
.DS_Store
```

Raw sequencing reads, large databases, and complete result directories should normally not be uploaded to GitHub.

Files that are suitable for GitHub include:

- README files;
- scripts;
- Conda environment files;
- small summary tables;
- figures;
- small example datasets.

---

# 20. Reproducibility

Record software versions:

```bash
{
  echo "FastQC: $(fastqc --version 2>&1)"
  echo "MultiQC: $(multiqc --version 2>&1)"
  echo "fastp: $(fastp --version 2>&1)"
  echo "SeqKit: $(seqkit version 2>&1)"
  echo "SPAdes: $(spades.py --version 2>&1)"
  echo "QUAST: $(quast.py --version 2>&1)"
  echo "Barrnap: $(barrnap --version 2>&1)"
  echo "BEDTools: $(bedtools --version 2>&1)"
  echo "FastANI: $(fastANI --version 2>&1)"
  echo "Unicycler: $(unicycler --version 2>&1)"
  echo "RagTag: $(ragtag.py --version 2>&1)"
} > software_versions.txt
```

Export the main Conda environment:

```bash
conda env export \
  --name wgs_bacillus \
  --no-builds \
  > envs/wgs_bacillus.lock.yml
```

Export the Bakta environment:

```bash
conda env export \
  --name bakta_env \
  --no-builds \
  > envs/bakta.lock.yml
```

---

# 21. Quick start

```bash
git clone <REPOSITORY_URL>
cd WGS_Bacillus

mamba env create -f envs/wgs_bacillus.yml
conda activate wgs_bacillus

mkdir -p data/raw data/reference

cp /path/to/<SAMPLE_ID>_R1.fastq.gz data/raw/
cp /path/to/<SAMPLE_ID>_R2.fastq.gz data/raw/
cp /path/to/<REFERENCE_GENOME>.fasta data/reference/

chmod +x scripts/run_wgs_bacillus.sh

bash scripts/run_wgs_bacillus.sh \
  "$(pwd)" \
  <SAMPLE_ID> \
  "$(pwd)/data/reference/<REFERENCE_GENOME>.fasta"
```

---

## Conclusion

This workflow provides a complete and reusable starting point for bacterial WGS analysis using Illumina paired-end data.

It includes:

- raw-read quality control;
- read trimming and filtering;
- de novo assembly;
- assembly quality assessment;
- 16S rRNA extraction;
- whole-genome ANI comparison;
- alternative assembly;
- reference-guided scaffolding;
- bacterial genome annotation.

The final interpretation should be based on multiple complementary metrics. Genome quality or taxonomic identity should not be determined from a single measurement such as N50, one 16S sequence, or ANI against only one reference genome.
