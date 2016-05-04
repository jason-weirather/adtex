# ADTEx 

The Aberration Detection in Tumour EXome (ADTEx) program was developed and published by
KC Amarasinge *et al* (see References below). It is a tool to detect copy number variation (CNV) 
in exome sample pairs, usually a tumor and control from one patient.

In a comparative analysis of exome CNV callers (A Alkodsi *et al*), ADTEx performed better than most other 
callers tested.

ADTEx consists of a set of R scripts run through a python wrapper. The code includes a **Circular 
Binary Segmentation** (CBS) step in which short segments are merged into larger ones if their scores 
are similar (defined as a number of standard deviations usually between 0.5 and 3.5). 
The DNAcopy module that provides this calculation requires log score input. However, ADTEx
does not generate log ratios, instead it supplies straight ratios to the CBS tool. 
This results in a somewhat fragmented output.

**This repository contains code to create a docker implementation of the ADTEx tool.**
The python wrapper has been rewritten, and a post processing CBS step added. 

A docker image can be downloaded directly from dockerhub using
`docker pull jeltje/adtex`

## Installation

After getting the code via git clone https://github.com/Jeltje/adtex.git, 
change to the adtex directory and run

``make``

This will do the following:
1. Build a docker image named adtexbuild, on which it will download and install all needed tools in one directory.
	This step takes a while
2. Start the adtexbuild container from the comand line, then copy the build directory from the container to the 
	/runtime directory
3. Build the jeltje/adtex container using the built tools
	This step should take a few minutes
4. Run a test to verify that the container works correctly

This two step build process makes it easier to edit the jeltje/adtex 'runtime' container because the necessary
programs have all been compiled.

Test the installed docker container:

``make test``

This will use input files from the `/test` directory and create output in `test/test_out`


## Running the docker container

For details on running docker containers in general, see the excellent tutorial at https://docs.docker.com/linux/

To see a usage statement, run

``
docker run jeltje/adtex
``

### Example input:

``
docker run --log-driver=none -v /path/to/input/files:/data jeltje/adtex --normal normal.bam --tumor tumor.bam --sampleid MyTumorSample --out myOutputDir --targetbed targets.bed --centromeres centromeres.bed
``

where

`normal.bam` and `tumor.bam`	are BAM format files of exome reads aligned to the genome. ADTEx will run the program bedtools to create coverage files. You can also supply coverage files directly, provided they were produced by bedtools v1.17 or below.

`sampleid` is an identifier for the patient. This will be used in the output.

`out` is the output directory, will be created in /path/to/input/files if it doesn't exist

`targetbed` is a list of exome targets in bed format

`centromeres.bed` is a bed format file containing centromere locations. This list is used to remove centromeres from the CBS calls. 

Centromeres for hg19 are provided ind the `/data` directory

>	You can find centromere locations for genomes via
>	http://genome.ucsc.edu/cgi-bin/hgTables
>	Using the following selections:
>	- group: Mapping and Sequencing
>	- track:gap
>	-	filter - goes to new page, look for 'type does match' and type centromere, submit
>	-	output format: bed
>	Submit, on the next page just press Get Bed

### Output

Since docker can only write to directories that are mounted via the `-v` parameter, the output directory is created inside
the input directory (`/path/to/input/files` in the above example). Inside the directory are the following files:

`cnv.result`	is the original ADTEx output. It lists scores for all exons tested (derived from the targetbed file) 
		and a segment start, end, and ratio score. 

`<sampleID>.cnv` is post processed CBS output. Here, the exon scores from the cnv.result file have been converted to log2 before
		running the CBS algorithm, and exons have been merged into segments. The num.mark column shows the number of 
		exons in the segment. Seg_mean is a score in log2-1 format, which means that negative scores indicate deletions
		and positive scores amplifications.
 
To get amplified or deleted segments from this file, a threshold must be applied. This is often set to `0.25/-0.25`, 
and with a minimum number of 10 markers per segment.

## Zygosity

ADTEx can also calculate ploidy and contamination, if given an input of B allele frequencies (BAF). BAF files can be extracted
from VCF records such as those generated by [MuTect](https://www.broadinstitute.org/gatk/download/auth?package=MuTect). 
These somatic point mutation callers **must** be run on the same input bamfiles.

The B allele is the non-reference genome allele of a SNP. ADTEx selects SNPs with two alleles that are heterozygous in the control
sample, meaning that about 50% of the reads map to one allele, and 50% to the other. By looking at the read ratio in the tumor
sample, ADTEx can predict which allele is preferentially amplified or lost. For instance, when 33% of reads map to allele P
and 67 represent allele Q, there has been a selective amplification of the Q segment. If 100% of reads show the P allele, then
the Q allele was lost. If ratios fall between these numbers, contamination of the tumor sample with non-tumor cells can be
calculated.

To create BAF from VCF, run `vcfToBaf.py` from the `/src` directory. *Note: This script has only been tested on MuTect output*

To run ADTEx with BAF input:

``
docker run --log-driver=none -v /path/to/input/files:/data jeltje/adtex --normal normal.bam --tumor tumor.bam --sampleid MyTumorSample --out myOutputDir --targetbed targets.bed --centromeres centromeres.bed --estimatePloidy --baf MySample.baf
``

### Zygosity output

The zygosity step is run after the regular CNV caller, so the same output files will be generated as described before.

In addition:

`zygosity.res` is a file with cnv and zygosity calls per input SNP. The last column contains a human readable call on the SNP: LOH (loss of an allele), HET(erozygous), or ASCNA (allele specific amplification)

`contamination` contains a single number <1, indicating a fraction of the sample. When studying tumor samples, this is the faction of non-tumor RNA present in the sample.

`ploidy` contains the inferred ploidy of the tumor sample (2, 3, or 4). ADTEx assumes ploidy is not higher than 4.


## References:
[Amarasinghe KC, Li J, Hunter SM, Ryland GL, Cowin PA, Campbell IG, Halgamuge SK. Inferring copy number and genotype in tumour exome data. BMC Genomics. 2014 Aug 28;15:732](http://www.ncbi.nlm.nih.gov/pubmed/25167919)

[Amarasinghe KC, Li J, Halgamuge SK. CoNVEX: copy number variation estimation in exome sequencing data using HMM. BMC Bioinformatics 2013;14 Suppl. 2:S2](http://www.ncbi.nlm.nih.gov/pubmed/23368785)

Amarasinghe KC, Li J, Halgamuge SK. Correction: CoNVEX: copy number variation estimation in exome sequencing data using HMM. BMC Bioinformatics 2013;14 Suppl. 2:S26

[Alkodsi A, Louhimo R, Hautaniemi S. Brief Bioinform. 2015 Mar;16(2):242-54. Comparative analysis of methods for identifying somatic copy number alterations from deep sequencing data](http://www.ncbi.nlm.nih.gov/pubmed/24599115)
