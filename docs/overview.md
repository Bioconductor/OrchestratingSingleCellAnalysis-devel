# (PART) Focus Topics {-}

# Overview

<script>
document.addEventListener("click", function (event) {
    if (event.target.classList.contains("aaron-collapse")) {
        event.target.classList.toggle("active");
        var content = event.target.nextElementSibling;
        if (content.style.display === "block") {
            content.style.display = "none";
        } else {
            content.style.display = "block";
        }
    }
})
</script>

<style>
.aaron-collapse {
  background-color: #eee;
  color: #444;
  cursor: pointer;
  padding: 18px;
  width: 100%;
  border: none;
  text-align: left;
  outline: none;
  font-size: 15px;
}

.aaron-content {
  padding: 0 18px;
  display: none;
  overflow: hidden;
  background-color: #f1f1f1;
}
</style>

## Introduction

This chapter provides an overview of the framework of a typical scRNA-seq analysis workflow (Figure \@ref(fig:scworkflow)).
Subsequent chapters will describe each analysis step in more detail.

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/Workflow.png" alt="Schematic of a typical scRNA-seq analysis workflow. Each stage (separated by dashed lines) consists of a number of specific steps, many of which operate on and modify a `SingleCellExperiment` instance."  />
<p class="caption">(\#fig:scworkflow)Schematic of a typical scRNA-seq analysis workflow. Each stage (separated by dashed lines) consists of a number of specific steps, many of which operate on and modify a `SingleCellExperiment` instance.</p>
</div>

## Experimental Design

Before starting the analysis itself, some comments on experimental design may be helpful.
The most obvious question is the choice of technology, which can be roughly divided into:

- Droplet-based: 10X Genomics, inDrop, Drop-seq
- Plate-based with unique molecular identifiers (UMIs): CEL-seq, MARS-seq
- Plate-based with reads: Smart-seq2
- Other: sci-RNA-seq, Seq-Well

Each of these methods have their own advantages and weaknesses that are discussed extensively elsewhere [@mereu2019benchmarking;@ziegenhain2017comparative].
In practical terms, droplet-based technologies are the current _de facto_ standard due to their throughput and low cost per cell.
Plate-based methods can capture other phenotypic information (e.g., morphology) and are more amenable to customization.
Read-based methods provide whole-transcript coverage, which is useful in some applications (e.g., splicing, exome mutations); otherwise, UMI-based methods are more popular as they mitigate the effects of PCR amplification noise.
The choice of method is left to the reader's circumstances - we will simply note that most aspects of our analysis pipeline are technology-agnostic.

The next question is how many cells should be captured, and to what depth they should be sequenced.
The short answer is "as much as you can afford to spend".
The long answer is that it depends on the aim of the analysis.
If we are aiming to discover rare cell subpopulations, then we need more cells.
If we are aiming to characterize subtle differences, then we need more sequencing depth.
As of time of writing, an informal survey of the literature suggests that typical droplet-based experiments would capture anywhere from 10,000 to 100,000 cells, sequenced at anywhere from 1,000 to 10,000 UMIs per cell (usually in inverse proportion to the number of cells).
Droplet-based methods also have a trade-off between throughput and doublet rate that affects the true efficiency of sequencing.

For studies involving multiple samples or conditions, the design considerations are the same as those for bulk RNA-seq experiments.
There should be multiple biological replicates for each condition and conditions should not be confounded with batch.
Note that individual cells are not replicates; rather, we are referring to samples derived from replicate donors or cultures.

## Obtaining a count matrix 

Sequencing data from scRNA-seq experiments must be converted into a matrix of expression values that can be used for statistical analysis.
Given the discrete nature of sequencing data, this is usually a count matrix containing the number of UMIs or reads mapped to each gene in each cell.
The exact procedure for quantifying expression tends to be technology-dependent:

* For 10X Genomics data, the `CellRanger` software suite provides a custom pipeline to obtain a count matrix.
This uses _STAR_ to align reads to the reference genome and then counts the number of unique UMIs mapped to each gene.
* Pseudo-alignment methods such as `alevin` can be used to obtain a count matrix from the same data with greater efficiency.
This avoids the need for explicit alignment, which reduces the compute time and memory usage.
* For other highly multiplexed protocols, the *[scPipe](https://bioconductor.org/packages/3.12/scPipe)* package provides a more general pipeline for processing scRNA-seq data.
This uses the *[Rsubread](https://bioconductor.org/packages/3.12/Rsubread)* aligner to align reads and then counts UMIs per gene.
* For CEL-seq or CEL-seq2 data, the *[scruff](https://bioconductor.org/packages/3.12/scruff)* package provides a dedicated pipeline for quantification.
* For read-based protocols, we can generally re-use the same pipelines for processing bulk RNA-seq data.
* For any data involving spike-in transcripts, the spike-in sequences should be included as part of the reference genome during alignment and quantification.

After quantification, we import the count matrix into R and create a `SingleCellExperiment` object.
This can be done with base methods (e.g., `read.table()`) followed by applying the `SingleCellExperiment()` constructor.
Alternatively, for specific file formats, we can use dedicated methods from the *[DropletUtils](https://bioconductor.org/packages/3.12/DropletUtils)* (for 10X data) or *[tximport](https://bioconductor.org/packages/3.12/tximport)*/*[tximeta](https://bioconductor.org/packages/3.12/tximeta)* packages (for pseudo-alignment methods).
Depending on the origin of the data, this requires some vigilance:

- Some feature-counting tools will report mapping statistics in the count matrix (e.g., the number of unaligned or unassigned reads).
While these values can be useful for quality control, they would be misleading if treated as gene expression values.
Thus, they should be removed (or at least moved to the `colData`) prior to further analyses.
- Be careful of using the `^ERCC` regular expression to detect spike-in rows in human data where the row names of the count matrix are gene symbols.
An ERCC gene family actually exists in human annotation, so this would result in incorrect identification of genes as spike-in transcripts.
This problem can be avoided by using count matrices with standard identifiers (e.g., Ensembl, Entrez).

## Data processing and downstream analysis

In the simplest case, the workflow has the following form:

1. We compute quality control metrics to remove low-quality cells that would interfere with downstream analyses.
These cells may have been damaged during processing or may not have been fully captured by the sequencing protocol.
Common metrics includes the total counts per cell, the proportion of spike-in or mitochondrial reads and the number of detected features.
2. We convert the counts into normalized expression values to eliminate cell-specific biases (e.g., in capture efficiency).
This allows us to perform explicit comparisons across cells in downstream steps like clustering.
We also apply a transformation, typically log, to adjust for the mean-variance relationship. 
3. We perform feature selection to pick a subset of interesting features for downstream analysis.
This is done by modelling the variance across cells for each gene and retaining genes that are highly variable.
The aim is to reduce computational overhead and noise from uninteresting genes.
4. We apply dimensionality reduction to compact the data and further reduce noise.
Principal components analysis is typically used to obtain an initial low-rank representation for more computational work,
followed by more aggressive methods like $t$-stochastic neighbor embedding for visualization purposes.
5. We cluster cells into groups according to similarities in their (normalized) expression profiles.
This aims to obtain groupings that serve as empirical proxies for distinct biological states.
We typically interpret these groupings by identifying differentially expressed marker genes between clusters.

Additional steps such as data integration and cell annotation will be discussed in their respective chapters.

## Quick start

Here, we use the a droplet-based retina dataset from @macosko2015highly, provided in the *[scRNAseq](https://bioconductor.org/packages/3.12/scRNAseq)* package.
This starts from a count matrix and finishes with clusters (Figure \@ref(fig:quick-start-umap)) in preparation for biological interpretation.
Similar workflows are available in abbreviated form in the Workflows.,


```r
library(scRNAseq)
sce <- MacoskoRetinaData()

# Quality control.
library(scater)
is.mito <- grepl("^MT-", rownames(sce))
qcstats <- perCellQCMetrics(sce, subsets=list(Mito=is.mito))
filtered <- quickPerCellQC(qcstats, percent_subsets="subsets_Mito_percent")
sce <- sce[, !filtered$discard]

# Normalization.
sce <- logNormCounts(sce)

# Feature selection.
library(scran)
dec <- modelGeneVar(sce)
hvg <- getTopHVGs(dec, prop=0.1)

# Dimensionality reduction.
set.seed(1234)
sce <- runPCA(sce, ncomponents=25, subset_row=hvg)
sce <- runUMAP(sce, dimred = 'PCA', external_neighbors=TRUE)

# Clustering.
g <- buildSNNGraph(sce, use.dimred = 'PCA')
colLabels(sce) <- factor(igraph::cluster_louvain(g)$membership)

# Visualization.
plotUMAP(sce, colour_by="label")
```

<div class="figure">
<img src="overview_files/figure-html/quick-start-umap-1.png" alt="UMAP plot of the retina dataset, where each point is a cell and is colored by the cluster identity." width="672" />
<p class="caption">(\#fig:quick-start-umap)UMAP plot of the retina dataset, where each point is a cell and is colored by the cluster identity.</p>
</div>

## Session Info {-}

<button class="aaron-collapse">View session info</button>
<div class="aaron-content">
```
R version 4.0.2 (2020-06-22)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Ubuntu 18.04.5 LTS

Matrix products: default
BLAS:   /home/biocbuild/bbs-3.12-bioc/R/lib/libRblas.so
LAPACK: /home/biocbuild/bbs-3.12-bioc/R/lib/libRlapack.so

locale:
 [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
 [3] LC_TIME=en_US.UTF-8        LC_COLLATE=C              
 [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
 [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
 [9] LC_ADDRESS=C               LC_TELEPHONE=C            
[11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       

attached base packages:
[1] parallel  stats4    stats     graphics  grDevices utils     datasets 
[8] methods   base     

other attached packages:
 [1] scran_1.17.15               scater_1.17.4              
 [3] ggplot2_3.3.2               scRNAseq_2.3.12            
 [5] SingleCellExperiment_1.11.6 SummarizedExperiment_1.19.6
 [7] DelayedArray_0.15.7         matrixStats_0.56.0         
 [9] Matrix_1.2-18               Biobase_2.49.0             
[11] GenomicRanges_1.41.6        GenomeInfoDb_1.25.10       
[13] IRanges_2.23.10             S4Vectors_0.27.12          
[15] BiocGenerics_0.35.4         BiocStyle_2.17.0           
[17] simpleSingleCell_1.13.16   

loaded via a namespace (and not attached):
 [1] bitops_1.0-6                  bit64_4.0.2                  
 [3] httr_1.4.2                    tools_4.0.2                  
 [5] R6_2.4.1                      irlba_2.3.3                  
 [7] vipor_0.4.5                   uwot_0.1.8                   
 [9] DBI_1.1.0                     colorspace_1.4-1             
[11] withr_2.2.0                   gridExtra_2.3                
[13] tidyselect_1.1.0              processx_3.4.3               
[15] bit_4.0.4                     curl_4.3                     
[17] compiler_4.0.2                graph_1.67.1                 
[19] BiocNeighbors_1.7.0           labeling_0.3                 
[21] bookdown_0.20                 scales_1.1.1                 
[23] callr_3.4.3                   rappdirs_0.3.1               
[25] stringr_1.4.0                 digest_0.6.25                
[27] rmarkdown_2.3                 XVector_0.29.3               
[29] pkgconfig_2.0.3               htmltools_0.5.0              
[31] limma_3.45.10                 dbplyr_1.4.4                 
[33] fastmap_1.0.1                 highr_0.8                    
[35] rlang_0.4.7                   RSQLite_2.2.0                
[37] shiny_1.5.0                   DelayedMatrixStats_1.11.1    
[39] farver_2.0.3                  generics_0.0.2               
[41] BiocParallel_1.23.2           dplyr_1.0.1                  
[43] RCurl_1.98-1.2                magrittr_1.5                 
[45] BiocSingular_1.5.0            GenomeInfoDbData_1.2.3       
[47] scuttle_0.99.12               Rcpp_1.0.5                   
[49] ggbeeswarm_0.6.0              munsell_0.5.0                
[51] viridis_0.5.1                 lifecycle_0.2.0              
[53] edgeR_3.31.4                  stringi_1.4.6                
[55] yaml_2.2.1                    zlibbioc_1.35.0              
[57] BiocFileCache_1.13.1          AnnotationHub_2.21.2         
[59] grid_4.0.2                    blob_1.2.1                   
[61] dqrng_0.2.1                   promises_1.1.1               
[63] ExperimentHub_1.15.1          crayon_1.3.4                 
[65] lattice_0.20-41               cowplot_1.0.0                
[67] locfit_1.5-9.4                CodeDepends_0.6.5            
[69] knitr_1.29                    ps_1.3.4                     
[71] pillar_1.4.6                  igraph_1.2.5                 
[73] codetools_0.2-16              XML_3.99-0.5                 
[75] glue_1.4.1                    BiocVersion_3.12.0           
[77] evaluate_0.14                 BiocManager_1.30.10          
[79] vctrs_0.3.2                   httpuv_1.5.4                 
[81] gtable_0.3.0                  purrr_0.3.4                  
[83] assertthat_0.2.1              xfun_0.16                    
[85] rsvd_1.0.3                    mime_0.9                     
[87] xtable_1.8-4                  RSpectra_0.16-0              
[89] later_1.1.0.1                 viridisLite_0.3.0            
[91] tibble_3.0.3                  AnnotationDbi_1.51.3         
[93] beeswarm_0.2.3                memoise_1.1.0                
[95] statmod_1.4.34                bluster_0.99.1               
[97] ellipsis_0.3.1                interactiveDisplayBase_1.27.5
```
</div>
