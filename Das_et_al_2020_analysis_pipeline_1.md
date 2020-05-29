
#Activating environment
```
conda activate qiime1
```
#Merging R1 and R2 reads together with Index.fastq
Available runs : 5 
Runs 1, 3, 5, 30, 36 and 52 and all runs processed individually
QIIME joined paired end reads was used, this uses Fastq-join tool and min overlap was chosen to be 10 nts. 

```
join_paired_ends.py -f R1.fastq.gz -r R2.fastq.gz -b Index.fastq.gz -j 10 -o joined_reads/
```

#Validate mapping file
```
validate_mapping_file.py -m map.txt -o validated_map
```
#Demultiplexing and stored in folder split_lib_out.
 phred quality score: 28 in at least 75% of each sequence.
```
split_libraries_fastq.py -i fastqjoin.join.fastq -m ../validated_map/map_corrected.txt -b fastqjoin.join_barcodes.fastq -o ../split_lib_out --barcode_type golay_12 --store_demultiplexed_fastq --phred_offset=33 -q 28 -p 0.75 --max_barcode_errors 2 --rev_comp_mapping_barcodes
```
#Split all sequences according to sample names and store in folder split_seq_out. This is done for selecting and filtering samples individually.

```
split_sequence_file_on_sample_ids.py -i seqs.fastq -o ../split_seq_out  --file_type fastq
```

#Merging and quality control
FastQC reports attached separately. Sequences trimmed 40 bp from ends and kept minimum length to 280
```
cat *.fastq > merged.fastq
fastx_trimmer -i merged.fastq -t 40 -m 280 -Q 33 -o merged_trimmed.fastq
```
#Converting fastq to fasta and quality files (this conversion may be also included in the script for vsearch sith fastx toolkit) 
```
convert_fastaqual_fastq.py -c fastq_to_fastaqual -f merged.fastq -o fastq2fasta/ 
```
#gold.fa database file was formatted to be a proper fasta file by using FASTX toolkit :http://hannonlab.cshl.edu/fastx_toolkit
```
fasta_formatter -i gold.fa.txt -o gold.fa
```

#Integrating cultured sequences - LUMICOL
LuMiCol and 16S seq data are going to be merged but only after dereplication step. refer to vsearch_mod.sh script
Tasks performed by the script:
1. Dereplication
2. Merged fasta and sort
3. proceed to clustering
4. Chimera check
5. OTU picking

#Permissions
```
chmod +x vsearch.sh 
./vsearch.sh 
```
#Script to execute vsearch based OTU picking: vsearch.sh 
Ran on the High Performance Computing Cluster, from Swiss Institute of Bioinformatics for EPFL and UNIL.
```
#!/bin/bash
exec &> vsearch_log.txt

fastq_to_fasta -Q33 -v -i merged.fastq -o merged.fna 

vsearch --derep_fulllength merged.fna --minuniquesize 2 --sizeout --output c1_derep.fa 

vsearch --derep_fulllength lumicol.fna --minuniquesize 1 --sizeout --output c1_derep_lumicol.fa

cat c1_derep.fa c1_derep_lumicol.fa > c1_merged.fa

vsearch --sortbysize c1_merged.fa --sizein --minsize 1 --output c1_merged_sorted.fa

vsearch --cluster_size c1_merged_sorted.fa --centroids c2_otus.fa --id 0.98 --sizein --sizeout --relabel OTU_ --maxaccepts 16 --wordlength 8 --strand both --log centroids.log --sizeorder --usersort --maxrejects 64

vsearch --uchime_denovo c2_otus.fa --chimeras c3_chimeras_denovo.fa --nonchimeras c3_otus_denovo.fa --uchimeout c3_uchime_denovo.tab

vsearch --uchime_ref c3_otus_denovo.fa --db /DIRECTORY/gold.fa --chimeras c4_chimeras_ref.fa --nonchimera c4_otus_final.fa --uchimeout c4_uchime_reference.tab

vsearch --sortbysize c4_otus_final.fa --sizein --minsize 2 --output c5_final_nosingle.fa

sed 's/;.*//g' c5_final_nosingle.fa > c0_otus_final_sorted.fa
```
#phiX spike-in removal
```
usearch -filter_phix c0_otus_final_sorted.fa --output c0_otus_final_sorted_phifil.fa -alnout c0_phix_hits.txt
```
#Merge LuMiCol before last step of mapping all sequences at 97% to clusters made at 98%.
```
cat merged.fna lumicol.fna > merged_lumicol.fna
vsearch --usearch_global merged_lumicol.fna --db c0_otus_final_sorted.fa --id 0.97 --self --maxaccepts 16 --wordlength 8 --strand both --log merged_lumicol.log --maxrejects 64 --uc map.uc
```

#converting to BIOM from generated mapping file 
```
biom from-uc -i map.uc -o otus.biom
biom convert -i otus.biom -o otus_biom.csv --to-tsv
sed 's/;/\t/g' otus_biom.csv > otus_biom.tsv
biom convert -i otus_biom.tsv -o otus_biom_tsv.biom --table-type="OTU table" --to-hdf5
biom validate-table -i otus_biom_tsv.biom 
```
#add metadata
```
biom add-metadata -i otus_biom_tsv.biom -o otus_biom_wData.biom --sample-metadata-fp metadata.txt 
```
#Adding taxonomy
```
biom add-metadata --sc-separated taxonomy --observation-header OTUID,taxonomy --observation-metadata-fp metadata.txt -i lotus_biom_wData.biom -o otus_biom_wData_wTaxa.biom
```

#SINA alignment
```
conda activate sina
sina -i /DIRECTORY/merged_lumicol.fna -o /DIRECTORY/merged_lumicol_aligned.fna --meta-fmt csv --db /DIRECTORY/SSURef_NR99_132_SILVA_13_12_17_opt.arb --search --search-db /DIRECTORY/SSURef_NR99_132_SILVA_13_12_17_opt.arb --lca-fields tax_slv
sed '/^[^>]/ y/uU/tT/' c0_otus_final_aligned.fa > c0_otus_final_sina_aligned.fa

```
#Mapping to eHOMD 
```
vsearch --usearch_global merged_lumicol.fna --db /DIRECTORY/HOMD_16S_rRNA_RefSeq_V15_1.fasta --id 0.97 --self --maxaccepts 16 --wordlength 8 --strand both --log ltx_homd_cluster.log --maxrejects 64 --uc homd_map.uc
```

#using sina alignment to make tree using FastTree
```
FastTree -gtr -nt c0_otus_final_sina_aligned.fa > c0_otus_final_tree.tre
```

#Extracting LUMICOL IDs from mapping data
```
grep 'LUMICOL' map.uc > lumicol_mapping_only_from_uc.txt 
```
#Extract the column with LUMICOL_*** 
```
 awk '{print $9}' lumicol_mapping_only_from_uc.txt > lumicol_numbers.txt
```


