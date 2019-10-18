# Installation of ArchR

```r

if (!requireNamespace("devtools", quietly = TRUE))
	install.packages("devtools")

if (!requireNamespace("BiocManager", quietly = TRUE))
	install.packages("BiocManager")

devtools::install_github("jgranja24/ArchR",
	auth_token = token, #please email me for a token
	repos = BiocManager::repositories()
)

```

# Additional Packages that are used from github


```r

# ggrastr is a package for plotting ggplots with rastr'd points which
# is super helpful for large UMAPs etc
# 
# You need to have Cairo for ggrastr
#
# On Mac OSx you need to have XQuartz (https://www.xquartz.org/)
#
devtools::install_github('VPetukhov/ggrastr')

# harmony is a package that can correct batch effects
devtools::install_github("immunogenomics/harmony")

# presto is a package that has efficient tools for wilcoxon tests on sparseMatrices
devtools::install_github('immunogenomics/presto')


```

# ArchR PBMC Tutorial

```r

#Input Libraries
library(ArchR)

#Set/Create Working Directory to Folder
#setwd("~/Documents/ArchR_Tutorial/")

#Download PBMC Tutorial Fragments
if(!file.exists("PBMC_Tutorial_fragments.tsv.gz")){
  download.file(
    url = "https://jeffgranja.s3.amazonaws.com/ArchR-Tutorial-Data/PBMC/PBMC_Tutorial_fragments.tsv.gz", 
    destfile = "PBMC_Tutorial_fragments.tsv.gz"
  )
}

#Load Genome Annotations
data("geneAnnoHg19")
data("genomeAnnoHg19")

#Set Threads to be used
threads <- 4

#Input Files
inputFiles <- c(
	"PBMC" = "PBMC_Tutorial_fragments.tsv.gz"
)

#Create Arrow Files
ArrowFiles <- createArrowFiles(
	inputFiles = inputFiles,
	sampleNames = names(inputFiles),
	outputNames = names(inputFiles),
	geneAnno = geneAnnoHg19,
	genomeAnno = genomeAnnoHg19,
	threads = threads,
  force = TRUE
)

#Add Infered Doublet Scores to each Arrow File
doubScores <- addDoubletScores(ArrowFiles, threads = threads)

#Create ArchRProject
proj <- ArchRProject(
  ArrowFiles = ArrowFiles, 
  genomeAnnotation = genomeAnnoHg19,
  geneAnnotation = geneAnnoHg19,
  outputDirectory = "PBMC_Tutorial",
)

#Filter Cells by TSS, Number of Fragments and Doublet Score
proj <- FilterCells(proj, list("TSSEnrichment" = 6, "nFrags" = 1000, "DoubletEnrichment" = c(-Inf, 3)))

#Reduce Dimensions with Iterative LSI
proj <- IterativeLSI(
  ArchRProj = proj, 
  useMatrix = "TileMatrix", 
  reducedDimsOut = "IterativeLSI",
  threads = threads
)

#Identify Clusters from Iterative LSI
proj <- IdentifyClusters(input = proj, reducedDims = "IterativeLSI", resolution = 0.4)

#Compute a UMAP embedding to visualize our tiled matrix
proj <- ComputeEmbedding(ArchRProj = proj, reducedDims = "IterativeLSI", embedding = "UMAP")

#Plot the UMAP Embedding with Metadata/GeneScores Overlayed
plotPDF("Plot-UMAP-TileLSI", width = 6, height = 6, ArchRProj = proj)
VisualizeEmbedding(ArchRProj = proj, colorBy = "colData", name = "Sample")
VisualizeEmbedding(ArchRProj = proj, colorBy = "colData", name = "Clusters")
VisualizeEmbedding(ArchRProj = proj, colorBy = "colData", name = "TSSEnrichment")
VisualizeEmbedding(ArchRProj = proj, colorBy = "GeneScoreMatrix", name = "CD8A")
VisualizeEmbedding(ArchRProj = proj, colorBy = "GeneScoreMatrix", name = "CD4")
VisualizeEmbedding(ArchRProj = proj, colorBy = "GeneScoreMatrix", name = "MS4A1")
dev.off()

#Plot Tracks at gene loci
plotPDF("Plot-Tracks", width = 6, height = 8, ArchRProj = proj)
ArchRRegionTrack(ArchRProj = proj, threads = threads, geneSymbol="CD4")
ArchRRegionTrack(ArchRProj = proj, threads = threads, geneSymbol="MS4A1")
ArchRRegionTrack(ArchRProj = proj, threads = threads, geneSymbol="CD8A")
dev.off()

#Create Group Coverage Files that can be used for downstream analysis
proj <- addGroupCoverages(ArchRProj = proj, groupBy = "Clusters", threads = threads)

#Call Peaks w/ Macs2
proj <- addReproduciblePeakSet(
  proj, 
  groupBy = "Clusters", 
  pathToMacs2 = "macs2",
  threads = threads
)

#Add Peak Matrix
proj <- addPeakMatrix(proj, threads = threads, force = TRUE)

#Motif Search in PeakSet
proj <- addMotifAnnotations(proj)

#Compute ChromVAR-Based Motif Deviations
proj <- addDeviationsMatrix(proj, threads = threads)

##################################
# Identifying Marker Features Across The Clusters
##################################

#Identify Marker Gene Scores
markersGS <- markerFeatures(proj, threads = threads, useMatrix = "GeneScoreMatrix")

plotPDF("GS-Marker-Heatmap", width = 8, height = 12, ArchRProj = proj)
markerHeatmap(markersGS, labelMarkers = c("CD4", "CD8A", "MS4A1", "CD3D", "CD56", "NKG7"))
dev.off()

#Identify Marker Peaks
markersPeaks <- markerFeatures(proj, threads = threads, useMatrix = "PeakMatrix")

plotPDF("Peak-Marker-Heatmap", width = 8, height = 12, ArchRProj = proj)
markerHeatmap(markersPeaks)
dev.off()

saveRDS(proj, "PBMC-Tutorial-ArchR-Project.rds")

```
