# Microbial ecology of the transplanted human lung

We combined amplicon sequencing and culturomics to characterize the viable bacterial community in 234 longitudinal bronchoalveolar lavage samples from 64 lung transplant recipients and established links to viral loads, host gene expression, lung function, and transplant health. 
We find that the lung microbiota post-transplant can be categorized into four distinct compositional states, ‘pneumotypes’. 

## Code conceptualization and development team 

* Sudip Das<sup>†
* Germán Bonilla-Rosso<sup>†
* Céline Pattaroni<sup>‡
* Alexis Rapin<sup>#
* Philipp Engel<sup>†

1 Department of Fundamental Microbiology, Biophore, University of Lausanne, Switzerland.

2 Department of Immunology and Pathology, Central Clinical School, Monash University, Australia.

3 Division of Pneumology, Centre Hospitalier Universitaire Vaudois (CHUV), Lausanne, Switzerland.

## Code environment and structure

**1. System Requirements** 

This pipeline has been tested on the following version of the required software.

* macOS 10.15.4 Catalina or Ubuntu 20.04 or MS Windows 10
* Python 2.7 
* Bash script 
* R version 3.6.3
* [Genocrunch](https://genocrunch.epfl.ch/home/doc) v 2 Release 2018/06/08
* [QIIME](http://qiime.org/install/install.html) v 1.9.1 
* R packages [phyloseq v. 1.26.1](https://github.com/joey711/phyloseq) and [ampvis2 v. 2.3.2](https://madsalbertsen.github.io/ampvis2/) are available in GitHub.

**2. Installation** 

* Python 2.7 comes pre-installed with macOS and Ubuntu. 
* For MS Windows, follow the [installation instructions](https://docs.python.org/3/using/windows.html).
* For MS Windows 10 now comes with Bash Shell scripting option. Initiate this by typing the following on the Command prompt window

```
#!/bin/bash
```
* For installation of **vsearch**, please follow the instructions in the github repository https://github.com/torognes/vsearch
* For installing **QIIME 1**, please follow the instructions on http://qiime.org/install/install.html
* **Genocrunch** is a web application. To use it, you need to create a free account. Further instructions can be found at https://genocrunch.epfl.ch/home/doc
* **phyloseq** and **ampvis2** can be installed via R following the given instructions.

**3. Demo**

All demo and instructions are provided in line with the code in the individual markdown documents.


**4. Instructions for use**

The analysis is divided into 3 parts:

* Part 1 (**Das_et_al_2020_analysis_pipeline_1**): Sequencing analysis pipeline using python (QIIME) and bash (vsearch, FastXToolkit, SINA, FastTree). Code from raw data processing, merging cultured sequences (LuMiCol), OTU picking and phylogeny.

* Import BIOM in R. For instructions see the R markdown

* Part 2 (**Das_et_al_2020_analysis_pipeline_2**): R markdown with BALF community analysis with starting OTUs from pipeline 1 with phyloseq, ampvis2 and vegan. Random Forest algorithms, Markov chain analysis and all statistical analysis and visualization plots.

* Part 3 - nested inside part 2- after applying appropriate filtering, the OTU table with taxonomy and metadata is exported 
as spreadsheet (.csv file). This was imported to the web application [Genocrunch](https://genocrunch.epfl.ch/home/doc) for performing the pneumotype discovery i.e. K-medoid based machine learning using Bray-Curtis diversity index. This generated Partition-around medoids (PAMs) and this information was added to the metadata file and re-imported to the R code. 

## Data availibity 

We have deposited the raw data from all samples used in the study to Short Read Archive, NCBI under the BioProject **PRJNA632552** and BioSample accession **SAMN14911405**. 

## Reference publication

**A prevalent and culturable microbiota links ecological balance to clinical stability of the human lung after transplantation**

Sudip Das, Eric Bernasconi, Angela Koutsokera, Daniel-Adrien Wurlod, Vishwachi Tripathi, Germán Bonilla-Rosso, John-David Aubert, Marie-France Derkenne, Louis Mercier, Céline Pattaroni, Alexis Rapin, Christophe von Garnier, Benjamin J. Marsland, Philipp Engel, Laurent P. Nicod

bioRxiv 2020.05.21.106211; doi: https://doi.org/10.1101/2020.05.21.106211

## Citations 
