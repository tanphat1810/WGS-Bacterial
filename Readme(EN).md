# Bacteria Whole-Genome Sequencing Pipeline

**I. Summary**

<div align="justify">
This is a Whole Genome Sequencing (WGS) data analysis workflow for bacterial samples. The main steps include quality assessment of the input data, adapter trimming and read filtering, de novo genome assembly, assembly quality evaluation, detection of rRNA genes (including 16S rRNA) and extraction of the corresponding sequences, comparison of genome similarity using Average Nucleotide Identity (ANI) against a reference genome for species identification, and, finally, genome annotation using Bakta. The figure below provides an overview of the workflow; the table summarizes the main analytical steps; and the subsequent sections describe each tool, command, input/output, and relevant implementation notes in detail.
</div>

![Quy trình lắp ráp và chú giải bộ gen](QUY%20TRÌNH%20LẮP%20RÁP%20VÀ%20CHÚ%20GIẢI%20BỘ%20GEN%20VI%20KHUẨN.svg) 

## II. Software Environment Setup

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

## III. Directory Structure and Input

Assume the working directory has the following structure:

```
project/
├── RawRead/              # Directory containing raw reads (R1 and R2 FASTQ files)
│   ├── sample_R1.fastq.gz
│   └── sample_R2.fastq.gz
├── QC/                   # Directory containing quality control results
│   ├── FastQC_raw/       # Raw data FastQC report
│   ├── MultiQC_raw/      # MultiQC report summarizing FastQC results for the raw data
│   ├── FastQC_clean/     # FastQC report for the filtered data
│   └── MultiQC_clean/    # MultiQC summary report for the filtered data
├── sample/               # Processing results for the sample
│   ├── sample_R1.clean.fastq    # Clean R1 reads
│   ├── sample_R2.clean.fastq    # Clean R2 reads
│   ├── sample_fastp.html        # fastp HTML report
│   ├── sample_fastp.json        # fastp JSON report
│   ├── contigs.fasta            # SPAdes output contigs file
│   ├── L1_unicycler/            # Unicycler output directory
│   ├── L1_ragtag/               # RagTag output directory
│   └── ...                      # Other temporary files and log files
├── README.md             # README file explaining this workflow
└── ...                   # Other data files and reports
```

Replace project/, RawRead/, and sample/ with the actual project and sample names, respectively. The output from each step should be saved in the corresponding directory as shown above.

## IV. Summary table of tools

| No.  | Tool            | Purpose                                                    
|:----:|:----------------|:-----------------------------------------------------------------------|
| 1    | FastQC          | Assess read quality                                                    |
| 2    | MultiQC         | Aggregate quality control reports                                      |
| 3    | Fastp           | Remove low-quality bases, short reads, and adapter sequences           |
| 4    | SeqKit stats    | Generate basic read statistics (e.g., number of reads and read length) |
| 5    | SPAdes          | Perform de novo assembly of bacterial genomes                          |
| 6    | QUAST           | Evaluate assembly quality, including genome length, N50, number of contigs, GC content, and assembly errors               |
| 7    | Barrnap         | Detect rRNA genes (5S, 16S, and 23S rRNA)                              |
| 8    | grep + bedtools | Extract 16S rRNA sequences based on GFF annotations                    |
| 9    | FastANI         | Calculate genome similarity against a reference genome using Average Nucleotide Identity (ANI)                                                                         |
| 10   | MEGA12          | Construct phylogenetic trees                                           |
| 11   | Unicycler       | Improve genome assembly using short-read sequencing data               |
| 12   | RagTag          | Scaffold and order contigs against a reference genome                  |
| 13   | Bakta           | Annotate bacterial genomes, including genes, CDSs, rRNAs, tRNAs, AMR genes, virulence factors, and other genomic features                                                     |

### V. Detailed description of the analysis workflow
### 5. 1. Raw data quality assessment (FastQC and MultiQC)
- [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)

<div align="justify">
FastQC assesses read quality, including per-base sequence quality, GC content, sequence duplication levels, adapter content, and other quality metrics. MultiQC aggregates the individual FastQC reports into a single summary report.
</div>

```bash
Fastqc <path/to/input.fastq.gz> -o <path/to/output_dir> -t 8
multiqc <path_to_fastqc_results> -o <path_to_multiqc_report>
```

<div align="justify">
Replace <path/to/input.fastq.gz> with the actual path to the raw FASTQ file. -t specifies the number of CPU threads to use.

The input data for FastQC, specified as `[Path to input FASTQ file]`, consist of raw sequencing reads in either uncompressed `.fastq` format or compressed `.fastq.gz format`. For each input FASTQ file, FastQC generates separate `fastqc.zip` and `fastqc.html` report files in the specified output directory. To improve processing speed, the number of CPU threads can be increased using the `-t` parameter. Subsequently, MultiQC takes `[Path to input directory]`, which is the directory containing the FastQC output files, as its input and generates a single consolidated `report.html` file in the specified output directory. The resulting report facilitates the review of quality-control warnings, such as residual adapter sequences, low-quality bases, abnormal GC content, and other potential sequencing quality issues.
</div>

### 5.2. Read Cleaning (fastp)
 [fastp](https://github.com/opengene/fastp) 

 - **Single-End**
- 
```bash
fastp \
  -i <path/to/input.fastq> \
  -o <path/to/output.fastq> \
  --cut_right --cut_right_window_size 4 --cut_right_window_size \
  --qualified_quality_phred 20 --unqualified_quality_phred \
  --length_required 50 \
  --thread 8 \
  --html <path/to/output.html> \
  --json <path/to/output.json>
```
  
- **Pair-end**

```bash
fastp \
  -i <path/to/inputR1.fastq> \
  -I <path/to/inputR2.fastq> \
  -o <path/to/outputR1.fastq> \
  -O <path/to/outputR2.fastq> \
  --detect_adapter_for_pe \
  --cut_right --cut_right_window_size 4 --cut_right_mean_quality 20 \
  --qualified_quality_phred 20 --unqualified_percent_limit 20 \
  --length_required 50 --correction \
  --thread 8 \
  --html <path/to/output.html> \
  --json <path/to/output.json>
```

<div align="justify">
The tool can process both single-end (SE) and paired-end (PE) sequencing data. In single-end sequencing, the sequencing instrument reads nucleotide bases from only one end of each target DNA fragment, typically starting from the 5′ end and proceeding in one direction. As a result, one read is generated for each DNA fragment.

In paired-end (PE) sequencing, both ends of the same DNA fragment are sequenced, generating two corresponding reads: the forward read (Read 1, R1) and the reverse read (Read 2, R2). Paired-end sequencing generally provides more sequence information and can improve read mapping and de novo genome assembly. Therefore, this workflow primarily focuses on paired-end sequencing data.

The input consists of two FASTQ files: R1 (forward reads) and R2 (reverse reads). The main parameters are described below:

> * **-i**: Path to the input forward-read file (R1), in .fastq or .fastq.gz format.
> * **-I**: Path to the input reverse-read file (R2).
> * **-o**: Path to the output file containing filtered and trimmed R1 reads.
> * **-O**: Path to the output file containing filtered and trimmed R2 reads.
> * **--detect_adapter_for_pe**: Enables automatic adapter sequence detection for paired-end data.
> * **--cut_right**: Enables sliding-window quality trimming from the 5′ end toward the 3′ end (left to right). When a window with insufficient mean quality is detected, the read is trimmed from that position toward the 3′ end.
> * **--cut_right_window_size**: Specifies the size of the sliding window used for quality-based trimming.
> * **--cut_right_mean_quality**: Specifies the minimum mean quality score required for each sliding window.
> * **--qualified_quality_phred**: Defines the Phred quality-score threshold at or above which an individual base is considered a qualified base.
> * **--unqualified_percent_limit**: Specifies the maximum percentage of unqualified bases allowed within a read. A base is considered unqualified if its Phred quality score is below the threshold specified by
> * **--qualified_quality_phred**: If the percentage of unqualified bases in a read exceeds this limit, the entire read is discarded.
> * **--length_required**: Specifies the minimum read length after trimming. Reads shorter than this threshold are discarded.
> * **--correction**: Enables base correction for paired-end data. fastp identifies overlapping regions between R1 and R2 and, when mismatched bases are detected within the overlapping region, uses the higher-quality base to correct the corresponding lower-quality base when appropriate.
> * **--thread**: Specifies the number of CPU threads used for parallel processing, allowing faster execution.
> * **--html**: Specifies the path for the HTML quality-control report, which can be opened in a web browser to view summary statistics and graphical results.
> * **--json**: Specifies the path for the JSON report, which stores quality-control results in a machine-readable format and can be used for subsequent automated or programmatic analyses.
</div>

### 5.3. Post-cleaning data quality assessment (FastQC, MultiQC, and SeqKit)
Use FastQC/MultiQC as described in Step 1, and use SeqKit to generate read statistics.
```bash
seqkit stats <path/to/inputR1.fastq> <path/to/inputR2.fastq> -o <path/to/input.txt>
```

<div align="justify">
FastQC and MultiQC are used to assess the cleaned data to ensure that read quality has improved, with fewer sequencing errors and reduced adapter contamination. The inputs for SeqKit are the two filtered FASTQ files, R1 and R2. The stats command in SeqKit calculates basic read statistics, including the number of reads, total number of bases, and minimum and maximum read lengths, allowing unusually short reads to be identified. If any issues are detected, such as an unexpectedly low number of reads or an unusual read-length distribution, the fastp parameters should be adjusted or the data should be examined further for potential problems.
</div>

### 5.4. De novo assembly (SPAdes)

[SPAdes](https://cab.spbu.ru/software/spades/) - genome assembler

```bash
  spades.py \
  --isolate \
  -1 <path/to/inputR1.fastq> \
  -2 <path/to/inputR2.fastq> \
  -o <path/to/output_dir> \
  -t 8 -m 16
  ```
<div align="justify">
The inputs for SPAdes are the two filtered FASTQ files, R1 and R2. The **--isolate** parameter indicates that the data represent a typical bacterial isolate genome. The **-t** parameter specifies the number of CPU threads, while **-m** specifies the maximum amount of RAM in GB; both can be adjusted according to the available hardware.

SPAdes processes the input reads and assembles them into multiple contigs. The results are saved in the specified output directory, including **contigs.fasta**, which contains the assembled contigs, and usually **scaffolds.fasta**, which contains scaffolded sequences if additional contig linking is possible.
</div>

### 5.5. Assembly quality assessment (QUAST)

[QUAST](http://quast.sourceforge.net/) – assembly quality assessment tool

 ```bash
  quast.py <path/to/input.fasta> \
    -o <path/to/output_dir> -t 8
  ```

<div align="justify">
(If a reference genome is available, add -r ref_genome.fasta to calculate additional reference-based assembly metrics.)

QUAST takes the FASTA assembly file generated by SPAdes as input and calculates a range of assembly statistics, including total assembly length, number of contigs, N50, GC content, and, when a reference genome is provided, reference-based metrics such as genome fraction and misassemblies. These metrics help assess whether the assembled genome is of sufficient quality for downstream analyses. QUAST can also evaluate assemblies without a reference genome.
</div>

### 5.6. rRNA gene detection (Barrnap)

Barrnap – a tool for detecting ribosomal RNA (rRNA) genes.

```bash
  barrnap --kingdom bac <path/to/input.fasta> > <path/to/output.gff>
  ```

<div align="justify">
Barrnap uses Hidden Markov Models (HMMs) to identify the locations of rRNA genes (5S, 16S, and 23S) in bacterial genomes. The `--kingdom bac` option specifies the kingdom of the target organism so that the appropriate trained models are applied. In this case, `bac` refers to Bacteria. The command searches for bacterial 5S, 16S, and 23S rRNA sequences. Other available options include `arc` for Archaea, `euk` for Eukaryota, and `mito` for animal mitochondrial sequences.

The results are written to a GFF3 file. Each GFF entry contains information such as the rRNA type, feature name, and its start and end positions in the genome.
</div>

### 5.7. 16S rRNA sequence extraction (grep + BEDTools)

[grep](https://en.wikipedia.org/wiki/Grep) (system command) và [BEDTools](https://bedtools.readthedocs.io/en/latest/content/tools/getfasta.html).

```bash
  grep "16S_rRNA" <path/to/input.gff> > <path/to/output16S.gff>
  bedtools getfasta -fi <path/to/input.fasta> \
                    -bed <path/to/input16S.gff> -s -name \
                    > <path/to/output16S.fasta>
  ```

<div align="justify">
In the `grep "16S_rRNA"` command, `"16S_rRNA"` is the keyword to be searched. The command scans the entire input file and retains only the lines containing this exact term. The input file contains all rRNA types, such as 5S, 16S, and 23S rRNA. The new output file therefore contains only the coordinates of the 16S rRNA genes.

Next, the `bedtools getfasta` command uses as input the FASTA file containing the complete contig sequences generated by the SPAdes assembly. The `-bed sample/16S.gff` option specifies the file containing the coordinates of the regions to be extracted, obtained from the output file generated in Section 5.6. Based on these coordinates, BEDTools determines which contig contains the 16S rRNA gene and the nucleotide positions at which the gene starts and ends.

The `-s` option stands for strand-specific extraction and allows BEDTools to automatically generate the reverse-complement sequence when the 16S rRNA gene is located on the negative (`-`) DNA strand. The `-name` option uses the gene name or ID provided in the GFF file as the header of each sequence in the output FASTA file, making it easier to identify which gene each extracted sequence corresponds to instead of displaying only generic genomic coordinates.
</div>

### 5.8. 16S rRNA sequence statistics (SeqKit stats)
```bash
  seqkit stats <path/to/input16s.fasta>
```

<div align="justify">
This tool reports the number of sequences (corresponding to the number of detected 16S rRNA genes), total sequence length, average sequence length, and other basic statistics. Full-length 16S rRNA genes are typically approximately 1,500 bp in length. The number of detected 16S rRNA copies may vary depending on the bacterial species and the quality and completeness of the genome assembly.
</div>

### 5.9. Whole-genome comparison (FastANI)
[FastANI](https://github.com/ParBLiSS/FastANI) – calculates Average Nucleotide Identity (ANI).
```bash
  fastANI -q <path/to/input.fasta> \
          -r <path/to/input_ref.fasta> \
          -o <path/to/output.txt>
```
<div align="justify">
FastANI takes a contig-containing genome file and a reference genome as input to measure the Average Nucleotide Identity (ANI) between the two genomes. An ANI value above approximately 95% generally indicates that the two genomes belong to the same species.

Rather than performing a full-length whole-genome alignment, FastANI uses a fast alignment-free approximation strategy based on MinHash-derived mapping, allowing genome comparisons to be performed efficiently.

The output is typically provided in a tab-delimited format containing the following fields: `query_genome`, `reference_genome`, `ANI%`, `bidirectional_mappings`, and `total_fragments`.
</div>

### 5.10. Phylogenetic tree construction (MEGA12)
<div align="justify">
The tool used for phylogenetic analysis is the desktop version of MEGA12. The input consists of the assembled 16S rRNA FASTA sequence together with 16S rRNA FASTA sequences from related species within the same genus or family. All sequences should first be aligned and combined into a single FASTA file.

MEGA12 then uses the aligned 16S rRNA sequences to construct a phylogenetic tree. The final output is a complete phylogenetic tree showing the evolutionary relationships among the analyzed sequences.
</div>

### 5.11. Chú giải genome (Bakta)

```bash
bakta \
  --input <path/to/input.fasta> \
  --output <path/to/output_dir> \
  --prefix SampleID \
  --threads 8
```

<div align="justify">
Bakta performs genome annotation by identifying CDSs, RNA genes (rRNA and tRNA), CRISPR regions, protein functions, DBxref links, and other genomic features. The `--prefix SampleID` option specifies the prefix used for the output files. For example, the generated files will be named `SampleID.gff3`, `SampleID.gbk`, etc., instead of using the default file names. `SampleID` can be replaced with the actual sample identifier.

Bakta generates comprehensive annotation results in multiple formats, including GFF3, GenBank, nucleotide and protein FASTA files, and TSV summary tables. Bakta is specifically designed for bacterial genome annotation and uses fast computational approaches, typically completing the annotation of a bacterial genome of approximately 5 Mb within a few minutes. It is recommended to check the Bakta log file to review the annotation records and the database version used.

* The `output` directory contains the following files:

  * `SampleID.gff` (GFF3 annotation),
  * `SampleID.gbk` (GenBank), `SampleID.embl` (EMBL),
  * `SampleID.ffn` (nucleotide sequences of CDSs), `SampleID.faa` (protein sequences),
  * `SampleID.tsv` (functional annotation summary), `SampleID.json`, etc.
</div>

### 5.12. Alternative assembly (Unicycler) [optional]

<div align="justify">
Unicycler is an assembly pipeline designed primarily for bacterial genomes and supports both short-read and hybrid short-read/long-read datasets. If the SPAdes assembly generated in Section 5.4 does not meet the desired quality criteria, for example due to an excessive number of contigs, a low N50 value, or substantial fragmentation, Unicycler can be used as an alternative assembly approach using the same short-read data.
</div>

```bash
unicycler -1 <path/to/inputR1.fastq> \
          -2 <path/to/inputR2.fastq> \
          -o <path/to/output_dir> \
          -t 8
```

<div align="justify">
Unicycler takes input data similar to SPAdes in Section 5.4. For short-read data, it internally uses SPAdes and then applies additional processing steps to optimize the assembly. The `-t` parameter specifies the number of CPU threads.

(If long-read data are available, the `-l longreads.fastq` parameter can be added to perform hybrid assembly. However, this workflow assumes that only short-read data are available.)

The output directory includes the `assembly.fasta` file together with log files and other intermediate outputs. After obtaining the Unicycler assembly, its quality should be evaluated again using QUAST.
</div>

### 5.13. Reference-guided scaffolding (RagTag) [optional]

<div align="justify">
RagTag uses a reference genome to order, orient, and scaffold the assembled contigs. The output includes a FASTA file and an AGP file containing information about scaffold structure and gaps. The `-t` option specifies the number of CPU threads.

The output directory contains the scaffolded FASTA file, `ragtag.agp` (which describes the positions and sizes of gaps between contigs), and log files. This scaffolded assembly can be used as the final assembly if the result is considered acceptable. After scaffolding, the assembly quality should be evaluated again using QUAST.
</div>

```bash
ragtag.py scaffold <path/to/input_ref.fasta> \
                   <path/to/input_dir> \
                   -o <path/to/output_dir> -t 8
```

<div align="justify">
RagTag takes as input a reference genome FASTA file and the contig assembly generated by SPAdes or Unicycler. It uses the reference genome to order, orient, and scaffold the contigs. The output includes a scaffolded FASTA file and an AGP file containing gap and scaffold-structure information. The `-t` option specifies the number of CPU threads.

The output directory contains the scaffolded FASTA file, `ragtag.agp` (which describes scaffold structure and gap positions), and log files. This scaffolded assembly can be considered the final assembly if the result is acceptable. After scaffolding, the assembly quality should be evaluated again using QUAST.

* **Note:** RagTag does not completely correct assembly errors. It can improve contig ordering and orientation when a suitable reference genome is available. The resulting scaffolds should be carefully evaluated to avoid potential misassemblies, particularly when the reference genome is not sufficiently closely related to the target genome.
</div>

### Completion criteria and implementation requirements
<div align="justify">
The workflow is considered complete when Bakta finishes genome annotation for the selected final assembly, either contigs or scaffolds. The results generated in the Bakta output directory are used as the final outputs.

**Assembly quality assessment:** After each assembly step (SPAdes, Unicycler, or RagTag), the assembly should always be evaluated using QUAST. If the assembly metrics, such as an excessively high number of contigs, low N50, abnormal total assembly size, or a high number of misassemblies, are not acceptable, an alternative approach should be considered.

If the SPAdes assembly meets the required quality criteria, Unicycler and RagTag can be skipped and the workflow can proceed directly to downstream analyses.

If the SPAdes assembly is of poor quality, Unicycler can be run for comparison. This is often performed early because Unicycler may produce a better assembly from short-read data.

If the assembly remains highly fragmented and a suitable reference genome is available, RagTag can be used for reference-guided scaffolding.

**Reference genome selection (FastANI, RagTag):** If FastANI (Step 12) produces a low ANI value (<90–95%), the genome may not belong to the same species as the selected reference. In this case, another reference genome should be selected, or RagTag should not be used. RagTag should only be applied when the reference genome is sufficiently closely related to the target genome.

**16S rRNA:** If Barrnap does not detect a 16S rRNA gene because the assembly does not contain a complete rRNA region, another assembly, such as the Unicycler assembly, can be tested. The cause of assembly failure should also be investigated, as rRNA regions are often difficult to assemble because of their repetitive nature. If 16S rRNA is not detected in either assembly, this may indicate substantial genome fragmentation.

**Note:** The order and implementation of the workflow can be adjusted depending on the characteristics of the data and the objectives of the analysis. For example, Unicycler can be run in parallel with SPAdes for comparison, with only the higher-quality assembly selected for downstream analyses. Important data should always be backed up, and reports from each step should be carefully reviewed to ensure that the workflow is proceeding correctly.
</div>

---

## Summary Table (English)

| Step | Tool     | Purpose                                      | Inputs                     | Outputs                                | Mandatory/Optional      | Key parameters                |
|------|----------|----------------------------------------------|--------------------------- |----------------------------------------|-------------------------|-------------------------------|
| 1    | FastQC   | Check raw read quality                       | Raw FASTQ files            | FastQC HTML/ZIP reports                | Mandatory               | `-t threads`                  |
| 1    | MultiQC  | Aggregate FastQC results                     | Folder of FastQC results   | `multiqc_report.html`, `multiqc_data/` | Mandatory               | —                             |
| 2    | fastp    | Trim adapters, filter low-quality reads      | Raw R1.fastq, R2.fastq     | Cleaned FASTQ files + HTML/JSON report | Mandatory               | `--detect_adapter_for_pe`, quality and length filters |
| 3    | FastQC   | Check trimmed-read quality                   | Cleaned FASTQ files        | FastQC reports                         | Mandatory               | `-t threads`                  |
| 3    | MultiQC  | Aggregate post-filter FastQC results         | Folder of FastQC results   | `multiqc_report.html`                  | Mandatory               | —                             |
| 4    | SeqKit   | Compute read statistics (count, length)      | Cleaned FASTQ files        | Stats table on stdout                  | Mandatory               | —                             |
| 5    | SPAdes   | De novo genome assembly                      | Cleaned R1.fastq, R2.fastq | `contigs.fasta` (assembly)             | Mandatory               | `--isolate`, `-t threads`, `-m memory` |
| 5    | QUAST    | Evaluate assembly quality                    | `contigs.fasta` (+ ref)    | Assembly quality reports               | Mandatory               | `-t threads`, `-r ref.fasta` |
| 6    | Barrnap  | Find rRNA genes (5S, 16S, 23S)               | `contigs.fasta`            | GFF3 with rRNA coordinates             | Mandatory               | `--kingdom bac`     |
| 7    | grep + bedtools | Extract 16S rRNA sequences from GFF   | `rrna.gff`                 | `16S.fasta`                            | Optional (if 16S found) | `grep "16S_rRNA"`, `bedtools getfasta -s -name` |
| 8    | SeqKit   | Stats on 16S sequences                       | `16S.fasta`                | Stats table on stdout                  | Optional                | —                             |
| 9    | FastANI  | Compute genome-wide ANI vs reference         | Assembly FASTA, ref FASTA  | ANI results (tab file)                 | Conditional (if ref.)   | `-q assembly -r reference -o output` |
| 10   | MEGA12   | Phylogenetic tree construction               | Multiple sequence alignment (MSA) FASTA file of 16S rRNA genes      | Complete phylogenetic tree file             | Conditional (For species identification) | `Desktop GUI (Manual execution)` |
| 11   | Bakta    | Annotate final genome (genes, proteins, etc.)| Final assembly FASTA       | GFF3, GenBank, FASTA, TSV, JSON      | Mandatory (final step) | `--input assembly.fa --output dir --prefix ID` |
| 12   | Unicycler| Alternative assembly (short-read only)       | Cleaned R1.fastq, R2.fastq | `assembly.fasta` (Unicycler output) | Optional             | `-1 reads -2 reads -o out -t threads` |
| 12   | QUAST    | Evaluate Unicycler assembly                  | Unicycler `assembly.fasta` | Assembly reports                    | Optional             | Same as Step 8                |
| 13   | RagTag   | Reference-guided scaffolding of contigs      | Assembly FASTA, ref FASTA  | `ragtag.scaffold.fasta`             | Conditional (if needed) | `ragtag.py scaffold ref.fa contigs.fa` |
| 13   | QUAST + SeqKit | Evaluate scaffolded assembly             | `ragtag.scaffold.fasta`   | Reports; contig stats (SeqKit)      | Conditional           | Same as Step 8 + count `N`    |

This table summarizes each step: **Tool**, **Purpose**, **Inputs/Outputs**, **When to run** (mandatory or conditional), and **Key parameters**. (Steps like Unicycler and RagTag are optional alternatives run only if needed. FastANI requires a reference genome.) The final step is always running Bakta on the chosen assembly to complete annotation.
