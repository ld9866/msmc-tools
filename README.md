# MSMC Tools

This repository contains helper scripts and utilities for [msmc](http://github.com/stschiff/msmc) and [msmc2](http://github.com/stschiff/msmc). Most scripts have a little built in help, which can be invoked with the option -h.

### generate_multihetsep.py
This script is the main workhorse to generate input files for msmc. Here is a short synopsis:

    ./generate_multihetsep.py --mask=covered_sites_sample1_chr1.bed.txt.gz \
                              --mask=covered_sites_sample1_chr1.bed.txt.gz \
                              --mask=mappability_mask_chr1.bed.txt.gz \
                              sample1_chr1.vcf.gz sample2_chr1.vcf.gz

In this example, you generate multihetsep files for two diploid individuals. For each individual you need a pair of sample-specific input files: One single-sample/single-chromosome VCF file, and one mask file in bed format, which gives the regions on the chromosome on which the genome of that individual was covered sufficiently. This pair of mask- and vcf files can for example be generated by one my caller scripts (`bamCaller.py` or `cgCaller.py`).

In addition to the mask- and vcf file for each individual, you should use one additional mask per chromosome, the mappability mask, which gives all regions on the chromosome, on which short sequencing reads can be uniquely mapped. For humans (GRCh37), you can download these masks from my ftp site, see the readme for msmc.

You can also apply masks in a negative way, using the option `--negative_mask`, which declares which regions should be _excluded_ from the analysis, instead of _included_, as with `--mask`. This is useful for example if you have an admixed sample and have a bed file which gives all regions of non-native ancestry that you would like to mask out. Any mask file, either by this option or by `--mask` can be given gzipped, using the standard file ending `.gz`.

### bamCaller.py

I assume that you have one bam file for each sample you want to study. You will need a reference file for this. This script reads samtools mpileup data from stdin, so you need to use it in a pipe. Here is an example line using the samtools 1.0 or higher (if using samtools 0.1.19 replace the `bcftools call -c -V indels` command e.g. with `bcftools view -cgI -`):

    samtools mpileup -q 20 -Q 20 -C 50 -u -r <chr> -f <ref.fa> <bam> | bcftools call -c -V indels |
    ./bamCaller.py <mean_cov> <out_mask.bed.gz> | gzip -c > <out.vcf.gz>

where you need to give the average sequencing depth as `<mean_cov>`. You can estimate the mean coverage using `samtools depth` and averaging over some region, e.g. chromosome 20 in humans:

    samtools depth -r 20 <in.bam> | awk '{sum += $3} END {print sum / NR}'

The samtools command line above will generate two files: `<out_mask.bed.gz>` and `<out.vcf.gz>`.

Further options are 

* `--minMapQ <float>` to set the minimum mapping quality, which defaults to 20.0
* `--minConsQ <float>` to set the minimum consensus quality, which defaults to 20.0
* `--legend_file <file>`: If you aim to phase your data against a reference panel, e.g. from 1000 Genomes (see section below about Phasing), you need your VCF to not only contain the variant sites of the sample, but also the genotypes at additional sites at which the panel is genotyped. This option takes a gzipped file of a format that is used in the IMPUTE and SHAPEIT reference panels. It is a simple tab-separated tabular file format with one header line which gets ignored. The only important columns for this purpose are: 1. the chromosome; 2. the position; 3. the reference allele; 4. the alternative allele; 5. the type of the variant, only sites of type `SNP` are considered here.

### cgCaller.py
When your data was generated by Complete Genomics, you need to have access to the masterVarBeta-file, as described in their [File Format Documentation](http://www.completegenomics.com/customer-support/documentation/100357139.html). You can then call the consensus sequence via:

    ./cgCaller.py <chr> <sample_id> <out_mask.bed.gz> <masterVarBeta> | gzip -c > <out.vcf.gz>.

Here, the sample_id is just the sample name to be used in the generated minimal vcf file. the masterVarBeta file can be given gzipped or b2zipped, using the correct file endings .gz or .bz2, respectively. Type `./cgCaller.py -h` to output options. As with the `bamCaller.py` script above, this command line will generate two files, one mask file (as specified as third positional argument) and a vcf file. Note that Complete Genomics normally uses chromosome labels such as "chr20" instead of just "20". This has to be given exactly as the first argument!

Further options for this script are:

* `--max_pos <int>`: this is for testing purposes and simply stops at some defined position in the chromosome.
* `--legend_file <file>`: see same option in `bamCaller.py` script above.

### run_shapeit.sh
This script is published to give you an idea how to phase a single-sample vcf file against a reference panel. You probably have to change a lot of paths and perhaps even executables in that script if you want to use it. I put it mainly to guide construction for your own custom phasing script. Note that this is only needed if you have unrelated individuals that you want to statistically phase. If you have trios, you can simply use the built in trio support of the `generate_multihetsep.py` script.

This scripts phases a vcf against the 1000 Genomes reference panel using [Shapeit2](http://www.shapeit.fr). You need to specify the correct input directories to the panel correctly in the script to make it work. You can then phase with

    ./run_shapeit.sh <VCF> <TMP_DIR> <CHR>

where `<TMP_DIR>` is a temporary directory to use. It is important for phasing against a reference panel to genotype your samples at the sites in the reference panel. See the option `--legend_file` in the two calling scripts above.


### getStats.d
This is a useful little program that just gives you various summary statistics given some msmc input files, as generated by `generate_multihetsep.py`. You need to compile it using `dmd getStats.d`. Then simply run it with

    ./getStats <input_msmc.chr1.txt> <input_msmc.chr2.txt> ...

and have a look at the output. It should look like this:

    called sites	2140477734
    segregating sites	2925425
    sites with ambiguous phase	80328
    pairwise hets(0,1)	1.60348e+06
    pairwise hets(0,2)	1.60956e+06
    pairwise hets(0,3)	1.59628e+06
    pairwise hets(1,2)	1.61854e+06
    pairwise hets(1,3)	1.60169e+06
    pairwise hets(2,3)	1.59281e+06
    Tajima's theta	0.000749238
    Watterson's theta	0.000745481

Most lines should be self-explanatory. The sites starting with `pairwise hets(x,y)` give the number of pairwise differences between haplotype x and haplotype y. This is useful to check whether for example you have some cryptic relatedness of consanguinity in your samples. If your haplotypes are reasonably random samples from a single population, you would expect all pairwise differences to be very similar, as in the example above.

### multihetsep_bootstrap.py
This script generates bootstrap samples from a set of msmc input sites. Type `--multihetsep_bootstrap.py -h` to display options.

### plot_utils.py
This python script (python2.7) contains some plotting functions you can use. Have a look at the doc-strings in each function. Import this module to your python plotting script, or import them into some other script that outputs plotting data for another tool.
