# pheniqs-splitseq-protocol

This repository contains results from processing the SRR6750056 SPLiTSeq experiment with a hybrid Pheniqs/STARsolo approach.
Command are assumed to be running on a machine with 16 cores.


The following are quoted from the [STAR manual](https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf).

```
nNinBarcode: number of reads with more than 2 Ns in cell barcode (CB)
nUMIhomopolymer: number of reads with homopolymer in CB
nNoMatch: number of reads with CBs that do not match whitelist even with one mismatch

```

```
Following reads are discarded from Solo output.
Remaining reads are checked for overlap with features.
nUnmapped: number of reads unmapped to the genome
nNoFeature: number of reads that map to the genome but do not belong to a feature
nAmbigFeature: number of reads that belong to more than one feature
nAmbigFeatureMultimap: number of reads that belong to more than one feature and are also multimapping to the genome (this is a subset of the nAmbigFeature)
nTooMany: number of reads with ambiguous CB (i.e. CB matches whitelist with one mismatch but with posterior probability ¡0.95)
nNoExactMatch: number of reads with CB that matches a whitelist barcode with 1 mismatch, but this whitelist barcode does not get any other reads with exact matches of CB
```

```
Following reads are output in feature (e.g. gene) / cell count matrices.
nExactMatch: number of reads with CB that match the whitelist exactly
nMatch: total number of reads that match CB with 0 or 1 mismatches (this is superset of nExactMatch)
nCellBarcodes: number of distinct CBs detected
nUMIs: number of distinct UMIs detected
```

```
nNinBarcode+nUMIhomopolymer+nNoMatch+nTooMany+nNoExactMatch = number of reads with CBs that do not match whitelist.
nUnmapped+nAmbigFeature = number of reads without defined feature (gene)
nMatch = number of reads that are output as solo counts
The three categoties above summed together should be equal to the total number of reads.
```
# STARsolo pipeline

## Configuration solo-A

This is a [Basic STARsolo configuration](https://github.com/biosails/pheniqs-splitseq-protocol/tree/main/usecase/solo/SRR6750056/A) for processing a SPLiTSeq experiment with the default strategy and produce the gene expression matrix.

```sh
STAR \
--runThreadN 16 \
--readFilesCommand bzcat \
--genomeDir /media/terminus/home/lg/split-seq-data/index/mm10_hg38 \
--soloFeatures Gene \
--readFilesPrefix /home/lg/split-seq-data/raw_fastq/ \
--readFilesIn SRR6750056_1.fastq.bz2 SRR6750056_2.fastq.bz2 \
--soloType CB_UMI_Complex \
--soloCBwhitelist /home/lg/code/moonwatcher/pheniqs-split-pool/raw/cb_whitelist \
/home/lg/code/moonwatcher/pheniqs-split-pool/raw/cb_whitelist \
/home/lg/code/moonwatcher/pheniqs-split-pool/raw/cb_whitelist \
--soloCBposition 0_10_0_17 0_48_0_55 0_86_0_93 \
--soloUMIposition 0_0_0_9 \
--soloCBmatchWLtype 1MM \
--outSAMtype BAM SortedByCoordinate \
--outSAMattributes CR CY UR UY GX GN CB UB sM sS sQ \
```

```sh
cat usecase/solo/SRR6750056/A/Solo.out/Gene/Features.stats

                                         nUnmapped       22655073
                                        nNoFeature      116973794
                                     nAmbigFeature        7457400
                             nAmbigFeatureMultimap        7431342
                                          nTooMany              0
                                     nNoExactMatch           1249
                                       nExactMatch        3442881
                                            nMatch       11316706
                                      nMatchUnique        3859306
                                     nCellBarcodes          73854
                                             nUMIs        1338194
```

```sh
cat solo/SRR6750056/A/Solo.out/Barcodes.stats
                                        nNoAdapter              0
                                            nNoUMI              0
                                             nNoCB              0
                                            nNinCB          64186
                                           nNinUMI         108727
                                   nUMIhomopolymer        3443228
                                          nTooMany              0
                                          nNoMatch       62544692
                               nMismatchesInMultCB        1575925
                                       nExactMatch      133336154
                                    nMismatchOneWL       17610668
                                 nMismatchToMultWL              0
```

# Hybrid STARSolo / Pheniqs pipeline

This configuration for processing a SPLiTSeq experiment decodes the cellular barcodes with Pheniqs PAMLD and feeds the output directly into STAR for further processing over a standard stream without a temporary file. All 3 cellular barcodes are written to the CB/CY tags, and UMIs to RX/QX. Pheniqs drops reads that where any of the cellular barcodes fail to decode so STARsolo will see a reduced number of input reads but reads will have error corrected cellular barcodes so STAR can use an exact match strategy.

Configuration `pheniqs-A` uses PAMLD with 0.99 `confidence threshold` and 0.05 `noise` prior in all 3 cellular decoders and the default STARsolo strategy and produce the gene expression matrix. Configuration `pheniqs-B` is similar but uses a 0.95 `confidence threshold` for cellular decoding. Configuration `pheniqs-C` uses the same configuration as `pheniqs-A` to estimate the barcode and noise priors for all 3 decoders and uses the adjusted configuration for when feeding decoded reads to STARSolo. Configuration `pheniqs-D` uses the same configuration as `pheniqs-B` to estimate the barcode and noise priors.

## Configuration pheniqs-A

This is a [STARsolo/Pheniqs hybrid configuration](https://github.com/biosails/pheniqs-splitseq-protocol/tree/main/usecase/pheniqs/SRR6750056/A) for processing a SPLiTSeq experiment with Pheniqs decoding the 3 cellular barcodes using PAMLD with 0.99 `confidence threshold` and 0.05 `noise` prior and the default STARsolo strategy and produce the gene expression matrix.

```sh
STAR \
--runThreadN 16 \
--readFilesCommand "pheniqs mux --threads 12 --config splitseq_decode.json --sense-input --input" \
--genomeDir /media/terminus/home/lg/split-seq-data/index/mm10_hg38 \
--soloFeatures Gene \
--readFilesIn /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--soloType CB_UMI_Simple \
--soloCBwhitelist None \
--soloBarcodeMate 2 \
--soloCBlen 24 \
--soloUMIlen 10 \
--soloUMIstart 25 \
--soloInputSAMattrBarcodeSeq CB RX \
--soloInputSAMattrBarcodeQual CY QX \
--soloCBmatchWLtype Exact \
--readFilesType SAM SE \
--outSAMtype BAM SortedByCoordinate \
--readFilesSAMattrKeep RX QX CB CR CY XC \
--outSAMattributes GX GN CB UB sM sS sQ \
```

```sh
cat pheniqs/SRR6750056/A/Solo.out/Gene/Features.stats

                                         nUnmapped       20218934
                                        nNoFeature      106847613
                                     nAmbigFeature        6920453
                             nAmbigFeatureMultimap        6896585
                                          nTooMany              0
                                     nNoExactMatch              0
                                       nExactMatch        3552197
                                            nMatch       10472650
                                      nMatchUnique        3552197
                                     nCellBarcodes          69728
                                             nUMIs        1265609
```

## Configuration pheniqs-B

This is a [STARsolo/Pheniqs hybrid configuration](https://github.com/biosails/pheniqs-splitseq-protocol/tree/main/usecase/pheniqs/SRR6750056/B) for processing a SPLiTSeq experiment with Pheniqs decoding the 3 cellular barcodes using PAMLD with 0.95 `confidence threshold` and 0.05 `noise` prior and the default STARsolo strategy and produce the gene expression matrix.

```sh
STAR \
--runThreadN 16 \
--readFilesCommand "pheniqs mux --threads 12 --config splitseq_decode.json --sense-input --input" \
--genomeDir /media/terminus/home/lg/split-seq-data/index/mm10_hg38 \
--soloFeatures Gene \
--readFilesIn /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--soloType CB_UMI_Simple \
--soloCBwhitelist None \
--soloBarcodeMate 2 \
--soloCBlen 24 \
--soloUMIlen 10 \
--soloUMIstart 25 \
--soloInputSAMattrBarcodeSeq CB RX \
--soloInputSAMattrBarcodeQual CY QX \
--soloCBmatchWLtype Exact \
--readFilesType SAM SE \
--outSAMtype BAM SortedByCoordinate \
--readFilesSAMattrKeep RX QX CB CR CY XC \
--outSAMattributes GX GN CB UB sM sS sQ \
```

```sh
cat pheniqs/SRR6750056/B/Solo.out/Gene/Features.stats

                                         nUnmapped       20335194
                                        nNoFeature      107246018
                                     nAmbigFeature        6945574
                             nAmbigFeatureMultimap        6921625
                                          nTooMany              0
                                     nNoExactMatch              0
                                       nExactMatch        3565063
                                            nMatch       10510637
                                      nMatchUnique        3565063
                                     nCellBarcodes          70192
                                             nUMIs        1270074
```

## Configuration pheniqs-C

This is a [STARsolo/Pheniqs hybrid configuration](https://github.com/biosails/pheniqs-splitseq-protocol/tree/main/usecase/pheniqs/SRR6750056/C) for processing a SPLiTSeq experiment with Pheniqs decoding the 3 cellular barcodes using PAMLD and estimated priors proceeded by the default STARsolo strategy to produce the gene expression matrix.

An initial pheniqs run with discarded output ( `--output /dev/null` ) uses the same configuration as `pheniqs-A` to estimate the barcode and noise priors.

```sh
pheniqs mux \
--threads 16 \
--config splitseq_decode.json \
--sense-input \
--input /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--output /dev/null \
--prior splitseq_adjusted.json
```

The result of this run is a new configuration file, [splitseq_adjusted.json](https://github.com/biosails/pheniqs-splitseq-protocol/blob/main/usecase/pheniqs/SRR6750056/C/splitseq_adjusted.json) that includes the imputed barcode and noise priors for all 3 cellular decoders. The bootstrapping configuration uses a 0.99 `confidence threshold` and 0.05 `noise` prior.

A subsequent run uses the new, prior adjusted, configuration file when executing pheniqs, overriding output to stdout (which would otherwise inherit the behavior from the initial run and redirect to `/dev/null`)

```sh
STAR \
--runThreadN 16 \
--readFilesCommand "pheniqs mux --threads 12 --config splitseq_adjusted.json --output /dev/stdout --sense-input --input" \
--genomeDir /media/terminus/home/lg/split-seq-data/index/mm10_hg38 \
--soloFeatures Gene \
--readFilesIn /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--soloType CB_UMI_Simple \
--soloCBwhitelist None \
--soloBarcodeMate 2 \
--soloCBlen 24 \
--soloUMIlen 10 \
--soloUMIstart 25 \
--soloInputSAMattrBarcodeSeq CB RX \
--soloInputSAMattrBarcodeQual CY QX \
--soloCBmatchWLtype Exact \
--readFilesType SAM SE \
--outSAMtype BAM SortedByCoordinate \
--readFilesSAMattrKeep RX QX CB CR CY XC \
--outSAMattributes GX GN CB UB sM sS sQ \
```

```sh
cat pheniqs/SRR6750056/C/Solo.out/Gene/Features.stats

                                         nUnmapped       19853238
                                        nNoFeature      105006515
                                     nAmbigFeature        6773231
                             nAmbigFeatureMultimap        6749822
                                          nTooMany              0
                                     nNoExactMatch              0
                                       nExactMatch        3490017
                                            nMatch       10263248
                                      nMatchUnique        3490017
                                     nCellBarcodes          68748
                                             nUMIs        1251339
```

## Configuration pheniqs-D

This is a [STARsolo/Pheniqs hybrid configuration](https://github.com/biosails/pheniqs-splitseq-protocol/tree/main/usecase/pheniqs/SRR6750056/D) for processing a SPLiTSeq experiment with Pheniqs decoding the 3 cellular barcodes using PAMLD and estimated priors proceeded by the default STARsolo strategy to produce the gene expression matrix.

An initial pheniqs run with discarded output ( `--output /dev/null` ) uses the same configuration as `pheniqs-A` to estimate the barcode and noise priors.

```sh
pheniqs mux \
--threads 16 \
--config splitseq_decode.json \
--sense-input \
--input /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--output /dev/null \
--prior splitseq_adjusted.json
```

The result of this run is a new configuration file, [splitseq_adjusted.json](https://github.com/biosails/pheniqs-splitseq-protocol/blob/main/usecase/pheniqs/SRR6750056/D/splitseq_adjusted.json) that includes the imputed barcode and noise priors for all 3 cellular decoders. The bootstrapping configuration uses a 0.95 `confidence threshold` and 0.05 `noise` prior.

A subsequent run uses the new, prior adjusted, configuration file when executing pheniqs, overriding output to stdout (which would otherwise inherit the behavior from the initial run and redirect to `/dev/null`)

```sh
STAR \
--runThreadN 16 \
--readFilesCommand "pheniqs mux --threads 12 --config splitseq_adjusted.json --output /dev/stdout --sense-input --input" \
--genomeDir /media/terminus/home/lg/split-seq-data/index/mm10_hg38 \
--soloFeatures Gene \
--readFilesIn /home/lg/split-seq-data/raw_cram/SRR6750056.cram \
--soloType CB_UMI_Simple \
--soloCBwhitelist None \
--soloBarcodeMate 2 \
--soloCBlen 24 \
--soloUMIlen 10 \
--soloUMIstart 25 \
--soloInputSAMattrBarcodeSeq CB RX \
--soloInputSAMattrBarcodeQual CY QX \
--soloCBmatchWLtype Exact \
--readFilesType SAM SE \
--outSAMtype BAM SortedByCoordinate \
--readFilesSAMattrKeep RX QX CB CR CY XC \
--outSAMattributes GX GN CB UB sM sS sQ \
```

```sh
cat pheniqs/SRR6750056/D/Solo.out/Gene/Features.stats

                                         nUnmapped       20232480
                                        nNoFeature      106929864
                                     nAmbigFeature        6923081
                             nAmbigFeatureMultimap        6899201
                                          nTooMany              0
                                     nNoExactMatch              0
                                       nExactMatch        3554858
                                            nMatch       10477939
                                      nMatchUnique        3554858
                                     nCellBarcodes          69745
                                             nUMIs        1266472
```

# Summary.csv from all 5 configurations

```
cat pheniqs/SRR6750056/A/Solo.out/Gene/Summary.csv

Number of Reads,137582581
Reads With Valid Barcodes,1
Sequencing Saturation,0.643711
Q30 Bases in CB+UMI,-nan
Q30 Bases in RNA read,0.940886
Reads Mapped to Genome: Unique+Multiple,0.852994
Reads Mapped to Genome: Unique,0.729604
Reads Mapped to Gene: Unique+Multipe Gene,0.076119
Reads Mapped to Gene: Unique Gene,0.0258187
Estimated Number of Cells,4632
Unique Reads in Cells Mapped to Gene,3041312
Fraction of Unique Reads in Cells,0.856178
Mean Reads per Cell,656
Median Reads per Cell,525
UMIs in Cells,1076856
Mean UMI per Cell,232
Median UMI per Cell,186
Mean Gene per Cell,127
Median Gene per Cell,112
Total Gene Detected,14662
```

```
cat pheniqs/SRR6750056/B/Solo.out/Gene/Summary.csv

Number of Reads,138136950
Reads With Valid Barcodes,1
Sequencing Saturation,0.643744
Q30 Bases in CB+UMI,-nan
Q30 Bases in RNA read,0.940398
Reads Mapped to Genome: Unique+Multiple,0.85274
Reads Mapped to Genome: Unique,0.729369
Reads Mapped to Gene: Unique+Multipe Gene,0.0760885
Reads Mapped to Gene: Unique Gene,0.0258082
Estimated Number of Cells,4638
Unique Reads in Cells Mapped to Gene,3053375
Fraction of Unique Reads in Cells,0.856472
Mean Reads per Cell,658
Median Reads per Cell,527
UMIs in Cells,1080803
Mean UMI per Cell,233
Median UMI per Cell,187
Mean Gene per Cell,127
Median Gene per Cell,112
Total Gene Detected,14695
```

```
cat pheniqs/SRR6750056/C/Solo.out/Gene/Summary.csv

Number of Reads,135162095
Reads With Valid Barcodes,1
Sequencing Saturation,0.641452
Q30 Bases in CB+UMI,-nan
Q30 Bases in RNA read,0.94326
Reads Mapped to Genome: Unique+Multiple,0.853072
Reads Mapped to Genome: Unique,0.729967
Reads Mapped to Gene: Unique+Multipe Gene,0.0759329
Reads Mapped to Gene: Unique Gene,0.025821
Estimated Number of Cells,4626
Unique Reads in Cells Mapped to Gene,2988451
Fraction of Unique Reads in Cells,0.856286
Mean Reads per Cell,646
Median Reads per Cell,516
UMIs in Cells,1064884
Mean UMI per Cell,230
Median UMI per Cell,184
Mean Gene per Cell,126
Median Gene per Cell,111
Total Gene Detected,14597
```

```
cat pheniqs/SRR6750056/D/Solo.out/Gene/Summary.csv

Number of Reads,137683698
Reads With Valid Barcodes,1
Sequencing Saturation,0.643735
Q30 Bases in CB+UMI,-nan
Q30 Bases in RNA read,0.940733
Reads Mapped to Genome: Unique+Multiple,0.853004
Reads Mapped to Genome: Unique,0.729618
Reads Mapped to Gene: Unique+Multipe Gene,0.0761015
Reads Mapped to Gene: Unique Gene,0.025819
Estimated Number of Cells,4633
Unique Reads in Cells Mapped to Gene,3044102
Fraction of Unique Reads in Cells,0.856322
Mean Reads per Cell,657
Median Reads per Cell,525
UMIs in Cells,1077748
Mean UMI per Cell,232
Median UMI per Cell,186
Mean Gene per Cell,127
Median Gene per Cell,112
Total Gene Detected,14671
```

```
cat solo/SRR6750056/A/Solo.out/Gene/Summary.csv

Number of Reads,218683580
Reads With Valid Barcodes,0.690246
Sequencing Saturation,0.653255
Q30 Bases in CB+UMI,0.860847
Q30 Bases in RNA read,0.937268
Reads Mapped to Genome: Unique+Multiple,0.764201
Reads Mapped to Genome: Unique,0.645298
Reads Mapped to Gene: Unique+Multipe Gene,0.0517492
Reads Mapped to Gene: Unique Gene,0.0176479
Estimated Number of Cells,4697
Unique Reads in Cells Mapped to Gene,3316833
Fraction of Unique Reads in Cells,0.859438
Mean Reads per Cell,706
Median Reads per Cell,567
UMIs in Cells,1142714
Mean UMI per Cell,243
Median UMI per Cell,195
Mean Gene per Cell,132
Median Gene per Cell,117
Total Gene Detected,14968
```
