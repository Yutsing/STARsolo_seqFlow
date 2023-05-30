# SPLiT-seq	
处理SPLiT-seq	x数据
SPLiT-seq	简介：

## 采用的实验数据
本文采用测序样本数据来源于出生后第 2 天的 11 个小鼠大脑和脊髓的 156,049 个单核转录组，数据来源于[ENA数据库](https://www.ebi.ac.uk/ena/browser/view/PRJNA434658?show=reads) 。参考基因组选用的是 mm10。经过测序后的 read1 文件 存储的是 cDNA，read2 文件包含 3 个长度为 8bp 的细胞条形码和长度为 10bp 的 UMI。在第一轮细胞条形码与第二轮细胞条形码之间，第二轮条形码与第三轮条形 码之间，分别还有一段 30bp 的其他序列。

我们使用来自以下论文的SPLiT-seq数据：
```
Rosenberg AB, Roco CM, Muscat RA, Kuchina A, Sample P, Yao Z, Gray L, Peeler DJ, Mukherjee S, Chen W, Pun SH, Sellers DL, Tasic B, Seelig G (2018) Single-cell profiling of the developing mouse brain and spinal cord with split-pool barcoding. Science 360:eaam8999. https://doi.org/10.1126/science.aam8999
```
我们使用论文中的SRR6750042数据进行方法展示。
```
#新建并下载实验数据到split-seq文件夹
mkdir -p mkdir -p split-seq/data
wget -P split-seq/data -c \
    ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR675/002/SRR6750042/SRR6750042_1.fastq.gz \
    ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR675/002/SRR6750042/SRR6750042_2.fastq.gz
```

## 准备白名单
完整的寡核苷酸序列可以在SPLiT-seq论文的补充表S12中找到。共有 96 个不同的Round2（8bp） 条码和 96 个不同的Round3（8bp） 条码。96个中Round1条形码前 48 个是 oligo-dT 引物，后 48 个是随机六聚体，它们被随机混合到48个孔中。因此，我们实际上有 48 个不同的Round1_barcodes。SPLiT-seq论文的补充表 S12中提供的寡核苷酸，应该具有48 * 96 * 96 * 8 = 3,538,944种组合。
可以通过以下代码下载本文中整理好的round1，round2，round3序列。
```
wget -P split-seq/data \
    https://teichlab.github.io/scg_lib_structs/data/SPLiT-seq_Round1_bc.csv \
    https://teichlab.github.io/scg_lib_structs/data/SPLiT-seq_Round2_bc.csv \
    https://teichlab.github.io/scg_lib_structs/data/SPLiT-seq_Round3_bc.csv
```
现在我们需要生成这三轮条形码的白名单。
可以通过以下代码生成：
```
tail -n +2 split-seq/data/SPLiT-seq_Round1_bc.csv | \
    cut -f 2 -d, > split-seq/data/round1_whitelist.txt

tail -n +2 split-seq/data/SPLiT-seq_Round2_bc.csv | \
    cut -f 2 -d, > split-seq/data/round2_whitelist.txt

tail -n +2 split-seq/data/SPLiT-seq_Round3_bc.csv | \
    cut -f 2 -d, > split-seq/data/round3_whitelist.txt
```


## 从FastQ到计数矩阵


## 相关命令解释

## 利用seruat进行下一步分析

