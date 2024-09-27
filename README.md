# The-amplicon-sequencing-analysis-of-big-dataset

#Merge paired-end reads into a single read, see https://github.com/jsh58/NGmerge

>NGmerge -1 sample1_R1.fastq.gz -2 sample1_R2.fastq.gz -o sample1.fastq.gz

#Quality trimming and adapter clipping, see https://github.com/usadellab/Trimmomatic for installation
trimmomatic SE -threads 10 -phred33 sample1.fastq.gz sample1_trim.fastq.gz ILLUMINACLIP:TruSeq3-SE.fa:2:30:10 LEADING:10 TRAILING:10 SLIDINGWINDOW:4:20 MINLEN:50
