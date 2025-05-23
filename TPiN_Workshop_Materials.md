2025 TPiN Workshop
================
2025-06-04

# Learning objectives for this workshop:

- Obtaining publicly available scRNA-seq data sets
- Annotating cell types
- Identify DEGs within each cell type
- Integrate multiple data sets

# Workshop Setup Instructions

We will be using R v4.3.1 and Bioconductor v3.18 and Seurat 5.2.1.

In RStudio create a new project named “2025_TPiN_Workshop”. This ensures
all the files for this workshop are placed in their own folder and
should match the paths here.

For some big objects, you may need to make sure you are using all
available memory on your computer. Can ensure this by running the
following in terminal:

``` bash
cd ~
touch .Renviron
open .Renviron
#Save R_MAX_VSIZE = __GB as the first line of .Renviron using an appropriate 
#number of GB based on your computer's available RAM.
```

# Package Installation

For this workshop, several packages need to be installed. Seurat is
relatively easy to install by itself and we will largely follow their
vignettes throughout this workshop. <https://satijalab.org/seurat/>

If having installation issues on a Mac, be sure you have Xcode developer
tools installed. May also need to install gfortran compiler for Mac
<https://cran.r-project.org/bin/macosx/tools/>

``` r
## Install required packages:

install.packages(c("Seurat", "stringr","dplyr","harmony","R.utils"))

setRepositories(ind = 1:3, addURLs = c('https://satijalab.r-universe.dev', 
                                       'https://bnprks.r-universe.dev/'))
install.packages(c("presto", "glmGamPoi"))

## Install Seuratwrapers and the remotes package
if (!requireNamespace("remotes", quietly = TRUE)) {
  install.packages("remotes")
}
remotes::install_github("satijalab/seurat-data", quiet = TRUE)
remotes::install_github("satijalab/azimuth", quiet = TRUE)
remotes::install_github("satijalab/seurat-wrappers", quiet = TRUE)

# Install Bioconductor
if (!require("BiocManager", quietly = TRUE))
  install.packages("BiocManager")

library(BiocManager)

BiocManager::install(version = "3.18")
BiocManager::install("GEOquery")
BiocManager::install("MAST")

# The version of Matrix package below is required for Azimuth cell type annotation 
# - dont update packages after installing.
remotes::install_version("Matrix", version = "1.6.4") 
```

After installing all packages, restart your RStudio. This allows for all
libraries to load cleanly and ensures that any cached or corrupted
session data doesn’t interfere with the newly installed packages.

``` r
#required for large datasets
library(future)
options(future.globals.maxSize = 1e9)
options(future.parallelization.factor = 1) 
plan("multicore") # plan("multisession") for Windows

library(Seurat)
library(harmony)
library(dplyr)
library(stringr)
library(purrr)
library(Matrix)
library(data.table)
library(readr)
library(SeuratDisk)
library(BiocManager)
library(GEOquery)
library(Azimuth)
library(ggplot2)
library(ggrepel)
library(MAST)
```

# *STOP HERE UNTIL DAY OF WORKSHOP*

# Download Data

We have made data available on Box for this workshop. Download the files
we will be using from Box into a folder in your Rproj directory named
“DataSets”.

The source datasets can be found at the links below and can be
downloaded as described. (This is not necessary for this workshop.)

## Datasets:

- Morabito et al. 2021
  - Paper: <https://pmc.ncbi.nlm.nih.gov/articles/PMC8766217/>
  - Prefrontal cortex: 11 late-stage AD and 7 controls
  - 61,472 cells in full dataset
- Li et al 2023 (Dracheva Lab)
  - Paper: <https://www.nature.com/articles/s41467-023-41033-y>
  - Data: <https://zenodo.org/records/8190317>
  - Prefrontal and motor cortex: 5 C9-FTD; 6 C9-ALS; 6 controls
  - 105,120 cells in full dataset
- BrainSCOPE Dataset
  - Paper: <https://www.science.org/doi/10.1126/science.adi5199>
  - Data (LIBD):
    <https://brainscope.gersteinlab.org/data/snrna_expr_matrices_zip/LIBD.zip>
  - Metadata:
    <https://brainscope.gersteinlab.org/data/sample_metadata/PEC2_sample_metadata.txt>
  - Prefrontal Cortex: 10 control samples
  - 52,214 cells in full dataset

## Read data into Seurat

First, identify what data type you have and the format of the data.
Can’t always tell by the file type so look at the data once you load it.

Possible file formats:

- .h5 file
- .mtx.txt file
- .rds file
- Other random formats?
  - Have seen .csv files deposited, so have to load and look!

More information on different data types can be found here:
<https://training.galaxyproject.org/training-material/topics/single-cell/tutorials/scrna-data-ingest/tutorial.html>

Things to consider:

- Are you exploring published data in isolation to ask a question you
  care about?
  - Depending on how raw the “processed” data is, determine if you need
    to do QC exactly as the paper did.
  - If so, look up the code used. If you can’t find it, may not want to
    use the processed data and instead start from raw.
- Are you combining data from multiple studies to do a meta analysis?
  - If so, start with the “rawest” data possible so all filtering and QC
    steps are the same.

# Think about batch effects:

From
<https://www.sc-best-practices.org/cellular_structure/integration.html>:

Batch correction methods deal with batch effects between samples in the
same experiment where cell identity compositions are consistent, and the
effect is often quasi-linear. In contrast, data integration methods deal
with complex, often nested, batch effects between datasets that may be
generated with different protocols and where cell identities may not be
shared across batches. Note that these terms are often used
interchangeably in general use.

# Start with Morabito Dataset:

Morabito dataset was already in .h5 format and is the easiest to load.

Example of how to read in full dataset *(DO NOT RUN)*:

``` r
# To read full dataset directly into Seurat:

data=Read10X_h5("~/Downloads/GSE174367/GSE174367_snRNA-seq_filtered_feature_bc_matrix.h5")
morabito_meta=read.csv("~/Downloads/GSE174367/GSE174367_snRNA-seq_cell_meta.csv",header=T)

# Have to match up barcodes (cells) as there are NAs in dataset
data=data[,which(colnames(data) %in% morabito_meta$Barcode)]

morabito=CreateSeuratObject(counts=data, meta.data = morabito_meta)
```

# Analyze downsampled Morabito dataset:

``` r
# Load Morabito Dataset:
morabito=readRDS("~/2025_TPiN_Workshop/DataSets/downsampled_Morabito.rds")
head(morabito@meta.data)
# Note that there are multiple batches in the data
# These must be integrated appropriately to account for batch effects 
# (not really much of one in this data set)

morabito[["RNA"]]<-split(morabito[["RNA"]],f=morabito$Batch)

# Calculate percent.mt:
morabito[["percent.mt"]] <- PercentageFeatureSet(morabito,pattern="^MT-")

# Filter based on QC metrics - original paper used a percent.mt cutoff of 
# 10, but we are using 5
# 22091 cells before QC; 21860 after
morabito<-subset(morabito, subset = nFeature_RNA < 10000 &   
               nFeature_RNA > 200 & percent.mt<5)
             
# Normalize and PCA:
# Regress out percent.mt as a technical covariate

morabito=SCTransform(morabito,vars.to.regress="percent.mt",verbose = F) %>% 
  RunPCA(npcs=30,verbose=F)

# Cluster and Plot:
morabito=morabito %>% 
  RunUMAP(reduction = "pca", dims = 1:30, reduction.name = "umap.pca", 
          reduction.key = "UMAP_") %>%
  FindNeighbors(reduction = "pca", dims = 1:30) %>%
  FindClusters()
```

## Plot UMAP before integration

``` r
par(mar = c(4, 4, .1, .1))
DimPlot(morabito, group.by="Batch",label = TRUE,reduction="umap.pca")
DimPlot(morabito, group.by="Cell.Type",label = TRUE,reduction="umap.pca")

# Batch looks pretty good before integration, cell type is...not great.
```

## Integrate via two methods: Harmony and CCA

``` r
# Harmony Integration
morabito <- IntegrateLayers(
  object = morabito, method = HarmonyIntegration,
  orig.reduction = "pca", new.reduction = "integrated.harmony",
  normalization.method = "SCT",
  verbose = FALSE
)
# UMAP and clustering after Harmony integration
morabito <- morabito %>%
  RunUMAP(reduction = "integrated.harmony", dims = 1:30, reduction.name = 
            "umap.int.harmony", reduction.key = "hUMAP_") %>%
  FindNeighbors(reduction = "integrated.harmony", dims = 1:30) %>%
  FindClusters()


# CCA Integration
morabito <- IntegrateLayers(
  object = morabito, method = CCAIntegration,
  orig.reduction = "pca", new.reduction = "integrated.cca",
  normalization.method = "SCT",
  verbose = FALSE
)

# UMAP and clustering after CCA integration
morabito <- morabito %>%
  RunUMAP(reduction = "integrated.cca", dims = 1:30, reduction.name = 
            "umap.cca", reduction.key = "hUMAP_") %>%
  FindNeighbors(reduction = "integrated.cca", dims = 1:30) %>%
  FindClusters()

# Re-join Layers after integration

morabito[["RNA"]]<-JoinLayers(morabito[["RNA"]])
```

## Plot UMAP after integration

``` r
par(mar = c(4, 4, .1, .1))
DimPlot(morabito,group.by="Batch",label = TRUE,reduction="umap.int.harmony")
DimPlot(morabito,group.by="Batch",label = TRUE,reduction="umap.cca")
```

We will continue with Harmony integration, but you can play with other
methods and compare resulting UMAPs/tSNEs.

# Identify cell types using Azimuth

``` r
DefaultAssay(morabito)<-"SCT"
morabito<-RunAzimuth(morabito, reference = "humancortexref", query.modality="SCT")

sort(table(morabito$predicted.class))
sort(table(morabito$predicted.subclass))
```

## Do these cell types match original annotations?

``` r
par(mar = c(4, 4, .1, .1))
DimPlot(morabito,group.by="predicted.class",label = TRUE,reduction="umap.int.harmony")
DimPlot(morabito,group.by="cell_type",label = TRUE,reduction="umap.int.harmony")
```

## The published cell type annotation do not match ours. Our new annotations are more consistent with the clustering.

Points to discuss:

- How could this happen?

- How concerned should we be?

# DEG Analysis

DEG analysis always uses raw counts not normalized/SCT values. What is
our question?

- What are the DEGs in each cell type between disease and control?
- Will focus just on AD vs control using the morabito dataset
- What is our n?
  - cells
  - metacells
  - donors

Using the default FindMarkers function:

- Which test to use? Most frequently used are:
  - wilcoxon rank sum
  - MAST
  - DESeq2

## Cells as n:

``` r
# DEG analysis with our correct cell types 
DefaultAssay(morabito)<-"RNA"
morabito[["RNA"]]<-JoinLayers(morabito[["RNA"]])
morabito=NormalizeData(morabito) %>% FindVariableFeatures() %>% ScaleData()

# Susbset by cell type and make list just of Ex, Inh neurons and Glia

Inhibitory<-subset(morabito, subset=predicted.class=="GABAergic")
Excitatory<-subset(morabito, subset=predicted.class=="Glutamatergic")
Glia<-subset(morabito, subset=predicted.class=="Non-Neuronal")

subs<-list(Inhibitory,Excitatory,Glia)

# Set the comparison group, in our case AD vs Control - "Diagnosis"
# For differential testing, use MAST as method
# only test genes that are expressed in at 25% of cells
# Control for differences due to age and sex

i=1
Idents(subs[[i]])<-subs[[i]]$Diagnosis
subs[[i]]<-NormalizeData(subs[[i]]) # normalize data within the cell type
# Start with MAST across all cells
ALL<-FindMarkers(subs[[i]], ident.1="AD",ident.2="Control",min.pct=0.25,
                 latent.vars=c("Age","Sex"),test.use="MAST", assay="RNA") 
ALL$celltype<-subs[[i]]$predicted.class[1]
ALL$gene<-rownames(ALL) #rownames will get duplicated if you don't do this!
for (i in 2:length(subs)){
    Idents(subs[[i]])<-subs[[i]]$Diagnosis
    subs[[i]]<-NormalizeData(subs[[i]])%>% ScaleData(features=row.names(data))
    markers<-FindMarkers(subs[[i]], ident.1="AD",ident.2="Control",min.pct=0.25, 
                         latent.vars=c("Age","Sex"), 
                         test.use="MAST",assay="RNA") 
    markers$celltype<-subs[[i]]$predicted.class[1]
    markers$gene<-rownames(markers)
    ALL<-rbind(ALL,markers)
}


ALL$cat<-ifelse(ALL$avg_log2FC >0, "up","down")

# set thresholds for your data:
significant<-ALL[which(abs(ALL$avg_log2FC)>0.25),]
significant<-significant[which(significant$p_val_adj<0.01),]
```

## Compare to wilcoxon rank sum test (default):

``` r
i=1
ALL_wilcox<-FindMarkers(subs[[i]], ident.1="AD",ident.2="Control",min.pct=0.25,
                        test.use="wilcox", assay="RNA") 
ALL_wilcox$celltype<-subs[[i]]$predicted.class[1]
ALL_wilcox$gene<-rownames(ALL_wilcox)
for (i in 2:length(subs)){
    Idents(subs[[i]])<-subs[[i]]$Diagnosis
    subs[[i]]<-NormalizeData(subs[[i]])%>% ScaleData(features=row.names(data))
    markers<-FindMarkers(subs[[i]], ident.1="AD",ident.2="Control",min.pct=0.25, 
                         test.use="wilcox",assay="RNA") 
    markers$celltype<-subs[[i]]$predicted.class[1]
    markers$gene<-rownames(markers)
    ALL_wilcox<-rbind(ALL_wilcox,markers)
}

ALL_wilcox$cat<-ifelse(ALL_wilcox$avg_log2FC >0, "up","down")

# set thresholds for your data:
significant_wilcox<-ALL_wilcox[which(abs(ALL_wilcox$avg_log2FC)>0.25),]
significant_wilcox<-significant_wilcox[which(significant_wilcox$p_val_adj<0.01),]
```

# Pseudobulk

We can aggregate all cells within the same cell type from each
sample/donor using the AggregateExpression() function. This returns a
Seurat object where each ‘cell’ represents the pseudobulk profile of one
cell type in one individual.

After we aggregate cells, we can perform cell type-specific differential
expression between healthy and AD samples using DESeq2.

``` r
# Psuedobulk
aggregated <- AggregateExpression(morabito, slot="counts", assays="RNA",
  group.by = c("predicted.class", "SampleID","Diagnosis"), return.seurat = TRUE)

# each 'cell' is a donor-condition-celltype pseudobulk profile
head(Cells(aggregated))

aggregated$celltype.diag <- paste(aggregated$predicted.class, 
                                  aggregated$Diagnosis, sep = "_")
table(aggregated$celltype.diag)

# Next, we perform DE testing on the pseudobulk level for Glutamatergic neurons, 
# and compare it against the previous single-cell-level DE results.

Idents(aggregated) <- "celltype.diag"

bulk.de <- FindMarkers(object = aggregated, 
                         ident.1 = "Glutamatergic_AD", 
                         ident.2 = "Glutamatergic_Control",
                         test.use = "DESeq2")
# look at your results - due to subsampling and low numbers of donors, 
# don't really expect any significant differences.

# set thresholds for your data - in this case just use p_val rather than adjusted p_val:
significant_bulk<-bulk.de[which(abs(bulk.de$avg_log2FC)>0.25),]
significant_bulk<-significant_bulk[which(significant_bulk$p_val<0.01),]
```

# Compare the DEG P-values between the single-cell level and the pseudobulk level results

``` r
names(bulk.de) <- paste0(names(bulk.de), ".bulk")
bulk.de$gene <- rownames(bulk.de)


names(ALL_wilcox) <- paste0(names(ALL_wilcox), ".sc")
ALL_wilcox$gene <- rownames(ALL_wilcox)

# subset to just glutamatergic neurons and combine the DEGs from both methods
sc_Gluta=ALL_wilcox[which(ALL_wilcox$celltype.sc=="Glutamatergic"),]
merge_dat <- merge(sc_Gluta, bulk.de, by = "gene").   # merging on gene
merge_dat <- merge_dat[order(merge_dat$p_val.bulk), ] # this orders by p-value


# Number of genes that are marginally significant in both; 
# marginally significant only in bulk; and marginally significant only in 
# single-cell

common <- merge_dat$gene[which(merge_dat$p_val.bulk < 0.05 & 
                                merge_dat$p_val.sc < 0.05)]
only_sc <- merge_dat$gene[which(merge_dat$p_val.bulk > 0.05 & 
                                  merge_dat$p_val.sc < 0.05)]
only_bulk <- merge_dat$gene[which(merge_dat$p_val.bulk < 0.05 & 
                                    merge_dat$p_val.sc > 0.05)]
print(paste0('# Common: ',length(common)))
print(paste0('# Only in single-cell: ',length(only_sc)))
print(paste0('# Only in bulk: ',length(only_bulk)))

# So what is real? Let's look at the data
# create a new column to annotate sample-condition-celltype in the single-cell dataset
morabito$donor_id.diag <- paste0(morabito$Diagnosis, "-", morabito$SampleID)
morabito$celltype.diag <- paste(morabito$predicted.class, morabito$Diagnosis, sep = "_")

# generate violin plot 
Idents(morabito) <- "celltype.diag"
print(merge_dat[merge_dat$gene%in%common[1:2],c('gene','p_val.sc.sc','p_val.bulk')])

VlnPlot(morabito, features = common[1:2], idents = c("Glutamatergic_Control", 
                                  "Glutamatergic_AD"), group.by = "Diagnosis") 
VlnPlot(morabito, features = common[1:2], idents = c("Glutamatergic_Control",
                                  "Glutamatergic_AD"), group.by = "SampleID") 

# Plot at the bulk level:
VlnPlot(aggregated, features = common[1:2], idents = c("Glutamatergic_Control",
                    "Glutamatergic_AD"), group.by = "Diagnosis") 


# sort data for largest log2FC instead and recall "commmon" to see maximum visual effect
merge_dat=merge_dat[order(abs(merge_dat$avg_log2FC.bulk),decreasing=T),]
common <- merge_dat$gene[which(merge_dat$p_val.bulk < 0.05 & 
                                merge_dat$p_val.sc < 0.05)]

VlnPlot(morabito, features = common[11:12], idents = c("Glutamatergic_Control", 
                                "Glutamatergic_AD"), group.by = "Diagnosis") 
VlnPlot(aggregated, features = common[11:12], idents = c("Glutamatergic_Control", 
                                  "Glutamatergic_AD"), group.by = "Diagnosis") 
```

# Integrate multiple datasets

## First, we will need to subset more for this to work on our local machines:

``` r
# Subset Morabito dataset:
Idents(morabito)="predicted.subclass"
morabito=subset(x=morabito,downsample=500) # 6742 cells left

# Load and downsample Dracheva Dataset:
dracheva=readRDS("~/2025_TPiN_Workshop/DataSets/downsampled_Dracheva.rds")
Idents(dracheva)="annotation_major_cell_type"
dracheva=subset(x=dracheva,downsample=500) # 6910 cells left
```

## Process BrainSCOPE LIBD dataset

``` r
# LIBD Dataset from BrainSCOPE 
# This is a unique format and barcode information is not available because they
# pre-annotated the cell type information:

dataset_paths <- list.files("~/2025_TPiN_Workshop/DataSets/LIBD/",full.names = TRUE)

# Get sample names
samples<- map_chr(dataset_paths, basename) 
samples<- sapply(strsplit(samples, "-"),"[[",1)
names(dataset_paths)<- samples

# Create a function to read counts into the correct format for Seurat since there
# isn't a ready function available.

# Function:
read_counts<- function(file,sample_name){
  x<- read_tsv(file)
  x<- as.data.frame(x)
  genes<- x[,1]
  x<- x[, -1]
  rownames(x)<- genes
  return(as.matrix(x))
}

# Run function to read in the data and save as individual h5 Seurat objects:
# Saving as h5 allows more efficient data storage and ability to read in 
# multiple larger files.

for (i in seq_along(samples)) {
  sample_name <- samples[i]
  file_path <- dataset_paths[i]

  # Read counts
  counts <- read_counts(file_path, sample_name)

  # Initialize Seurat object for each sample
  seurat_obj <- CreateSeuratObject(counts)
  
  # Have to convert to assay to save as h5
  seurat_obj[["RNA"]] <- as(object = seurat_obj[["RNA"]], Class = "Assay")
  
  # Annotate metadata with cell types
  cell_ids <- colnames(seurat_obj)
  seurat_obj@meta.data$cell_type <- sapply(strsplit(cell_ids, "[.]"), "[[", 1)

  # Save Seurat object as an .h5 file
  SaveH5Seurat(seurat_obj, filename = sample_name, verbose=F)

}

h5_files <- list.files(pattern = ".h5seurat$", full.names = TRUE)

# Create function to read in these files:

read_h5Seurat_file <- function(file_path) {
  # Load the Seurat object from the .h5Seurat file
  seurat_obj <- LoadH5Seurat(file_path)
  return(seurat_obj)
}

# Read 5 of the .h5Seurat files into a list of Seurat objects.
# If you hit a vector memory limit, read in 2 or 3 files.
seurat_objects_list <- map(h5_files[1:5], read_h5Seurat_file)

# Name the objects list based on the filenames
names(seurat_objects_list) <- sapply(basename(h5_files[1:5]), 
                                     function(x) sub("\\.h5seurat$", "", x))
# Clean up to keep space open
rm(i,file_path,sample_name,samples,
   dataset_paths,h5_files,read_counts,read_h5Seurat_file)

# Merge the samples into one object with 23,525 cells
LIBD=merge(x = seurat_objects_list[[1]], y = seurat_objects_list[2:5],
           add.cell.ids=names(seurat_objects_list),project="LIBD")

rm(seurat_objects_list)

# Fix metadata to name the dataset and add sample name column 
LIBD@meta.data$dataset="LIBD"
LIBD@meta.data$sample=sapply(strsplit(rownames(LIBD@meta.data),"[_]"),"[[",1)

# Subsample the data
# Set Idents to cell types so sample across each
Idents(LIBD)="cell_type"

# Look at cell types before subsetting:
table(LIBD@meta.data$cell_type)

LIBD=subset(x=LIBD,downsample=500) # down to 8,331 cells

# Look at cell types after subsetting:
table(LIBD@meta.data$cell_type)
```

## Look at the data sets and meta data:

``` r
head(LIBD@meta.data)
# Not much extra information included here.

# Need to calculate percent.mt
LIBD[["percent.mt"]] <- PercentageFeatureSet(LIBD,pattern="^MT-")

# It is 0 for all as no MT- genes are in the data set, but need to populate for
# filtering later

head(dracheva@meta.data)
# Note that there are also batches,
# doublets have already been removed, and percent_mt has been calculated

# Have they already been filtered on percent_mt?
quantile(dracheva@meta.data$percent_mt) # YES, but rename column to be consistent
dracheva@meta.data$percent.mt=dracheva@meta.data$percent_mt
dracheva@meta.data$percent_mt=NULL

# Full dataset had 2 brain regions, but already subset to just frontal cortex

table(dracheva@meta.data$seq_batch) 
# Dataset has 6 seq batches - rename column "Batch" to match morabito
dracheva@meta.data$Batch=dracheva@meta.data$seq_batch
dracheva@meta.data$seq_batch=NULL

# Give LIBD dataset a batch name
LIBD@meta.data$Batch="A"
# Can we integrate these data together? If want to, then adjust meta.data to have same
# colnames for same categories (mainly cell_type vs Cell.Type), but make sure 
# levels match
table(dracheva@meta.data$annotation_major_cell_type)
table(LIBD@meta.data$cell_type)
table(morabito@meta.data$Cell.Type)
# These are all about the same level of cell type descriptor
# (except morabito which is a little more vague)

# Name each of these columns "cell_type" for easier subsetting/plotting later on 
dracheva@meta.data$cell_type=dracheva@meta.data$annotation_major_cell_type
dracheva@meta.data$annotation_major_cell_type=NULL

morabito@meta.data$cell_type=morabito@meta.data$Cell.Type
morabito@meta.data$Cell.Type=NULL

# Add diagnosis info to each dataset to be consistent
dracheva@meta.data$Diagnosis=dracheva@meta.data$disease
dracheva@meta.data$disease=NULL
LIBD@meta.data$Diagnosis="Control"

# Add dataset to each and check batches
morabito@meta.data$dataset="morabito"
dracheva@meta.data$dataset="dracheva"
```

# Integrate the datasets:

``` r
# First, split by batch for the integration.
 morabito[["RNA"]]<-split(morabito[["RNA"]],f=morabito$Batch) 
 dracheva[["RNA"]]<-split(dracheva[["RNA"]],f=dracheva$Batch)

 # This is not necessary if regressing out batch with SCT 
 # (like we did with percent.mt) but takes too much memory for this.

# Next, merge them together:
# For more than two samples:
data <- merge(x = LIBD, y = c(dracheva,morabito), project = "merged_project")
# 21,983 cells total

rm(dracheva, LIBD) #to free up space - will use morabito later so keep.
gc()
# For just two samples:
# data <- merge(x = LIBD, y = morabito, project = "merged_project")

# Filter based on QC metrics - original morabito paper used a percent.mt cutoff of 10,
# and dracheva of 1, but we are using 5
# 21983 before QC; 15140 after mainly bc of nFeature cutoff  

data<-subset(data, subset = nFeature_RNA < 10000 &   
               nFeature_RNA > 200 & percent.mt<5)
             
# Normalize and PCA:

data=SCTransform(data,vars.to.regress=c("percent.mt"),verbose = F) %>% 
  RunPCA(npcs=30,verbose=F)

# Cluster and Plot:
data=data %>% 
  RunUMAP(reduction = "pca", dims = 1:30, reduction.name = "umap.pca", 
          reduction.key = "UMAP_") %>%
  FindNeighbors(reduction = "pca", dims = 1:30) %>%
  FindClusters()
```

## Plot UMAP before integration

``` r
par(mar = c(4, 4, .1, .1))
DimPlot(data, group.by="Batch",label = TRUE,reduction="umap.pca")
DimPlot(data, group.by="cell_type",label = TRUE,reduction="umap.pca")
# Batch is clearly a big problem as expected.
```

## Integrate via Harmony

``` r
# Harmony Integration
data <- IntegrateLayers(
  object = data, method = HarmonyIntegration,
  orig.reduction = "pca", new.reduction = "integrated.harmony",
  normalization.method = "SCT",
  verbose = FALSE
)

# UMAP and clustering after Harmony batch correction
data <- data %>%
  RunUMAP(reduction = "integrated.harmony", dims = 1:30, reduction.name = 
            "umap.int.harmony", reduction.key = "hUMAP_") %>%
  FindNeighbors(reduction = "integrated.harmony", dims = 1:30) %>%
  FindClusters()

# Re-join Layers after integration and run NormalizeData to create scale.data layer

data[["RNA"]]<-JoinLayers(data[["RNA"]])
data=NormalizeData(data) %>% FindVariableFeatures() %>% ScaleData()
```

## Plot UMAP after integration

``` r
DimPlot(data,group.by="Batch",label = TRUE,reduction="umap.int.harmony")
DimPlot(data,group.by="cell_type",label = TRUE,reduction="umap.int.harmony")

# Much better
```

# Identify a common set of cell types using Azimuth

``` r
DefaultAssay(data)<-"SCT"
data<-RunAzimuth(data, reference = "humancortexref", query.modality="SCT")

sort(table(data$predicted.class))
sort(table(data$predicted.subclass))
```

``` r
DimPlot(data,group.by="Batch",label = TRUE,reduction="umap.int.harmony")
DimPlot(data,group.by="predicted.class",label = TRUE,reduction="umap.int.harmony")
DimPlot(data,group.by="predicted.subclass",label = TRUE,reduction="umap.int.harmony")


# So it's better but not great as we see a lot of clusters with multiple cell types. 
# Multiple options to clean this up.
```

## Can plot genes of interest if desired. Will look at some marker genes.

More options for visualization found here:
<https://satijalab.org/seurat/articles/visualization_vignette>

``` r
# Choose your favorite gene(s)!
features=c("GAD2","MBP","GFAP")
FeaturePlot(data, features = features, 
            pt.size = 0.2, ncol = 3)
# Can look at specific genes across each cell type category in violin plots:

VlnPlot(data,features = features, idents="predicted.class")

DoHeatmap(data, features = VariableFeatures(data)[1:20], cells=1:500, size = 3,
          group.by= "Diagnosis")
```

## Clustering and cell type assignment still has mixed cell type clusters.

What are the options to address this?

- Adjust clustering and integration parameters

# Questions and Discussion
