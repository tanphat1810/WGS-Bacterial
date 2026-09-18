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

## Summary

This pipeline automates **read QC → cleaning → assembly → evaluation → annotation** for *Bacillus* genomes. Adjust parameters and reference genomes as needed for different samples or sequencing technologies. Each step’s intermediate outputs allow you to inspect data quality and assembly progress. The final result is a high-quality annotated *Bacillus* genome assembly.

