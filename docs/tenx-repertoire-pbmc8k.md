# Human PBMC with surface proteins (10X Genomics)

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

Here, we describe a brief analysis of _yet another_ peripheral blood mononuclear cell (PBMC) dataset from 10X Genomics [@zheng2017massively].
Data are publicly available from the [10X Genomics website](https://support.10xgenomics.com/single-cell-vdj/datasets/3.0.0/vdj_v1_mm_c57bl6_pbmc_5gex), from which we download the filtered gene/barcode count matrices for gene expression and cell surface proteins.
Note that most of the repertoire-related steps will be discussed in Chapter \@ref(repertoire-seq), this workflow mostly provides the baseline analysis for the expression data.

## Data loading


```r
library(BiocFileCache)
bfc <- BiocFileCache(ask=FALSE)
exprs.data <- bfcrpath(bfc, file.path(
    "http://cf.10xgenomics.com/samples/cell-vdj/3.1.0",
    "vdj_v1_hs_pbmc3",
    "vdj_v1_hs_pbmc3_filtered_feature_bc_matrix.tar.gz"))
untar(exprs.data, exdir=tempdir())

library(DropletUtils)
sce.pbmc <- read10xCounts(file.path(tempdir(), "filtered_feature_bc_matrix"))
sce.pbmc <- splitAltExps(sce.pbmc, rowData(sce.pbmc)$Type)
```

## Quality control


```r
unfiltered <- sce.pbmc
```

We discard cells with high mitochondrial proportions and few detectable ADT counts.


```r
library(scater)
is.mito <- grep("^MT-", rowData(sce.pbmc)$Symbol)
stats <- perCellQCMetrics(sce.pbmc, subsets=list(Mito=is.mito))

high.mito <- isOutlier(stats$subsets_Mito_percent, type="higher")
low.adt <- stats$`altexps_Antibody Capture_detected` < nrow(altExp(sce.pbmc))/2

discard <- high.mito | low.adt
sce.pbmc <- sce.pbmc[,!discard]
```

We examine some of the statistics:


```r
summary(high.mito)
```

```
##    Mode   FALSE    TRUE 
## logical    6660     571
```

```r
summary(low.adt)
```

```
##    Mode   FALSE 
## logical    7231
```

```r
summary(discard)
```

```
##    Mode   FALSE    TRUE 
## logical    6660     571
```

We examine the distribution of each QC metric (Figure \@ref(fig:unref-pbmc-adt-qc)).


```r
colData(unfiltered) <- cbind(colData(unfiltered), stats)
unfiltered$discard <- discard

gridExtra::grid.arrange(
    plotColData(unfiltered, y="sum", colour_by="discard") +
        scale_y_log10() + ggtitle("Total count"),
    plotColData(unfiltered, y="detected", colour_by="discard") +
        scale_y_log10() + ggtitle("Detected features"),
    plotColData(unfiltered, y="subsets_Mito_percent",
        colour_by="discard") + ggtitle("Mito percent"),
    plotColData(unfiltered, y="altexps_Antibody Capture_detected",
        colour_by="discard") + ggtitle("ADT detected"),
    ncol=2
)
```

<div class="figure">
<img src="tenx-repertoire-pbmc8k_files/figure-html/unref-pbmc-adt-qc-1.png" alt="Distribution of each QC metric in the PBMC dataset, where each point is a cell and is colored by whether or not it was discarded by the outlier-based QC approach." width="672" />
<p class="caption">(\#fig:unref-pbmc-adt-qc)Distribution of each QC metric in the PBMC dataset, where each point is a cell and is colored by whether or not it was discarded by the outlier-based QC approach.</p>
</div>

We also plot the mitochondrial proportion against the total count for each cell, as one does (Figure \@ref(fig:unref-pbmc-adt-qc-mito)).


```r
plotColData(unfiltered, x="sum", y="subsets_Mito_percent",
    colour_by="discard") + scale_x_log10()
```

<div class="figure">
<img src="tenx-repertoire-pbmc8k_files/figure-html/unref-pbmc-adt-qc-mito-1.png" alt="Percentage of UMIs mapped to mitochondrial genes against the totalcount for each cell." width="672" />
<p class="caption">(\#fig:unref-pbmc-adt-qc-mito)Percentage of UMIs mapped to mitochondrial genes against the totalcount for each cell.</p>
</div>

## Normalization

Computing size factors for the gene expression and ADT counts.


```r
library(scran)

set.seed(1000)
clusters <- quickCluster(sce.pbmc)
sce.pbmc <- computeSumFactors(sce.pbmc, cluster=clusters)
altExp(sce.pbmc) <- computeMedianFactors(altExp(sce.pbmc))
sce.pbmc <- logNormCounts(sce.pbmc, use_altexps=TRUE)
```

We generate some summary statistics for both sets of size factors:


```r
summary(sizeFactors(sce.pbmc))
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##   0.074   0.719   0.908   1.000   1.133   8.858
```

```r
summary(sizeFactors(altExp(sce.pbmc)))
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##    0.10    0.70    0.83    1.00    1.03  227.36
```

We also look at the distribution of size factors compared to the library size for each set of features (Figure \@ref(fig:unref-norm-pbmc-adt)).


```r
par(mfrow=c(1,2))
plot(librarySizeFactors(sce.pbmc), sizeFactors(sce.pbmc), pch=16,
    xlab="Library size factors", ylab="Deconvolution factors", 
    main="Gene expression", log="xy")
plot(librarySizeFactors(altExp(sce.pbmc)), sizeFactors(altExp(sce.pbmc)), pch=16,
    xlab="Library size factors", ylab="Median-based factors", 
    main="Antibody capture", log="xy")
```

<div class="figure">
<img src="tenx-repertoire-pbmc8k_files/figure-html/unref-norm-pbmc-adt-1.png" alt="Plot of the deconvolution size factors for the gene expression values (left) or the median-based size factors for the ADT expression values (right) compared to the library size-derived factors for the corresponding set of features. Each point represents a cell." width="672" />
<p class="caption">(\#fig:unref-norm-pbmc-adt)Plot of the deconvolution size factors for the gene expression values (left) or the median-based size factors for the ADT expression values (right) compared to the library size-derived factors for the corresponding set of features. Each point represents a cell.</p>
</div>

## Dimensionality reduction

We omit the PCA step for the ADT expression matrix, given that it is already so low-dimensional,
and progress directly to $t$-SNE and UMAP visualizations.


```r
set.seed(100000)
altExp(sce.pbmc) <- runTSNE(altExp(sce.pbmc))

set.seed(1000000)
altExp(sce.pbmc) <- runUMAP(altExp(sce.pbmc))
```

## Clustering

We perform graph-based clustering on the ADT data and use the assignments as the column labels of the alternative Experiment.


```r
g.adt <- buildSNNGraph(altExp(sce.pbmc), k=10, d=NA)
clust.adt <- igraph::cluster_walktrap(g.adt)$membership
colLabels(altExp(sce.pbmc)) <- factor(clust.adt)
```

We examine some basic statistics about the size of each cluster, 
their separation (Figure \@ref(fig:unref-clustmod-pbmc-adt))
and their distribution in our $t$-SNE plot (Figure \@ref(fig:unref-tsne-pbmc-adt)).


```r
table(colLabels(altExp(sce.pbmc)))
```

```
## 
##    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16 
##  160  507  662   39  691 1415   32  650   76 1037  121   47   68   25   15  562 
##   17   18   19   20   21   22   23   24 
##  139   32   44  120   84   65   52   17
```


```r
mod <- clusterModularity(g.adt, clust.adt, as.ratio=TRUE)
library(pheatmap)
pheatmap::pheatmap(log10(mod + 10), cluster_row=FALSE, cluster_col=FALSE,
    color=colorRampPalette(c("white", "blue"))(101))
```

<div class="figure">
<img src="tenx-repertoire-pbmc8k_files/figure-html/unref-clustmod-pbmc-adt-1.png" alt="Heatmap of the pairwise cluster modularity scores in the PBMC dataset, computed based on the shared nearest neighbor graph derived from the ADT expression values." width="672" />
<p class="caption">(\#fig:unref-clustmod-pbmc-adt)Heatmap of the pairwise cluster modularity scores in the PBMC dataset, computed based on the shared nearest neighbor graph derived from the ADT expression values.</p>
</div>


```r
plotTSNE(altExp(sce.pbmc), colour_by="label", text_by="label", text_col="red")
```

<div class="figure">
<img src="tenx-repertoire-pbmc8k_files/figure-html/unref-tsne-pbmc-adt-1.png" alt="Obligatory $t$-SNE plot of PBMC dataset based on its ADT expression values, where each point is a cell and is colored by the cluster of origin. Cluster labels are also overlaid at the median coordinates across all cells in the cluster." width="672" />
<p class="caption">(\#fig:unref-tsne-pbmc-adt)Obligatory $t$-SNE plot of PBMC dataset based on its ADT expression values, where each point is a cell and is colored by the cluster of origin. Cluster labels are also overlaid at the median coordinates across all cells in the cluster.</p>
</div>

We perform some additional subclustering using the expression data to mimic an _in silico_ FACS experiment.


```r
set.seed(1010010)
subclusters <- quickSubCluster(sce.pbmc, clust.adt,
    prepFUN=function(x) {
        dec <- modelGeneVarByPoisson(x)
        top <- getTopHVGs(dec, prop=0.1)
        denoisePCA(x, dec, subset.row=top)
    }, 
    clusterFUN=function(x) {
        g.gene <- buildSNNGraph(x, k=10, use.dimred = 'PCA')
        igraph::cluster_walktrap(g.gene)$membership
    }
)
```

We counting the number of gene expression-derived subclusters in each ADT-derived parent cluster.


```r
data.frame(
    Cluster=names(subclusters),
    Ncells=vapply(subclusters, ncol, 0L),
    Nsub=vapply(subclusters, function(x) length(unique(x$subcluster)), 0L)
)
```

```
##    Cluster Ncells Nsub
## 1        1    160    3
## 2        2    507    4
## 3        3    662    5
## 4        4     39    1
## 5        5    691    5
## 6        6   1415    7
## 7        7     32    1
## 8        8    650    7
## 9        9     76    2
## 10      10   1037    8
## 11      11    121    2
## 12      12     47    1
## 13      13     68    2
## 14      14     25    1
## 15      15     15    1
## 16      16    562    9
## 17      17    139    3
## 18      18     32    1
## 19      19     44    1
## 20      20    120    4
## 21      21     84    3
## 22      22     65    2
## 23      23     52    3
## 24      24     17    1
```

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
 [1] pheatmap_1.0.12             scran_1.17.15              
 [3] scater_1.17.4               ggplot2_3.3.2              
 [5] DropletUtils_1.9.10         SingleCellExperiment_1.11.6
 [7] SummarizedExperiment_1.19.6 DelayedArray_0.15.7        
 [9] matrixStats_0.56.0          Matrix_1.2-18              
[11] Biobase_2.49.0              GenomicRanges_1.41.6       
[13] GenomeInfoDb_1.25.10        IRanges_2.23.10            
[15] S4Vectors_0.27.12           BiocGenerics_0.35.4        
[17] BiocFileCache_1.13.1        dbplyr_1.4.4               
[19] BiocStyle_2.17.0            simpleSingleCell_1.13.16   

loaded via a namespace (and not attached):
 [1] bitops_1.0-6              bit64_4.0.2              
 [3] RColorBrewer_1.1-2        RcppAnnoy_0.0.16         
 [5] httr_1.4.2                tools_4.0.2              
 [7] R6_2.4.1                  irlba_2.3.3              
 [9] vipor_0.4.5               HDF5Array_1.17.3         
[11] uwot_0.1.8                DBI_1.1.0                
[13] colorspace_1.4-1          rhdf5filters_1.1.2       
[15] withr_2.2.0               gridExtra_2.3            
[17] tidyselect_1.1.0          processx_3.4.3           
[19] bit_4.0.4                 curl_4.3                 
[21] compiler_4.0.2            graph_1.67.1             
[23] BiocNeighbors_1.7.0       labeling_0.3             
[25] bookdown_0.20             scales_1.1.1             
[27] callr_3.4.3               rappdirs_0.3.1           
[29] stringr_1.4.0             digest_0.6.25            
[31] rmarkdown_2.3             R.utils_2.9.2            
[33] XVector_0.29.3            pkgconfig_2.0.3          
[35] htmltools_0.5.0           highr_0.8                
[37] limma_3.45.10             rlang_0.4.7              
[39] RSQLite_2.2.0             DelayedMatrixStats_1.11.1
[41] farver_2.0.3              generics_0.0.2           
[43] BiocParallel_1.23.2       dplyr_1.0.1              
[45] R.oo_1.23.0               RCurl_1.98-1.2           
[47] magrittr_1.5              BiocSingular_1.5.0       
[49] GenomeInfoDbData_1.2.3    scuttle_0.99.12          
[51] Rcpp_1.0.5                ggbeeswarm_0.6.0         
[53] munsell_0.5.0             Rhdf5lib_1.11.3          
[55] viridis_0.5.1             lifecycle_0.2.0          
[57] R.methodsS3_1.8.0         stringi_1.4.6            
[59] yaml_2.2.1                edgeR_3.31.4             
[61] zlibbioc_1.35.0           Rtsne_0.15               
[63] rhdf5_2.33.7              grid_4.0.2               
[65] blob_1.2.1                dqrng_0.2.1              
[67] crayon_1.3.4              lattice_0.20-41          
[69] cowplot_1.0.0             locfit_1.5-9.4           
[71] CodeDepends_0.6.5         knitr_1.29               
[73] ps_1.3.4                  pillar_1.4.6             
[75] igraph_1.2.5              codetools_0.2-16         
[77] XML_3.99-0.5              glue_1.4.1               
[79] evaluate_0.14             BiocManager_1.30.10      
[81] vctrs_0.3.2               gtable_0.3.0             
[83] purrr_0.3.4               assertthat_0.2.1         
[85] xfun_0.16                 rsvd_1.0.3               
[87] RSpectra_0.16-0           viridisLite_0.3.0        
[89] tibble_3.0.3              beeswarm_0.2.3           
[91] memoise_1.1.0             statmod_1.4.34           
[93] bluster_0.99.1            ellipsis_0.3.1           
```
</div>
