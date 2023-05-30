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
输入命令：
```
STAR --runThreadN 4 \
     --genomeDir mm10/star_index \
     --readFilesCommand zcat \
     --outFileNamePrefix split-seq/star_outs/ \
     --readFilesIn split-seq/data/SRR6750042_1.fastq.gz split-seq/data/SRR6750042_2.fastq.gz \
     --soloType CB_UMI_Complex \
     --soloCBposition 0_10_0_17 0_48_0_55 0_86_0_93 \
     --soloUMIposition 0_0_0_9 \
     --soloCBwhitelist split-seq/data/round3_whitelist.txt split-seq/data/round2_whitelist.txt split-seq/data/round1_whitelist.txt \
     --soloCBmatchWLtype 1MM \
     --soloCellFilter EmptyDrops_CR \
     --soloStrand Forward \
     --outSAMattributes CB UB \
     --outSAMtype BAM SortedByCoordinate
```

## 相关命令解释
`--runThreadN 4`
>>>采用四个线程运行程序.

`--genomeDir mm10/star_index`
>>>指向star的索引的目录。本实验的数据来自上述论文的公开数据小鼠大脑的单细胞测序。

`--readFilesCommand zcat`
>>>fastq文件是.gz的被压缩状态，所以用zcat来提取内容。

`--outFileNamePrefix split-seq/star_outs/`
>>>指向输出文件目录。所有的输出内容都将被放入split-seq/star_outs/目录下。

`--readFilesIn split-seq/data/SRR6750042_1.fastq.gz split-seq/data/SRR6750042_2.fastq.gz`
>>>我们应该在这里放两个文件，第一个文件是来自 cDNA 的读数，第二个文件应包含细胞条形码和 UMI。在SPLiT-seq中，cDNA reads 来自 Read 1，cell barcode 和 UMI 来自 Read 2。

`--soloType CB_UMI_Complex`
>>>由于 Read 2 不仅有细胞条形码和 UMI，还有常见的 linker sequences。细胞条形码是非连续的，由连接序列分隔。在这种情况下，我们须使用CB_UMI_Complex选项。

`--soloCBposition和--soloUMIposition`
>>>这些选项指定了我们传给的第二个 fastq 文件中细胞条形码和 UMI 的位置。

`--soloCBwhitelist`
>>>细胞条形码由三个不连续的部分组成：三轮条形码。这里的白名单是这三个列表的组合。我们应该按照指定的顺序分别提供它们，并star负责组合。

`--soloCBmatchWLtype 1MM`
>>>设置细胞条形码reads与白名单匹配的严格程度。简单起见这里设置为1MM。

`--soloCellFilter EmptyDrops_CR`
>>>实验从来都不是完美的。即使是从空液滴中我们也能获得一些读数。一般来说，这些空液滴的读数数量应该比带有细胞的液滴小几个数量级。为了识别出真正的细胞，可以应用不同的算法。在这里我们使用最常用的参数EmptyDrops_CR。

`--soloStrand Forward`
>>>该参数的选择取决于 cDNA 读数的来源。如果 cDNA 读数来自与 mRNA 相同的链（编码链），则此参数将为Forward（这是默认值）。如果它们来自与 mRNA 相反的链，通常称为第一链，则此参数将为Reverse。对于SPLiT-seq，cDNA 读数来自 Read 1 文件。在实验过程中，mRNA 分子被包含 UMI 的条形码 oligo-dT 引物捕获，随后 Illumina Read 2 序列将连接到这一端。因此，Read 2 由 RT 条形码和 UMI 组成。它们来自第一链，与编码链互补。Read 1 来自编码链。因此，对SPLiT -seq数据应使用Forward。

`--outSAMattributes CB UB`
>>>我们希望在输出的属性中分别包含细胞条形码CB和UMI序列UB，这些信息将对下游分析有很大帮助。

`--outSAMtype BAM SortedByCoordinate`
>>>对输出的sam格式进行bam处理以便其他程序处理。

如果程序顺利执行，输出的目录格式应该如下：
```
test/split-seq/
├── data
│   ├── round1_whitelist.txt
│   ├── round2_whitelist.txt
│   ├── round3_whitelist.txt
│   ├── SPLiT-seq_Round1_bc.csv
│   ├── SPLiT-seq_Round2_bc.csv
│   ├── SPLiT-seq_Round3_bc.csv
│   ├── SRR6750042_1.fastq.gz
│   └── SRR6750042_2.fastq.gz
└── star_outs
    ├── Aligned.sortedByCoord.out.bam
    ├── Log.final.out
    ├── Log.out
    ├── Log.progress.out
    ├── SJ.out.tab
    └── Solo.out
        ├── Barcodes.stats
        └── Gene
            ├── Features.stats
            ├── filtered
            │   ├── barcodes.tsv
            │   ├── features.tsv
            │   └── matrix.mtx
            ├── raw
            │   ├── barcodes.tsv
            │   ├── features.tsv
            │   └── matrix.mtx
            ├── Summary.csv
            └── UMIperCellSorted.txt

```




## 利用seruat进行下一步分析

