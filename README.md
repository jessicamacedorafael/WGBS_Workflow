# WGBS_Workflow

Bioinformatics workflow used to process public whole-genome bisulfite sequencing (WGBS) data from bovine samples, from raw read download to region-level DNA methylation tables. This repository documents the commands and parameters used in the master's thesis and in the associated article (see [How to cite](#-how-to-cite)).

> **Note:** this repository documents the workflow for transparency and reproducibility. It is not an automated pipeline. Paths, file names and computational resources must be adapted to your environment. Long-running commands were executed in the background with `nohup <command> &` and monitored with `tail -f nohup.out`.

---

## 📌 Requirements

- **Operating system:** Linux (Ubuntu 24.04.3 LTS)
- **Reference genome:** *Bos taurus* ARS-UCD1.2, obtained from NCBI (RefSeq GCF_002263795.1; UCSC bosTau9), restricted to the 29 autosomes (chrX, the mitochondrial genome and unplaced scaffolds were removed)

| Tool | Version(s) used | Purpose |
|------|-----------------|---------|
| [sradownloader](https://github.com/s-andrews/sradownloader) | 3.11 | Download of raw reads (ENA/SRA) |
| [SRA Toolkit](https://github.com/ncbi/sra-tools) | 3.4.1 | Used by sradownloader for SRA downloads |
| [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) | 0.11.9 | Read quality control |
| [Trim Galore](https://github.com/FelixKrueger/TrimGalore) | 0.6.6; 0.6.10 | Adapter and quality trimming |
| [Cutadapt](https://github.com/marcelm/cutadapt) | 1.18; 2.8; 4.9 | Used internally by Trim Galore |
| [Bismark](https://github.com/FelixKrueger/Bismark) | 0.24.1 | Genome preparation, bisulfite alignment and deduplication |
| [Bowtie 2](https://github.com/BenLangmead/bowtie2) | 2.5.5 | Aligner used by Bismark |
| [SAMtools](https://github.com/samtools/samtools) | 1.10 | Merging, sorting and indexing BAM files |
| [wgbstools](https://github.com/nloyfer/wgbs_tools) | 0.2.2 | PAT/beta files and region-level methylation |
| [UCSC LiftOver](https://genome.ucsc.edu/cgi-bin/hgLiftOver) | web tool | Mapping human ICR coordinates to bosTau9 |

---

## 📂 Workflow overview

| Step | Description |
|------|-------------|
| 00 | Reference genome preparation |
| 01 | Download of samples |
| 02 | Quality control and trimming |
| 03 | Alignment to the reference genome |
| 04 | Removal of PCR duplicates |
| 05 | Merging of BAM files (when needed) |
| 06 | Sorting and indexing of BAM files |
| 07 | BAM to PAT/beta conversion |
| 08 | Definition of genomic regions (ICRs) |
| 09 | Region-level methylation table |

---

## 🚀 Workflow

### 00. Reference genome preparation

```bash
# Bisulfite-converted genome index for Bismark (Bowtie 2 is the default aligner).
# The folder contains the bosTau9 FASTA file restricted to the 29 autosomes.
bismark_genome_preparation --verbose /path/to/bosTau9_genome_folder/

# Reference genome for wgbstools
wgbstools init_genome bosTau9
```

### 01. Download of samples

Samples were selected in the [SRA Run Selector](https://www.ncbi.nlm.nih.gov/Traces/study/) and the list of run accessions was saved to a text file (one accession per line).

```bash
sradownloader --outdir /path/to/output/ --threads 4 samples_list.txt
```

When a downloaded sample could not be trimmed correctly, it was downloaded again directly from the SRA (skipping ENA and GEO):

```bash
sradownloader --noena --nogeo --threads 8 samples_list.txt
```

### 02. Quality control and trimming

Adapter removal and quality trimming were performed with Trim Galore, in single-end or paired-end mode according to the library layout of each dataset. Read quality was assessed with FastQC (`--fastqc`) after trimming. Because the datasets were processed over time, different parameter sets were used.

#### Single-end libraries

```bash
trim_galore -q 20 --fastqc --length 50 --illumina --phred33 --gzip --cores 4 \
  sample.fastq.gz
```

Output: `sample_trimmed.fq.gz`.

#### Paired-end libraries

**Set 1** (most datasets):

```bash
trim_galore --paired --illumina --phred33 -q 20 --length 20 \
  --clip_R1 5 --clip_R2 5 --three_prime_clip_R1 3 --three_prime_clip_R2 3 \
  --fastqc --gzip --cores 4 \
  sample_1.fastq.gz sample_2.fastq.gz
```

**Set 2:**

```bash
trim_galore --paired --illumina --phred33 -q 30 --length 50 --max_n 1 \
  --fastqc --gzip --cores 8 \
  sample_1.fastq.gz sample_2.fastq.gz
```

**Set 3:**

```bash
trim_galore --paired --illumina --phred33 -q 20 --length 50 --max_n 1 \
  --fastqc --gzip --cores 8 \
  sample_1.fastq.gz sample_2.fastq.gz
```

**Set 4:**

```bash
trim_galore --paired --illumina --phred33 -q 20 --length 20 \
  --fastqc --gzip --cores 8 \
  sample_1.fastq.gz sample_2.fastq.gz
```

Outputs: `sample_1_val_1.fq.gz` and `sample_2_val_2.fq.gz`.

### 03. Alignment to the reference genome (Bismark)

Single-end:

```bash
bismark -q --genome /path/to/bosTau9_genome_folder/ \
  --multicore 12 --bam --gzip \
  sample_trimmed.fq.gz
```

Paired-end:

```bash
bismark -q --genome /path/to/bosTau9_genome_folder/ \
  --multicore 12 --bam --gzip \
  -1 sample_1_val_1.fq.gz -2 sample_2_val_2.fq.gz
```

Outputs: `*_bismark_bt2.bam` (single-end) or `*_bismark_bt2_pe.bam` (paired-end), plus alignment reports. BAM integrity was checked with `samtools quickcheck sample.bam`.

### 04. Removal of PCR duplicates

Single-end:

```bash
deduplicate_bismark -s --bam aligned_file.bam
```

Paired-end:

```bash
deduplicate_bismark -p --bam aligned_file.bam
```

Outputs: `*.deduplicated.bam` and `*.deduplication_report.txt`.

### 05. Merging of BAM files (when needed)

Used when a biological sample was sequenced in more than one run:

```bash
samtools cat -o sample_merged.dedup.bam \
  run1*.deduplicated.bam \
  run2*.deduplicated.bam
```

### 06. Sorting and indexing of BAM files

```bash
samtools sort input.deduplicated.bam -o output_sorted.bam
samtools index output_sorted.bam
```

### 07. BAM to PAT/beta conversion (wgbstools)

```bash
for bam in *sorted.bam; do
  wgbstools bam2pat "$bam"
done
```

Outputs for each sample: `*.pat.gz`, `*.pat.gz.csi` and `*.beta`.

### 08. Definition of genomic regions (ICRs)

Human ICR coordinates were mapped to the bovine genome (bosTau9) with the UCSC LiftOver web tool. The resulting BED file (columns: chr, start, end) was annotated with CpG indices:

```bash
wgbstools convert -L ICRs.bed --out_path ICRs_wgbs.bed
```

Output columns: chr, start, end, startCpG, endCpG.

### 09. Region-level methylation table

```bash
wgbstools beta_to_table ICRs_wgbs.bed --betas *.beta > ICRs_methylation.csv
```

The output contains the average methylation of each region in each sample.

---

## 📎 Notes

- Raw data are not included in this repository. All samples are publicly available in the NCBI Sequence Read Archive (SRA); accession numbers are listed in the article.
- Links to all tools are provided in the Requirements section.

---

## 📖 How to cite

If you use this workflow, please cite:

> Arruda, J.M.R., 2026. WGBS_Workflow (v1.0.0). Zenodo. https://doi.org/10.5281/zenodo.23778117

---

## 🎓 Academic context

This workflow was developed in the context of the master's thesis:

**Title:** *Mapping and characterization of novel candidate ICRs in the bovine genome*  
**Author:** Jéssica Macedo Rafael de Arruda  
**Program:** Graduate Program in Biosciences and Biotechnology (PGBB)  
**Institution:** Darcy Ribeiro North Fluminense State University (UENF)  
**Years:** 2024–2026  
**Advisor:** Prof. Dr. Álvaro Fabrício Lopes Rios

---

## 📄 License

This repository is distributed under the MIT License. See the `LICENSE` file for details.
