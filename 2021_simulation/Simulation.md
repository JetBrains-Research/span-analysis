Simulation
==========

**Chips** is available from bioconda and on [GitHub](https://github.com/gymreklab/chips).
Paper in [Bioinformatics](https://link.springer.com/article/10.1186/s12859-021-04097-5).

Works only with fasta and peaks on chromosome 15, otherwise fails.

# Data
Download data for human CD14-positive monocyte cells.

ChIP-seq
* [H3K4me1](https://www.encodeproject.org/files/ENCFF076WOE/)
* [H3K4me3](https://www.encodeproject.org/files/ENCFF001FYS/)
* [H3K27ac](https://www.encodeproject.org/files/ENCFF000CEN/)
* [H3K27me3](https://www.encodeproject.org/files/ENCFF001FYR/)
* [H3K36me3](https://www.encodeproject.org/files/ENCFF000CFB/)

Control
* [ENCFF825XKT](https://www.encodeproject.org/files/ENCFF825XKT/) (for H3K4me1)
* [ENCFF001HUV](https://www.encodeproject.org/files/ENCFF001HUV/) (for H3K4me3 and H3K27me3)
* [ENCFF692GVG](https://www.encodeproject.org/files/ENCFF692GVG/) (for H3K27ac and H3K36me3)

Reference
* [hg38 reference fa](https://www.encodeproject.org/files/GRCh38_no_alt_analysis_set_GCA_000001405.15/)

# Download peaks data
```
WORK_DIR=/mnt/stripe/shpynov/2021_chips
cd $WORK_DIR
wget https://www.encodeproject.org/files/ENCFF366GZW/@@download/ENCFF366GZW.bed.gz -O H3K4me1.bed.gz
wget https://www.encodeproject.org/files/ENCFF651GXK/@@download/ENCFF651GXK.bed.gz -O H3K4me3.bed.gz  
wget https://www.encodeproject.org/files/ENCFF039XWV/@@download/ENCFF039XWV.bed.gz -O H3K27ac.bed.gz
wget https://www.encodeproject.org/files/ENCFF666NYB/@@download/ENCFF666NYB.bed.gz -O H3K27me3.bed.gz      
wget https://www.encodeproject.org/files/ENCFF213IBM/@@download/ENCFF213IBM.bed.gz -O H3K36me3.bed.gz
gunzip *.gz
```

For MACS2, SICER, SPAN peaks launch chipseq snakemake pipeline from fastq files.

# Learn models

```
# Ensure that chips is available!
bash learn.sh
bash frip.sh
```

# Simulate reads

```
# Ensure that chips is available!
bash simulate.sh
mkdir fastq
mv *.fastq fastq/
```

# Prepare input for peak calling (align control and filter chr15 only)
```
for F in $(ls input*.bam | grep -v chr15); do 
    echo $F; 
    samtools view $F chr15 -b > ${F/.bam/_chr15.bam}; 
done
 
for F in input*chr15.bam; do 
    echo $F; 
    bedtools bamtofastq -i $F -fq ${F/.bam/.fastq}; 
done
```

# Launch peak callers
```
cd /mnt/stripe/shpynov/chipseq-smk-pipeline
conda activate snakemake

# Perform peak calling using chipseq snakemake pipeline
for FDR in 0.1 0.05 0.01 1e-3 1e-4 1e-5 1e-6; do
  echo "FDR $FDR"
  
  echo "MACS2 narrow"
  snakemake all --cores 24 --use-conda --directory $WORK_DIR \--config genome=hg38 \
    fastq_dir=$WORK_DIR/fastq fastq_ext=fastq macs2_params="-q $FDR" macs2_mode=narrow macs2_suffix=q$FDR
  
  echo "MACS2 broad"
  snakemake all --cores 24 --use-conda --directory $WORK_DIR --config genome=hg38 \
    fastq_dir=$WORK_DIR/fastq fastq_ext=fastq macs2_params="--broad --broad-cutoff $FDR" macs2_suffix=broad$FDR;
  
  echo "SPAN"
  snakemake all --cores 24 --use-conda --directory $WORK_DIR --config genome=hg38 \
    fastq_dir=$WORK_DIR/fastq fastq_ext=fastq span_fdr=$FDR
  
  echo "SPAN Gap 0"
  snakemake all --cores 24 --use-conda --directory $WORK_DIR --config genome=hg38 \
    fastq_dir=$WORK_DIR/fastq fastq_ext=fastq span_fdr=$FDR span_gap=0    
done
```


# Launch SPAN modifications
SPAN modification `span234.jar` is built from the branch span234.

```
cd $WORK_DIR
for FDR in 0.1 0.05 0.01 1e-3 1e-4 1e-5 1e-6; do 
    snakemake all --cores 24 --config fdr=$FDR; 
done
```


#Analyze 
Prepare data for visualization and analysis in jupyter notebook.
`bash analyze.sh`

#Merge No Control and tracks with Control 
```
cat ../2021_chips_span_modifications/report.tsv | grep Macs2$'\t' | sed 's/Macs2/Macs2-NC/g' >> report.tsv
cat ../2021_chips_span_modifications/report.tsv | grep Macs2Broad | sed 's/Macs2Broad/Macs2Broad-NC/g'  >> report.tsv  
cat ../2021_chips_span_modifications/report.tsv | grep SPAN-GAP5 | sed 's/SPAN-GAP5/SPAN-GAP5-NC/g' >> report.tsv
cat ../2021_chips_span_modifications/report.tsv | grep SPAN-GAP0 | sed 's/SPAN-GAP0/SPAN-GAP0-NC/g' >> report.tsv
cat ../2021_chips_span_modifications/report.tsv | grep SPAN-Islands | sed 's/SPAN-Islands/SPAN-Islands-NC/g' >> report.tsv
cat ../2021_chips_span_modifications/report.tsv | grep SPAN-NZ >> report.tsv
cat ../2021_chips_span_modifications/report.tsv | grep SPAN-Z >> report.tsv
```