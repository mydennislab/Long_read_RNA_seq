
# Long-read RNA-seq (Kinnex) Pipeline

### 1. Download the `yml` file and create the conda environment

```         
wget https://raw.githubusercontent.com/mydennislab/Long_read_RNA_seq/refs/heads/main/Long_read_RNA_seq_env.yml
conda env create -f Long_read_RNA_seq_env.yml
conda activate Long_read_RNA_seq_env
```

**Isoseq pipeline documented here: <https://isoseq.how/clustering/cli-workflow.html>**

### 2. Download MAS adapter fasta.

Download MAS adapter fasta. These will either be the MAS16 set or the MAS8 set

**MAS16 barcodes:**

```         
wget https://raw.githubusercontent.com/mydennislab/Long_read_RNA_seq/refs/heads/main/mas16_primers.fasta
```

**MAS8 barcodes:**

```         
wget https://raw.githubusercontent.com/mydennislab/Long_read_RNA_seq/refs/heads/main/mas8_primers.fasta
```

### 3. Run skera split to generate segmented reads

Use Skera to split Kinnex PacBio HiFi reads at adapter positions generating segmented reads

```         
skera split -j <threads> <hifi_reads.bam> <adapter_primers.fasta> segmented_MASadapters.bam
```

### 4. Download barcoded Iso-Seq primers

barcoded Iso-Seq primers are here:

```         
wget https://raw.githubusercontent.com/mydennislab/Long_read_RNA_seq/refs/heads/main/barcoded_IsoSeq_primers.fa
```

### 5. Primer removal and demultiplexing

Use `lima` to create Full length (FL) reads.

```         
lima -j 60 --isoseq segmented_MASadapters.bam barcoded_IsoSeq_primers.fa fl_reads.bam
```

This will create a variety of files depending on the barcodes detected. Use `cat fl_reads.lima.counts` to see counts and identify which adapter was used. There will be a file with significantly more reads than the other barcodes Use this file for `isoseq refine`

### 6. Refine

Refinement involves:

-   Trimming of poly(A) tails

-   Rapid concatemer identification and removal

This results in creation of Full length non-concatemer (FLNC) reads

```         
isoseq refine -j 60 --require-polya fl_reads.barcoded_IsoSeq_adapter_ID.bam barcoded_IsoSeq_primers.fa flnc_require-polya.bam
```

### 7. Oarfish: transcript quantification from long-read RNA-seq data


Use `Oarfish` to get transcript counts

**Oarfish docs: <https://github.com/COMBINE-lab/oarfish>**

Check oarfish flags:

```         
oarfish -h
```

Run Oarfish. Oarfish will take the FLNC reads as input as well as a transcriptome

```         
oarfish --verbose --output <output_folder/output_header> -j <threads> --reads flnc_require-polya.fastq --reference <transcriptome_ref.fa> --index-out <folder_with_transcriptome_ref.fa> --seq-tech <ont-cdna, ont-drna, pac-bio, pac-bio-hifi>
```

Oarfish output should be a folder with these files:

-   `Sample.ambig_info.tsv`

-   `Sample.meta_info.json`

-   `Sample.quant`


`Sample.quant` will contain transcript name, transcript length, and normalized counts.

### 8. Summarise transcript counts to get gene level counts:

```         
Gene_sum_oarfish_trancript_counts = fread("Sample.quant") %>%
    group_by(tname) %>%
    summarise(gene_sum = sum(num_reads)) 
head(Gene_sum_oarfish_trancript_counts)
```
