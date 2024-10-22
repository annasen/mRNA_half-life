# mRNA half-life

Here we present a pipeline to identify newly transcribed transcripts.
The cells were treated 4 hours with the 4sU. This is a uracil analog, which upon incorporation into newly synthesized RNA can be chemically converted to a C (we see T->C). After subsequent library prep and sequencing these conversions can be mapped back to the genome identifying newly synthesized RNAs

A combination of bulk PAS-seq and GRAND-SLAM data analysis.
There are several crucial steps that have to be done, listed below.

### 1 Demultiplex pooled fastq data
The main difference from the usual config file will be the demultiplexing step. During the library preparation, in order to save reagents and time, the samples were pooled after RT reaction since each sample was already indexed by the CS2 poly dT primer. This step, however, requires demultiplexing on my own as bcl2fastq step takes into account only samples sorted upon the second PCR indexed primer.

![R2_like-CELseq2](https://github.com/user-attachments/assets/aaac2b9e-b857-4bfe-9c9a-67c74a6536de)

I used the demultiplex package in my bash_env, I listed first R2 file as that is the place where the barcodes are stored (see in the picture, 8nt UMI, 6nt cell barcode, polyT). https://demultiplex.readthedocs.io/en/latest/usage.html Depending on the original fastq.gz size, the command can take some time. I recommend using screen command. There might be quite a big file sample_R*_UNKNOWN.fastq.gz.
```
demultiplex demux -r -s 9 -e 14 barcodes.tsv sample_R2.fastq.gz sample_R1.fastq.gz
```

The demultiplexed R1 sequences can be copied into a new folder. In order to get rid off the "sample_R1_" at the beginning of each (for clarity and for seq2science to run the pipeline as single-end), one can use this bash loop below:
```
for file in *fastq.gz; do mv "$file" "${file/sample_R1_/}"; done
```

### 2 Mapping by seq2science
Link to seq2science documentation https://vanheeringen-lab.github.io/seq2science/index.html   
Download annotation/index for mapping. Current version is 107 (2022/08/16) (https://github.com/erhard-lab/gedi/wiki/Preparing-genomes)
```
gedi -e IndexGenome -organism homo_sapiens -version 107 -p
```

seq2science to be run as single-end on the R1 samples, the config file needs to have this adjustment when creating the BAM files (otherwise GRAND-SLAM won't be able to create a cit file):
```
aligner:
  star:
    index: '--limitGenomeGenerateRAM 37000000000 --genomeSAsparseD 1'
    align: '--outSAMattributes MD NH'
```

Another thing to adjust in the config file might be removing the duplicates:
```
remove_dups: true # keep duplicates (check dupRadar in the MultiQC) true if you want to remove dupl
```

### 3 GRAND-SLAM
Link to GRAND-SLAM documentation https://github.com/erhard-lab/gedi/wiki/GRAND-SLAM  
One needs to apply for the software at the Erhart lab first or use one I already have /vol/moldevbio/veenstra/asenovska/GRAND-SLAM/GRAND-SLAM_2.0.5f/
1. activate a conda environment with java and generate a .cit file, use full pathways
```
/path/to/GRAND-SLAM_2.0.5f/gedi -e Bam2CIT -p mapped.cit /path/to/seq2science/results/final_bam/*.samtools-coordinate.bam
```
There might be an error "No index is available for this BAM file” which means there are missing the .bai files. You can index the bam files with samtools:
```
samtools index *.bam
```

2. run GRAND-SLAM, -full flag to obtain Coverage and Conversion data as well
```
/path/to/GRAND-SLAM_2.0.5f/gedi -e Slam -full -genomic homo_sapiens.107 -prefix date_and_projectname-full/4sU -progress -plot -D -reads mapped.cit
```
