# Lawlor human pancreas (SMARTer)

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

This performs an analysis of the @lawlor2017singlecell dataset,
consisting of human pancreas cells from various donors.

## Data loading


```r
library(scRNAseq)
sce.lawlor <- LawlorPancreasData()
```


```r
library(AnnotationHub)
edb <- AnnotationHub()[["AH73881"]]
anno <- select(edb, keys=rownames(sce.lawlor), keytype="GENEID", 
    columns=c("SYMBOL", "SEQNAME"))
rowData(sce.lawlor) <- anno[match(rownames(sce.lawlor), anno[,1]),-1]
```

## Quality control


```r
unfiltered <- sce.lawlor
```


```r
library(scater)
stats <- perCellQCMetrics(sce.lawlor, 
    subsets=list(Mito=which(rowData(sce.lawlor)$SEQNAME=="MT")))
qc <- quickPerCellQC(stats, percent_subsets="subsets_Mito_percent",
    batch=sce.lawlor$`islet unos id`)
sce.lawlor <- sce.lawlor[,!qc$discard]
```


```r
colData(unfiltered) <- cbind(colData(unfiltered), stats)
unfiltered$discard <- qc$discard

gridExtra::grid.arrange(
    plotColData(unfiltered, x="islet unos id", y="sum", colour_by="discard") +
        scale_y_log10() + ggtitle("Total count") +
        theme(axis.text.x = element_text(angle = 90)),
    plotColData(unfiltered, x="islet unos id", y="detected", 
        colour_by="discard") + scale_y_log10() + ggtitle("Detected features") +
        theme(axis.text.x = element_text(angle = 90)), 
    plotColData(unfiltered, x="islet unos id", y="subsets_Mito_percent",
        colour_by="discard") + ggtitle("Mito percent") +
        theme(axis.text.x = element_text(angle = 90)),
    ncol=2
)
```

<div class="figure">
<img src="lawlor-pancreas_files/figure-html/unref-lawlor-qc-dist-1.png" alt="Distribution of each QC metric across cells from each donor of the Lawlor pancreas dataset. Each point represents a cell and is colored according to whether that cell was discarded." width="960" />
<p class="caption">(\#fig:unref-lawlor-qc-dist)Distribution of each QC metric across cells from each donor of the Lawlor pancreas dataset. Each point represents a cell and is colored according to whether that cell was discarded.</p>
</div>


```r
plotColData(unfiltered, x="sum", y="subsets_Mito_percent",
    colour_by="discard") + scale_x_log10()
```

<div class="figure">
<img src="lawlor-pancreas_files/figure-html/unref-lawlor-qc-comp-1.png" alt="Percentage of mitochondrial reads in each cell in the 416B dataset compared to the total count. Each point represents a cell and is colored according to whether that cell was discarded." width="672" />
<p class="caption">(\#fig:unref-lawlor-qc-comp)Percentage of mitochondrial reads in each cell in the 416B dataset compared to the total count. Each point represents a cell and is colored according to whether that cell was discarded.</p>
</div>


```r
colSums(as.matrix(qc))
```

```
##              low_lib_size            low_n_features high_subsets_Mito_percent 
##                         9                         5                        25 
##                   discard 
##                        34
```

## Normalization


```r
library(scran)
set.seed(1000)
clusters <- quickCluster(sce.lawlor)
sce.lawlor <- computeSumFactors(sce.lawlor, clusters=clusters)
sce.lawlor <- logNormCounts(sce.lawlor)
```


```r
summary(sizeFactors(sce.lawlor))
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##   0.295   0.781   0.963   1.000   1.182   2.629
```


```r
plot(librarySizeFactors(sce.lawlor), sizeFactors(sce.lawlor), pch=16,
    xlab="Library size factors", ylab="Deconvolution factors", log="xy")
```

<div class="figure">
<img src="lawlor-pancreas_files/figure-html/unref-lawlor-norm-1.png" alt="Relationship between the library size factors and the deconvolution size factors in the Lawlor pancreas dataset." width="672" />
<p class="caption">(\#fig:unref-lawlor-norm)Relationship between the library size factors and the deconvolution size factors in the Lawlor pancreas dataset.</p>
</div>

## Variance modelling

Using age as a proxy for the donor.


```r
dec.lawlor <- modelGeneVar(sce.lawlor, block=sce.lawlor$`islet unos id`)
chosen.genes <- getTopHVGs(dec.lawlor, n=2000)
```


```r
par(mfrow=c(4,2))
blocked.stats <- dec.lawlor$per.block
for (i in colnames(blocked.stats)) {
    current <- blocked.stats[[i]]
    plot(current$mean, current$total, main=i, pch=16, cex=0.5,
        xlab="Mean of log-expression", ylab="Variance of log-expression")
    curfit <- metadata(current)
    curve(curfit$trend(x), col='dodgerblue', add=TRUE, lwd=2)
}
```

<div class="figure">
<img src="lawlor-pancreas_files/figure-html/unnamed-chunk-4-1.png" alt="Per-gene variance as a function of the mean for the log-expression values in the Lawlor pancreas dataset. Each point represents a gene (black) with the mean-variance trend (blue) fitted separately for each donor." width="672" />
<p class="caption">(\#fig:unnamed-chunk-4)Per-gene variance as a function of the mean for the log-expression values in the Lawlor pancreas dataset. Each point represents a gene (black) with the mean-variance trend (blue) fitted separately for each donor.</p>
</div>

## Dimensionality reduction


```r
library(BiocSingular)
set.seed(101011001)
sce.lawlor <- runPCA(sce.lawlor, subset_row=chosen.genes, ncomponents=25)
sce.lawlor <- runTSNE(sce.lawlor, dimred="PCA")
```

## Clustering


```r
snn.gr <- buildSNNGraph(sce.lawlor, use.dimred="PCA")
colLabels(sce.lawlor) <- factor(igraph::cluster_walktrap(snn.gr)$membership)
```


```r
table(colLabels(sce.lawlor), sce.lawlor$`cell type`)
```

```
##    
##     Acinar Alpha Beta Delta Ductal Gamma/PP None/Other Stellate
##   1      1     0    0    13      2       16          2        0
##   2      0     1   76     1      0        0          0        0
##   3      0   161    1     0      0        1          2        0
##   4      0     1    0     1      0        0          5       19
##   5      0     0  175     4      1        0          1        0
##   6     22     0    0     0      0        0          0        0
##   7      0    75    0     0      0        0          0        0
##   8      0     0    0     1     20        0          2        0
```


```r
table(colLabels(sce.lawlor), sce.lawlor$`islet unos id`)
```

```
##    
##     ACCG268 ACCR015A ACEK420A ACEL337 ACHY057 ACIB065 ACIW009 ACJV399
##   1       8        2        2       4       4       4       9       1
##   2      14        3        2      33       3       2       4      17
##   3      36       23       14      13      14      14      21      30
##   4       7        1        0       1       0       4       9       4
##   5      34       10        4      39       7      23      24      40
##   6       0        2       13       0       0       0       5       2
##   7      32       12        0       5       6       7       4       9
##   8       1        1        2       1       2       1      12       3
```


```r
gridExtra::grid.arrange(
    plotTSNE(sce.lawlor, colour_by="label"),
    plotTSNE(sce.lawlor, colour_by="islet unos id"),
    ncol=2
)
```

<div class="figure">
<img src="lawlor-pancreas_files/figure-html/unref-grun-tsne-1.png" alt="Obligatory $t$-SNE plots of the Lawlor pancreas dataset. Each point represents a cell that is colored by cluster (left) or batch (right)." width="672" />
<p class="caption">(\#fig:unref-grun-tsne)Obligatory $t$-SNE plots of the Lawlor pancreas dataset. Each point represents a cell that is colored by cluster (left) or batch (right).</p>
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
 [1] BiocSingular_1.5.0          scran_1.17.15              
 [3] scater_1.17.4               ggplot2_3.3.2              
 [5] ensembldb_2.13.1            AnnotationFilter_1.13.0    
 [7] GenomicFeatures_1.41.2      AnnotationDbi_1.51.3       
 [9] AnnotationHub_2.21.2        BiocFileCache_1.13.1       
[11] dbplyr_1.4.4                scRNAseq_2.3.12            
[13] SingleCellExperiment_1.11.6 SummarizedExperiment_1.19.6
[15] DelayedArray_0.15.7         matrixStats_0.56.0         
[17] Matrix_1.2-18               Biobase_2.49.0             
[19] GenomicRanges_1.41.6        GenomeInfoDb_1.25.10       
[21] IRanges_2.23.10             S4Vectors_0.27.12          
[23] BiocGenerics_0.35.4         BiocStyle_2.17.0           
[25] simpleSingleCell_1.13.16   

loaded via a namespace (and not attached):
  [1] Rtsne_0.15                    ggbeeswarm_0.6.0             
  [3] colorspace_1.4-1              ellipsis_0.3.1               
  [5] scuttle_0.99.12               bluster_0.99.1               
  [7] XVector_0.29.3                BiocNeighbors_1.7.0          
  [9] farver_2.0.3                  bit64_4.0.2                  
 [11] interactiveDisplayBase_1.27.5 codetools_0.2-16             
 [13] knitr_1.29                    Rsamtools_2.5.3              
 [15] graph_1.67.1                  shiny_1.5.0                  
 [17] BiocManager_1.30.10           compiler_4.0.2               
 [19] httr_1.4.2                    dqrng_0.2.1                  
 [21] assertthat_0.2.1              fastmap_1.0.1                
 [23] lazyeval_0.2.2                limma_3.45.10                
 [25] later_1.1.0.1                 htmltools_0.5.0              
 [27] prettyunits_1.1.1             tools_4.0.2                  
 [29] igraph_1.2.5                  rsvd_1.0.3                   
 [31] gtable_0.3.0                  glue_1.4.1                   
 [33] GenomeInfoDbData_1.2.3        dplyr_1.0.1                  
 [35] rappdirs_0.3.1                Rcpp_1.0.5                   
 [37] vctrs_0.3.2                   Biostrings_2.57.2            
 [39] ExperimentHub_1.15.1          rtracklayer_1.49.5           
 [41] DelayedMatrixStats_1.11.1     xfun_0.16                    
 [43] stringr_1.4.0                 ps_1.3.4                     
 [45] mime_0.9                      lifecycle_0.2.0              
 [47] irlba_2.3.3                   statmod_1.4.34               
 [49] XML_3.99-0.5                  edgeR_3.31.4                 
 [51] zlibbioc_1.35.0               scales_1.1.1                 
 [53] hms_0.5.3                     promises_1.1.1               
 [55] ProtGenerics_1.21.0           yaml_2.2.1                   
 [57] curl_4.3                      memoise_1.1.0                
 [59] gridExtra_2.3                 biomaRt_2.45.2               
 [61] stringi_1.4.6                 RSQLite_2.2.0                
 [63] highr_0.8                     BiocVersion_3.12.0           
 [65] BiocParallel_1.23.2           rlang_0.4.7                  
 [67] pkgconfig_2.0.3               bitops_1.0-6                 
 [69] evaluate_0.14                 lattice_0.20-41              
 [71] purrr_0.3.4                   labeling_0.3                 
 [73] GenomicAlignments_1.25.3      CodeDepends_0.6.5            
 [75] cowplot_1.0.0                 bit_4.0.4                    
 [77] processx_3.4.3                tidyselect_1.1.0             
 [79] magrittr_1.5                  bookdown_0.20                
 [81] R6_2.4.1                      generics_0.0.2               
 [83] DBI_1.1.0                     pillar_1.4.6                 
 [85] withr_2.2.0                   RCurl_1.98-1.2               
 [87] tibble_3.0.3                  crayon_1.3.4                 
 [89] rmarkdown_2.3                 viridis_0.5.1                
 [91] progress_1.2.2                locfit_1.5-9.4               
 [93] grid_4.0.2                    blob_1.2.1                   
 [95] callr_3.4.3                   digest_0.6.25                
 [97] xtable_1.8-4                  httpuv_1.5.4                 
 [99] openssl_1.4.2                 munsell_0.5.0                
[101] beeswarm_0.2.3                viridisLite_0.3.0            
[103] vipor_0.4.5                   askpass_1.1                  
```
</div>
