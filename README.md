# mRNA half-life

Here we present a pipeline to identify newly transcribed mRNA.
The cells were treated 4 hours with the 4sU. This is a uracil analog, which upon incorporation into newly synthesized RNA can be chemically converted to a C (we observe T->C conversion in newly transcribed mRNA). After subsequent library prep and sequencing these conversions can be mapped back to the genome identifying newly synthesized RNAs

![image](https://github.com/user-attachments/assets/a8ac81bb-2ed1-432f-8f18-06f554201543)  
_Herzog et al., 2017_

This workflow combines bulk **PAS-seq2, CEL-Seq2, and GRAND-SLAM**.
> Yoon Y, Soles LV, Shi Y. PAS-seq 2: A fast and sensitive method for global profiling of polyadenylated RNAs. Methods Enzymol. 2021;655:25-35. doi: 10.1016/bs.mie.2021.03.013. Epub 2021 Apr 23. PMID: 34183125. Hashimshony, T., Senderovich, N., Avital, G. et al.
> 
> CEL-Seq2: sensitive highly-multiplexed single-cell RNA-Seq. Genome Biol 17, 77 (2016). https://doi.org/10.1186/s13059-016-0938-8
> 
> Herzog, V., Reichholf, B., Neumann, T. et al. Thiol-linked alkylation of RNA to assess expression dynamics. Nat Methods 14, 1198–1204 (2017). https://doi.org/10.1038/nmeth.4435
>
> https://github.com/erhard-lab/gedi/wiki/GRAND-SLAM


### 1 Demultiplex pooled fastq data
During the library preparation, in order to save reagents and time, the samples were pooled after RT reaction since each sample was already indexed by the CEL-Seq2 poly-dT primer. This step, however, requires demultiplexing on our own as bcl2fastq step takes into account only samples sorted upon the second PCR indexed primer.

![R2_like-CELseq2](https://github.com/user-attachments/assets/aaac2b9e-b857-4bfe-9c9a-67c74a6536de)

I used the demultiplex package in my bash_env, I listed first R2 file as that is the place where the barcodes are stored (see in the picture, 8nt UMI, 6nt cell barcode, polyT). https://demultiplex.readthedocs.io/en/latest/usage.html  
Depending on the original fastq.gz size, the command can take some time. I recommend using screen command.  
There might be quite a big file sample_R*_UNKNOWN.fastq.gz.
```
demultiplex demux -r -s 9 -e 14 barcodes.tsv sample_R2.fastq.gz sample_R1.fastq.gz
```

The demultiplexed R1 sequences can be copied into a new folder because we will be using only R1 reads (as shown in the picture above, R2 reads contain only few nucleotides of the actual sequence at the very end, so we will not use them anymore, their main role was about having the cell barcode for demultiplexing). In order to get rid off the "sample_R1_" at the beginning of each (for clarity and for seq2science to run the pipeline as single-end), one can use this bash loop below:
```
for file in *fastq.gz; do mv "$file" "${file/sample_R1_/}"; done
```

Note: In shallow seq low RNA-quality experiments, the sorted fastq files can still contain a lot of R2 reads without the pattern -8 UMI, 6 barcode, polyT- (check with `zcat sample_R2.fastq.gz | head -n 40`). One can apply more stringent filtering by adding TTTTT at the end of each cell barcode sequence in the barcode_CEL-Seq2_48.tab file. Such sample could look like this: (barcode TGTCGA, algorithm allows one mismatch, only reads 3 and 6 are mRNA reads)
![Picture1](https://github.com/user-attachments/assets/3e1f8b7a-d4b8-4a9f-9091-7f3682ce857c)


### 2 Mapping by seq2science
Link to seq2science documentation https://vanheeringen-lab.github.io/seq2science/index.html   


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
One needs to apply for the software at the Erhart lab first (takes around 2 weeks) or use one I already have `/vol/moldevbio/veenstra/asenovska/GRAND-SLAM/GRAND-SLAM_2.0.5f/`  
First time users need to download annotation/index for mapping. Current version is 107 (2022/08/16) (https://github.com/erhard-lab/gedi/wiki/Preparing-genomes)
```
gedi -e IndexGenome -organism homo_sapiens -version 107 -p
```


1. activate a conda environment with java and generate a .cit file, use full pathways
```
/path/to/GRAND-SLAM_2.0.5f/gedi -e Bam2CIT -p mapped.cit /path/to/seq2science/results/final_bam/*.samtools-coordinate.bam
```
There might be an error `"No index is available for this BAM file”` which means there are missing the .bai files. You can index the bam files with samtools:
```
samtools index *.bam
```

2. run GRAND-SLAM, -full flag to obtain Coverage and Conversion data as well
```
/path/to/GRAND-SLAM_2.0.5f/gedi -e Slam -full -genomic homo_sapiens.107 -prefix date_and_projectname-full/4sU -progress -plot -D -reads mapped.cit
```

3. Data generated by GRAND-SLAM
The grandslam command generates multiple files which provide a first glance into the quality of the data in a pdf file. For the following analysis steps I will mainly focus on the generated .tsv file. This files contains per gene information with columns referring to a number of statistics that have been generated per treatment.
The description of all statistics can be found here: https://github.com/erhard-lab/gedi/wiki/GRAND-SLAM

  - Categories used in this experiment:  
    1. Gene  
    2. Symbol  
    3. Readcount : The total number of reads mapped to this gene in condition  
    4. MAP : The mode of the posterior distribution for the NTR (this should usually be used as the NTR)  
    5. Conversions: The total number of conversions in this gene  
    6. Coverage: The total number of U covered by any reads (if all U were converted, the Conversions=Coverage)

### 4 Analysis in R
Script attached, see the _grandslam-4sU.Rmd_ file.
