
#Available runs : 5 

Runs 1, 3, 5, 30, 36 and 52 

All runs processed individually

#Merging R1 and R2 reads together with Index.fastq
#Hence, QIIME joined paired end reads was used, this uses Fastq-join tool and min overlap was chosen to be 10 nts. 

`join_paired_ends.py -f R1.fastq.gz -r R2.fastq.gz -b Index.fastq.gz -j 10 -o joined_reads/
