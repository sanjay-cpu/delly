<p align="center">
  <a href="https://academic.oup.com/bioinformatics/article/28/18/i333/245403/DELLY-structural-variant-discovery-by-integrated">
    <img height="150" src="https://raw.githubusercontent.com/dellytools/assets/master/delly-logo/delly-logo-539x600.png">
  </a>
  <h1 align="center">Delly</h1>
</p>

[![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg?style=flat-square)](http://bioconda.github.io/recipes/delly/README.html)
[![Anaconda-Server Badge](https://anaconda.org/bioconda/delly/badges/downloads.svg)](https://anaconda.org/bioconda/delly)
[![C/C++ CI](https://github.com/dellytools/delly/workflows/C/C++%20CI/badge.svg)](https://github.com/dellytools/delly/actions)
[![Docker CI](https://github.com/dellytools/delly/workflows/Docker%20CI/badge.svg)](https://hub.docker.com/r/dellytools/delly/)
[![GitHub license](https://img.shields.io/badge/License-BSD%203--Clause-blue.svg)](https://github.com/dellytools/delly/blob/master/LICENSE)
[![GitHub Releases](https://img.shields.io/github/release/dellytools/delly.svg)](https://github.com/dellytools/delly/releases)

Delly is an integrated structural variant (SV) prediction method that can discover, genotype and visualize deletions, tandem duplications, inversions and translocations at single-nucleotide resolution in short-read massively parallel sequencing data. It uses paired-ends, split-reads and read-depth to sensitively and accurately delineate genomic rearrangements throughout the genome. Structural variants can be visualized using [Delly-maze](https://github.com/dellytools/maze) and [Delly-suave](https://github.com/dellytools/suave).


Installing Delly
----------------

The easiest way to get Delly is to download a statically linked binary or the singularity container (SIF file) from the [Delly github release page](https://github.com/dellytools/delly/releases/). Alternatively, you can download Delly from [Bioconda](https://anaconda.org/bioconda/delly). You can also build Delly from source using a recursive clone and make. 

`git clone --recursive https://github.com/dellytools/delly.git`

`cd delly/`

`make all`

There is a Delly discussion group [delly-users](http://groups.google.com/d/forum/delly-users) for usage and installation questions and a dockerized [delly](https://hub.docker.com/r/dellytools/delly/).



Delly multi-threading mode
--------------------------
Delly supports parallel computing using the OpenMP API (www.openmp.org).

`make PARALLEL=1 -B src/delly`

You can set the number of threads using the environment variable OMP_NUM_THREADS.

`export OMP_NUM_THREADS=2`

Delly primarily parallelizes on the sample level. Hence, OMP_NUM_THREADS should be always smaller or equal to the number of input samples. 


Running Delly
-------------

Delly needs a sorted, indexed and duplicate marked bam file for every input sample. An indexed reference genome is required to identify split-reads. The output is in [BCF](http://samtools.github.io/bcftools/) format with a csi index. Delly supports germline and somatic SV discovery, genotyping and filtering. Because of that, Delly has been modularized and common workflows for germline and somatic SV calling are outlined below. If you do need VCF output you need a recent version of [BCFtools](http://samtools.github.io/bcftools/) for file conversion
.

`delly call -x hg19.excl -o delly.bcf -g hg19.fa input.bam`

`bcftools view delly.bcf > delly.vcf`


Delly for long reads from PacBio or ONT (experimental)
------------------------------------------------------

Delly also has a long-read SV discovery mode.

`delly lr -y ont -g hg19.fa -x hg19.excl input.bam`

`delly lr -y pb -g hg19.fa -x hg19.excl input.bam`


Somatic SV calling
------------------

* At least one tumor sample and a matched control sample are required for SV discovery

`delly call -x hg19.excl -o t1.bcf -g hg19.fa tumor1.bam control1.bam`

* Somatic pre-filtering requires a tab-delimited sample description file where the first column is the sample id (as in the VCF/BCF file) and the second column is either tumor or control.

`delly filter -f somatic -o t1.pre.bcf -s samples.tsv t1.bcf`

* Genotype pre-filtered somatic sites across a larger panel of control samples to efficiently filter false postives and germline SVs. For performance reasons, this can be run in parallel for each sample of the control panel and you may want to combine multiple pre-filtered somatic site lists from multiple tumor samples.

`delly call -g hg19.fa -v t1.pre.bcf -o geno.bcf -x hg19.excl tumor1.bam control1.bam ... controlN.bam`

* Post-filter for somatic SVs using all control samples.

`delly filter -f somatic -o t1.somatic.bcf -s samples.tsv geno.bcf`



Germline SV calling
-------------------

* SV calling is done by sample for high-coverage genomes or in small batches for low-coverage genomes

`delly call -g hg19.fa -o s1.bcf -x hg19.excl sample1.bam`

* Merge SV sites into a unified site list 

`delly merge -o sites.bcf s1.bcf s2.bcf ... sN.bcf`

* Genotype this merged SV site list across all samples. This can be run in parallel for each sample.

`delly call -g hg19.fa -v sites.bcf -o s1.geno.bcf -x hg19.excl s1.bam`

`delly call -g hg19.fa -v sites.bcf -o sN.geno.bcf -x hg19.excl sN.bam`

* Merge all genotyped samples to get a single VCF/BCF using bcftools merge

`bcftools merge -m id -O b -o merged.bcf s1.geno.bcf s2.geno.bcf ... sN.geno.bcf`

* Apply the germline SV filter which requires at least 20 unrelated samples

`delly filter -f germline -o germline.bcf merged.bcf`


Read-depth profiles
-------------------

You can generate read-depth profiles with delly. This requires a mappability map which can be downloaded here:

[Mappability Maps](https://gear.embl.de/data/delly/)

The command to count reads in 10kbp windows and normalize the coverage is:

`delly rd -a -g hg19.fa -m hg19.map input.bam`

The output file can be plotted using R to generate normalized copy-number profiles:

`Rscript R/rd.R out.cov.gz`

The GC bias can be visualized using the stats output.

`delly rd -s stats.gz -g hg19.fa -m hg19.map input.bam`

`zgrep "^GC" stats.gz  > gc.table`

`Rscript R/gcbias.R gc.table`


FAQ
---
* What is the smallest SV size Delly can call?  
This depends on the sharpness of the insert size distribution. For an insert size of 200-300bp with a 20-30bp standard deviation, Delly starts to call reliable SVs >=300bp. Delly also supports calling of small InDels using soft-clipped reads only, the smallest SV size called is 15bp.

* Can Delly be used on a non-diploid genome?  
Yes and no. The SV site discovery works for any ploidy. However, Delly's genotyping model assumes diploidy (hom. reference, het. and hom. alternative).

* Delly is running too slowly what can I do?    
You should exclude telomere and centromere regions and also all unplaced contigs. Delly ships with such an exclude list for human and mouse samples. In addition, you can filter input reads more stringently using -q 20 and -s 15.

* Are non-unique alignments, multi-mappings and/or multiple split-read alignments allowed?  
Delly expects two alignment records in the bam file for every paired-end, one for the first and one for the second read. Multiple split-read alignment records of a given read are allowed if and only if one of them is a primary alignment whereas all others are marked as secondary or supplementary (flag 0x0100 or flag 0x0800). This is the default for bwa mem.

* What pre-processing of bam files is required?    
Bam files need to be sorted, indexed and ideally duplicate marked.

* Usage/discussion mailing list?         
There is a delly discussion group [delly-users](http://groups.google.com/d/forum/delly-users).

* Docker/Singularity support?            
There is a dockerized delly available [here](https://hub.docker.com/r/dellytools/delly/) and singularity containers (*.sif files) are part of the [delly release](https://github.com/dellytools/delly/releases).

* Bioconda support?             
Delly is available via [bioconda](http://bioconda.github.io/recipes/delly/README.html).


Citation
--------

Tobias Rausch, Thomas Zichner, Andreas Schlattl, Adrian M. Stuetz, Vladimir Benes, Jan O. Korbel.      
DELLY: structural variant discovery by integrated paired-end and split-read analysis.     
Bioinformatics. 2012 Sep 15;28(18):i333-i339.       
[https://doi.org/10.1093/bioinformatics/bts378](https://doi.org/10.1093/bioinformatics/bts378)

License
-------
Delly is distributed under the BSD 3-Clause license. Consult the accompanying [LICENSE](https://github.com/dellytools/delly/blob/master/LICENSE) file for more details.
