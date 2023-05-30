# sci-RNA-seq3
处理sci-RNA-seq3数据
sci-RNA-seq3简介：

## 采用的实验数据
本文采用测序样本数据来源于 61 个小鼠的胚胎细胞。参考基因组选用的是 mm10。经过测序后的 read1 文件存储的是 9 或 10 个 bp 的发夹条形码加一段， “CAGAGC”序列，8 个 bp 的 UMI 和 10 个 bp 的 RT 条形码，read2 存储的是 cDNA。
本实验使用来自以下论文的sci-RNA-seq3数据：
```
Cao J, Spielmann M, Qiu X, Huang X, Ibrahim DM, Hill AJ, Zhang F, Mundlos S, Christiansen L, Steemers FJ, Trapnell C, Shendure J (2019) 哺乳动物器官发生的单细胞转录景观。 自然566:496–502。https://doi.org/10.1038/s41586-019-0969-x
```
fastq文件可以直接从[ENA数据库](https://www.ebi.ac.uk/ena/browser/view/PRJNA490754?show=reads) 获取。

```
#新建并下载实验数据到sci-rna-seq3/文件夹
mkdir -p sci-rna-seq3/data
wget -P sci-rna-seq3/data \
    ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR782/006/SRR7827206/SRR7827206_1.fastq.gz \
    ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR782/006/SRR7827206/SRR7827206_2.fastq.gz
```

## 准备白名单

完整的核苷酸序列可以在sci-RNA-seq3论文的补充表 S11中找到。共有 384 个不同的 10 bp RT 条条形码码、384 个不同的 9 或 10 bp发夹条码。
可以用以下命令下载已经经过整理的条形码序列。
```
wget -P sci-rna-seq3/data \
    https://teichlab.github.io/scg_lib_structs/data/sci-RNA-seq3_RT_bc.csv \
    https://teichlab.github.io/scg_lib_structs/data/sci-RNA-seq3_hairpin_bc.csv
```
现在我们需要生成这两种条形码的白名单。 可以通过以下代码生成：
```
# hairpin barcode whitelist
tail -n +2 sci-rna-seq3/data/sci-RNA-seq3_hairpin_bc.csv | \
    cut -f 2 -d, > sci-rna-seq3/data/hairpin_whitelist.txt

# RT barcode whitelist
tail -n +2 sci-rna-seq3/data/sci-RNA-seq3_RT_bc.csv | \
    cut -f 2 -d, > sci-rna-seq3/data/RT_whitelist.txt
```

## 从FastQ到计数矩阵
输入命令：
```
STAR --runThreadN 4 \
     --genomeDir mm10/star_index \
     --readFilesCommand zcat \
     --outFileNamePrefix sci-rna-seq3/star_outs/ \
     --readFilesIn sci-rna-seq3/data/SRR7827206_2.fastq.gz sci-rna-seq3/data/SRR7827206_1.fastq.gz \
     --soloType CB_UMI_Complex \
     --soloAdapterSequence CAGAGC \
     --soloCBposition 0_0_2_-1 3_9_3_18 \
     --soloUMIposition 3_1_3_8 \
     --soloCBwhitelist sci-rna-seq3/data/hairpin_whitelist.txt sci-rna-seq3/data/RT_whitelist.txt \
     --soloCBmatchWLtype 1MM \
     --soloCellFilter EmptyDrops_CR \
     --soloStrand Forward \
     --outSAMattributes CB UB \
     --outSAMtype BAM SortedByCoordinate
```

## 相关命令解释
`--runThreadN 4`
>>>采用四个线程运行程序。

`--genomeDir mm10/star_index`
>>>指向star的索引的目录。本实验的数据来自上述论文的公开数据小鼠胚胎的单细胞测序。

`--readFilesCommand zcat`
>>>fastq文件是.gz的被压缩状态，所以用zcat来提取内容。

`--outFileNamePrefix sci-rna-seq3/star_outs/`
>>>指向输出文件目录。所有的输出内容都将被放入sci-rna-seq3/star_outs/目录下。

`--readFilesIn sci-rna-seq3/data/SRR7827206_2.fastq.gz sci-rna-seq3/data/SRR7827206_1.fastq.gz`
>>>第一个文件应该来自 cDNA 的读数，第二个文件应包含细胞条形码和 UMI。在sci-RNA-seq3中，cDNA reads 来自 Read 2，cell barcode 和 UMI 来自 Read 1。

`--soloType CB_UMI_Complex`
>>>由于 Read 1 不仅有细胞条形码和 UMI，还有linker sequences。细胞条形码是非连续的，由linker sequences分隔。在这种情况下，我们需要使用该CB_UMI_Complex选项。

`--soloAdapterSequence CAGAGC`
>>>Read 1开头发夹条形码是可变长度（9 或 10 bp），这会使得情况变复杂，因为每次读取中RT 条形码和UMI的绝对位置会有所不同。然而，通过指定一个adapter序列，就可以使用这个序列作为锚点，并告诉程序细胞条形码和 UMI 相对于锚点的位置。在这里中间的恒定连接子序列`CAGAGC`将发夹条形码和UMI分开。

`--soloCBposition和--soloUMIposition`
>>>这些选项指定了我们传递给的第二个 fastq 文件中单元格条形码和 UMI 的位置。由于 9 或 10 bp发夹条形码，中间的RT 条形码和UMI的绝对位置是可变的。因此，使用 Read start 作为锚点对他们不起作用。我们需要使用adapter作为锚点，并指定相对于锚点的位置。

`--soloCBwhitelist sci-rna-seq3/data/hairpin_whitelist.txt sci-rna-seq3/data/RT_whitelist.txt`
>>>由于细胞条码由两个不连续的部分组成：发夹条码和RT条码，所以这里的白名单是两个子文件的组合。我们应该按照指定的顺序分别提供它们，并star负责组合。

`--soloCBmatchWLtype 1MM`
>>>设置细胞条形码reads与白名单匹配的严格程度。简单起见这里设置为1MM。

`--soloCellFilter EmptyDrops_CR`
>>>实验从来都不是完美的。即使是从空液滴中我们也能获得一些读数。一般来说，这些空液滴的读数数量应该比带有细胞的液滴小几个数量级。为了识别出真正的细胞，可以应用不同的算法。在这里我们使用最常用的参数EmptyDrops_CR。

`--soloStrand Forward`
>>>该参数的选择取决于 cDNA 读数的来源。如果 cDNA 读数来自与 mRNA 相同的链（编码链），则此参数将为Forward（这是默认值）。如果它们来自与 mRNA 相反的链，通常称为第一链，则此参数将为Reverse。在sci-RNA-seq3的情况下，cDNA 读数来自 Read 2 文件。在实验过程中，mRNA 分子被包含 UMI 和 Illumina Read 1 序列的带条形码的 oligo-dT 引物捕获。因此，Read 1 由 RT 条形码和 UMI 组成。它们来自第一链，与编码链互补。Read 2 来自编码链。因此，对于sci-RNA-seq3数据因使用Forward。

`--outSAMattributes CB UB`
>>>我们希望在输出的属性中分别包含细胞条形码CB和UMI序列UB，这些信息将对下游分析有很大帮助。

`--outSAMtype BAM SortedByCoordinate`
>>>对输出的sam格式进行bam处理以便其他程序处理。

如果程序顺利执行，输出的目录格式应该如下：

```
test/sci-rna-seq3/
├── data
│   ├── hairpin_whitelist.txt
│   ├── RT_whitelist.txt
│   ├── sci-RNA-seq3_hairpin_bc.csv
│   ├── sci-RNA-seq3_RT_bc.csv
│   ├── SRR7827206_1.fastq.gz
│   └── SRR7827206_2.fastq.gz
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
