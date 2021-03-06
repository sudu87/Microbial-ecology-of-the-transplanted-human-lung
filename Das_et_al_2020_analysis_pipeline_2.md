
# Microbial community analysis of the lung - Part 2"

### 1. Set working directory
```{r}
setwd("/Users/sdas/16S_data/all_run_240219/merged_060319")
```

### 2. Load packages
```{r}
library(ggplot2)
library(rmarkdown)
library(vegan) # ecological diversity analysis
library(dplyr)
library(scales) # scale functions for vizualizations
library(grid)
library(reshape2) # data manipulation package
library(ape)
library(devtools) 
library(biomformat)
library(phyloseq)
library(ampvis2)
library(plotly)
library(genefilter)
library(Rtsne)
library(microbiome)
library(BiocInstaller)
library(metacoder)
library(taxa)
library(edgeR)
library(grid)
library(plyr)
library(genefilter)
library(metagenomeSeq)
library(fso)
library(philentropy)
library(gplots)
library(DESeq2)
library(ampvis2)
library(RColorBrewer)
library(ComplexHeatmap)
library(pheatmap)
library(superheat)
library(car)
library(intergraph)
library(GGally)
library(network)
library(ggnet)
library(ggnetwork)
library(divo)
library(extrafont)
library(apeglm)
library(EnhancedVolcano)
library(gridExtra)
library(grid)
library(IHW)
library(gtx)
library(PMCMR)
library(ggpubr)
library(EnvStats)
library(randomForestExplainer)
library(randomForest)
library(markovchain)
library(tidyverse)
library(forcats)
library(ARTool)
library(emmeans)
library(multcomp)
library(phia)
library(markovchain)
library(randomForest)
library(dplyr) # for the "arrange" function
library(rfUtilities) # to test model significance
```
### 3. Functions
```{r}
#Summary SE function
summarySE <- function(data=NULL, measurevar, groupvars=NULL, na.rm=FALSE,
                      conf.interval=.95, .drop=TRUE) {
  library(plyr)
  
  
  length2 <- function (x, na.rm=FALSE) {
    if (na.rm) sum(!is.na(x))
    else       length(x)
  }
  datac <- ddply(data, groupvars, .drop=.drop,
                 .fun = function(xx, col) {
                   c(N    = length2(xx[[col]], na.rm=na.rm),
                     mean = mean   (xx[[col]], na.rm=na.rm),
                     sd   = sd     (xx[[col]], na.rm=na.rm)
                   )
                 },
                 measurevar
  )
  datac <- rename(datac, c("mean" = measurevar))
  
  datac$se <- datac$sd / sqrt(datac$N)  
  
  ciMult <- qt(conf.interval/2 + .5, datac$N-1)
  datac$ci <- datac$se * ciMult
  
  return(datac)
}

```

### 4. Set plotting theme and font
```{r}
theme_set(theme_bw())
choose_font("Arial")
```

### 5. Creating a phyloseq object
```{r}
ltx_otus<-import_biom("otus.biom",parseFunction = parse_taxonomy_default)
sample<-import_qiime_sample_data("metadata.txt")
#Adding tree
ltx_tree=ape::read.tree("c0_otus_final_tree.tre.tre")
#merging
ltx_otus = merge_phyloseq(ltx_otus,sample,ltx_tree)
ltx_otus
```

### 6. Renaming taxonomy column names
```{r}
colnames(tax_table(ltx_otus)) <- c("Kingdom", "Phylum", "Class", "Order", "Family", "Genus","Species")
rank_names(ltx_otus)
```
### 7. Create table, number of features for each phyla
```{r}
table(tax_table(ltx_otus)[, "Phylum"], exclude = NULL)
```
### 8. Exploring metadata
```{r}
sample_names(ltx_otus)
```
### 9. Save workspace
```{r}
save.image(file = "FILE.RData")
load(file = "FILE.RData")
```
### 10. Storing Lumicol in another object for further exploration
```{r}
lumicol<-subset_samples(ltx_otus,Sample_SD=="LUMICOL")
lumicol
```

## Data exploration
### 11. Quality control of data
```{r}
readsumsdf = data.frame(nreads = sort(taxa_sums(ltx_otus), TRUE), sorted = 1:ntaxa(ltx_otus), type = "OTUs")
readsumsdf = rbind(readsumsdf, data.frame(nreads = sort(sample_sums(ltx_otus),TRUE), sorted = 1:nsamples(ltx_otus), type = "Samples"))
title = "Total number of reads"
p = ggplot(readsumsdf, aes(x = sorted, y = nreads)) + geom_point()
p + ggtitle(title) + scale_y_log10() + facet_wrap(~type, 1, scales = "free")
```

### 12. Converting phyloseq to ampvis2
```{r}
obj1<-ltx_otus
otutable <- data.frame(OTU = rownames(phyloseq::otu_table(obj1)@.Data),
                       phyloseq::otu_table(obj1)@.Data,
                       phyloseq::tax_table(obj1)@.Data,
                       check.names = FALSE
                       )

meta2<- data.frame(phyloseq::sample_data(obj1), check.names = FALSE)
object1<-amp_load(otutable,meta2,fasta = "c0_otus_final_sorted_phifil_test.fa ")

#alternative when direct import fails
meta2<-read.table("metadata.txt",header = T) 
object1<- amp_load(otutable,meta2,fasta = "c0_otus_final_sorted_phifil_test.fa ")
```

### 13. Analysis of negative control
```{r}
negatives<- amp_subset_samples(object1, NegCtrl %in% "1")
negative_otus<-cbind(negatives$abund,negatives$tax)
write.csv(negative_otus,"negative_control_OTU_table_wTaxa.csv")
```
### 14. Boxplots of the most abundant taxa according to read counts.
```{r}
neg_ctrl<-amp_boxplot(negatives,adjust_zero=T,tax_add="OTU",detailed_output=T,normalise =F,tax_show = 30)
neg_ctrl
```
### 15. Boxplots of the most abundant taxa according to read counts and grouped by samples
```{r}
neg_sample<-amp_boxplot(negatives,adjust_zero=T,tax_add="OTU",plot_type = "point",detailed_output=TRUE,normalise = F,group_by = "Sample_SD")
neg_sample$plot
write.table(neg_sample$data,file = "negative_non_normalized_by_sample.txt")
```
### 16. Heatmap of the most abundant taxa according to read counts and grouped by samples
```{r}
neg_heat<-amp_heatmap(negatives,normalise = F,tax_add="OTU",tax_aggregate = "Genus",tax_show = 40,plot_values =F,color_vector = c("white","red","darkred"),min_abundance = 0.1,measure = "median")
```

### 17. Removing negative controls from phyloseq object.
```{r}
ltx_noneg<-subset_samples(ltx_otus,NegCtrl!=1)
ltx_noneg
```
### 18. Take out samples with NA in 16S copy column.
```{r}
ltx_otus2<-subset_samples(ltx_noneg,Cop16SPerMLBAL!="NA")
sample_data(ltx_otus2)$Cop16SPerMLBAL<-as.numeric(as.character(sample_data(ltx_otus2)$Cop16SPerMLBAL))#converting the variable to numeric
ltx_otus3<-subset_samples(ltx_otus2,Cop16SPerMLBAL>0)
ltx_otus3
```
### 19. Removal of contaminating OTUs
```
As we saw before, there are some OTUs that come from the negative controls. We will remove them.
```
```{r}
#Wherever genera cannot be used since it will remove undesired taxa
badTaxa=c("OTU_10","OTU_124","OTU_19","OTU_44","OTU_50","OTU_5","OTU_10055","OTU_101")
allTaxa = taxa_names(ltx_otus3)
allTaxa <- allTaxa[!(allTaxa %in% badTaxa)]
ltx_otus4<-prune_taxa(allTaxa,ltx_otus3)
ltx_otus5<-subset_taxa(ltx_otus4, !is.na(Phylum) & !Phylum %in% c("", "NA"))
ltx_otus5
sums<-data.frame(nreads = sort(sample_sums(ltx_otus5),TRUE))
```

### 20. Removal of low read samples
```
We will remove all ambigous phyla, sample read amounts less than 10,000,samples
```
```{r}
high_read<-prune_samples(sample_sums(ltx_otus5)>=10000,ltx_otus5)#object with high read samples, this will be used to filter our low 16s copies
high_read #contains samples with at least 10000 reads

```

### 21.Rarefaction of data to minimum read counts 
```
This only helps diversity and will not be used for differential abundances
```

```{r}
ltx_rare<-rarefy_even_depth(high_read, sample.size = min(sample_sums(high_read)),rngseed = 10000, replace = TRUE, trimOTUs = TRUE, verbose = TRUE)
```
### 22. Compositional data
```{r}
ltx_rare_rel<-transform_sample_counts(ltx_rare, function(x) x/sum(x))
ltx_rare_rel
```
### 23. Exporting OTU and Metadata for Genocrunch and PAM analysis
```{r}
write_phyloseq(ltx_rare_rel,'OTU')
write_phyloseq(ltx_rare_rel,'METADATA')
```
```{r}
tax_summary<-table(tax_table(eric.june2019)[, "Phylum"], exclude = NULL)
write.csv(tax_summary,"tables/tax_summary_S1.csv")
```

### 24. Compute prevalence of each feature, store as data.frame
```{r}
prevdf = apply(X = otu_table(ltx_rare),
               MARGIN = ifelse(taxa_are_rows(ltx_rare), yes = 1, no = 2),
               FUN = function(x){sum(x > 0)})
```
### 25. Add taxonomy and total read counts to this data.frame
```{r}
prevdf = data.frame(Prevalence = prevdf,TotalAbundance = taxa_sums(ltx_rare),tax_table(ltx_rare))
plyr::ddply(prevdf, "Phylum", function(df1){cbind(mean(df1$Prevalence),sum(df1$Prevalence))})
```

### 26. Subset to the remaining phyla
```{r}
prevdf1 = subset(prevdf, Phylum %in% get_taxa_unique(ltx_rare, "Phylum"))
prev_plot<-ggplot(prevdf1, aes(TotalAbundance, Prevalence / nsamples(ltx_rare_rel),color=Phylum)) +
geom_hline(yintercept = 0.5, alpha = 0.5, linetype = 2) +geom_point(size = 2, alpha = 0.7) + xlab("Cumulative Abundance") + ylab("Prevalence [Frac. Samples]")  + theme(legend.position="none")+facet_wrap(~Phylum)
prev_plot
```
### 27. Define prevalence threshold as 50% of total samples
```{r}
prevalenceThreshold = 0.5 * nsamples(ltx_rare_rel)
prevalenceThreshold
```
### 28. Execute prevalence filter, using `prune_taxa()` function.
```{r}
keepTaxa = rownames(prevdf1)[(prevdf1$Prevalence >= prevalenceThreshold)]
hi_prev_rel = prune_taxa(keepTaxa, ltx_rare_rel)
sample_sums(hi_prev_rel)
write.csv(sample_sums_prev,"tables/sample_sums_prev.csv")
prevdf2 = subset(prevdf1, Genus %in% get_taxa_unique(hi_prev_rel, "Genus"))
prev_plot2<-ggplot(prevdf2, aes(TotalAbundance, Prevalence / nsamples(hi_prev_rel),color=Genus,size=TotalAbundance)) +geom_hline(yintercept = 0.5, alpha = 0.5, linetype = 2) + geom_point(alpha = 0.7) +geom_vline(xintercept = 2000,alpha = 0.5, linetype = 1)+ xlab("total rarefied reads") + ylab("Prevalence (fraction of samples)")+ theme(legend.text = element_text(size = 8),legend.position="bottom",axis.text.y= element_text(size = 20),axis.text.x= element_text(angle = 90),strip.text = element_text(size=15))+scale_y_continuous(limits = c(0,1))+facet_grid(~Phylum)
write.csv(prevdf1,"tables/final_tables/new_prevalence_table.csv")
#extracting prevalent taxa from eric new
prev_otus<-as(otu_table(hi_prev_rel),"matrix")
prev_otus<-t(prev_otus)
write.csv(prev_otus,"tables/final_tables/prev_otus.csv")
prev_meta<-as(sample_data(hi_prev_rel),"data.frame")
sample_data(hi_prev_rel)
tax_summary<-table(tax_table(hi_prev_rel)[, "Phylum"], exclude = NULL)
View(tax_summary)
prev_plot2
```
### 29. Converting ltx_rare_rel object to ampvis object
```{r}
obj3<-ltx_rare_rel
otutable_exp <- data.frame(OTU = rownames(phyloseq::otu_table(obj3)@.Data),
                       phyloseq::otu_table(obj3)@.Data,
                       phyloseq::tax_table(obj3)@.Data,
                       check.names = FALSE
                       )
erc.tree=phy_tree(ltx_rare_rel)
meta_exp<- data.frame(phyloseq::sample_data(obj3), check.names = FALSE)
amp_exp<-amp_load(otutable_exp,meta_exp,tree = erc.tree)
amp_exp
amp_box_phyla<-amp_boxplot(amp_exp,sort_by = "median",adjust_zero = T,detailed_output = T)+theme(axis.text.y = element_text(face="italic",size=12),axis.text.x = element_text(size=20),axis.title = element_text(size = 20))
detailed_taxa<-amp_frequency(amp_exp, tax_aggregate = "Genus",detailed_output = T)
View(detailed_taxa$data)
write.csv(detailed_taxa$data,"tables/amp_freq_detailed_genus.csv")
amp_box_median<-amp_boxplot(amp_exp,sort_by = "median",tax_aggregate = "OTU",tax_add = "Genus",tax_show = 22)+theme(axis.text.y = element_text(face="italic",size=12),axis.text.x = element_text(size=20),axis.title = element_text(size = 18))
amp_box_median
```

### 30. Richness vs bacterial copy
```{r}
alpha_summary<-amp_alphadiv(amp_exp)
alpha_summary$log_copy<-log10(alpha_summary$Cop16SPerMLBAL)
mode(alpha_summary$log_copy)
otu_bact_copy<-ggplot(alpha_summary,aes(log_copy,ObservedOTUs))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))+scale_y_continuous(limits = c(0,4))+scale_x_continuous(limits = c(1,7),breaks = c(1,2,3,4,5,6,7))
otu_copy_lm<-(lm(log_copy~ObservedOTUs,data=alpha_summary))
summary(otu_copy_lm)
# Create the function.
getmode <- function(v) {
   uniqv <- unique(v)
   uniqv[which.max(tabulate(match(v, uniqv)))]
}

getmode(alpha_summary$log_copy)
alpha_summary$pielou<-alpha_summary$Shannon/log(alpha_summary$ObservedOTUs)
otu_bact_copy
```
### 31. Core analysis using eric's data at OTU level
```{r}
amp_exp
core<-amp_core(amp_exp,tax_empty = "OTU",tax_aggregate = "OTU",abund_thrh = 1,detailed_output=T,plotly = T)
write.csv(eric_core$data,"tables/incidence_abundance.csv")
core_data<-read.csv("tables/incidence_abundance.csv")
```

### 32. Creating a simple incidence plot
```{r}
incidence<-read.csv("tables/incidence_abundance.csv")
incidence$taxa_info<-paste(incidence$Genus,";",incidence$OTU)
incidence$col=ifelse(incidence$Freq_percent>50,paste(incidence$taxa_info),"NA")
incidence_eric
freq_plot<-ggplot(incidence,aes(x=reorder(OTU,-Freq_percent),y=Freq_percent))+geom_point(aes(color=factor(col)),size=3)+theme(legend.position =c(0.3,0.5),legend.text = element_text(size = 10,face="italic"),axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid=element_blank())+guides(fill=guide_legend(ncol = 3,byrow = T))+ylim(0,100)
freq_plot
```
### 33. Refining the incidence and creating rank abundance plot
```{r}
clusterData_eric = filter(incidence,Frequency >=23)
View(clusterData)
clusterData$col=ifelse(clusterData$Abundance>1,paste(clusterData$taxa_info),"NA")
rank_eric_otus<-ggplot(clusterData,aes(x=reorder(OTU,-Abundance),y=Abundance))+geom_point(aes(color=factor(col)),size=3)+theme(legend.position =c(0.1,0.5),axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid=element_blank(),legend.text = element_text(size = 10,face="italic"),panel.border =element_rect(colour="black"))+geom_hline(yintercept = 1, linetype = 2,color="red")
rank_eric_otus
```
### 34. Normalized phyloseq object
```{r}
ltx_rare_rel
ltx.rare.rel.abs <- ltx_rare_rel
for(n in 1:nsamples(ltx_rare_rel))
{
otu_table(ltx.rare.rel.abs)[,n] <- otu_table(ltx_rare_rel)[,n]*sample_data(ltx_rare_rel)$Cop16SPerMLBAL [n]
}
ltx.rare.rel.abs
```
### 35. Extracting lumicol bacteria data
```{r}
lumicol_otus<-c("OTU_69",
"OTU_30",
"OTU_38",
"OTU_3427",
"OTU_27",
"OTU_219",
"OTU_2",
"OTU_1",
"OTU_3227",
"OTU_592",
"OTU_9786",
"OTU_513",
"OTU_107",
"OTU_1474",
"OTU_263",
"OTU_699",
"OTU_328",
"OTU_501",
"OTU_39",
"OTU_20",
"OTU_34",
"OTU_7",
"OTU_143",
"OTU_5553",
"OTU_11",
"OTU_228",
"OTU_55",
"OTU_41",
"OTU_9424",
"OTU_205",
"OTU_57",
"OTU_67",
"OTU_3",
"OTU_5034",
"OTU_113",
"OTU_115",
"OTU_6",
"OTU_58",
"OTU_1567",
"OTU_7830",
"OTU_42",
"OTU_336",
"OTU_17",
"OTU_161",
"OTU_10031",
"OTU_4021",
"OTU_5247")
lumicol_extraction<-prune_taxa(lumicol_otus,lumicol)
lumicol_tree<-phy_tree(lumicol_extraction)
ape::write.tree(lumicol_tree,file="lumicol_singles.tre")
```

### 36. Extracting the most important contributors - 30 OTUs

```{r}
imp2<-c("OTU_11",
"OTU_3",
"OTU_6",
"OTU_15",
"OTU_30",
"OTU_20",
"OTU_8",
"OTU_17",
"OTU_4",
"OTU_7",
"OTU_39",
"OTU_1",
"OTU_41",
"OTU_27",
"OTU_34",
"OTU_107",
"OTU_57",
"OTU_26",
"OTU_42",
"OTU_69",
"OTU_21",
"OTU_46",
"OTU_2",
"OTU_16",
"OTU_24",
"OTU_49",
"OTU_78",
"OTU_6768",
"OTU_234",
"OTU_63")
imp2
impTaxa2<-prune_taxa(imp2,ltx.rare.rel.abs)
impTaxa2
#Contribution of 30 taxa on total reads
sum(sample_sums(impTaxa2))/sum(sample_sums(ltx_rare_rel))*100

```
### 37. Converting most important OTUs to ampvis object
```{r}
imp_obj<-impTaxa
otutable_imp <- data.frame(OTU = rownames(phyloseq::otu_table(imp_obj)@.Data),
                       phyloseq::otu_table(imp_obj)@.Data,
                       phyloseq::tax_table(imp_obj)@.Data,
                       check.names = FALSE
                       )
imp_exp<- data.frame(phyloseq::sample_data(imp_obj),check.names = FALSE)
amp_imp<-amp_load(otutable_imp,imp_exp)
imp_heat<-amp_heatmap(amp_imp,tax_aggregate = "OTU",tax_show = 29,color_vector = c("white","red","darkred"),min_abundance = 1.0,group_by = "kmed_2",textmap = F)

```
### 38. Extracting LuMiCol OTU phylogeny
```{r}
lumicol_otus<-read.table("/Users/sdas/16S_data/all_run_240219/merged_060319/tables/final_tables/lumicol_otus_to_extract.txt",stringsAsFactors = FALSE)
lumicol<-lumicol_otus$V1
lumicol_singles<-prune_taxa(lumicol,ltx_rare_rel)
lumicol_singles
ape::write.tree(phy_tree(lumicol_singles), "phylogeny/lumicol_singles_tree.tre")
```
```
PAM colors
"#de2d26","#41ab5d","#0c2c84","#fe9929"
```

## Subset samples and analysis of according to pams into its phyloseq and ampvis2 objects
### 39. PAM1 as example, applied to all PAMs
```{r}

pam1<-subset_samples(ltx.rare.rel,kmed_2=="pam1")
pam1.obj<-pam1
pam1.otutable <- data.frame(OTU = rownames(phyloseq::otu_table(pam1.obj)@.Data),
                       phyloseq::otu_table(pam1.obj)@.Data,
                       phyloseq::tax_table(pam1.obj)@.Data,
                       check.names = FALSE
                       )
pam1.tree=phy_tree(pam1)
pam1.meta<- data.frame(phyloseq::sample_data(pam1.obj), check.names = FALSE)
amp_pam1<-amp_load(pam1.otutable,pam1.meta,tree = pam1.tree)
pam1_core<-amp_core(amp_pam1,tax_empty = "OTU",tax_aggregate = "OTU",abund_thrh = 1,detailed_output=T)
write.csv(pam1_core$data,"tables/pam1_incidence_abundance.csv")

pam_freq<-read.csv("tables/final_tables/pam1_core_incidence.csv",header = T)
head(pam_freq)

melted_pam_freq<-melt(pam_freq)

my_levels<-c("OTU_11",
"OTU_3",
 "OTU_6",
 "OTU_15",
 "OTU_30",
 "OTU_20",
 "OTU_8",
 "OTU_17",
 "OTU_4",
 "OTU_7",
 "OTU_39",
 "OTU_1",
 "OTU_41",
 "OTU_27",
 "OTU_34",
 "OTU_107",
 "OTU_57",
 "OTU_26",
 "OTU_42",
 "OTU_69",
 "OTU_21",
 "OTU_46",
 "OTU_2",
 "OTU_24",
 "OTU_78",
 "OTU_16",
 "OTU_49",
 "OTU_234",
 "OTU_63",
 "OTU_6768")

melted_pam_freq$taxa_info<- factor(taxa_info, levels=c("OTU_11","OTU_3","OTU_6","OTU_15","OTU_30","OTU_20","OTU_8","OTU_17",
 "OTU_4","OTU_7","OTU_39","OTU_1","OTU_41","OTU_27","OTU_34","OTU_107","OTU_57","OTU_26","OTU_42",
"OTU_69","OTU_21","OTU_46","OTU_2","OTU_24","OTU_78","OTU_16","OTU_49","OTU_234","OTU_63","OTU_6768"))

my_levels<-pam_freq$taxa_info<- as.factor(as.character(
pam_freq$taxa_info))

pam_core_plot<-ggplot(melted_pam_freq,aes(group=variable,fill=variable,x=taxa_info,y=value))+geom_bar(position="dodge",stat="identity")+geom_hline(yintercept = 50, linetype = 2,color="#006600")+theme(axis.text.x = element_text(size=12,angle = 90,face="italic"),axis.text.y = element_text(size = 15),axis.title = element_blank(),legend.position = "none",panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_fill_manual(values=c("grey40","#DE2D26"))+scale_x_discrete(limits=my_levels)+ylim(0,100)

```

```{r}
pam1_ra<-subset_samples(ltx.rare.rel.abs,kmed_2=="pam1")
pam1_imp_ra<-prune_taxa(imp2,pam1_ra)
pam1_imp_ra
pam1.reduced.otus<-as(otu_table(pam1_imp_ra),"matrix")
pam1.reduced.otus<-t(pam1.reduced.otus)
write.csv(pam1.reduced.otus,"tables/final_tables/pam1_reduced_otus.csv")
pam1_full_compare<-read.csv("tables/final_tables/enrichment_analysis/full_pam1_compare_v2.csv",header = T)

heads<-factor(heads, levels=c("OTU_11","OTU_3","OTU_6","OTU_15","OTU_30","OTU_20","OTU_8","OTU_17","OTU_4","OTU_7","OTU_39","OTU_1","OTU_41","OTU_27","OTU_34","OTU_107","OTU_57","OTU_26","OTU_42","OTU_69","OTU_21","OTU_46","OTU_2","OTU_24","OTU_78","OTU_16","OTU_49","OTU_234","OTU_63","OTU_6768"))

melted_full_compare<-melt(pam1_full_compare)
melted_full_compare
pd <- position_dodge(1) 
pam1_ab_plot<-ggplot(melted_full_compare,aes(fill=group,x=fct_inorder(variable),y=log10(value)))+geom_boxplot(outlier.size = 0.2,position = pd)+geom_hline(yintercept = 1, linetype = 2,color="black")+theme(axis.text.x = element_text(size=12,angle = 90,face="italic"),axis.text.y = element_text(size = 15),axis.title = element_blank(),legend.position = "none",panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_fill_manual(values=c("grey40","#de2d26"))
pam1_ab_plot
```
### 40. Statistical testing for PAM1 enrichment
```{r}
artool.model.pam1 <- art(value ~ group*variable, data = melted_full_compare)
anova(artool.model.pam1)
artool.model.lm.pam1 <- artlm(artool.model.pam1, "group:variable")
artool.model.lm.pam1.marginal <- emmeans(artool.model.lm.pam1, ~ group:variable)
pairs_otu_pam1<-contrast(artool.model.lm.pam1.marginal,method="pairwise",adjust="fdr")
write.csv(pairs_otu_pam1,"tables/final_tables/posthoc_pam1_full_otus.csv")
```


### 41. Bacterial load plotting
```{r}
copy_plot<-ggplot(df,aes(x=pam,y=log10(Cop16SPerMLBAL),fill=kmed_2))+geom_violin(scale = "width")+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title.x= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_y_continuous(expand = c(0, 0), limits = c(0, 7),breaks = c(0,1,2,3,4,5,6,7),name = "Bacterial cells/ml (Log10)")

bact_copy_plot<-ggplot(df,aes(x=reorder(LabSampleNumber, -Cop16SPerMLBAL),y=log10(Cop16SPerMLBAL),fill=kmed_2))+geom_bar(width = 0.9,stat = "identity")+theme(axis.text = element_text(size=20),axis.ticks.x = element_blank(),axis.text.x =element_blank(),panel.grid = element_blank())+xlab("BALF Samples (n=233)")+scale_y_continuous(expand = c(0, 0), limits = c(0, 7),breaks = c(0,1,2,3,4,5,6,7),name = "Bacterial cells/ml (Log10)")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))

bact_copy_plain<-ggplot(df,aes(x=reorder(LabSampleNumber, -Cop16SPerMLBAL),y=log10(Cop16SPerMLBAL)))+geom_bar(width = 0.9,stat = "identity")+theme(axis.text = element_text(size=20),axis.ticks.x = element_blank(),axis.text.x =element_blank(),panel.grid = element_blank(),axis.title = element_text(size = 20))+xlab("BALF Samples (n=233)")+scale_y_continuous(expand = c(0, 0), limits = c(0, 7),breaks = c(0,1,2,3,4,5,6,7),name = "Bacterial cells/ml (Log10)")
leveneTest(log10(Cop16SPerMLBAL)~kmed_2,data = df)
TukeyHSD(aov(log10(Cop16SPerMLBAL)~kmed_2,data = df))
```

### 42. Renyi and Hill index
```{r}
OTU1 = as(otu_table(ltx.rare.rel), "matrix")
# transpose if necessary
if(taxa_are_rows(ltx.rare.rel)){OTU1 <- t(OTU1)}
# Coerce to data.frame
OTUdf = as.data.frame(OTU1)

renyi_eric_nohill<-renyi(OTUdf, scales = c(0, 1, 2, Inf), hill = FALSE)
renyi_eric_hill<-renyi(OTUdf, scales = c(0, 1, 2, Inf), hill = TRUE)

write.csv(renyi_hill,"tables/renyi_hill.csv")
write.csv(renyi_nohill,"tables/renyi_nohill.csv")
#read modified table

renyi_hill<-read.csv("tables/renyi_hill_table.csv",header = T)
renyi_hill


renyi_hill$pielou<-renyi_hill$hill_1/log(renyi_hill$hill_0)
alpha_summary
pielou

hill_0<-ggplot(renyi_hill,aes(x=kmed_2,y=hill_0,fill=pam))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+ggtitle("Hill 0")
hill_0
hill_1<-ggplot(renyi_hill,aes(x=kmed_2,y=hill_1,fill=pam))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+ggtitle("Hill 1")

hill_2<-ggplot(renyi_hill,aes(x=kmed_2,y=hill_2,fill=pam),axis.text = element_text(size = 20))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+ggtitle("Hill 2")+ylim(0,200)

hill_Inf<-ggplot(renyi_hill,aes(x=kmed_2,y=hill_Inf,fill=pam),axis.text = element_text(size = 20))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20))+ggtitle("Hill Inf")

maxp<-ggplot(renyi_hill,aes(x=kmed_2,y=maxp,fill=pam),axis.text = element_text(size = 20))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+ggtitle("Max(p)")

pielou<-ggplot(renyi_hill,aes(x=kmed_2,y=pielou,fill=kmed_2),axis.text = element_text(size = 20))+geom_violin(scale = "width",trim=F)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.title= element_blank(),axis.text.x = element_blank(),axis.text = element_text(size = 20),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+ggtitle("Evenness")


#Statistics
plot(lm(hill_0~kmed_2,data=renyi_hill))
car::leveneTest(hill_0~kmed_2,data=renyi_hill)
TukeyHSD(aov(hill_0~kmed_2,data=renyi_hill))
kruskal.test(maxp~kmed_2,data=renyi_hill)
posthoc.kruskal.dunn.test(renyi_hill$maxp,renyi_hill$kmed_2,p.adjust.method = "BH")
renyi_hill0_summary<-summarySE(renyi_hill,measurevar = "hill_0",groupvars = "kmed_2")
renyi_hill1_summary<-summarySE(renyi_hill,measurevar = "hill_1",groupvars = "kmed_2")
renyi_hill2_summary<-summarySE(renyi_hill,measurevar = "hill_2",groupvars = "kmed_2")
renyi_hillinf_summary<-summarySE(renyi_hill,measurevar = "hill_Inf",groupvars = "kmed_2")
renyi_maxp_summary<-summarySE(renyi_hill,measurevar = "maxp",groupvars = "kmed_2")
renyi_maxp_summary

```

### 43. Distances for entire dataset
```{r}
bc.dist.bin<-phyloseq::distance(ltx.rare.rel,"bray",binary=T)
hn.dist<-phyloseq::distance(ltx.rare.rel,"horn")
uni.dist<-phyloseq::distance(ltx.rare.rel,"unifrac")
wuni.dist<-phyloseq::distance(ltx.rare.rel,"wunifrac")
bc.dist.bin

```

### 44. Distances of PAM based subsets
```{r}
pam1.set<-subset_samples(ltx.rare.rel,kmed_2=="pam1")
pam2.set<-subset_samples(ltx.rare.rel,kmed_2=="pam2")
pam3.set<-subset_samples(ltx.rare.rel,kmed_2=="pam3")
pam4.set<-subset_samples(ltx.rare.rel,kmed_2=="pam4")

pam1.bc.dist<-phyloseq::distance(pam1.set,"bray",binary=TRUE)
pam2.bc.dist<-phyloseq::distance(pam2.set,"bray",binary=TRUE)
pam3.bc.dist<-phyloseq::distance(pam3.set,"bray",binary=TRUE)
pam4.bc.dist<-phyloseq::distance(pam4.set,"bray",binary=TRUE)


pam1.hn.dist<-phyloseq::distance(pam1.set,"horn")
pam2.hn.dist<-phyloseq::distance(pam2.set,"horn")
pam3.hn.dist<-phyloseq::distance(pam3.set,"horn")
pam4.hn.dist<-phyloseq::distance(pam4.set,"horn")

pam1.uni.dist<-phyloseq::distance(pam1.set,"unifrac")
pam2.uni.dist<-phyloseq::distance(pam2.set,"unifrac")
pam3.uni.dist<-phyloseq::distance(pam3.set,"unifrac")
pam4.uni.dist<-phyloseq::distance(pam4.set,"unifrac")

pam1.wuni.dist<-phyloseq::distance(pam1.set,"wunifrac")
pam2.wuni.dist<-phyloseq::distance(pam2.set,"wunifrac")
pam3.wuni.dist<-phyloseq::distance(pam3.set,"wunifrac")
pam4.wuni.dist<-phyloseq::distance(pam4.set,"wunifrac")

melted.pam1.bc<-melt(as.matrix(pam1.bc.dist))
melted.pam2.bc<-melt(as.matrix(pam2.bc.dist))
melted.pam3.bc<-melt(as.matrix(pam3.bc.dist))
melted.pam4.bc<-melt(as.matrix(pam4.bc.dist))

write.csv(melted.pam1.bc,"tables/distances/melted.pam1.bc.csv")
write.csv(melted.pam2.bc,"tables/distances/melted.pam2.bc.csv")
write.csv(melted.pam3.bc,"tables/distances/melted.pam3.bc.csv")
write.csv(melted.pam4.bc,"tables/distances/melted.pam4.bc.csv")

pam.all.bc<-read.csv("tables/pam.bray.csv")


melted.pam1.hn<-melt(as.matrix(pam1.hn.dist))
melted.pam2.hn<-melt(as.matrix(pam2.hn.dist))
melted.pam3.hn<-melt(as.matrix(pam3.hn.dist))
melted.pam4.hn<-melt(as.matrix(pam4.hn.dist))

write.csv(melted.pam1.hn,"tables/distances/melted.pam1.hn.csv")
write.csv(melted.pam2.hn,"tables/distances/melted.pam2.hn.csv")
write.csv(melted.pam3.hn,"tables/distances/melted.pam3.hn.csv")
write.csv(melted.pam4.hn,"tables/distances/melted.pam4.hn.csv")

pam.all.hn<-read.csv("tables/pam.horn.csv")


melted.pam1.uni<-melt(as.matrix(pam1.uni.dist))
melted.pam2.uni<-melt(as.matrix(pam2.uni.dist))
melted.pam3.uni<-melt(as.matrix(pam3.uni.dist))
melted.pam4.uni<-melt(as.matrix(pam4.uni.dist))

write.csv(melted.pam1.uni,"tables/distances/melted.pam1.uni.csv")
write.csv(melted.pam2.uni,"tables/distances/melted.pam2.uni.csv")
write.csv(melted.pam3.uni,"tables/distances/melted.pam3.uni.csv")
write.csv(melted.pam4.uni,"tables/distances/melted.pam4.uni.csv")

pam.all.uni<-read.csv("tables/pam.unifrac.csv")


melted.pam1.wuni<-melt(as.matrix(pam1.wuni.dist))
melted.pam2.wuni<-melt(as.matrix(pam2.wuni.dist))
melted.pam3.wuni<-melt(as.matrix(pam3.wuni.dist))
melted.pam4.wuni<-melt(as.matrix(pam4.wuni.dist))

write.csv(melted.pam1.wuni,"tables/distances/melted.pam1.wuni.csv")
write.csv(melted.pam2.wuni,"tables/distances/melted.pam2.wuni.csv")
write.csv(melted.pam3.wuni,"tables/distances/melted.pam3.wuni.csv")
write.csv(melted.pam4.wuni,"tables/distances/melted.pam4.wuni.csv")

pam.all.wuni<-read.csv("tables/pam.wuni.csv")


melted.all.bc.pam<-melt(pam.all.bc)
melted.all.hn.pam<-melt(pam.all.hn)
melted.all.uni.pam<-melt(pam.all.uni)
melted.all.wuni.pam<-melt(pam.all.wuni)

summarySE(melted.all.bc.pam,groupvars = "variable",measurevar = "value")

sd(pam.all.wuni$pam4,na.rm=T)

bray.pam.plot<-ggplot(melted.all.bc.pam,aes(variable,value))+geom_violin(aes(fill=variable),scale = "width",trim = FALSE)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.text = element_text(size=20),axis.title.x = element_blank(),axis.text.x = element_blank(),axis.title = element_blank(),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_y_continuous(limits = c(0,1))

horn.pam.plot<-ggplot(melted.all.hn.pam,aes(variable,value))+geom_violin(aes(fill=variable),scale = "width",trim = FALSE)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.text = element_text(size=20),axis.title.x = element_blank(),axis.text.x = element_blank(),axis.title = element_blank(),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_y_continuous(limits = c(0,1))

unifrac.pam.plot<-ggplot(melted.all.uni.pam,aes(variable,value))+geom_violin(aes(fill=variable),scale = "width",trim = FALSE)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.text = element_text(size=20),axis.title.x = element_blank(),axis.text.x = element_blank(),axis.title = element_blank(),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_y_continuous(limits = c(0,1))

wunifrac.pam.plot<-ggplot(melted.all.wuni.pam,aes(variable,value))+geom_violin(aes(fill=variable),scale = "width",trim = FALSE)+geom_boxplot(width=0.1,fill="white")+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+theme(axis.text = element_text(size=20),axis.title.x = element_blank(),axis.text.x = element_blank(),axis.title = element_blank(),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),
panel.background = element_blank())+scale_y_continuous(limits = c(0,1))

ggarrange(bray.pam.plot,horn.pam.plot,unifrac.pam.plot,wunifrac.pam.plot,
          ncol = 4, nrow = 1, label.x = 0.33, align = "v",
          common.legend = TRUE, legend = "right")

```
### 45. PERMANOVAs
```{r}
adonis(bc.dist.bin~pam,data = df,permutations = 10000)
adonis(hn.dist~ pam,data = df,permutations = 10000)
adonis(uni.dist ~ pam,data = df,permutations = 10000)
adonis(wuni.dist ~ pam,data = df,permutations = 10000)
```

### 46. Paired adonis for entire datasets: 
```{r}
require(pairwiseAdonis)
paired.bc.kmed<-pairwise.adonis(bc.dist.bin,df$pam,p.adjust.m = "BH")
paired.hn.kmed<-pairwise.adonis(hn.dist,df$pam,p.adjust.m = "BH")
paired.uni.kmed<-pairwise.adonis(uni.dist,df$pam,p.adjust.m = "BH")
paired.wuni.kmed<-pairwise.adonis(wuni.dist,df$pam,p.adjust.m = "BH")
```
### 47. Summary of paired adonis
```{r}
paired.bc.kmed$distance<-"bray"
paired.hn.kmed$distance<-"horn"
paired.uni.kmed$distance<-"unifrac"
paired.wuni.kmed$distance<-"wunifrac"
paired.adonis.summary<-rbind(paired.bc.kmed,paired.hn.kmed,paired.uni.kmed,paired.wuni.kmed)
paired.adonis.summary
write.csv(paired.adonis.summary,"tables/final_tables/paired_adonis_summary.csv")
```
### 48. Virome of lung transplant
```{r}
#Virome vs Immunosuppresants
summary(lm(log10(AnelloAllPerMLBAL)~conc,data=df))

#Virome vs Rejection
summary(lm(log10(AnelloAllPerMLBAL)~clad,data=df))
plot(log10(df$AnelloAllPerMLBAL)~df.july2019$FEV1PercentOfBaseline)
table(is.na(df$FEV1PercentBaseline))

copy_virus_lm<-ggplot(df,aes(AnelloAllPerMLBAL,Cop16SPerMLBAL))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
virus_pred_lm<-ggplot(df,aes(log10(AnelloAllPerMLBAL),PrednisoneDosis))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
virus_tacro_lm<-ggplot(df.july2019,aes(log10(AnelloAllPerMLBAL),TacrolimusConc))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
ggarrange(virus_pred_lm,virus_tacro_lm,ncol = 2, nrow = 1, label.x = 0.33, align = "v")
```



```
Gene expression variables:
COXcrude+MRC1crude+DCSIGNcrude+TNFcrude+IDOcrude+IL10crude+IL1RNcrude+MMP12crude+TIMP1crude+FN1crude+PDGFDcrude+COL6A2crude+MMP7crude+MMP9crude+CHI3L1crude+THBS1crude+IGF1crude+IGFBP2crude+SPP1crude+LY96crude+TLR5crude+CAMPcrude+NLRP3crude+IFITM2crude+MAVScrude+TLR7crude+S100A12crude+TLR2crude+TLR3crude+IFNLR1crude+RSAD2crude

Other variables:
AnelloAllPerMLBAL+TTMVperMLBAL+TTMDVperMLBAL+BALMacroNumber+BALNeutroNumber+BALAllLymphoNumber+BloodNeutroNumber+PrednisoneDosis+CurrentBactInfection+TimeWindows5+DiagnosisTx+CurrentATB

```

### 49. Gene expression ploting : kmed2 is PAM
```{r}
#COX
cox_pam<-ggplot(df.eric,aes(kmed_2,log10(COXcrude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#TNF
tnf_pam<-ggplot(df.eric,aes(kmed_2,log10(TNFcrude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))
#IDO
ido_pam<-ggplot(df.eric,aes(kmed_2,log10(IDOcrude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#IL1RN
il1_pam<-ggplot(df.infect,aes(kmed_2,log10(IL1RNcrude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#TIMP1
timp_pam<-ggplot(df.eric,aes(kmed_2,log10(TIMP1crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#PDGFD
pgd_pam<-ggplot(df.eric,aes(kmed_2,log10(PDGFDcrude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#IFNLR1
ifnlr_pam<-ggplot(df.eric,aes(kmed_2,log10(IFNLR1crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#MMP9
mmp9_pam<-ggplot(df.eric,aes(kmed_2,log10(MMP9crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#LY96
ly_pam<-ggplot(df.eric,aes(kmed_2,log10(LY96crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#TLR5
tlr5_pam<-ggplot(df.eric,aes(kmed_2,log10(TLR5crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#NLRP3
nlrp_pam<-ggplot(df.eric,aes(kmed_2,log10(NLRP3crude)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#TLR2
tlr2_pam<-ggplot(df.nona,aes(kmed,TLR2))+geom_violin(aes(fill=kmed),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))
```
### 50. Immunocompetence and physiology
```{r}
bal_cells_pam<-ggplot(df,aes(PamMicrobLevel2,log10(TotalBALCellsPerML)))+geom_violin(aes(fill=PamMicrobLevel2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=3,angle=90)+scale_y_continuous(limits = c(0,7),breaks = c(1,2,3,4,5,6,7))
bal_cells_pam

bal_macro_pam<-ggplot(df,aes(PamMicrobLevel2,log10(BALMacroNumber)))+geom_violin(aes(fill=PamMicrobLevel2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=3,angle=90)+scale_y_continuous(limits = c(0,7),breaks = c(1,2,3,4,5,6,7))

bal_pmn_pam<-ggplot(df,aes(PamMicrobLevel2,log10(BALNeutroNumber)))+geom_violin(aes(fill=PamMicrobLevel2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=2,angle=90)+scale_y_continuous(limits = c(0,7),breaks = c(1,2,3,4,5,6,7))


blood_bcell_pam<-ggplot(df,aes(PamMicrobLevel2,BloodBcellNumber))+geom_violin(aes(fill=PamMicrobLevel2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=4,angle=90)


lungfunc_pam<-ggplot(df,aes(PamMicrobLevel2,FEV1PercentBaseline))+geom_violin(aes(fill=PamMicrobLevel2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+geom_hline(yintercept = 80, alpha = 0.5, linetype = 2)+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=20,angle=90)+scale_y_continuous(limits = c(0,120),breaks = c(0,30,60,90,120))



#Statistics
pmn<-data.frame(df.sept2019$PamMicrobLevel2,log10(df.sept2019$BALNeutroNumber))
colnames(pmn)<-c("pam","log_pmn")
str(pmn)
df.pmn<-subset(pmn,!is.na(log_pmn))
df.pmn<-subset(df.pmn,log_pmn!="-Inf")
df.pmn
leveneTest(BloodBcellNumber~PamMicrobLevel2,data=df.sept2019)
kruskal.test(ClinicalInfection~PamMicrobLevel2,data=df.sept2019)

inf_table<-table(df.sept2019$ClinicalInfection,df.sept2019$PamMicrobLevel2)
contrasts(inf_table,contrasts = T)
posthoc.kruskal.dunn.test(df.sept2019$ClinicalInfection,df.sept2019$PamMicrobLevel2,p.adjust.method = "BH")

TukeyHSD(aov(BloodBcellNumber~PamMicrobLevel2,data=df.sept2019))
mymodel<-glm(ClinicalInfection~PamMicrobLevel2,data=df.sept2019, family="binomial")
summary(mymodel)

```

### 51. Immunosuppresants
```{r}
pred_pam<-ggplot(df.july2019,aes(kmed_2,PrednisoneDosis))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits = c(0,80),breaks = c(0,20,40,60,80))
tacro_pam<-ggplot(df.july2019,aes(kmed_2,TacrolimusConc))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits = c(0,40),breaks = c(0,10,20,30,40))

tacro_pam<-ggplot(df.july2019,aes(kmed_2,TacrolimusConc))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits = c(0,40),breaks = c(0,10,20,30,40))

```
### 52. Current ATBs
```{r}
fisher.test(df.sept2019$CurrentABXnumber,df.sept2019$PamMicrobLevel2,simulate.p.value=TRUE,B=1e7)
fisher.test(df.sept2019$ClinicalInfection,df.sept2019$PamMicrobLevel2,simulate.p.value=TRUE,B=1e7)
atb_data$CurrentABXnumber<-as.factor(atb_data$CurrentABXnumber)
summary(glm(CurrentABXnumber~PamMicrobLevel2,data=df.sept2019,family = "poisson"))
```

### 53. Longitudinal analysis
```{r}
virome_data<-data.frame(df.july2019$AnelloAllPerMLBAL,df.july2019$TTV1to5perMLBAL,df.july2019$TTMVperMLBAL,df.july2019$TTMDVperMLBAL,as.factor(df.july2019$TimeWindows5))
head(virome_data)
colnames(virome_data)<-c("anello","alpha","beta","gamma","time")
melted_virus<-melt(virome_data)
melted_virus$log_value<-log10(melted_virus$value)
colnames(melted_virus)<-c("time","virus","value","log")

melted_virus<-na.omit(melted_virus)
melted_virus_summary<-summarySE(melted_virus,measurevar="log",groupvar=c("virus","time"))
melted_virus_summary

virome_time_plot<-ggplot(melted_virus_summary,aes(as.factor(time),log,group=virus))+geom_errorbar(aes(ymin=log-se, ymax=log+se,colour=virus),size=1,width=.5, position=pd)+geom_path(aes(color=virus,linetype=virus),position=pd,size=1.2)+geom_point(aes(color=virus),position = pd,size=3)+theme(axis.text = element_text(size = 20),axis.title.x = element_blank())+scale_y_continuous(limits = c(0,6),breaks = c(0,2,4,6))

tac_plot<-ggplot(df.july2019,aes(x=TimeWindows5,y=TacrolimusConc))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 20),axis.title.x = element_blank())+stat_n_text(y.pos=30,size=5)
tac_plot
pred_plot<-ggplot(df.july2019,aes(x=TimeWindows5,y=PrednisoneDosis))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 20),axis.title.x = element_blank())+stat_n_text(y.pos=60,size=3)+ylim(0,75)

anello_time<-ggplot(df.july2019,aes(x=TimeWindows5,y=log10(AnelloAllPerMLBAL)))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_y_continuous(limits = c(0,10),breaks = c(0,2,4,6,8,10))+stat_n_text(y.pos=1,size=4,angle = 90)

alphavir<-ggplot(df.july2019,aes(x=TimeWindows5,y=log10(TTV1to5perMLBAL)))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_y_continuous(limits = c(0,10),breaks = c(0,2,4,6,8,10))+stat_n_text(y.pos=1,size=4,angle = 90)

betavir<-ggplot(df.july2019,aes(x=TimeWindows5,y=log10(TTMVperMLBAL)))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_y_continuous(limits = c(0,10),breaks = c(0,2,4,6,8,10))+stat_n_text(y.pos=1,size=4,angle = 90)

gammavir<-ggplot(df.july2019,aes(x=TimeWindows5,y=log10(TTMDVperMLBAL)))+geom_violin(fill="grey")+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_y_continuous(limits = c(0,10),breaks = c(0,2,4,6,8,10))+stat_n_text(y.pos=1,size=4,angle = 90)

```
### 54. Markov chain analysis
```{r}
statenames= c("pam1", "pam2", "pam3","pam4")
pam_trans<-new("markovchain",state=statenames,
transitionMatrix = matrix(c(0.625,0.046875,0.25,0.078125,
0.666666667,0.111111111,0.222222222,0,
0.393939394,0.090909091,0.424242424,0.090909091,
0.444444444,0,0.333333333,0.222222222), nrow=4,byrow = TRUE, dimnames = list(statenames, statenames)))

summary(pam_trans)
initial<-c(0.25,0.25,0.25,0.25)
after1<-initial*(pam_trans)
after1
after5<-initial*(pam_trans^2)
after5
# Initiate probability vectors
pam1prob<- c()
pam2prob<- c()
pam3prob<- c()
pam4prob<-c()
# Calculate probabilities for 24 steps.
for(k in 1:5){
  nsteps <- initial*pam_trans^k
  pam1prob[k] <- nsteps[1,1]
  pam2prob[k] <- nsteps[1,2]
  pam3prob[k] <- nsteps[1,3]
  pam4prob[k] <- nsteps[1,4]
  }

# Make dataframes and merge them
pam1prob <- as.data.frame(pam1prob)
 pam1prob$Group <- 'pam1'
 pam1prob$Iter <- 1:5
names( pam1prob)[1] <- 'Value'

pam2prob <- as.data.frame(pam1prob)
 pam2prob$Group <- 'pam2'
 pam2prob$Iter <- 1:5
names( pam2prob)[1] <- 'Value'


pam3prob <- as.data.frame(pam3prob)
 pam3prob$Group <- 'pam3'
 pam3prob$Iter <- 1:5
names( pam3prob)[1] <- 'Value'


pam4prob <- as.data.frame(pam4prob)
 pam4prob$Group <- 'pam4'
 pam4prob$Iter <- 1:5
names( pam4prob)[1] <- 'Value'

steps <- rbind(pam1prob,pam2prob,pam3prob,pam4prob)
steps
# Plot the probabilities using ggplot
ggplot(steps, aes(x = Iter, y = Value, col = Group))+
  geom_line() +geom_point()+
  xlab('Chain Step') +
  ylab('Probability') +
  ggtitle('5 Chain Probability Prediction')


trans_data<-read.csv("tables/hmm_data.csv",header = T)
trans_data
sequence<-read.table("tables/pam_sequence.txt")
seq<-sequence$V1
seq
verifyMarkovProperty(seq,verbose = T)
assessStationarity(seq,nblocks = 4)
steadyStates(pam_trans)
absorbingStates(pam_trans)
transientStates(pam_trans)
communicatingClasses(pam_trans)
period(pam_trans)

mcFit<-markovchainFit(data=seq,method = "bootstrap",nboot = 100)
mcFit$estimate
predict(mcFit$estimate, newdata=c("pam1","pam4"),n.ahead=4)
#direct conversion
myMc<-as(pam_trans, "markovchain")
myMc
#example of summary
summary(simpleMc)
## Not run: plot(simpleMc)

markov_actual<-plot(pam_trans)
markov_stats<-plot(mcFit$estimate)
round(mcFit$estimate@transitionMatrix,2)

stochastic_matrix_to_plot <- round(as(pam_trans, "matrix"),2)
plotmat(stochastic_matrix_to_plot,relsize = 0.7)
```
### 55. Virome plots
```{r}
annello_pam<-ggplot(df.july2019$,aes(,log10(AnelloAllPerMLBAL)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=1,angle=90)+scale_y_continuous(limits = c(0,8),breaks = c(0,2,4,6,8))

ttv_pam<-ggplot(df.july2019,aes(kmed_2,log10(TTV1to5perMLBAL)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text = element_text(size = 15),axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=1,angle=90)+scale_y_continuous(limits = c(0,8),breaks = c(0,2,4,6,8))

ttmv_pam<-ggplot(df.july2019,aes(kmed_2,log10(TTMVperMLBAL)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text.x = element_blank(),axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=1,angle=90)+scale_y_continuous(limits = c(0,8),breaks = c(0,2,4,6,8))

ttmdv_pam<-ggplot(df.july2019,aes(kmed_2,log10(TTMDVperMLBAL)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.15,fill="white")+theme(axis.text.x = element_blank(),axis.text = element_text(size = 15),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+stat_n_text(y.pos=1,angle=90)+scale_y_continuous(limits = c(0,8),breaks = c(0,2,4,6,8))

car::leveneTest(log10(AnelloAllPerMLBAL)~CLADonDayOfBALsampling,data=df.july2019)
shapiro.test(log10(df.virus$anello))
df.july2019$TTV1to5perMLBAL
df.virus<-data.frame(df.july2019$AnelloAllPerMLBAL,df.july2019$TTV1to5perMLBAL,df.july2019$TTMVperMLBAL,df.july2019$TTMDVperMLBAL,df.july2019$CLADonDayOfBALsampling)
df.virus<-na.omit(df.virus)
colnames(df.virus)<-c("anello","alpha","beta","gamma","clad")
pairwise.wilcox.test(log10(df.virus$anello),df.virus$clad,p.adjust.method = "BH",paired = FALSE)
summary(df.virus)

alpha_plot<-ggplot(df.virus,aes(y=log10(alpha),x=clad))+geom_boxplot()+scale_y_continuous(limits = c(0,8))+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())
beta_plot<-ggplot(df.virus,aes(y=log10(beta),x=clad))+geom_boxplot()+scale_y_continuous(limits = c(0,8))+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())
gamma_plot<-ggplot(df.virus,aes(y=log10(gamma),x=clad))+geom_boxplot()+scale_y_continuous(limits = c(0,8))+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())

anello_plot<-ggplot(df.virus,aes(y=log10(anello),x=clad))+geom_boxplot()+scale_y_continuous(limits = c(0,8))+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())

anello_plot
df.sept2019$TTV1to5perMLBAL
total_virus<-subset(df.sept2019,select=AnelloAllPerMLBAL:TTMDVperMLBAL)

virus_stats<-total_virus<-subset(df.sept2019,select=TTV1to5perMLBAL:TTMDVperMLBAL)
boxplot(log10(virus_stats),las=2)
virus_stats_melted<-melt(virus_stats)

leveneTest(log10(value)~variable,data=virus_stats_melted)

posthoc.kruskal.dunn.test(virus_stats_melted$value, virus_stats_melted$variable,p.adjust.method = "BH")
virus_summary<-summarySE(total_virus_melted,measurevar = "value",groupvar="variable" )
virus_summary

ggplot(df.sept2019,aes(x=reorder(SOP100.number, -AnelloAllPerMLBAL),y=log10(AnelloAllPerMLBAL)))+geom_bar(width = 0.9,stat = "identity")+theme(axis.text = element_text(size=20),axis.ticks.x = element_blank(),axis.text.x =element_blank(),panel.grid = element_blank(),axis.title = element_text(size = 20))


summary(lm(log10(TotalBALCellsPerML)~log10(AnelloAllPerMLBAL),data = df.sept2019))

ggplot(df.sept2019,aes(log10(AnelloAllPerMLBAL),log10(TotalBALCellsPerML)))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))+ylim(0,8)


total_virus_plot<-ggplot(df.may2019,aes(x=reorder(Sample_SD, -AnelloAllPerMLBAL),y=log10(AnelloAllPerMLBAL)))+geom_bar(width = 0.9,stat = "identity")+theme(axis.text = element_text(size=20),axis.ticks.x = element_blank(),axis.text.x =element_blank(),panel.grid = element_blank(),axis.title = element_text(size = 20))+scale_y_continuous(expand = c(0, 0),limits = c(0, 9),breaks = c(0,1,2,3,4,5,6,7,8))

```

## Random Forest application 
### 56. Setting up data
```{r}
predictors <- read.table("tables/expression_meta_median_trans.csv", sep=",",header=T, row.names=1)  
metadata <- read.table("tables/metadata_RF", sep="\t", header=T, row.names=1, stringsAsFactors=TRUE, comment.char="")
response<-metadata$pam

rf.data <- data.frame(, predictors)

```


## Optimization of random forest parameters 

To optimize the randomly selected predictors at every step i.e. mtry parameter of random forest. This was done with the help of the "carat" package in R. The algorithms were first trained with the random control dataset (created by the algorithm itself). The model is trained with the actual data (pneumotype and gene expression). 

mtry: Number of variable is randomly collected to be sampled at each split time.

ntree: Number of branches will grow after each time split.

tutorial:https://rstudio-pubs-static.s3.amazonaws.com/389752_a0e0b14d14ea40ba8a7729fbd59cd5b5.html

### 57. Prediction of pneumotypes using host gene expression using classification model

**Optimization of splits per try (mtry)**

The first step was to make a grid search for optimizing mtry i.e. splits per try, which is basically the number of variables randomly selected gene predictors to be sampled at each iteration. For this we created a control function with search method 'grid' that performs 10 folds cross-validation and repeats the step 3 times. 

```{r}
control <- trainControl(method='repeatedcv', 
                        number=10, 
                        repeats=3, 
                        search='grid')

tunegrid <- expand.grid(.mtry = (5:20)) 

rf_gridsearch <- train(response~ ., 
                       data = rf.data,
                       method = 'rf',
                       metric = 'Accuracy',
                       tuneGrid = tunegrid)
plot(rf_gridsearch)

```
**Results**

From here, we could see that mtry= 5 gave the best accuracy results. 

**Optimization of number of decision trees (ntrees)**

The next step was to manually tune for optimize the number of decision trees (ntrees) needed for good prediction, using mtry=5 and applying a loop to stepwise vary the ntrees. 

```{r}
control <- trainControl(method="repeatedcv", number=10, repeats=3, search="grid")
tunegrid <- expand.grid(.mtry=5)
modellist <- list()
for (ntree in c(500,1000,2000, 3000, 4000, 5000)) {
	set.seed(123) 
	fit <- train(response~., data= rf.data, method="rf", tuneGrid=tunegrid, trControl=control, ntree=ntree)
	key <- toString(ntree)
	modellist[[key]] <- fit
}

results <- resamples(modellist)
dotplot(results) 

```
**Results**

Here we used mtry= 5 and varied the ntree= 500 to 5000 and saw little difference in Accuracy or sensivitiy (Kappa) amongst all. Hence, we decided to keep the minimum number of trees i.e 500, which is also the default in the "randomForest" function. 

### 58. Prediction of bacterial and viral numbers by host gene expression by using regression model. 

For regression models, a random search was performed for optimizing mtry i.e. number of variable is randomly collected to be sampled at each split time. Unlike the classification model (section 57), grid search cannot be applied here. First the model was trained using control data created by the algorithm itself and then the actual data (bacteria/virus counts and gene expression) with parameter "tuneLength" (here given as 30, which randomy generates given number of mtry values) was used.

Starting value of mtry used was taken as the square root of number of columns present in the dataset. With each step an increment of 3 decision trees (ntree) were used. 

**Optimization of splits per try (mtry)**
```{r}
mtry <- sqrt(ncol(rf.data))


ntree <- 3

control <- trainControl(method='repeatedcv', 
                        number=10, 
                        repeats=3,
                        search = 'random',allowParallel = T)


set.seed(1)
rf_random <- train(response~ .,
                   data =  rf.data,
                   method = 'rf',
                   tuneLength  = 30, 
                   trControl = control)
plot(rf_random)

```
**Results**

From these steps, we could observe the mtry value that provided with the lowest Root Mean Square Error (RMSE) values indicating better accuracy. In case of bacterial numbers, mtry = 21 was the best value and it was also used by the random forest algorithm as default in this case when used without setting mtry parameter. Although the differences were not large after 10 considering the order of magnitude of 0.01. Hence, we chose mtry = 21 to use for this model. 

Unlike bacterial numbers, for viral numbers we couldn't deduce any best mtry value as the differences were minute and were of the order of magnitude 0.005. We chose mtry= 10, since it was also used by the random forest algorithm as default. 

**Optimization of number of decision trees (ntrees)**

Similar to the classification model in section A, the next step was to manually tune for optimize the number of decision trees (ntrees). We optimized the number of decision trees (ntrees) by appyling a loop to vary different number of decision trees (ntrees) needed for good prediction. However, the parameters that provide us with the accuracy were different, as discussed below.

```{r}
control <- trainControl(method="repeatedcv", number=10, repeats=3, search="grid")
tunegrid <- expand.grid(.mtry=5)
modellist <- list()
for (ntree in c(500,1000,2000, 3000, 4000, 5000)) {
	set.seed(123) 
	fit <- train(response~., data= rf.data, method="rf", tuneGrid=tunegrid, trControl=control, ntree=ntree)
	key <- toString(ntree)
	modellist[[key]] <- fit
}

results <- resamples(modellist)
dotplot(results) 

```
**Results**

Further using the mtry = 21 for bacterial numbers, we optimized the number of decision trees (ntrees) needed for good prediction. This generated values for Mean Absolute Error (MAE), RMSE and regression value i.e. R2 for all ntrees used. We observed little difference in RMSE and R2 amongst all. Hence, we decided to use the minimum number of trees i.e 1000.

Using the mtry = 10 for viral numbers, we optimized the number of decision trees (ntrees) needed for good prediction similar to bacterial numbers. Here as well, we observed little difference in RMSE and R2 amongst all. Hence, we decided to use the minimum number of trees i.e 500.


### 59. Running the actual model for gene exp vs pams
```{r}
set.seed(123)
rf_lung<-randomForest(response~., data = rf.data)
print(rf_lung)
boruta_lung<-Boruta(response~.,data = rf.data, ntree =500,mtry=5,mcAdj = TRUE,
pValue=0.01)
plot(boruta_lung,xlab="",las=2,colCode = c("#009E73", "yellow", "#D55E00", "#999999"))
levels(rf.data$response)
str(rf.data)
```
### 60. Making detailed comparison with selected features
```{r}
#Set 1

#IFNLR1
ifn_pam<-ggplot(rf.data,aes(response,log2(IFNLR1)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

ifn_pam

#MRC1
mrc_pam<-ggplot(rf.data,aes(response,log2(MRC1)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

mrc_pam

#IL10
il10_pam<-ggplot(rf.data,aes(response,log2(IL10)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

il10_pam

#IL1RN
il1rn_pam<-ggplot(rf.data,aes(response,log2(IL1RN )))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

il1rn_pam

#LY96
ly96_pam<-ggplot(rf.data,aes(response,log2(LY96 )))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

ly96_pam


#IDO1
ido_pam<-ggplot(rf.data,aes(response,log2(IDO)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

ido_pam

#PDGFD
pg_pam<-ggplot(rf.data,aes(response,log2(PDGFD)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text = element_text(size=20),axis.text.x = element_blank(),axis.title.x = element_blank(),axis.line = element_line(colour = "black"),panel.grid.major = element_blank(), panel.grid.minor = element_blank(),panel.border = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))+scale_y_continuous(limits=c(-9,3),breaks = c(-9,-6,-3,0,3))

#Set 2

#DC-SIGN
dc_pam<-ggplot(rf.data,aes(response,log2(DCSIGNMedTr)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))
dc_pam

#TNFa
tnf_pam<-ggplot(rf.data,aes(response,log2(TNFMedTr)))+geom_violin(aes(fill=response),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank())+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))

ggarrange(ifn_pam,mrc_pam,il10_pam,il1rn_pam,ly96_pam,ido_pam,
          ncol = 3, nrow = 2,
          common.legend = TRUE, legend = "right")
```
### 61. Statistical testing of the selected features
```{r}
leveneTest(IDO~response,data = rf.data,center=mean)
TukeyHSD(aov(IDO~response,data = rf.data))
posthoc.kruskal.dunn.test(LY96~response,data = rf.data,p.adjust.method = "BH")
```

### 62. Running regression model with copy number vs gene exp 
```{r}
copy<-log10(metadata$Cop16SPerMLBAL)
exp_copy_data<-data.frame(copy, predictors)
str(exp_copy_data)
set.seed(123)
rf_lung_copy<-randomForest(copy~., data = exp_copy_data,ntree=1000,mtry=21)
rf_lung_copy
set.seed(123)
boruta_lung_copy_geneexp<-Boruta(copy~., data = exp_copy_data,ntree=1000,mtry=21,mcAdj = TRUE,pValue=0.01)
plot(boruta_lung_copy_geneexp,las=2,xlab="")
boruta_lung_copy_geneexp$finalDecision

```
### 63. Statistical tests for gene exp and copy number
```{r}
# Stepwise Regression
library(MASS)
fit <- lm(copy~PDGFD+IFNLR1+TNF+IL10+LY96,data=exp_copy_data)
step <- stepAIC(fit, direction="both")
step$anova# display results
summary(step)
# All Subsets Regression
library(leaps)

fit <- leaps::regsubsets(copy~PDGFD+IFNLR1+TNF+IL10+LY96,data=exp_copy_data,nbest=10)
# view results 
plot(fit,scale="adjr2")
fit

copy_pdgfd_lm<-ggplot(exp_copy_data,aes(copy,log2(PDGFD)))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
copy_pdgfd_lm

copy_ifn_lm<-ggplot(exp_copy_data,aes(copy,log2(IFNLR1)))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
copy_ifn_lm


#PDGFD_pam
pd_pam<-ggplot(df.july2019,aes(kmed_2,log2(PDGFD)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank(),axis.text = element_text(size = 20))+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))
pd_pam

#IFNLR1_pam
ifnlr_pam<-ggplot(df.july2019,aes(kmed_2,log2(IFNLR1)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank(),axis.text = element_text(size = 20))+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))
ifnlr_pam


leveneTest(IFNLR1~kmed_2,data = df.july2019,center=mean)
summary(aov(IDO~response,data = rf.data))
posthoc.kruskal.dunn.test(IFNLR1~kmed_2,data =df.july2019,p.adjust.method = "BH")

```


### 64. Running regression model with virus number vs gene exp
```{r}
virus<-log10(metadata$AnelloAllPerMLBAL)
exp_virus_data<-data.frame(virus, predictors)
exp_virus_data<-na.omit(exp_virus_data)
str(exp_virus_data)
set.seed(123)
rf_lung_virus<-randomForest(virus~., data = exp_virus_data)
rf_lung_virus
set.seed(123)
boruta_lung_virus_geneexp<-Boruta(virus~., data = exp_virus_data,mcAdj = TRUE,pValue=0.01)
boruta_lung_virus_geneexp
TentativeRoughFix(boruta_lung_virus_geneexp)
par(mar=c(7, 5, 3, 1))
plot(boruta_lung_virus_geneexp,las=2,xlab="",colCode = c("#009E73", "yellow", "#D55E00", "#999999"))
```

### 65. Statistical tests for gene exp and anellovirus copies 
```{r}
# Stepwise Regression
library(MASS)
fit <- lm(virus~IFITM2+IGF1+RSAD2+TLR3,data=exp_virus_data)
summary(fit)
step <- stepAIC(fit, direction="both")
step$anova # display results
summary(step)
# All Subsets Regression
library(leaps)

fit <- regsubsets(virus~IFITM2+IGF1+RSAD2+TLR3,data=exp_virus_data,nbest=10)
# view results 
plot(fit,scale="adjr2")

metadata$AnelloAllPerMLBAL

summary(lm(virus~TLR3,data=exp_virus_data))

virus_tlr3_lm<-ggplot(exp_virus_data,aes(virus,log2(TLR3)))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))
virus_tlr3_lm

virus_ifitm_lm<-ggplot(exp_virus_data,aes(virus,log2(IFITM2)))+geom_point()+geom_smooth(method = "lm",formula = y ~ log(x))+theme(axis.text = element_text(size = 20))

virus_ifitm_lm
#TLR3 pam
tlr3_pam<-ggplot(df.july2019,aes(kmed_2,log2(TLR3)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank(),axis.text = element_text(size = 20))+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))
tlr3_pam

tlr3_pam<-ggplot(df.july2019,aes(kmed_2,log2(TLR3)))+geom_violin(aes(fill=kmed_2),alpha=0.8)+geom_boxplot(width=0.1,fill="white")+theme(axis.text.x = element_blank(),axis.title.x = element_blank(),axis.text = element_text(size = 20))+scale_fill_manual(values=c("#de2d26","#41ab5d","#0c2c84","#fe9929"))

leveneTest(IFITM2~kmed_2,data = df.july2019,center=mean)
posthoc.kruskal.dunn.test(IGF1~kmed_2,data = df.july2019,p.adjust.method = "BH")

summary(aov(IFITM2~kmed_2,data = df.july2019))
```



### 66. Using BORUTA package
```{r}
boruta_lung_copy<-Boruta(response~.,data = rf.data,mcAdj = TRUE,
pValue=0.01)

boruta_lung_copy

boruta_lung_bacteria<-Boruta(pams~.,data = rf_lung_bacteria_data,mcAdj = TRUE,
pValue=0.05,ntree = 1000,mtry=6)
str(rf_lung_filtered_data)
rf_lung_bacteria_data

set.seed(123)
rf_lung_bacteria<-randomForest(pams_filtered~., data = rf_lung_filtered_data,ntree = 5000)
rf_lung_bacteria
library(Boruta)
boruta_lung_filtered<-Boruta(pams_filtered~.,data = rf_lung_filtered_data,mcAdj = TRUE,
pValue=0.01,ntree = 5000)
plot(boruta_lung_filtered,las=2)

TentativeRoughFix(
boruta_lung_copy)

attStats(boruta_lung_bacteria)
getConfirmedFormula(boruta_lung)
lung_boruta_stats<-as.data.frame(attStats(boruta_lung))
lung_boruta_stats
write.csv(lung_boruta_stats,"tables/randomforest/lung_boruta_stats.csv")
par(mar=c(7, 3, 3, 1))
plot(boruta_lung,las=2)
```

