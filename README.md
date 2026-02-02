# WGBS_Workflow

Workflow bioinformático para análise de metilação por sequenciamento completo do genoma por bissulfito (WGBS), passo a passo desenvolvido na dissertação de mestrado

Este repositório documenta, passo a passo, o fluxo de trabalho utilizado para processamento das amostras, desde o download dos dados brutos até a geração da tabela final de porcentagem de metilação.

---

## 📌 Sistema e Ferramentas

- Sistema operacional: Linux
- Genoma de referência: Bos taurus (bosTau9)
- Ferramentas:
  - SRAToolkit (https://github.com/ncbi/sratoolkit.git) 
  - Trim Galore (https://github.com/FelixKrueger/TrimGalore.git)
  - Bismark (https://github.com/FelixKrueger/Bismark.git)
  - Samtools (https://github.com/samtools/samtools.git)
  - WGBStools (https://github.com/nloyfer/wgbs_tools.git)

---

## 📂 Estrutura do pipeline

| Etapa | Descrição |
|-------|-----------|
| 01 | Download das amostras do SRA |
| 02 | Controle de qualidade e trimming |
| 03 | Alinhamento ao genoma |
| 04 | Remoção de duplicatas |
| 05 | Concatenação de arquivos BAM |
| 06 | Conversão BAM → PAT |
| 07 | Definição de regiões genômicas |
| 08 | Geração da tabela final de metilação |

---

## 🚀 Fluxo de trabalho

### 1. Download das amostras (SRA)
```bash
nohup ./sradownloader \
--outdir /CAMINHO_DO_DIRETORIO/ \
samples_list.txt &



### 2. Controle de qualidade e trimming
Single-end

nohup trim_galore \
  --q 20 --fastqc --length 50 --illumina --phred33 --gzip --cores 4 \
  amostra.fastq &
Paired-end
nohup trim_galore \
  --paired --gzip -q 20 --phred33 --length 50 --max_n 1 --illumina --fastqc --cores 8 \
  amostra_R1.fastq amostra_R2.fastq &

3. Alinhamento ao genoma (Bismark)

Single-end
nohup bismark \
  -q --genome /caminho/genoma/referência \
  --multicore 12 --bam --gzip \
  AMOSTRA_val.fq.gz &
Paired-end
nohup bismark \
  -q --genome /caminho/genoma/referência \
  --multicore 12 --bam --gzip \
  -1 SRRamostra_val_1.fq.gz -2 SRRamostra_val_2.fq.gz &
4. Remoção de duplicatas

Single-end
nohup deduplicate_bismark -s --bam arquivo_alinhado.bam &
Paired-end
nohup deduplicate_bismark -p --bam arquivo_alinhado.bam &

5. Concatenação de BAMs (se necessário)
nohup samtools cat -o output.bam input1.bam input2.bam ... n.bam &


6. Conversão BAM → PAT (WGBStools)
nohup wgbstools bam2pat SRRamostra_sorted.bam &

7. Conversão de regiões genômicas
wgbstools convert -L input.bed --out _pat output.bed

8. Geração da tabela final de metilação
nohup wgbstools beta_to_table output.bed --betas *.beta > output.csv &

📎 Observações

Este pipeline tem finalidade documental e reprodutível, não automatizada.

Os caminhos e nomes de arquivos devem ser ajustados conforme o ambiente local.

Os dados brutos não são disponibilizados neste repositório.











































