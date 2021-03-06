genoscapeRtools
================
17 November, 2016

-   [Example running through missing data visualization](#example-running-through-missing-data-visualization)
    -   [Reading data in](#reading-data-in)
    -   [Doing the missing data calcs](#doing-the-missing-data-calcs)
    -   [Pulling out the data](#pulling-out-the-data)
    -   [Read it back in](#read-it-back-in)
-   [Getting the full VCF of those retained indv's and positions](#getting-the-full-vcf-of-those-retained-indvs-and-positions)

<!-- README.md is generated from README.Rmd. Please edit that file -->
This is an R package of tools in devlopment. At the moment it consists mostly of some utilities for investigating amounts of missing data and visualizing where a good cutoff might be.

Example running through missing data visualization
--------------------------------------------------

We have some RAD data from Willow Flycatcher from numerous wonderful contributors and processed bioinformatically by Rachael Bay and Kristen Ruegg. These files are `wifl-10-15-16.012.gz`, `wifl-10-15-16.012.indv`, and `wifl-10-15-16.012.pos`. I have put all of those into the directory `/Documents/UnsyncedData/WIFL_10-15-16` on my computer.

Notice that the actual "012" file itself has been gzipped to save space. The latest version of this package allows that (so long as you are on a system like Unix...)

First we gotta load the library:

``` r
library(genoscapeRtools)
```

### Reading data in

There's a function for that

``` r
wifl <- read_012(prefix = "~/Documents/UnsyncedData/WIFL_10-15-16/wifl-10-15-16", gz = TRUE)
#> 
Read 0.0% of 219 rows
Read 219 rows and 349015 (of 349015) columns from 0.165 GB file in 00:00:09
```

### Doing the missing data calcs

We can do this in a locus centric or an individual centric manner. Here we first do it in an indiv-centric way:

``` r
indv <- miss_curves_indv(wifl)
indv$plot
```

![](readme-figs/indivs-1.png)

And in a locus-centric view:

``` r
loci <- miss_curves_locus(wifl)
loci$plot
```

![](readme-figs/loci-1.png)

### Pulling out the data

So, in theory, if we look at this and decide that we want to keep 175 individuals and roughly 105,000 positions, we should be able to do so in such a way that we have no individuals with more than about 17% missing data, and no loci that should have more than say 7% missing data.

Let's see:

``` r
clean <- miss_curves_indv(wifl, clean_pos = 105000, clean_indv = 175)
#> Picking out clean_pos and clean_indv and writing them to cleaned_indv175_pos105000.012
```

We can make a picture of the result

``` r
clean$plot
```

![](readme-figs/clean-pic-1.png)

It is hard to resolve the line at 104,704 from the one at 105,000.

### Read it back in

Look at the files that were produced:

``` r
dir(pattern = "cleaned_indv175_pos105000*")
#> [1] "cleaned_indv175_pos105000.012"     
#> [2] "cleaned_indv175_pos105000.012.indv"
#> [3] "cleaned_indv175_pos105000.012.pos"
```

To check, just for fun, we can read the thing back in:

``` r
wifl_clean <- read_012("cleaned_indv175_pos105000")
```

And count up the distribution of missing data in indivs and markers and make sure it looks like it should:

``` r
wifl_clean_miss <- wifl_clean == -1
missing_perc_in_indvs <- rowSums(wifl_clean_miss) / ncol(wifl_clean_miss)
missing_perc_in_loci <- colSums(wifl_clean_miss) / nrow(wifl_clean_miss)
par(mfrow=c(1,2))
hist(missing_perc_in_indvs, main = "Individuals")
hist(missing_perc_in_loci, main = "Positions")
```

![](readme-figs/count-missing-1.png)

Booyah! That looks right!

Getting the full VCF of those retained indv's and positions
-----------------------------------------------------------

The original VCF file that the original WIFL 012 file came from is about 16 Gb on the server at UCLA. It is a beast. But it would be nice to be able to have a VCF of our selected markers and individuals. The `.indv` and `.pos` files that are made `miss_curves_indv` when given the `clean_pos` and `clean_indv` arguments can be used with `vcftools` to select everything from the original VCF.

This is what it looks like on Hoffman after copying the `.indv` and `.pos` files there:

``` sh
# first, if you haven't yet, do qrsh to get a compute node...
[kruegg@login1 WIFL_10.15.16]$ qrsh
[kruegg@n2189 WIFL_10.15.16]$ pwd
/u/home/k/kruegg/nobackup-klohmuel/WIFL/WIFL_10.15.16
[kruegg@n2189 WIFL_10.15.16]$ module load vcftools
[kruegg@n2189 WIFL_10.15.16]$ vcftools --vcf WIFL_10.15.16.recode.vcf \
                                       --out cleaned-175-105K  \
                                       --keep cleaned_indv175_pos105000.012.indv \
                                       --positions cleaned_indv175_pos105000.012.pos \
                                       --recode 

VCFtools - 0.1.14
(C) Adam Auton and Anthony Marcketta 2009

Parameters as interpreted:
    --vcf WIFL_10.15.16.recode.vcf
    --keep cleaned_indv175_pos105000.012.indv
    --out cleaned-175-105K
    --positions cleaned_indv175_pos105000.012.pos
    --recode

Keeping individuals in 'keep' list
After filtering, kept 175 out of 219 Individuals
Outputting VCF file...
After filtering, kept 105000 out of a possible 4997594 Sites
Run Time = 210.00 seconds

# now, we might as well gzip that too, since we will be bringing it
# to our laptop.  PLINK can read it directly when gzipped, too...
[kruegg@n2189 WIFL_10.15.16]$ gzip -9 cleaned-175-105K.recode.vcf 
```

After doing all of that, we can read it in with PLINK and make a binary version of the SNPs out of it. This is much faster to read. This should be done with PLINK 1.9 or greater.

``` sh
# this is the plink command to make a bed file from vcf
plink --vcf cleaned-175-105K.recode.vcf.gz \
      --out clean-wifl \
      --make-bed \
      --allow-extra-chr \
      --double-id
      
# this is the output it shows...
PLINK v1.90b3.42 64-bit (20 Sep 2016)      https://www.cog-genomics.org/plink2
(C) 2005-2016 Shaun Purcell, Christopher Chang   GNU General Public License v3
Logging to clean-wifl.log.
Options in effect:
  --allow-extra-chr
  --double-id
  --make-bed
  --out clean-wifl
  --vcf cleaned-175-105K.recode.vcf.gz

4096 MB RAM detected; reserving 2048 MB for main workspace.
--vcf: clean-wifl-temporary.bed + clean-wifl-temporary.bim +
clean-wifl-temporary.fam written.
105000 variants loaded from .bim file.
175 people (0 males, 0 females, 175 ambiguous) loaded from .fam.
Ambiguous sex IDs written to clean-wifl.nosex .
Using 1 thread (no multithreaded calculations invoked).
Before main variant filters, 175 founders and 0 nonfounders present.
Calculating allele frequencies... done.
Total genotyping rate is 0.976517.
105000 variants and 175 people pass filters and QC.
Note: No phenotypes present.
--make-bed to clean-wifl.bed + clean-wifl.bim + clean-wifl.fam ... done.

# look at how nice and small the bed and bim files are:
/WIFL_10-15-16/--% du -h *
4.4M    clean-wifl.bed
3.8M    clean-wifl.bim
8.0K    clean-wifl.fam
4.0K    clean-wifl.log
4.0K    clean-wifl.nosex
 84M    cleaned-175-105K.recode.vcf.gz
 15M    wifl-10-15-16.012.gz
4.0K    wifl-10-15-16.012.indv
 10M    wifl-10-15-16.012.pos
```
