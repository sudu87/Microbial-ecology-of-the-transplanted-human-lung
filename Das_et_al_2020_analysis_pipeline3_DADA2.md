# Community analysis using DADA2 pipeline - Das et al 

```{r, echo=FALSE}
knitr::opts_chunk$set(message=FALSE,echo=FALSE,eval=FALSE)
```

## Amplicon sequencing set up 
Directory : `/Users/sdas/<PATH>`
- Illumina adapter file
- Raw data location : `/Users/sdas/16S_data/dada2/`
contains :
- mock-files
- fastq.gz
- metadata


## Trimming reads 

Trimmomatic is used to trim reads see [Trimmomatic](:/8608841e30af4bd4b5279fa25114fb74)

```{bash}
mkdir 01_trimmed_R1
mkdir 01_trimmed_R2
mkdir unpaired

for f in *R1.fastq; do trimmomatic PE  -phred33 ${f} ${f/R1.fastq/R2.fastq} 01_trimmed_R1/${f/.fastq/.trim.fastq} unpaired/${f/.fastq/.unpaired.fastq} 01_trimmed_R2/${f/R1.fastq/R2.trim.fastq} unpaired/${f/R1.fastq/R2.unpaired.fastq} HEADCROP:40 MINLEN:180; done

```

Trimming improve base quality but none of the others parameters - Still over-represented sequences, per sequence GC content, .... 


## DADA2 pipeline
### First, install required packages
```{r}
 if (!requireNamespace("BiocManager", quietly = TRUE))
   install.packages("BiocManager")
 BiocManager::install(version = "3.10")
BiocManager::install("dada2", version = "3.10")
BiocManager::install("DECIPHER")
BiocManager::install("phyloseq")
BiocManager::install("biomformat")
BiocManager::install("microbiome/microbiome")
BiocManager::install("Biostrings")
BiocManager::install("Biostrings")
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("DESeq2")
if (!requireNamespace("BiocManager", quietly = TRUE))
install.packages("BiocManager")
BiocManager::install("decontam")

if(!require("devtools"))
install.packages("devtools")
source the phyloseq_to_ampvis2() function from the gist
devtools::source_gist("8d0ca4206a66be7ff6d76fc4ab8e66c6")

remotes::install_github("MadsAlbertsen/ampvis2")
```
### Load libraries
```{r}
library(dada2)
library(ShortRead)
library(Biostrings)
library(ggplot2)
library(biomformat)
library(microbiome)
library(DESeq2)
library(remotes)
library(ggplot2)
library(vegan) # ecological diversity analysis
library(dplyr)
library(scales) # scale functions for vizualizations
library(grid)
library(reshape2) # data manipulation package
library(cowplot)
library(phyloseq)
library(devtools)
library(ampvis2)
library(tidyverse)
library(readxl)

```
### Save and load environment
```{r}
getwd()
save.image("SD_DADA2.RData")
load("SD_DADA2.RData")
```

### Get files path and sample names
```{r}
library(dada2); packageVersion("dada2")
path<-"/Users/sdas/16S_data/dada2/01_trimmed"
# List all the files in trimmed directory
list.files(path)
# Forward and reverse fastq filenames have format: SAMPLENAME_R1_001.trim.fastq and SAMPLENAME_R2_001.fastq
FWDfiles <- sort(list.files(path, pattern="_R1.trim.fastq", full.names = TRUE))
REVfiles <- sort(list.files(path, pattern="_R2.trim.fastq", full.names = TRUE))
# Extract sample names, assuming filenames have format: SAMPLENAME_XXX.fastq
sample.names <- sapply(strsplit(basename(FWDfiles), "_"), `[`, 1)
str(sample.names)
write.csv(sample.names,"samplenames.csv")
```

### Quality scores

Median quality score is the green line. Quartile quality scores are the orange lines. 
The red line (bottom) is the proportion of reads that reach the position (length). 

Here we can see that the reads are shorter than 250bp (due to the cutadapt step)
The overall quality of the reads is good, median and quartiles quality scores are above 30 (phred score).
Reverse reads have 'lower' scores but still >30

```{r}
# Quality scores of R1 reads
plotQualityProfile(FWDfiles[100:130]) 
# Quality scores of R2 reads
plotQualityProfile(REVfiles[20:30])

```


### Trim the data

Trimming should be adapted to the data-type and quality of the reads.
`truncLen` must be large enough to maintain an overlap between forward and reverse reads of at least `20 + biological.length.variation` nucleotides.

`derepFastq` Dereplication step : 
all identical sequences are combiend in "unique sequences" that is associates with "abundance" (number of reads that have this unique sequence)
```{r}
# Place filtered files in filtered/subdirectory
filtFWD <- file.path(path,"filtered", paste0(sample.names, "_F_filt.fastq"))
filtREV<- file.path(path,"filtered", paste0(sample.names, "_R_filt.fastq"))
names(filtFWD) <- sample.names
names(filtREV) <- sample.names
length(filtFWD)
out <- filterAndTrim(FWDfiles, filtFWD, REVfiles, filtREV, truncLen=c(200,180), 
                     maxN=0, maxEE=c(2,4), truncQ=2, rm.phix=TRUE,      
                     compress=TRUE, multithread=TRUE) 
# On Windows set multithread=FALSE
# all parameters but truncLen are default DADA2 params

#any(duplicated(c(FWDfiles, filtFWD)))
tail(out)
save.image("SD_DADA2.RData")
```

```{r}
derepFWD <- derepFastq(filtFWD)
derepREV <- derepFastq(filtREV)
sam.names <- sapply(strsplit(basename(filtFWD),"_"),`[`,1)
names(derepFWD) <- sam.names
names(derepREV) <- sam.names
save.image("SD_DADA2.RData")
derepFWD$ZWER77$map
```


### Learn the error rates
Data is used to model the probability of transistions and transversion (errors) in function of the read quality.
Each run has its specific error rates (cannot combine data from two different runs)

- black dots : observed error rates for each consensus quality score.
- black lines : estimates error rate after convergence of the algorithm
- red line : error rates expected under the nominal definition of the Q-score

! Parameter learning is computationally intensive, so by default the learnErrors function uses only a subset of the data (the first 100M bases = 1e8). If you are working with a large dataset and the plotted error model does not look like a good fit, you can try increasing the nbases parameter to see if the fit improves !


```{r}
# Took 20 min (with other process running )
# Took 7 minutes with no other process runing
sys_str <- Sys.time()
errF <- learnErrors(derepFWD, randomize=TRUE,nbases = 5e+08 ,multithread=TRUE)  
errR <- learnErrors(derepREV, randomize=TRUE,nbases = 5e+08, multithread=TRUE) 
plotErrors(errR, nominalQ=TRUE)
# In the plots, the black line is the error model, the dots are the actual errors
sys_str[2] <- Sys.time()
sys_str
save.image("SD_DADA2.RData")

```

### Sample inference
The DADA2 algorithm divides the reads in ASVs

!Remember that there is a DADA2 workflow for big dataset - to process the samples one by one!
```{r}
sys_str <- Sys.time()
dadaFs <- dada(filtFWD, err=errF, multithread=TRUE) # we need to incorporate "selfconsist" and "pool=TRUE"
dadaRs <- dada(filtREV, err=errR, multithread=TRUE) # we need to incorporate "selfconsist" and "pool=TRUE"
sys_str[2] <- Sys.time()
sys_str
#rm(sys_str)
save.image("SD_DADA2.RData")
dadaFs[[1]]
dadaRs[[1]]
```

### Merging paired reads

Merging reads to obtain full denoised sequences. Merged sequences are output if the overlap is at least of 12 *identical* nucleotides.
! Most of your reads should successfully merge. If that is not the case upstream parameters may need to be revisited: Did you trim away the overlap between your reads? !

merger contains a list of data.frames. Each data.frame contains the merged `$sequence`, `$abundance`, the indices of FWD and REV sequences variant that were merged. Paired-reads that did not exactly match were removed by the `mergePairs` function.

```{r}
mergers <- mergePairs(dadaFs, derepFWD, dadaRs, derepREV, verbose=TRUE, trimOverhang=TRUE)
save.image("SD_DADA2.RData")
# Inspect the merger data.frame from the first sample
head(mergers[[1]])
```

### Construct sequence table

! some sequences may be shorter or longer than what is expected - here ~250 bp.
```{r}
seqtab <- makeSequenceTable(mergers)
dim(seqtab)
print(seqtab)
# Inspect distribution of sequence lengths
table(nchar(getSequences(seqtab)))
```

### Remove sequences with length too distant from amplified region

selection of sequences with +/- 4bp -> 51 sequences are removed
```{r}
seqtab2 <- seqtab[,nchar(colnames(seqtab)) %in% 200:250]
dim(seqtab2)

```

### Remove chimeras
Chimeric sequences are identified if they can be exactly reconstructed by combining a left-segment and a right-segment from two more abundant “parent” sequences.
Most of your reads should remain after chimera removal (it is not uncommon for a majority of sequence variants to be removed though). If most of your reads were removed as chimeric, upstream processing may need to be revisited. In almost all cases this is caused by primer sequences with ambiguous nucleotides that were not removed prior to beginning the DADA2 pipeline.

```{r}
seqtab.nochim <- removeBimeraDenovo(seqtab2, method="consensus", multithread=TRUE, verbose=TRUE)
dim(seqtab.nochim)
rownames(seqtab.nochim)
sum(seqtab.nochim)/sum(seqtab)
# save sequences
sequences <- data.frame(colnames(seqtab.nochim))
colnames(sequences) <- 'sequences'
write.csv(sequences, 'SB_sequences.csv')
save.image("SD_DADA2.RData")
```
### Track the number of reads after each filtering steps
```{r}
getN <- function(x) sum(getUniques(x))
reads_counts <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim))

colnames(reads_counts) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim")
rownames(reads_counts) <- sample.names
head(reads_counts)
```

### Results
After the filtering steps (trimmings, denoising, removal of artifact and chimeras) the remaining reads per species is sufficient to continue the analysis

### Assign taxonomy
Fasta release files from the UNITE ITS database can be used as is. To follow along, download the silva_nr_v132_train_set.fa.gz
Considerations for your own data: If your reads do not seem to be appropriately assigned, for example lots of your bacterial 16S sequences are being assigned as Eukaryota NA NA NA NA NA, your reads may be in the opposite orientation as the reference database. Tell dada2 to try the reverse-complement orientation with assignTaxonomy(..., tryRC=TRUE) and see if this fixes the assignments
```{r}
#assignTaxonomy using DADA2/Silva 
taxa <- assignTaxonomy(seqtab.nochim, "SILVA/silva_nr_v132_train_set.fa", multithread=TRUE)
taxa <- addSpecies(taxa,"SILVA/silva_species_assignment_v132.fa")
taxa.print <- taxa # Removing sequence rownames for display only
rownames(taxa.print) <- NULL
head(taxa.print)
path_trim<-paste(path, "03_Taxonomy", sep="/")
write.csv2(file=paste(path_trim, "Taxtable_dada2.csv", sep="/"),taxa)
write.csv2(file=paste(path_trim, "ASV_sequences.csv", sep="/"),seqtab.nochim)
save.image("SD_DADA2.RData")
```

### Convert to fasta and send to SILVA
Fasta files have sequence as header (so that header is unique)
```{bash eval=FALSE}
awk -F';'  'NR>1{ print  ">ASV"++i "\n" $1 }' Taxtable_dada2.csv  > ASV_sequences.fasta
sed 's/\"//g' ASV_sequences.fasta | sed 's/NA//g' > ASV_sequences2.fasta


# Upload file to https://www.arb-silva.de/aligner/
sina -i ASV_sequences2.fasta -o ASV_sequences2_aligned.fasta --meta-fmt csv --db SILVA_138_SSURef_NR99_05_01_20_opt.arb --search --search-db SILVA_138_SSURef_NR99_05_01_20_opt.arb --lca-fields tax_slv


echo '"";"Kingdom";"Phylum";"Class";"Order";"Family";"Genus";"Species"' > sina_taxonomy.csv
cat ASV_sequences2_aligned.csv | sed 's/,/#/g' | awk -F# '{print $1 ";" $10 }' | sed 's/;/";"/g' | sed 's/^/"/g' | sed 's/;"$//g' | grep -v "lca_tax_slv" >> sina_taxonomy.csv
```


## Data analysis in phyloseq
```{r}
# Set plotting theme
theme_set(theme_bw())

#Data frame containing sample information
   samdf = read.table(file="metadata_dada2_updated261020.txt", sep="\t",header = T, fill=TRUE) # fill=TRUE allows to read a table with missing entries
rownames(samdf) = samdf$Sample_SD
head(samdf)
head(seqtab.nochim)
sample.names
setdiff(sample.names,samdf$Sample_SD)

# Import SINA taxonomy
taxa.sina <- read.csv(file="/Users/sdas/16S_data/dada2/01_trimmed/03_Taxonomy/sina_taxonomy.csv",sep=";",header = TRUE, row.names = 1)
dim(taxa.sina)
dim(seqtab.nochim)
#Create a phyloseq object
ps_raw <- phyloseq(otu_table(seqtab.nochim, taxa_are_rows=F), 
               sample_data(samdf), 
               tax_table(taxa))
otu_table(ps_raw)
sample_data(ps_raw)
sample_names(ps_raw)
tax_table(ps_raw)
ps_raw
# save sequences as refseq and give new names to ASV's
dna <- Biostrings::DNAStringSet(taxa_names(ps_raw))
names(dna) <- taxa_names(ps_raw)
ps_raw <- merge_phyloseq(ps_raw, dna)
taxa_names(ps_raw) <- paste0("ASV", seq(ntaxa(ps_raw)))
ps_raw
ps <- ps_raw
ps
# Export ASV table\ 
table = merge( tax_table(ps),t(otu_table(ps)), by="row.names")
write.table(table, "/Users/sdas/16S_data/dada2/01_trimmed/03_Taxonomy/ASVtable.txt", sep="\t")

# Export to FASTA with Biostrings
writeXStringSet(refseq(ps), "/Users/sdas/16S_data/dada2/01_trimmed/03_Taxonomy/phyloseq_ASVs.fasta",append=FALSE, format="fasta")
# Then align it with SINA/SILVA, and edit the taxonomy table to get it 

# To save phyloseq objects:
# With Biostrings
writeXStringSet(refseq(ps), "/Users/sdas/16S_data/dada2/01_trimmed/03_Taxonomy/outfile.fasta",append=FALSE, format="fasta")
```

### Data exploration and quality control
```{r}
readsumsdf = data.frame(nreads = sort(taxa_sums(ps), TRUE), sorted = 1:ntaxa(ps), type = "ASVs")
readsumsdf = rbind(readsumsdf, data.frame(nreads = sort(sample_sums(ps),TRUE), sorted = 1:nsamples(ps), type = "Samples"))
title = "Total number of reads"
p = ggplot(readsumsdf, aes(x = sorted, y = nreads)) + geom_point()
p + ggtitle(title) + scale_y_log10() + facet_wrap(~type, 1, scales = "free")
```

### Create table, number of features for each phyla
```{r}
table(tax_table(ps)[, "Phylum"], exclude = NULL)
```
### Remove junk reads
```{r}
ps <- subset_taxa(ps, !is.na(Phylum) & !Phylum %in% c("", "uncharacterized"))
sample_data(ps)
```
### Subset negative controls to seperate object, convert to amp_vis object

```{r}
ps_neg<-subset_samples(ps,NegCtrl==1)
neg_amp<- phyloseq_to_ampvis2(ps_neg)
neg_amp
amp_boxplot(neg_amp)
```

### Removing negative controls

```{r}
ps2<-subset_samples(ps,NegCtrl!=1)
ps2
```

### Rarefaction
```{r}
ps3<-rarefy_even_depth(ps2,rngseed = 10000, replace = TRUE, trimOTUs = TRUE, verbose = TRUE)
ps3

```
### New Data frame as sample data with pam info
```{r}
samdf2 = read.table(file="metadata_dada2_updated16112020.txt", sep="\t",header = T, fill=TRUE) # fill=TRUE allows to read a table with missing entries
rownames(samdf2) = samdf2$Sample_SD
samdf2<-sample_data(samdf2)
head(samdf2)
sample_data(ps3)<-samdf2
```

A useful next step is to explore feature prevalence in the dataset, which we will define here as the number of samples in which a taxon appears at least once.

### Compute prevalence of each feature, store as data.frame and add taxonomy and total read counts to this data.frame

Are there phyla that are comprised of mostly low-prevalence features? Compute the total and average prevalences of the features in each phylum.
```{r}
prevdf = apply(X = otu_table(ps3),
               MARGIN = ifelse(taxa_are_rows(ps3), yes = 1, no = 2),
               FUN = function(x){sum(x > 0)})

prevdf = data.frame(Prevalence = prevdf,
                    TotalAbundance = taxa_sums(ps3),
                    tax_table(ps3))

plyr::ddply(prevdf, "Phylum", function(df1){cbind(mean(df1$Prevalence),sum(df1$Prevalence))})
save.image("SD_DADA2.RData")
```
First, explore the relationship of prevalence and total read count for each feature. Sometimes this reveals outliers that should probably be removed, and also provides insight into the ranges of either feature that might be useful. This aspect depends quite a lot on the experimental design and goals of the downstream inference, so keep these in mind. It may even be the case that different types of downstream inference require different choices here. There is no reason to expect ahead of time that a single filtering workflow is appropriate for all analysis.

### Subset to the remaining phyla
```{r}
prevdf1 = subset(prevdf, Phylum %in% get_taxa_unique(ps3, "Phylum"))
ggplot(prevdf1, aes(TotalAbundance, Prevalence / nsamples(ps3),color=Phylum)) +
  # Include a guess for parameter
  geom_hline(yintercept = 0.05, alpha = 0.5, linetype = 2) +  geom_point(size = 2, alpha = 0.7) +
  scale_x_log10() +  xlab("Total Abundance") + ylab("Prevalence [Frac. Samples]") +
  facet_wrap(~Phylum) + theme(legend.position="none")

```
### Transforming counts
```{r}
ps4<-transform_sample_counts(ps3, function(x) x/sum(x))
sample_data(ps4)
```
### Conversion to amp_vis object

```{r}
amp_ps<-phyloseq_to_ampvis2(ps3)
```
### Output OTU table
```{r}
OTU1 = as(otu_table(ps4), "matrix")
if(taxa_are_rows(ps4)){OTU1 <- t(OTU1)}
# Coerce to data.frame
OTUdf = as.data.frame(OTU1)
write.csv(OTUdf,"ps4_otu_table.csv")
ps4_taxa<-as.data.frame(tax_table(ps4))
head(ps4_taxa)
write.csv(ps4_taxa,"ps4_taxa_table.csv")
```

### Add new metadata after Genocrunch based clustering
```{r}
df.nov20<-read.table(file="metadata_dada2_updated16112020.txt", sep="\t",header = T, fill=TRUE) # fill=TRUE allows to read a table with missing entries
rownames(df.nov20) = df.nov20$Sample_SD
```

### Bacterial load plotting after ASV based clustering
```{r}
df.nov20$clustering_level_2_OTU
ggplot(df.nov20,aes(x=clustering_level_2_ASV_AdaptedNumber,y=log10(Cop16SPerMLBAL),fill=clustering_level_2_ASV_AdaptedNumber))+geom_violin(scale = "width")+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))
```

### PAM subsets
```{r}
pam1_asv<-subset_samples(ps3,clustering_level_2_ASV_AdaptedNumber=="pam1")
pam2_asv<-subset_samples(ps3,clustering_level_2_ASV_AdaptedNumber=="pam2")
pam3_asv<-subset_samples(ps3,clustering_level_2_ASV_AdaptedNumber=="pam3")
pam4_asv<-subset_samples(ps3,clustering_level_2_ASV_AdaptedNumber=="pam4")

amp_pam1_asv<-phyloseq_to_ampvis2(pam1_asv)
amp_pam2_asv<-phyloseq_to_ampvis2(pam2_asv)
amp_pam3_asv<-phyloseq_to_ampvis2(pam3_asv)
amp_pam4_asv<-phyloseq_to_ampvis2(pam4_asv)
 
class(amp_pam1_asv)
amp_boxplot(amp_pam1_asv)
```

