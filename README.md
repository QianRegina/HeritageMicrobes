# The-amplicon-sequencing-analysis-of-big-dataset

#Merge paired-end reads into a single read, see https://github.com/jsh58/NGmerge
>NGmerge -1 sample1_R1.fastq.gz -2 sample1_R2.fastq.gz -o ./merge/sample1.fastq.gz

#Quality trimming and adapter clipping of one sample, see https://github.com/usadellab/Trimmomatic for installation
>trimmomatic SE -threads 10 -phred33 ./merge/sample1.fastq.gz ./trimm/sample1_trim.fastq.gz ILLUMINACLIP:TruSeq3-SE.fa:2:30:10 LEADING:10 TRAILING:10 SLIDINGWINDOW:4:20 MINLEN:50

#Do the trimming step for multiple sample 
>for i in `ls ./merge/*.fastq.gz`
>
>do
>
>x=${i/*\//}
>
>echo trimmomatic SE -threads 10 -phred33 $i $x ILLUMINACLIP:TruSeq3-SE.fa:2:30:10 LEADING:10 TRAILING:10 SLIDINGWINDOW:4:20 MINLEN:50
>
>done > command.trimmomatic.list
>
>sh command.trimmomatic.list 

#Quality assessment, see https://github.com/s-andrews/FastQC
>fastqc ./trimm/*fastq.gz -t 2

#**Analysis using QIIME2 version 2024.5**, see https://github.com/qiime2

#inpot data
>qiime tools import  --type 'SampleData[SequencesWithQuality]' \
>--output-path ./demux-single-end.qza  \
>--input-format CasavaOneEightSingleLanePerSampleDirFmt \
>--input-path ./trimm/

#Quality filtering with default parameters
>qiime quality-filter q-score --i-demux demux-single-end.qza \
>--o-filtered-sequences demux_filtered.qza \
>--o-filter-stats demux-filter-stats.qza

#Viewing filter quality of metadata
>qiime metadata tabulate --m-input-file demux-filter-stats.qza --o-visualization demux-filter-stats.qzv

#Apply Deblur workflow for denoise based on demux-filter-stats.qzv
>qiime deblur denoise-16S --i-demultiplexed-seqs demux_filtered.qza \
>--p-trim-length 205 --o-representative-sequences rep-seqs-deblur.qza \
>--o-table table-deblur.qza --p-sample-stats \
>--o-stats deblur-stats.qza --p-jobs-to-start 10

#Generate summary statistics for Deblur results
>qiime deblur visualize-stats --i-deblur-stats deblur-stats.qza --o-visualization deblur-stats.qzv

#cluster sequences according to closed references
> qiime vsearch cluster-features-closed-reference --i-table table-deblur.qza --i-sequences rep-seqs-deblur.qza \
 --i-reference-sequences ./database/silva-138.1-ssu-nr97-seqs-derep.qza \
 --p-perc-identity 0.97 --o-clustered-table table-cr-97.qza \
 --o-clustered-sequences repseqs-cr-97.qza --o-unmatched-sequences unmatched-cr-97.qz  --p-threads 10

#Obtain OTU table
>qiime tools export --input-path table-cr-97.qza --output-path ./
>biom convert -i feature-table.biom -o table-cr-97.tsv --to-tsv

#Extract represented sequence
>qiime tools export --input-path repseqs-cr-97.qza --output-path ./ 
>mv dna-sequences.fasta repseqs-cr-97.fasta

#Taxnomy annotation
>qiime feature-classifier classify-sklearn --i-classifier database/silva-138.1-ssu-nr97-classifier.qza \
--i-reads repseqs-cr-97.qza --o-classification taxonomy_97.qza \
--p-n-jobs 10 --p-reads-per-batch 1000

#Obtain Taxnomic table
>qiime tools export --input-path taxonomy_97.qza --output-path ./ 

#Extract optimum depth. Based on deblur-stats.qzv to comfirm the value of sampling-depth (eg. 3012)
>qiime feature-table rarefy --o-rarefied-table ./resampled_table_97.qza \
>--i-table table-cr-97.qza --p-sampling-depth 3012 &

#Export the table for diversity and UMAP analysis
>qiime tools export --input-path resampled_table_97.qza --output-path ./ 
>biom convert -i feature-table.biom -o resampled_table_97.tsv --to-tsv
