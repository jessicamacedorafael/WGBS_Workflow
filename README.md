# WGBS_Workflow

Bioinformatics workflow for methylation analysis via Whole Genome Bisulfite Sequencing (WGBS), a step-by-step guide developed during a master's thesis. This repository documents the step-by-step workflow used for sample processing, from downloading raw data to generating the final methylation percentage table.
---

## 📌 System and Tools

- Operating system: Linux
- Tools:
  - SRAToolkit (https://github.com/ncbi/sratoolkit.git) 
  - Trim Galore (https://github.com/FelixKrueger/TrimGalore.git)
  - Bismark (https://github.com/FelixKrueger/Bismark.git)
  - Samtools (https://github.com/samtools/samtools.git)
  - WGBStools (https://github.com/nloyfer/wgbs_tools.git)

---

## 📂 Pipeline structure

| Step/Stage | Description |
|-------|-----------|
| 01 | SRA sample download |
| 02 | Quality control and trimming |
| 03 | Genome alignment |
| 04 | Duplicate removal |
| 05 | BAM file concatenation |
| 06 | BAM → PAT conversion |
| 07 | Genomic region definition |
| 08 | Final methylation table generation |

---

## 🚀 Workflow

### 1. Sample download (SRA)
```bash
nohup ./sradownloader \
--outdir /DIRECTORY_PATH/ \
samples_list.txt &

```

### 2. Quality control and trimming
Single-end
```bash
nohup trim_galore \
  --q 20 --fastqc --length 50 --illumina --phred33 --gzip --cores 4 \
  sample.fastq &
```

Paired-end
```bash
nohup trim_galore \
  --paired --gzip -q 20 --phred33 --length 50 --max_n 1 --illumina --fastqc --cores 8 \
  sample_R1.fastq sample_R2.fastq &
```
### 3. Genome alignment (Bismark)

Single-end
```bash
nohup bismark \
  -q --genome /path/genome/reference \
  --multicore 12 --bam --gzip \
  sample_val.fq.gz &
```
Paired-end
```bash
nohup bismark \
  -q --genome /path/genome/reference \
  --multicore 12 --bam --gzip \
  -1 SRRsample_val_1.fq.gz -2 SRRsample_val_2.fq.gz &
```
### 4. Duplicate removal

Single-end
```bash
nohup deduplicate_bismark -s --bam aligned_file.bam &
```
Paired-end
```bash
nohup deduplicate_bismark -p --bam aligned_file.bam &
```
### 5. Concatenation of BAMs (if necessary)
```bash
nohup samtools cat -o output.bam input1.bam input2.bam ... n.bam &
```

### 6. BAM → PAT conversion (WGBStools)
```bash
nohup wgbstools bam2pat SRRsample_sorted.bam &
```
### 7. Genomic region conversion
```bash
wgbstools convert -L input.bed --out _pat output.bed
```
### 8. Final methylation table generation
```bash
nohup wgbstools beta_to_table output.bed --betas *.beta > output.csv &
```

📎 Observations

This pipeline is for documentation and reproducibility purposes, not for automation.

Paths and filenames should be adjusted according to the local environment.

Raw data is not made available in this repository.

There are direct links to all tools used.


## 🎓 Academic Context

This pipeline was developed in the context of the master's thesis:

**Title:** *Mapping and characterization of novel candidate ICRs in the bovine genome*  
**Author:** Jéssica Macedo Rafael de Arruda  
**Program:** Graduate Program in Biosciences and Biotechnology (PGBB)  
**Institution:** Darcy Ribeiro North Fluminense State University (UENF)  
**Year:** 2024-2026  
**Advisor:** Prof. Dr. Álvaro Fabrício Lopes Rios
