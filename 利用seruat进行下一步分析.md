# 利用R中的Seurat对count matrix画图

UMAP: 是一种降维技术，可用于类似于 t-SNE 的可视化，但也可用于一般的非线性降维。

count matrix: raw data经过starsolo 处理后，在star_outs/Solo.out/Gene/filtered的文件夹中有三个文件：barcodes.tsv features.tsv matrix.mtx。将这三个文件压缩成barcodes.tsv.gz features.tsv.gz matrix.mtx.gz后，这三个文件分别储存着条形码文件，特征文件，以及表达矩阵文件。在R中用Seurat进行处理。

下面代码以处理10X数据为例。

**读取文件**
```
library(dplyr)
library(Seurat)
library(patchwork)


pbmc.data <- Read10X(data.dir = "/Users/memewarrior/Desktop/毕业设计/pbmc10x")
pbmc <- CreateSeuratObject(counts = pbmc.data, project = "pbmc3k", min.cells = 3, min.features = 30)
pbmc
```
**Quality Control (QC) and Filter Out of Cells**

质量控制对于消除单细胞库制备过程中产生的噪音至关重要。例如，空的液滴导致检测到的基因很少；细胞双胞胎或多胞胎会导致基因数量异常的增加。细胞碎片和垂死的细胞会导致大量的线体污染。考虑到这些参数在该数据集中的分布，我们保留了具有超过200个独特特征/基因计数（nFeature_RNA）的细胞，小于26,000个UMI计数（nCount_RNA），以及小于15%的线粒体计数：
使用 PercentageFeatureSet() 函数计算线粒体 QC 指标，该函数计算源自一组特征的计数百分比  
我们使用以 MT- 开头的所有基因的集合作为一组线粒体基因。
```
pbmc[["percent.mt"]] <- PercentageFeatureSet(pbmc, pattern = "^MT-")
plot1 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "percent.mt")
plot2 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "nFeature_RNA")
plot1 + plot2
pbmc <- subset(pbmc, subset = nFeature_RNA > 200 & nFeature_RNA < 2500 & percent.mt < 5)
```

**Normalizing the data**

```
pbmc <- NormalizeData(pbmc, normalization.method = "LogNormalize", scale.factor = 10000)
pbmc <- NormalizeData(pbmc)
pbmc <- FindVariableFeatures(pbmc, selection.method = "vst", nfeatures = 2000)
VariableFeaturePlot(pbmc)
```

**scanling**

使每个基因的权重相同，使得高表达的基因在以后的分析中不占优势。使用ScaleData()
`pbmc <- ScaleData(pbmc, features = rownames(pbmc))`

**PCA降维**

```
pbmc <- RunPCA(pbmc, features = VariableFeatures(object = pbmc))
VizDimLoadings(pbmc, dims = 1:2, reduction = "pca")
```
`DimPlot(pbmc, reduction = "pca")`

**Cluster the cells**

```
pbmc <- FindNeighbors(pbmc, dims = 1:10)
pbmc <- FindClusters(pbmc, resolution = 0.5)
```

**umap**

```
pbmc <- RunUMAP(pbmc, dims = 1:10)
DimPlot(pbmc, reduction = "umap")
```







