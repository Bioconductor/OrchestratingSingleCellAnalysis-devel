---
output: html_document
bibliography: ref.bib
---

# Interactive data exploration {#interactive-sharing}

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



## Motivation

Exploratory data analysis (EDA) and visualization are crucial for many aspects of data analysis such as quality control, hypothesis generation and contextual result interpretation.
Single-cell 'omics datasets generated with modern high-throughput technologies are no exception, especially given their increasing size and complexity.
The need for flexible and interactive platforms to explore those data from various perspectives has contributed to the increasing popularity of graphical user interfaces (GUIs) for interactive visualization.

In this chapter, we illustrate how the Bioconductor package *[iSEE](https://bioconductor.org/packages/3.12/iSEE)* can be used to perform some common exploratory tasks during single-cell analysis workflows.
We note that these are examples only; in practice, EDA is often context-dependent and driven by distinct motivations and hypotheses for every new data set.
To this end, `iSEE` provides a flexible framework that is immediately compatible with a wide range of genomics data modalities and can be easily customized to focus on key aspects of individual data sets.

## Quick start {#interactive-quickstart}

An instance of an interactive `iSEE` application can be launched with any data set that is stored in an object of the `SummarizedExperiment` class (or any class that extends it, e.g., `SingleCellExperiment`, `DESeqDataSet`, `MethylSet`).
In its simplest form, this is done simply by calling `iSEE(sce)` with the `sce` data object as the sole argument, as demonstrated here with the 10X PBMC dataset (Figure \@ref(fig:iSEE-default)).

<button class="aaron-collapse">View history</button>
<div class="aaron-content">
   
```r
#--- loading ---#
library(BiocFileCache)
bfc <- BiocFileCache("raw_data", ask = FALSE)
raw.path <- bfcrpath(bfc, file.path("http://cf.10xgenomics.com/samples",
    "cell-exp/2.1.0/pbmc4k/pbmc4k_raw_gene_bc_matrices.tar.gz"))
untar(raw.path, exdir=file.path(tempdir(), "pbmc4k"))

library(DropletUtils)
fname <- file.path(tempdir(), "pbmc4k/raw_gene_bc_matrices/GRCh38")
sce.pbmc <- read10xCounts(fname, col.names=TRUE)

#--- gene-annotation ---#
library(scater)
rownames(sce.pbmc) <- uniquifyFeatureNames(
    rowData(sce.pbmc)$ID, rowData(sce.pbmc)$Symbol)

library(EnsDb.Hsapiens.v86)
location <- mapIds(EnsDb.Hsapiens.v86, keys=rowData(sce.pbmc)$ID, 
    column="SEQNAME", keytype="GENEID")

#--- cell-detection ---#
set.seed(100)
e.out <- emptyDrops(counts(sce.pbmc))
sce.pbmc <- sce.pbmc[,which(e.out$FDR <= 0.001)]

#--- quality-control ---#
stats <- perCellQCMetrics(sce.pbmc, subsets=list(Mito=which(location=="MT")))
high.mito <- isOutlier(stats$subsets_Mito_percent, type="higher")
sce.pbmc <- sce.pbmc[,!high.mito]

#--- normalization ---#
library(scran)
set.seed(1000)
clusters <- quickCluster(sce.pbmc)
sce.pbmc <- computeSumFactors(sce.pbmc, cluster=clusters)
sce.pbmc <- logNormCounts(sce.pbmc)

#--- variance-modelling ---#
set.seed(1001)
dec.pbmc <- modelGeneVarByPoisson(sce.pbmc)
top.pbmc <- getTopHVGs(dec.pbmc, prop=0.1)

#--- dimensionality-reduction ---#
set.seed(10000)
sce.pbmc <- denoisePCA(sce.pbmc, subset.row=top.pbmc, technical=dec.pbmc)

set.seed(100000)
sce.pbmc <- runTSNE(sce.pbmc, dimred="PCA")

set.seed(1000000)
sce.pbmc <- runUMAP(sce.pbmc, dimred="PCA")

#--- clustering ---#
g <- buildSNNGraph(sce.pbmc, k=10, use.dimred = 'PCA')
clust <- igraph::cluster_walktrap(g)$membership
colLabels(sce.pbmc) <- factor(clust)
```

</div>


```r
library(iSEE)
app <- iSEE(sce.pbmc)
```

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_default.png" alt="Screenshot of the _iSEE_ application with its default initialization."  />
<p class="caption">(\#fig:iSEE-default)Screenshot of the _iSEE_ application with its default initialization.</p>
</div>

The default interface contains up to eight built-in panels, each displaying a particular aspect of the data set. 
The layout of panels in the interface may be altered interactively - panels can be added, removed, resized or repositioned using the "Organize panels" menu in the top right corner of the interface. 
The initial layout of the application can also be altered programmatically as described in the rest of this Chapter.

To familiarize themselves with the GUI, users can launch an interactive tour from the menu in the top right corner.
In addition, custom tours can be written to substitute the default built-in tour.
This feature is particularly useful to disseminate new data sets with accompanying bespoke explanations guiding users through the salient features of any given data set (see Section \@ref{dissemination}).

It is also possible to deploy "empty" instances of `iSEE` apps, where any `SummarizedExperiment` object stored in an RDS file may be uploaded to the running application.
Once the file is uploaded, the application will import the `sce` object and initialize the GUI panels with the contents of the object for interactive exploration.
This type of `iSEE` applications is launched without specifying the `sce` argument, as shown in Figure \@ref(fig:iSEE-landing).


```r
app <- iSEE()
```

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_landing.png" alt="Screenshot of the _iSEE_ application with a landing page."  />
<p class="caption">(\#fig:iSEE-landing)Screenshot of the _iSEE_ application with a landing page.</p>
</div>

## Usage examples {#isee-examples}

### Quality control

In this example, we demonstrate that an `iSEE` app can be configured to focus on quality control metrics.
Here, we are interested in two plots:

- The library size of each cell in decreasing order.
An elbow in this plot generally reveals the transition between good quality cells and low quality cells or empty droplets.
- A dimensionality reduction result (in this case, we will pick $t$-SNE) where cells are colored by the log-library size.
This view identifies trajectories or clusters associated with library size and can be used to diagnose QC/normalization problems.
Alternatively, it could also indicate the presence of multiple cell types or states that differ in total RNA content.

In addition, by setting the `ColumnSelectionSource` parmaeter, any point selection made in the _Column data plot_ panel will highlight the corresponding points in the _Reduced dimension plot_ panel.
A user can then select the cells with either large or small library sizes to inspect their distribution in low-dimensional space.


```r
copy.pbmc <- sce.pbmc

# Computing various QC metrics; in particular, the log10-transformed library
# size for each cell and the log-rank by decreasing library size.
library(scater)
copy.pbmc <- addPerCellQC(copy.pbmc, exprs_values="counts")
copy.pbmc$log10_total_counts <- log10(copy.pbmc$total)
copy.pbmc$total_counts_rank <- rank(-copy.pbmc$total)

initial.state <- list(
    # Configure a "Column data plot" panel
    ColumnDataPlot(YAxis="log10_total_counts",
        XAxis="Column data",
        XAxisColumnData="total_counts_rank",
        DataBoxOpen=TRUE,
        PanelId=1L),

    # Configure a "Reduced dimension plot " panel
    ReducedDimensionPlot(
        Type="TSNE",
        VisualBoxOpen=TRUE,
        DataBoxOpen=TRUE,
        ColorBy="Column data",
        ColorByColumnData="log10_total_counts",
        SelectionBoxOpen=TRUE,
        ColumnSelectionSource="ColumnDataPlot1")
)

# Prepare the app
app <- iSEE(copy.pbmc, initial=initial.state)
```

The configured Shiny app can then be launched with the `runApp()` function or by simply printing the `app` object (Figure \@ref(fig:iSEE-qc)).

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_qc.png" alt="Screenshot of an _iSEE_ application for interactive exploration of quality control metrics."  />
<p class="caption">(\#fig:iSEE-qc)Screenshot of an _iSEE_ application for interactive exploration of quality control metrics.</p>
</div>

This app remains fully interactive, i.e., users can interactively control the settings and layout of the panels.
For instance, users may choose to color data points by percentage of UMI mapped to mitochondrial genes (`"pct_counts_Mito"`) in the _Reduced dimension plot_.
Using the transfer of point selection between panels, users could select cells with small library sizes in the _Column data plot_ and highlight them in the _Reduced dimension plot_, to investigate a possible relation between library size, clustering and proportion of reads mapped to mitochondrial genes.

### Annotation of cell populations

In this example, we use `iSEE` to interactively examine the marker genes to conveniently determine cell identities.
We identify upregulated markers in each cluster (Chapter \@ref(marker-detection)) and collect the log-$p$-value for each gene in each cluster.
These are stored in the `rowData` slot of the `SingleCellExperiment` object for access by `iSEE`.


```r
copy.pbmc <- sce.pbmc

library(scran)
markers.pbmc.up <- findMarkers(copy.pbmc, direction="up", 
    log.p=TRUE, sorted=FALSE)

# Collate the log-p-value for each marker in a single table
all.p <- lapply(markers.pbmc.up, FUN = "[[", i="log.p.value")
all.p <- DataFrame(all.p, check.names=FALSE)
colnames(all.p) <- paste0("cluster", colnames(all.p))

# Store the table of results as row metadata
rowData(copy.pbmc) <- cbind(rowData(copy.pbmc), all.p)
```

The next code chunk sets up an app that contains:

1. A table of feature statistics, including the log-transformed FDR of cluster markers computed above.
2. A plot showing the distribution of expression values for a chosen gene in each cluster.
3. A plot showing the result of the UMAP dimensionality reduction method overlaid with the expression value of a chosen gene.

Moreover, we configure the second and third panel to use the gene (i.e., row) selected in the first panel.
This enables convenient examination of important markers when combined with sorting by $p$-value for a cluster of interest.


```r
initial.state <- list(
    RowDataTable(PanelId=1L),

    # Configure a "Feature assay plot" panel
    FeatureAssayPlot(
        YAxisFeatureSource="RowDataTable1",
        XAxis="Column data",
        XAxisColumnData="label",
        Assay="logcounts",
        DataBoxOpen=TRUE
    ),

    # Configure a "Reduced dimension plot" panel
    ReducedDimensionPlot(
        Type="UMAP",
        ColorBy="Feature name",
        ColorByFeatureSource="RowDataTable1",
        ColorByFeatureNameAssay="logcounts"
    )
)

# Prepare the app
app <- iSEE(copy.pbmc, initial=initial.state)
```

After launching the application (Figure \@ref(fig:iSEE-anno)), we can then sort the table by ascending values of `cluster1` to identify genes that are strong markers for cluster 1.
Then, users may select the first row in the _Row statistics table_ and watch the second and third panel automatically update to display the most significant marker gene on the y-axis (_Feature assay plot_) or as a color scale overlaid on the data points (_Reduced dimension plot_).
Alternatively, users can simply search the table for arbitrary gene names and select known markers for visualization.

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_anno.png" alt="Screenshot of the _iSEE_ application initialized for interactive exploration of population-specific marker expression."  />
<p class="caption">(\#fig:iSEE-anno)Screenshot of the _iSEE_ application initialized for interactive exploration of population-specific marker expression.</p>
</div>

### Querying features of interest

So far, the plots that we have examined have represented each column (i.e., cell) as a point.
However, it is straightforward to instead represent rows as points that can be selected and transmitted to eligible panels.
This is useful for more gene-centric exploratory analyses.
To illustrate, we will add variance modelling statistics to the `rowData()` of our `SingleCellExperiment` object.


```r
copy.pbmc <- sce.pbmc

# Adding some mean-variance information.
dec <- modelGeneVarByPoisson(copy.pbmc)
rowData(copy.pbmc) <- cbind(rowData(copy.pbmc), dec) 
```

The next code chunk sets up an app (Figure \@ref(fig:iSEE-hvg)) that contains:

1. A plot showing the mean-variance trend, where each point represents a cell.
2. A table of feature statistics, similar to that generated in the previous example.
3. A heatmap for the genes in the first plot.

We again configure the second and third panels to respond to the selection of points in the first panel.
This allows the user to select several highly variable genes at once and examine their statistics or expression profiles.
More advanced users can even configure the app to start with a brush or lasso to define a selection of genes at initialization.


```r
initial.state <- list(
    # Configure a "Feature assay plot" panel
    RowDataPlot(
        YAxis="total",
        XAxis="Row data",
        XAxisRowData="mean",
        PanelId=1L
    ),

    RowDataTable(
        RowSelectionSource="RowDataPlot1"
    ),

    # Configure a "ComplexHeatmap" panel
    ComplexHeatmapPlot(
        RowSelectionSource="RowDataPlot1",
        CustomRows=FALSE,
        ColumnData="label",
        Assay="logcounts",
        ClusterRows=TRUE,
        PanelHeight=800L,
        AssayCenterRows=TRUE
    )
)

# Prepare the app
app <- iSEE(copy.pbmc, initial=initial.state)
```

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_hvg.png" alt="Screenshot of the _iSEE_ application initialized for examining highly variable genes."  />
<p class="caption">(\#fig:iSEE-hvg)Screenshot of the _iSEE_ application initialized for examining highly variable genes.</p>
</div>

It is entirely possible for these row-centric panels to exist alongside the column-centric panels discussed previously.
The only limitation is that row-based panels cannot transmit multi-row selections to column-based panels and vice versa.
That said, a row-based panel can still transmit a single row selection to a column-based panel for, e.g., coloring by expression;
this allows us to set up an app where selecting a single HVG in the mean-variance plot causes the neighboring $t$-SNE to be colored by the expression of the selected gene (Figure \@ref(fig:iSEE-rowcol)).


```r
initial.state <- list(
    # Configure a "Feature assay plot" panel
    RowDataPlot(
        YAxis="total",
        XAxis="Row data",
        XAxisRowData="mean",
        PanelId=1L
    ),

    # Configure a "Reduced dimension plot" panel
    ReducedDimensionPlot(
        Type="TSNE",
        ColorBy="Feature name",
        ColorByFeatureSource="RowDataPlot1",
        ColorByFeatureNameAssay="logcounts"
    )
)

# Prepare the app
app <- iSEE(copy.pbmc, initial=initial.state)
```

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_rowcol.png" alt="Screenshot of the _iSEE_ application containing both row- and column-based panels."  />
<p class="caption">(\#fig:iSEE-rowcol)Screenshot of the _iSEE_ application containing both row- and column-based panels.</p>
</div>

## Reproducible visualizations

The state of the `iSEE` application can be saved at any point to provide a snapshot of the current view of the dataset.
This is achieved by clicking on the "Display panel settings" button under the "Export" dropdown menu in the top right corner and saving an RDS file containing a serialized list of panel parameters.
Anyone with access to this file and the original `SingleCellExperiment` can then run `iSEE` to recover the same application state. 
Alternatively, the code required to construct the panel parameters can be returned, which is more transparent and amenable to further modification.
This facility is most obviously useful for reproducing a perspective on the data that leads to a particular scientific conclusion;
it is also helpful for collaborations whereby different views of the same dataset can be easily transferred between analysts.

`iSEE` also keeps a record of the R commands used to generate each figure and table in the app.
This information is readily available via the "Extract the R code" button under the "Export" dropdown menu.
By copying the code displayed in the modal window and executing it in the R session from which the `iSEE` app was launched, a user can exactly reproduce all plots currently displayed in the GUI.
In this manner, a user can use `iSEE` to rapidly prototype plots of interest without having to write the associated boilerplate, after which they can then copy the code in an R script for fine-tuning.
Of course, the user can also save the plots and tables directly for further adjustment with other tools.

## Dissemination of analysis results {#dissemination}

`iSEE` provides a powerful avenue for disseminating results through a "guided tour" of the dataset.
This involves writing a step-by-step walkthrough of the different panels with explanations to facilitate their interpretation.
All that is needed to add a tour to an `iSEE` instance is a data frame with two columns named "element" and "intro"; the first column declares the UI element to highlight in each step of the tour, and the second one contains the text to display at that step.
This data frame must then be provided to the `iSEE()` function via the `tour` argument.
Below we demonstrate the implementation of a simple tour that takes users through the two panels that compose a GUI and trains them to use the collapsible boxes.


```r
tour <- data.frame(
    element = c(
        "#Welcome",
        "#ReducedDimensionPlot1",
        "#ColumnDataPlot1",
        "#ColumnDataPlot1_DataBoxOpen",
        "#Conclusion"),
    intro = c(
        "Welcome to this tour!",
        "This is a <i>Reduced dimension plot.</i>",
        "And this is a <i>Column data plot.</i>",
        "<b>Action:</b> Click on this collapsible box to open and close it.",
        "Thank you for taking this tour!"),
    stringsAsFactors = FALSE)

initial.state <- list(
    ReducedDimensionPlot(PanelWidth=6L), 
    ColumnDataPlot(PanelWidth=6L)
)
```

The preconfigured Shiny app can then be loaded with the tour and launched to obtain Figure \@ref(fig:iSEE-tour).
Note that the viewer is free to leave the interactive tour at any time and explore the data from their own perspective.
Examples of advanced tours showcasing a selection of published data sets can be found at https://github.com/iSEE/iSEE2018.


```r
app <- iSEE(sce.pbmc, initial = initial.state, tour = tour)
```

<div class="figure">
<img src="https://raw.githubusercontent.com/Bioconductor/OSCABase/images/images/iSEE_tour.png" alt="Screenshot of the _iSEE_ application initialized with a tour."  />
<p class="caption">(\#fig:iSEE-tour)Screenshot of the _iSEE_ application initialized with a tour.</p>
</div>

## Additional resources

For demonstration and inspiration, we refer readers to the following examples of deployed applications:

- Use cases accompanying the published article: https://marionilab.cruk.cam.ac.uk/ (source code: https://github.com/iSEE/iSEE2018)
- Examples of `iSEE` in production: http://www.teichlab.org/singlecell-treg 
- Other examples as source code:
    - Gallery of examples notebooks to reproduce analyses on public data: https://github.com/iSEE/iSEE_instances
    - Gallery of example custom panels: https://github.com/iSEE/iSEE_custom 

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
 [3] ggplot2_3.3.2               iSEE_2.1.12                
 [5] SingleCellExperiment_1.11.6 SummarizedExperiment_1.19.6
 [7] DelayedArray_0.15.7         matrixStats_0.56.0         
 [9] Matrix_1.2-18               Biobase_2.49.0             
[11] GenomicRanges_1.41.6        GenomeInfoDb_1.25.10       
[13] IRanges_2.23.10             S4Vectors_0.27.12          
[15] BiocGenerics_0.35.4         BiocStyle_2.17.0           
[17] simpleSingleCell_1.13.16   

loaded via a namespace (and not attached):
 [1] ggbeeswarm_0.6.0          colorspace_1.4-1         
 [3] rjson_0.2.20              ellipsis_0.3.1           
 [5] circlize_0.4.10           scuttle_0.99.12          
 [7] bluster_0.99.1            XVector_0.29.3           
 [9] GlobalOptions_0.1.2       BiocNeighbors_1.7.0      
[11] clue_0.3-57               DT_0.15                  
[13] codetools_0.2-16          splines_4.0.2            
[15] knitr_1.29                jsonlite_1.7.0           
[17] cluster_2.1.0             png_0.1-7                
[19] shinydashboard_0.7.1      graph_1.67.1             
[21] shiny_1.5.0               BiocManager_1.30.10      
[23] compiler_4.0.2            dqrng_0.2.1              
[25] fastmap_1.0.1             limma_3.45.10            
[27] later_1.1.0.1             BiocSingular_1.5.0       
[29] htmltools_0.5.0           tools_4.0.2              
[31] rsvd_1.0.3                igraph_1.2.5             
[33] gtable_0.3.0              glue_1.4.1               
[35] GenomeInfoDbData_1.2.3    dplyr_1.0.1              
[37] Rcpp_1.0.5                vctrs_0.3.2              
[39] nlme_3.1-148              rintrojs_0.2.2           
[41] DelayedMatrixStats_1.11.1 xfun_0.16                
[43] stringr_1.4.0             ps_1.3.4                 
[45] mime_0.9                  miniUI_0.1.1.1           
[47] lifecycle_0.2.0           irlba_2.3.3              
[49] statmod_1.4.34            XML_3.99-0.5             
[51] shinyAce_0.4.1            edgeR_3.31.4             
[53] zlibbioc_1.35.0           scales_1.1.1             
[55] colourpicker_1.0          promises_1.1.1           
[57] RColorBrewer_1.1-2        ComplexHeatmap_2.5.5     
[59] yaml_2.2.1                gridExtra_2.3            
[61] stringi_1.4.6             highr_0.8                
[63] BiocParallel_1.23.2       shape_1.4.4              
[65] rlang_0.4.7               pkgconfig_2.0.3          
[67] bitops_1.0-6              evaluate_0.14            
[69] lattice_0.20-41           purrr_0.3.4              
[71] CodeDepends_0.6.5         htmlwidgets_1.5.1        
[73] processx_3.4.3            tidyselect_1.1.0         
[75] magrittr_1.5              bookdown_0.20            
[77] R6_2.4.1                  generics_0.0.2           
[79] pillar_1.4.6              withr_2.2.0              
[81] mgcv_1.8-31               RCurl_1.98-1.2           
[83] tibble_3.0.3              crayon_1.3.4             
[85] shinyWidgets_0.5.3        rmarkdown_2.3            
[87] viridis_0.5.1             GetoptLong_1.0.2         
[89] locfit_1.5-9.4            grid_4.0.2               
[91] callr_3.4.3               digest_0.6.25            
[93] xtable_1.8-4              httpuv_1.5.4             
[95] munsell_0.5.0             beeswarm_0.2.3           
[97] viridisLite_0.3.0         vipor_0.4.5              
[99] shinyjs_1.1              
```
</div>
