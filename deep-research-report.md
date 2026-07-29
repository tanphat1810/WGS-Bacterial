# Bacillus Whole-Genome Sequencing Pipeline

This workflow describes the steps to process paired-end Illumina reads from a *Bacillus* isolate, assemble its genome, and annotate the final assembly. It includes quality control, adapter trimming, de novo assembly, assembly evaluation, rRNA gene identification, reference-based scaffolding (optional), and genome annotation.

## Prerequisites

- **Operating System:** Linux (tested on Ubuntu/Fedora)  
- **Hardware:** Multi-core CPU (≥8 cores recommended), ≥16 GB RAM  
- **Data:** Paired-end raw FASTQ files for each sample (e.g. `RawRead/L1_1.fq` and `RawRead/L1_2.fq`)  
- **Tools and Environment:** Install the following software (via Conda or package manager)  
  - FastQC (for read quality control)  
  - MultiQC (for aggregating QC reports)  
  - fastp (for trimming and filtering)  
  - SeqKit (for FASTQ/FASTA utilities)  
  - SPAdes (for de novo assembly)  
  - QUAST (for assembly quality assessment)  
  - Barrnap (for rRNA gene detection)  
  - BEDTools (for sequence extraction by coordinates)  
  - FastANI (for average nucleotide identity)  
  - Unicycler (for hybrid or alternative assembly)  
  - RagTag (for reference-guided scaffolding)  
  - Bakta (for prokaryotic genome annotation)  

  For example, you can create a Conda environment with necessary tools:

  ```bash
  conda create -n wgs_bacillus python=3.9
  conda activate wgs_bacillus
  conda install fastqc multiqc fastp seqkit spades quast barrnap bedtools fastani unicycler ragtag bakta
  ```

## Directory Structure and Input

Assume the working directory has the following structure (example for sample `L1`):

```
/path/to/workdir
├── RawRead/                # Raw paired-end reads directory
│   ├── L1_1.fq
│   └── L1_2.fq
└── [Pipeline outputs will be organized here]
```

You can organize outputs into subdirectories, e.g. `QC/`, `L1/`, etc. Adapt paths in the commands below as needed.

## Workflow Steps

The pipeline consists of several sequential steps. Each step includes key commands and expected outputs.

### 1. Quality Control of Raw Reads

Perform initial quality checks on the raw FASTQ files to assess read quality, GC content, adapter contamination, duplication levels, etc.

```bash
mkdir -p QC/FastqcRaw
fastqc RawRead/*.fq -o QC/FastqcRaw -t 8
```

- **Input:** Raw FASTQ files (`RawRead/*.fq`)  
- **Tool:** FastQC  
- **Output:** Individual FastQC HTML and ZIP files in `QC/FastqcRaw/`  

Then aggregate the FastQC reports with MultiQC:

```bash
multiqc QC/FastqcRaw -o QC/MultiqcRaw
```

- **Input:** FastQC output directory  
- **Tool:** MultiQC  
- **Output:** `QC/MultiqcRaw/multiqc_report.html` (summarized QC report)  

Review the MultiQC report to check if the sequencing run has any issues (low-quality reads, adapter content, etc.) before proceeding.

### 2. Adapter Trimming and Read Filtering

Trim adapters and low-quality bases, and filter out poor-quality reads using **fastp**. This step improves assembly quality by removing artifacts.

```bash
fastp \
  -i RawRead/L1_1.fq \
  -I RawRead/L1_2.fq \
  -o L1/L1_1.clean.fastq \
  -O L1/L1_2.clean.fastq \
  --detect_adapter_for_pe \
  --cut_right \
  --cut_right_window_size 4 \
  --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 \
  --unqualified_percent_limit 20 \
  --length_required 50 \
  --correction \
  --thread 8 \
  --html L1/L1_fastp.html \
  --json L1/L1_fastp.json
```

- **Input:** Raw FASTQ files (`L1_1.fq`, `L1_2.fq`)  
- **Tool:** fastp  
- **Key Options:**  
  - `--detect_adapter_for_pe`: auto-detect paired-end adapters  
  - Quality trimming from 3′ end (`--cut_right` with window and mean quality)  
  - Filter reads with >20% low-quality bases (`--unqualified_percent_limit 20`)  
  - Minimum read length 50 bp after trimming  
  - `--correction`: overlap-based error correction  
- **Output:** Cleaned FASTQ (`L1_1.clean.fastq`, `L1_2.clean.fastq`), plus an HTML/JSON report (`L1_fastp.html`, `L1_fastp.json`)  

Review the fastp HTML report to confirm adapter removal and quality statistics.

### 3. Quality Control of Cleaned Reads

Repeat quality checks on the cleaned reads to ensure preprocessing was successful.

```bash
mkdir -p QC/FastqcClean
fastqc L1/*.clean.fastq -o QC/FastqcClean -t 8
multiqc QC/FastqcClean -o QC/MultiqcClean
```

- **Input:** Cleaned FASTQ files (`L1_1.clean.fastq`, `L1_2.clean.fastq`)  
- **Tool:** FastQC, MultiQC  
- **Output:** Post-trimming QC reports (`QC/MultiqcClean/multiqc_report.html`)  

You should see improved quality profiles (e.g. higher per-base quality) after trimming.

Also, generate basic statistics of the cleaned reads:

```bash
seqkit stats L1/L1_1.clean.fastq L1/L1_2.clean.fastq
```

- **Tool:** SeqKit  
- **Output:** Tabular stats printed to console (total reads, total bases, read lengths, etc.)  

### 4. De novo Assembly (SPAdes)

Assemble the cleaned reads into contigs using SPAdes.

```bash
spades.py \
  --isolate \
  -1 L1/L1_1.clean.fastq \
  -2 L1/L1_2.clean.fastq \
  -o L1/SPAdes_output \
  -t 8 \
  -m 16
```

- **Input:** Cleaned paired-end FASTQ files  
- **Tool:** SPAdes (with `--isolate` mode for bacterial genomes)  
- **Options:** `-t 8` (threads), `-m 16` (GB RAM)  
- **Output:** Assembly directory (`L1/SPAdes_output/`) containing:  
  - `contigs.fasta`, `scaffolds.fasta` (assembled sequences)  
  - Assembly graph files and logs  

List the assembled contigs file:

```bash
ls -lh L1/SPAdes_output/contigs.fasta
```

### 5. Assembly Quality Assessment (QUAST)

Evaluate assembly quality metrics with QUAST.

```bash
quast.py \
  L1/SPAdes_output/contigs.fasta \
  -o L1/SPAdes_quast \
  -t 8
```

- **Input:** `contigs.fasta` (from SPAdes)  
- **Tool:** QUAST  
- **Output:** QUAST reports in `L1/SPAdes_quast/` including HTML, PDF, and text summaries (N50, number of contigs, total length, GC%, etc.)  

Review the QUAST report (`report.txt` or HTML) to check metrics:
- **Total length:** Roughly expected genome size for *Bacillus* (~4–5 Mbp).  
- **N50/L50:** Indicates assembly contiguity.  
- **# contigs:** Fewer contigs usually means more contiguous assembly.  

#### Choosing Next Steps Based on Assembly

- If assembly metrics are satisfactory (e.g. low number of contigs, high N50), proceed to annotation.  
- If assembly is fragmented or incomplete, consider alternative assembly or scaffolding (next sections).

### 6. Ribosomal RNA Gene Detection (Barrnap)

Identify rRNA genes in the assembly to verify completeness and for downstream analysis.

```bash
barrnap --kingdom bac L1/SPAdes_output/contigs.fasta > L1/L1_rrna.gff
```

- **Input:** `contigs.fasta`  
- **Tool:** Barrnap (rRNA predictor)  
- **Option:** `--kingdom bac` specifies bacterial rRNA models  
- **Output:** GFF3 file (`L1_rrna.gff`) listing predicted rRNA genes (5S, 16S, 23S) with coordinates.

Check if 16S rRNA was found:

```bash
grep "16S_rRNA" L1/L1_rrna.gff > L1/L1_16S.gff
cat L1/L1_16S.gff
```

If no 16S rRNA is detected, it may indicate assembly fragmentation in the rRNA region. In that case, ensure assembly completeness or try an alternative assembly method (see steps 9–10).

### 7. 16S rRNA Sequence Extraction (BEDTools)

If a 16S rRNA gene is identified, extract its nucleotide sequence:

```bash
bedtools getfasta \
  -fi L1/SPAdes_output/contigs.fasta \
  -bed L1/L1_16S.gff \
  -s \
  -name \
  > L1/L1_16S.fasta
```

- **Input:** Assembly FASTA and filtered GFF of 16S gene (`L1_16S.gff`)  
- **Tool:** BEDTools `getfasta`  
- **Options:** `-s` to respect strand orientation, `-name` to use feature name as FASTA header  
- **Output:** `L1_16S.fasta` containing the 16S rRNA sequence(s)

Inspect the FASTA (e.g. with SeqKit) to verify length (~1500 bp for full-length 16S).

### 8. Average Nucleotide Identity (FastANI)

Compute Average Nucleotide Identity (ANI) between the assembly and a reference genome to help confirm the species.

```bash
fastANI \
  -q L1/SPAdes_output/contigs.fasta \
  -r references/Bacillus_velezensis_FZB42.fasta \
  -o L1/L1_vs_FZB42.ani.txt
```

- **Input:** Query assembly and reference genome FASTA  
- **Tool:** FastANI  
- **Output:** Tab-delimited file with ANI percentage and alignment information (`.ani.txt`)

Low ANI values (<95%) suggest a distant reference or potential misclassification. Use this to select an appropriate reference for scaffolding if needed.

### 9. Alternative Assembly (Unicycler) [Optional]

If the SPAdes assembly is unsatisfactory (e.g., too fragmented), run Unicycler for an alternative assembly.

```bash
unicycler \
  -1 L1/L1_1.clean.fastq \
  -2 L1/L1_2.clean.fastq \
  -o L1/Unicycler_output \
  -t 8
```

- **Input:** Cleaned paired-end reads  
- **Tool:** Unicycler (short-read mode by default)  
- **Output:** Assembly in `L1/Unicycler_output/` (similar to SPAdes outputs)

Assess Unicycler assembly with QUAST as in Step 5. Compare SPAdes vs Unicycler (fewer contigs, higher N50). Choose the better assembly for downstream analysis.

### 10. Reference-Guided Scaffolding (RagTag) [Optional]

If the chosen assembly is still fragmented and you have a closely related complete reference genome, use RagTag to order and orient contigs.

```bash
ragtag.py scaffold \
  reference.fa \
  L1/Best_assembly_contigs.fasta \
  -o L1/Ragtag_output \
  -t 8
```

- **Input:** Reference genome FASTA and assembly contigs FASTA  
- **Tool:** RagTag (scaffolding)  
- **Output:** `L1/Ragtag_output/ragtag.scaffold.fasta` (scaffolded assembly) and AGP file.

Evaluate the scaffolded assembly:

```bash
quast.py \
  L1/Ragtag_output/ragtag.scaffold.fasta \
  -r reference.fa \
  -o L1/Ragtag_quast \
  -t 8
```

- **Tool:** QUAST on scaffolded assembly  
- **Output:** QUAST report in `L1/Ragtag_quast/`

Compare scaffolded vs original assembly metrics. If RagTag introduced large gaps (Ns), consider whether it improved contiguity overall.

### 11. Final Assembly Selection

Choose the best assembly for annotation:

- If both Unicycler and RagTag were used, decide based on continuity and correctness.  
- Ensure the final assembly is likely complete: ideally a single circular chromosome (common for *Bacillus*), or a small number of scaffolds.

Rename or copy the chosen assembly FASTA to a clear filename (e.g., `L1_final_assembly.fasta`).

### 12. Genome Annotation (Bakta)

Annotate the final assembly with Bakta to generate standardized prokaryotic genome annotations.

First, ensure the `bakta_env` (or environment with Bakta) is activated:

```bash
conda activate bakta_env
```

Then run Bakta:

```bash
bakta \
  --db <path_to_bakta_db> \
  --cpus 8 \
  --outdir L1_bakta_annotation \
  L1/L1_final_assembly.fasta
```

- **Input:** Final assembly FASTA  
- **Tool:** Bakta (bacterial genome annotation)  
- **Output:** Annotation files in `L1_bakta_annotation/`, including GFF3, GenBank, protein FASTA (.faa), nucleotide FASTA (.ffn), and summary tables.

Review the Bakta summary and GFF to ensure annotation completeness (genes, RNAs, etc.). The pipeline is then complete.

## Summary

This pipeline automates **read QC → cleaning → assembly → evaluation → annotation** for *Bacillus* genomes. Adjust parameters and reference genomes as needed for different samples or sequencing technologies. Each step’s intermediate outputs allow you to inspect data quality and assembly progress. The final result is a high-quality annotated *Bacillus* genome assembly.

