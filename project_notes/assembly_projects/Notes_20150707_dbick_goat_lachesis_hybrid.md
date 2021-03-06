# Lachesis reverse mapping for hybrid assembly
---

*7/7/2015*

These are my notes on mapping the pacbio contigs back to the Lachesis scaffolds in order to generate comparisons for the final hybrid assembly.

## Table of Contents

* [Preparation](#preparation)
* [Order File Alignment](#order)
* [Summary Table of Results](#summary)
* [Oriented RH map assignments](#orient)
* [Gap information scan](#gap)
* [Pilon error correction](#pilon)
* [Penultimate assembly correction and splitting](#penultimate)
* [Lachesis read depth error correction](#rderrorcorr)
* [Filling gaps with pbJelly](#pbjelly)
	* [Associating pbJelly filled gaps with SV signal](#pbjellysv)
	* [Known problem site gap fill status](#probsitegapfill)
	* [The columns of the perl_associated_named_gap_file.gap.sorted.bed](#gapfilecols)
* [Creating pre-final draft](#prefinal)

<a name="preparation"></a>
## Preparation

I'm going to do this locally because I don't want to wait on the slow file transfer to Blade14. 

> pwd: /home/dbickhart/share/goat_assembly_paper/super_scaffolding/lachesis

```bash
# It's inefficient, but I'm going to use BWA mem for quick contig alignments
bwa index goat_sds_31scaffolds.fasta

# BWA mem has an upper limit of 1 Mbp for alignments. I'm going to have to split some of the larger PacBio contigs to make this work
# I wrote a script to split each PacBio contig into chunks under 9900000
perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaByLength.pl -f goat_split_36ctg_assembly.fa -o lachesis/goat_sub_1mb_assembly.fa

# Just checking my work
samtools faidx lachesis/goat_sub_1mb_assembly.fa
wc -l lachesis/goat_sub_1mb_assembly.fa.fai
	5080 lachesis/goat_sub_1mb_assembly.fa.fai	# far fewer than I was expecting!
perl -lane 'if($F[1] > 990000){print $_;}' < lachesis/goat_sub_1mb_assembly.fa.fai
# Returned nothing
```

OK, things are prepared for alignment. I'm going to try to leave this running all night and hope it doesn't suck up all of the memory on my machine.

```bash
bwa mem goat_sds_31scaffolds.fasta goat_sub_1mb_assembly.fa > goat_sub_1mb_assembly_lachesis_mappings.sam
```

*7/8/2015*

--

OK, this took longer than I expected. I'm going to have to move this to the server because of HD IO issues. Let's first ensure that I'm not redoing the same reads:

> pwd: /home/dbickhart/share/goat_assembly_paper/super_scaffolding/lachesis

```bash
perl -lane 'if($F[0] =~ /^@/){next;} if($F[1] < 2000){print "$F[0]\t$F[1]\t$F[2]\t$F[3]\t$F[4]\t$F[5]";}' < goat_sub_1mb_assembly_lachesis_mappings.sam | cut -f1 | sort | uniq > finished_bwa_mem_first_run.txt

# Now to remove the segments that finished last night
perl ../../../programs_source/Perl/perl_toolchain/sequence_data_scripts/filterFastaByName.pl -f goat_sub_1mb_assembly.fa -l finished_bwa_mem_first_run.txt -o goat_sub_1mb_assembly_restart2.fa
```

I then gzipped both files and shipped them to Blade14 via the nfs mount.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/

```bash
# I couldn't transfer the bwa indicies, so I will regenerate them
bwa index goat_sds_31scaffolds.fasta
bwa mem -t 10 goat_sds_31scaffolds.fasta goat_sub_1mb_assembly_restart2.fa > goat_sub_1mb_assembly_lachesis_restart_mappings.sam
```

*7/10/2015*

--

It turns out that I just needed to ask Shawn for the ordering list, rather than brute-forcing contig assignments! I'm going to try to do direct comparisons between the RH mappings and the ordering files.

<a name="order"></a>
## Order file comparisons
Let's first pull the exact contig names from Brian's RH map, then I'm going to do a modified Smith-Waterman alignment in order to find the optimal order. 

> pwd: /home/dbickhart/share/goat_assembly_paper/super_scaffolding/lachesis

```bash
perl -lane 'if($F[0] eq "RH" || !($F[5] =~ /^utg.+/)){next;}else{$F[5] =~ /(utg\d+)\_.+/; print "$F[1]\t$1";}' < ../RHmap_to_PacBio-v3_contigs.txt | uniq > RH_map_pacbio_contig_order.tab

# To do alignments, I really need to segregate these by chromosome
for i in `seq 1 29`; do echo $i; grep "^$i\s" RH_map_pacbio_contig_order.tab | perl -lane 'print $F[1];' > chr${i}.rh_map.order; done

grep "^X" RH_map_pacbio_contig_order.tab | perl -lane 'print $F[1];' > chrX.rh_map.order

mkdir rh_map_order
mv chr*.order ./rh_map_order/
```

Now, I want to make a modification of the Smith-Waterman alignment to generate contig comparisons. 

*7/11/2015*

--

I made the modification program, but I must be very careful about the alignment gap scoring method. I did not assign a gap opening penalty in order to be very conscious of the affline penalty (as numerous single contig gaps are likely between the methods). I may need to revisit this after consideration.

I may also need to do better space formatting on the output. 

Let's start with the input formatting on Shawn's order files.

> pwd: /home/dbickhart/share/goat_assembly_paper/super_scaffolding/lachesis

```bash
for i in Order_files-2015-07-10/Order_files/*.ordering; do echo $i; outname=`basename $i | cut -d'.' -f1`; perl -lane 'if($F[0] =~ /^#/){next;}else{$F[1] =~ s/\.\d+//g; print $F[1];}' < $i > Order_files-2015-07-10/${outname}.list; done

# Just making sure that the line numbers add up
grep -v "^#" Order_files-2015-07-10/Order_files/group0.ordering | wc -l
	71

wc -l Order_files-2015-07-10/group0.list
	71 Order_files-2015-07-10/group0.list

# Now for the test run of the SW program
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr1.rh_map.order -b Order_files-2015-07-10/group0.list -o test.out

# The first set of conditions (match = 5, gap = -4, mismatch = -1) wasn't permissive enough for gapping the A region. Let's reduce it to a more simplistic setting: (match = 1, gap = -1, mismatch = 0)
# It worked! I also set the max values to the lower diagonal of the matrix to force all values to be accounted in the matrix
```

It's not perfect, and needs manual editing, but it will be far faster than scoring the matches by hand!

The script is located on my perl_toolchain [github repository](https://github.com/njdbickhart/perl_toolchain/blob/master/assembly_scripts/assemblyScaffoldSWAlign.pl).

Let's automate all of this and generate all of the contig alignments.

```bash
mkdir alignments
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr1.rh_map.order -b Order_files-2015-07-10/group0.list -o alignments/order0_chr1.align
...

# Oops! Ran into the first problem! The group orders do not frequently match
# Let's try to get my venn program in action
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group1.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr13.rh_map.order 36
	rh_map_order/chr19.rh_map.order 26
	rh_map_order/chr22.rh_map.order 7
	rh_map_order/chr2.rh_map.order 8

# OK, so group1 is split between more than 4 chromosomes on the RH map
# Let's see if we can get alignments for them
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr13.rh_map.order -b Order_files-2015-07-10/group1.list -o alignments/order1_chr13.align

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr19.rh_map.order -b Order_files-2015-07-10/group1.list -o alignments/order1_chr19.align

# This one was a clear forward alignment:
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr22.rh_map.order -b Order_files-2015-07-10/group1.list -o alignments/order1_chr22.align

# Some contig order issues:
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr2.rh_map.order -b Order_files-2015-07-10/group1.list -o alignments/order1_chr2.align

# It is actually quite fragmented! There are clear inversions and segregated fragments here! Perhaps BioNano superscaffolds could resolve this?
```

Using my [venn comparison script](https://github.com/njdbickhart/perl_toolchain/blob/master/bed_cnv_fig_table_pipeline/nameListVennCount.pl) was very handy here.

My thoughts: the venn components are clearly important for identifying which chromosome "cluster" the Lachesis data should be compared against. Also, the inter-chromosome associations are clearly biasing the clustering data. This is clearly important biological data, but it interferes with the scaffolding. 

There may be a way to associate superscaffolds in a node-based way to resolve these issues. 

Let's finalize the data and send it out to the group.

```bash
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group2.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr2.rh_map.order 43
	rh_map_order/chr3.rh_map.order 2
	rh_map_order/chr5.rh_map.order 2

# We'll just do chr2 alignment
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr2.rh_map.order -b Order_files-2015-07-10/group2.list -o alignments/order2_chr2.align -d
	# The end of the RH chr2 is missing in this cluster

# group3
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group3.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr3.rh_map.order 63
	# and a few "1's" on other chrs

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr3.rh_map.order -b Order_files-2015-07-10/group3.list -o alignments/order3_chr3.align
	# There appears to be an inversion in the middle of the chromosome

# group4
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group4.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr4.rh_map.order 32
	# and a few "1's"

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr4.rh_map.order -b Order_files-2015-07-10/group4.list -o alignments/order4_chr4.align

# group5
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group5.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr5.rh_map.order 49
	rh_map_order/chr9.rh_map.order 3

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr5.rh_map.order -b Order_files-2015-07-10/group5.list -o alignments/order5_chr5.align
	# Inversion at the end of the chromosome

# group6
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group6.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr6.rh_map.order 51
	# from now on, "1's" will be noted if absent in the association

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr6.rh_map.order -b Order_files-2015-07-10/group6.list -o alignments/order6_chr6.align
	# Another inversion, but most of it is here

# group7
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group7.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr8.rh_map.order 56

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr8.rh_map.order -b Order_files-2015-07-10/group7.list -o alignments/order7_chr8.align
	# Another inversion

# group8
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group8.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr7.rh_map.order 43

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr7.rh_map.order -b Order_files-2015-07-10/group8.list -o alignments/order8_chr7.align
	# Most intact assembly (apart from chr1) so far

# group9
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group9.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr11.rh_map.order 52

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr11.rh_map.order -b Order_files-2015-07-10/group9.list -o alignments/order9_chr11.align
	# Inversion

# group10
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group10.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr10.rh_map.order 45

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr10.rh_map.order -b Order_files-2015-07-10/group10.list -o alignments/order10_chr10.align
	# Very good forward alignment

# group11
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group11.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr14.rh_map.order 46

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr14.rh_map.order -b Order_files-2015-07-10/group11.list -o alignments/order11_chr14.align
	# Inversion

# group12
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group12.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr9.rh_map.order 32

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr9.rh_map.order -b Order_files-2015-07-10/group12.list -o alignments/order12_chr9.align
	# Inversion at the beginning

# group13
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group13.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr12.rh_map.order 21

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr12.rh_map.order -b Order_files-2015-07-10/group13.list -o alignments/order13_chr12.align
	# Lots of genomic segment islands that are not represented in the alignment

# group14
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group14.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr15.rh_map.order 43

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr15.rh_map.order -b Order_files-2015-07-10/group14.list -o alignments/order14_chr15.align

# group15
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group15.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr16.rh_map.order 45

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr16.rh_map.order -b Order_files-2015-07-10/group15.list -o alignments/order15_chr16.align
	# inversion 

# group16
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group16.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chrX.rh_map.order 222
	# Very few "1's"

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chrX.rh_map.order -b Order_files-2015-07-10/group16.list -o alignments/order16_chrX.align
	# The alignments are very small -- lots of misarranged regions
	# Likely a problem with both the RH map and the Lachesis? maybe Lachesis is better here?

# group17
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group17.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr17.rh_map.order 45
	# Very few "1's"

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr17.rh_map.order -b Order_files-2015-07-10/group17.list -o alignments/order17_chr17.align
	# Inversion

# group18
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group18.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr20.rh_map.order 17

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr20.rh_map.order -b Order_files-2015-07-10/group18.list -o alignments/order18_chr20.align
	# Lots of rearranged chromosome segments

# group 19
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group19.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr21.rh_map.order 43

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr21.rh_map.order -b Order_files-2015-07-10/group19.list -o alignments/order19_chr21.align
	# Inversion

# group20
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group20.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr18.rh_map.order 46

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr18.rh_map.order -b Order_files-2015-07-10/group20.list -o alignments/order20_chr18.align
	# Lots of rearranged segments

# group21
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group21.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr24.rh_map.order 23

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr24.rh_map.order -b Order_files-2015-07-10/group21.list -o alignments/order21_chr24.align
	# Inversion

# group22
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group22.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr23.rh_map.order 26

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr23.rh_map.order -b Order_files-2015-07-10/group22.list -o alignments/order22_chr23.align
	# Inversion

# group23
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group23.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr29.rh_map.order

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr29.rh_map.order -b Order_files-2015-07-10/group23.list -o alignments/order23_chr29.align
	# Inversion

# group24
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group24.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr26.rh_map.order 21

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr26.rh_map.order -b Order_files-2015-07-10/group24.list -o alignments/order24_chr26.align
	# Inversion

# group25
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group25.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chrX.rh_map.order 196  <- maybe this is part of the X and Y??

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chrX.rh_map.order -b Order_files-2015-07-10/group25.list -o alignments/order25_chrX.align
	# Again, more disorder -- perhaps the PAR has confused the clustering?

# group26
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group26.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr22.rh_map.order 20

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr22.rh_map.order -b Order_files-2015-07-10/group26.list -o alignments/order26_chr22.align
	# slight gap towards the end

# group27
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group27.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr28.rh_map.order

# Note, I fixed a bug with my assembly scaffold alignment script here
perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr28.rh_map.order -b Order_files-2015-07-10/group27.list -o alignments/order27_chr28.align
	# Inversion

# group28
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group28.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr27.rh_map.order 23

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr27.rh_map.order -b Order_files-2015-07-10/group28.list -o alignments/order28_chr27.align -d
	# Inversion

# group29
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group29.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr25.rh_map.order 31

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr25.rh_map.order -b Order_files-2015-07-10/group29.list -o alignments/order29_chr25.align
	#Inversion

# group30
for i in rh_map_order/chr*.order; do echo -n "$i "; value=`perl ../../../programs_source/Perl/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl $i Order_files-2015-07-10/group30.list | grep "1;2" | cut -f2`; echo $value; done
	rh_map_order/chr19.rh_map.order 2
	rh_map_order/chr2.rh_map.order 1
	# Novel scaffolds?!

perl ../../../programs_source/Perl/perl_toolchain/assembly_scripts/assemblyScaffoldSWAlign.pl -a rh_map_order/chr19.rh_map.order -b Order_files-2015-07-10/group30.list -o alignments/order30_chr19.align
	# Only two contigs
```
<a name="summary"></a>
## Alignment Summary Table

Now to summarize all of that in a table:

| Cluster # | RHmap chrs | Counts | Comments |
| :--- | :--- | :--- | :--- |
group0 | Chr1 | Chr1:63 | Most map to Chr1 -- some single contig mappings to other chrs |
group1 | Chr13,Chr19,Chr22,Chr2 | Chr13:36; Chr19:26; Chr22:7, Chr2:8| The most fragmented cluster
group2 |  Chr2,Chr3,Chr5 | Chr2:43; Chr3:2, Chr5:2 | a bit more reasonable 
group3 | Chr3 | Chr3:63 | |
group4 | Chr4 | Chr4:32 | |
group5 | Chr5,Chr9 | Chr5:43; Chr9:3 | Clear split + Inversion
group6 | Chr6 | Chr6:51 | Inversion
group7 | Chr8 | Chr8:56 | Inversion
group8 | Chr7 | Chr7:43 | Intact -- no inversion
group9 | Chr11 | Chr11:52 | Inversion
group10 | Chr10 | Chr10:45 | Intact -- no inversion
group11 | Chr14 | Chr14:46 | Inversion
group12 | Chr9 | Chr9:42 | Inversion at the beginning
group13 | Chr12 | Chr12:21 | Most contigs were not in contiguous alignment
group14 | Chr15 | Chr15:43 | 
group15 | Chr16 | Chr16:45 | Inversion
group16 | ChrX | ChrX:222 | Lots of small sub-alignments -- very discontinuous
group17 | Chr17 | Chr17:45 | Inversion
group18 | Chr20 | Chr20:17 | Most contigs were not in contiguous alignment
group19 | Chr21 | Chr21:43 | Inversion
group20 | Chr18 | Chr18:46 | Most contigs were not in contiguous alignment
group21 | Chr24 | Chr24:23 | Inversion
group22 | Chr23 | Chr23:26 | Inversion
group23 | Chr29 | Chr29:30 | Inversion
group24 | Chr26 | Chr26:21 | Inversion
group25 | ChrX | ChrX:196 | Lots of small sub-alignments -- very discontinuous
group26 | Chr22 | Chr22:20 | Gap of contigs at the end
group27 | Chr28 | Chr28:11 | Inversion
group28 | Chr27 | Chr27:23 | Inversion
group29 | Chr25 | Chr25:31 | Inversion
group30 | Chr19,Chr2 | Chr19:2; Chr2:1 | Hard to tell!


In light of this information, it's clear that group1 is a chimeric scaffold derived from inter-chromosome interactions from several autosomes. Also, the two groups that have the X chromosome contigs COULD be the two sex chromosomes, but it is difficult to scaffold them from there! Another possibility is that the scaffolding "forked" once it reached the PAR and that one cluster is part of the X and the other is a chimeric Y + X? 

Ivan and Shawn will know best how to reanalyze the data.

<a name="orient"></a>
## Oriented RH map assignments

OK, I have simple contig order lists, but I don't have the contigs assigned in terms of their orientation on the RH map. I will write a quick one-shot perl script to do this.

Here's the script (it's so singular in purpose that it's not worth archiving in my github repository):

```perl
#!/usr/bin/perl
# This is a one shot script designed to quickly interrogate and order the pacbio contigs on the RH map
# Output format:
#	chr	contigname	start	end	orientation["+", "-", "?"]

use strict;

my $output = "ordered_rh_map_contigs.tab";
my $input = "RHmap_to_PacBio-v3_contigs.txt";

open(my $IN, "< $input") || die "Could not find input file!\n";
open(my $OUT, "> $output");
my $header = <$IN>;
my @current;
my %data; # {chr}->[]->[contigname, min, max, orient]
my $chrcontig = "0"; my $curchr = "0";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[5] =~ /^\*/){next;}
	if($segs[5] =~ /\#N\/A/){next;}
	
	# get the simple contig name
	my @ctgsegs = split(/\_/, $segs[5]);
	$segs[5] = $ctgsegs[0];
	
	if($segs[5] ne $chrcontig && $chrcontig ne "0" && scalar(@current) > 0){
		# new contig, and we have data to sort through
		push(@{$data{$curchr}}, determineOrder(\@current));
		@current = ();
	}elsif($segs[5] ne $chrcontig && scalar(@current) == 0){
		# new contig and there's only one entry for it!
		push(@{$data{$curchr}}, [$chrcontig, $segs[6], $segs[6], "?"]);
	}
	
	push(@current, \@segs);
	$chrcontig = $segs[5];
	$curchr = $segs[1];
}
if(scalar(@current) >= 1){
	push(@{$data{$curchr}}, determineOrder(\@current));
}

foreach my $chr (sort {$a <=> $b} keys(%data)){
	foreach my $row (@{$data{$chr}}){
		print {$OUT} "$chr\t" . $row->[0] . "\t" . $row->[1] . "\t" . $row->[2] . "\t" . $row->[3] . "\n";
	}
}

close $IN;
close $OUT;
	
exit;

sub determineOrder{
	my ($array) = @_;
	my $min = 20000000;
	my $max = 0;
	my $orient = ($array->[0]->[6] - $array->[1]->[6] < 0)? "+" : "-";
	# I'm using empty string concatenation and redundant math to ensure that the Perl interpreter
	# Doesn't just store the address of the array in memory
	my $ctgname = $array->[0]->[5] . "";
	foreach my $v (@{$array}){
		if($v->[6] < $min){
			$min = $v->[6] + 1 - 1;
		}
		if($v->[6] > $max){
			$max = $v->[6] + 1 - 1;
		}
	}
	return [$ctgname, $min, $max, $orient];
}
```

I ran this on my local virtualbox installation:

> pwd: /home/dbickhart/share/goat_assembly_paper/super_scaffolding

```bash
perl generate_oriented_rhmap_contigs.pl

# Gzipping original RHmap file just in case
gzip RHmap_to_PacBio-v3_contigs.txt
```

The output file has the following columns:

1. Chromosome
2. PacBio contig abbreviation
3. minimum alignment position (start coordinate)
4. maximum alignment position (end coordinate)
5. Orientation (can be "+", "-" or "?" if there is only one point of reference)

<a name="gap"></a>
## Gap information scan

*8/24/2015*

I need to pull information from the assemblies so that gap regions and lengths can be easily determined. I'm going to write a Java program to pull this from the fasta quickly.

--

*8/25/2015*

OK, I wrote the program [GetMaskBedFasta](https://github.com/njdbickhart/GetMaskBedFasta) and ran it from my IDE in console mode. It produced 10,712 gap regions, and 10,520 of which were greater than 100bp in length.

Example settings for running the program:

> pwd: /home/dbickhart/share/goat_assembly_paper/bng_scaffolds

```bash
java -jar GetMaskBedFasta.jar -f Goat333HybScaffolds1242contigs0723.fasta -o bng_gaps.bed -s bng_stats.tab
```

I'm going to try splitting the BNG superscaffolds to help Shawn with the Lachesis alignment in the meantime. I'll select only the gap regions that are larger than 100 bp (larger than the read length?) and I've updated my fasta splitting script, [splitFastaWBreakpointBed.pl](https://github.com/njdbickhart/perl_toolchain/blob/master/sequence_data_scripts/splitFastaWBreakpointBed.pl), so that it does not pull gap sequence from the beginning and ends of scaffold sequence.

> pwd: /home/dbickhart/share/goat_assembly_paper/bng_scaffolds

```bash
# Going to extract only the gap regions above 100 bp
perl -lane 'if($F[2] - $F[1] > 100){print $_;}' < bng_gaps.bed > bng_gaps_above100bp.bed

perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f Goat333HybScaffolds1242contigs0723.fasta -o bng_split_gap.fa -b bng_gaps_above100bp.bed

# Using samtools to get summary statistics from the split fasta
samtools faidx bng_split_gap.fa

wc -l *.fai
  2349 bng_split_gap.fa.fai			<-774 more segments than the original BNG super scaffolds
  1575 Goat333HybScaffolds1242contigs0723.fasta.fai

tail *.fai
==> bng_split_gap.fa.fai <==
...
utg40992        23779   2,657,212,952      23779   23780	<- a loss of 39 megabases

==> Goat333HybScaffolds1242contigs0723.fasta.fai <==
...
utg40992        23779   2,696,524,662      23779   23780
```

OK, there were far fewer split regions than I expected, but this can be attributed to close gaps in the scaffolds that probably corresponded to nickase binding sites? I'll check to confirm:

```bash
# From bng_gaps_above100bp.bed
	Super-Scaffold_1        2673354 2677638
	Super-Scaffold_1        2677646 2682610

# So the extract region would be Super-Scaffold_1:2677638-2677646
samtools faidx Goat333HybScaffolds1242contigs0723.fasta Super-Scaffold_1:2677638-2677646
	>Super-Scaffold_1:2677638-2677646
	NGCTCTTCN

# From Alex's email, the nickase site is: GCTCTTC, so that's a site!
```

I think that I'm ready to gzip this and send it to Shawn.

Wait... getting a bed file with the exact split coordinates would help substantially, I think. Let's modify the perl script and run it again to get the right coordinates.

```bash
perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f Goat333HybScaffolds1242contigs0723.fasta -o bng_split_gap.fa -b bng_gaps_above100bp.bed -s bng_split_gap_coords.bed

# The bng_split_gap_coords.bed file will contain the coordinates that the split fasta entry has on the original scaffold

# Now to set the old fai file in a new, sorted location
mv bng_split_gap.fa.fai bng_split_gap.fa.fai.bak
sort -k1,1 bng_split_gap.fa.fai.bak | cut -f1,2 > bng_split_gap.fa.fai.bak.sorted

samtools faidx bng_split_gap.fa
sort -k1,1 bng_split_gap.fa.fai | cut -f1,2 > bng_retest.fa.fai.sorted

diff bng_retest.fa.fai.sorted bng_split_gap.fa.fai.bak.sorted
# Nothing -- so we're good!
```

--

*8/26/2015*

Just generating some quick summary statistics on the split assembly.

> pwd: /home/dbickhart/share/goat_assembly_paper/bng_scaffolds

```bash
# Calculating contig N50
perl ../../programs_source/Perl/perl_toolchain/assembly_scripts/calculateContigN50.pl bng_split_gap.fa
	N50 length:     1309271228
	N50 value:      12,149,468
	L50 value:      58
```

#### Initial check for 2kb gap split cutoff

Alex mentioned that a 100bp cutoff would be too stringent. Let's try a 2kb gap cutoff and see if that appreciably improves the statistics.

> pwd: /home/dbickhart/share/goat_assembly_paper/bng_scaffolds

```bash
perl -lane 'if($F[2] - $F[1] > 2000){print $_;}' < bng_gaps.bed > bng_gaps_above2kb.bed

wc -l bng_gaps*
  10520 bng_gaps_above100bp.bed
   9329 bng_gaps_above2kb.bed
  10712 bng_gaps.bed

# OK, that was a substantial decrease. Let's make the split fasta now
perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f Goat333HybScaffolds1242contigs0723.fasta -o bng_split_2kb_gap.fa -b bng_gaps_above2kb.bed -s bng_split_2kb_gap_coords.bed

samtools faidx bng_split_2kb_gap.fa

wc -l *.fai
  2583 bng_split_2kb_gap.fa.fai
  2349 bng_split_gap.fa.fai
  1242 bng_unscaffolded_pacbio.fai
  1575 Goat333HybScaffolds1242contigs0723.fasta.fai

# Hmm... that made more segments! How is that possible?
# Let's look at the data
grep -P 'Super-Scaffold_336\.' *.fai | perl -lane '$F[0] =~ s/\:/\|/g; pop(@F); pop(@F); print join("|", @F);'
```

| File | Scaffold | ScaffLen | total Bases |
| :--- | :--- | :--- | ---: |
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.1|**177701**|22
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.2|**155403**|180707
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.3|869399|338723
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.4|1523|1222634
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.5|80668|1224205
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.6|3279|1306240
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.7|72515|1309596
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.8|103326|1383342
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.9|1765|1488413
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.10|1773|1490231
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.11|**326432**|1492057
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.12|1856|1823953
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.13|**105876**|1825863
bng_split_2kb_gap.fa.fai|Super-Scaffold_336.14|198|1933527
bng_split_gap.fa.fai|Super-Scaffold_336.1|**177701**|293574308
bng_split_gap.fa.fai|Super-Scaffold_336.2|**154708**|293754993
bng_split_gap.fa.fai|Super-Scaffold_336.3|96051|293912302
bng_split_gap.fa.fai|Super-Scaffold_336.4|554331|294009976
bng_split_gap.fa.fai|Super-Scaffold_336.5|148739|294573568
bng_split_gap.fa.fai|Super-Scaffold_336.6|69431|294724808
bng_split_gap.fa.fai|Super-Scaffold_336.7|78872|294795419
bng_split_gap.fa.fai|Super-Scaffold_336.8|72515|294875628
bng_split_gap.fa.fai|Super-Scaffold_336.9|102867|294949374
bng_split_gap.fa.fai|Super-Scaffold_336.10|**326432**|295053979
bng_split_gap.fa.fai|Super-Scaffold_336.11|**105876**|295385875

Bold values are the same as in the previous file. It seems obvious that smaller distances between nickase sites allowed my scripts's internal "100bp" length filter for splitting to kick in. Hell, the end of the chromosome filter failed for the 2kb gap file because there was 200bp uninterrupted!

Calculating the N50:

```bash
perl ../../programs_source/Perl/perl_toolchain/assembly_scripts/calculateContigN50.pl bng_split_2kb_gap.fa
N50 length:     1316999836
N50 value:      14,451,513
L50 value:      47
```

#### Implementation of minimum size filter for scaffolds

Hmm, despite the increase in number of fragments, the N50 went up by 2Mb! Let's try improving the Perl script so that the minimum size filter is customizable.

```bash
# 2kb minimum size filter
perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f Goat333HybScaffolds1242contigs0723.fasta -o bng_split_2kb_2kbf_gap.fa -b bng_gaps_above2kb.bed -s bng_split_2kb_2kbf_gap_coords.bed -m 2000

samtools faidx bng_split_2kb_2kbf_gap.fa

wc -l *.fai
  2146 bng_split_2kb_2kbf_gap.fa.fai
  2583 bng_split_2kb_gap.fa.fai
  2349 bng_split_gap.fa.fai
  1242 bng_unscaffolded_pacbio.fai
  1575 Goat333HybScaffolds1242contigs0723.fasta.fai
  9895 total

perl -lane 'print $F[1];' < bng_split_2kb_2kbf_gap.fa.fai | statStd.pl
	total   2146
	Minimum 435
	Maximum 66863457
	Average 1220070.247903
	Median  58891.5
	Standard Deviation      4774317.508145
	Mode(Highest Distributed Value) 21821

# I wanted to see if the smaller size contigs contained a copious amount of N's
samtools faidx Goat333HybScaffolds1242contigs0723.fasta Super-Scaffold_880:11738277-11741706
>Super-Scaffold_880:11738277-11741706
	# all N's apart from a nickase site that breaks up the distance

# N50 size calculation
perl ../../programs_source/Perl/perl_toolchain/assembly_scripts/calculateContigN50.pl bng_split_2kb_2kbf_gap.fa
	N50 length:     1316999836
	N50 value:      14,451,513
	L50 value:      47
```

So, the 2kb split contains far more contigs with only "N's" due to nickase site interruptions. Here is a table with the scaffold N50 values:

| filter criteria | N50 | 
| :--- | ---: |
100bp gap, 100bp min | 12,149,468
2kb gap, 100bp min | 14,451,513
2kb gap, 2kb min | 14,451,513

#### Implementation of 'N' ratio filter for smaller split contigs

Let's see if we can filter the 2kb gap split file by removing segments that have mostly N's. I rewrote the [splitFastaWBreakpointBed.pl](https://github.com/njdbickhart/perl_toolchain/blob/master/sequence_data_scripts/splitFastaWBreakpointBed.pl) script again to accommodate an N base filter.

> pwd: /home/dbickhart/share/goat_assembly_paper/bng_scaffolds

```bash
# 2kb gaps, 1000bp min size filter, 50% N base filter
perl ../../programs_source/Perl/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f Goat333HybScaffolds1242contigs0723.fasta -o bng_split_2kb_1kbf_50n_gap.fa -b bng_gaps_above2kb.bed -s bng_split_2kb_1kbf_50n_gap_coords.bed -m 1000 -n 0.5

samtools faidx bng_split_2kb_1kbf_50n_gap.fa

wc -l *.fai
  2075 bng_split_2kb_1kbf_50n_gap.fa.fai
  2146 bng_split_2kb_2kbf_gap.fa.fai
  2583 bng_split_2kb_gap.fa.fai
  2349 bng_split_gap.fa.fai
  1242 bng_unscaffolded_pacbio.fai
  1575 Goat333HybScaffolds1242contigs0723.fasta.fai

perl ../../programs_source/Perl/perl_toolchain/assembly_scripts/calculateContigN50.pl bng_split_2kb_1kbf_50n_gap.fa
	N50 length:     1316999836
	N50 value:      14451513
	L50 value:      47

perl -lane 'print $F[1];' < bng_split_2kb_1kbf_50n_gap.fa.fai | statStd.pl
	total   2075
	Minimum 1501
	Maximum 66863457
	Average 1261708.550843
	Median  61520
	Standard Deviation      4849948.272566
	Mode(Highest Distributed Value) 51884

```

OK, so ultimately this looks to be the best split fasta. Let's package it up and send it to Alex and Shawn.

<a name="pilon"></a>
## Pilon error correction
*9/24/2015*

Given that Pilon is a dependency-free JAR and given that we need error-corrected fasta sequence for Ben to do the annotation, I'm going to try to run Pilon on the Lachesis data so that he can begin work as soon as possible.

From the help menu of Pilon, here's a list of options that I want to use to generate output that can be parseable later:

* INPUTS:
	* --genome genome.fasta
	  *	The input genome we are trying to improve, which must be the reference used
	  *	for the bam alignments.  At least one of --frags or --jumps must also be given.
	* --frags frags.bam
	  * A bam file consisting of fragment paired-end alignments, aligned to the --genome
	  * argument using bwa or bowtie2.  This argument may be specifed more than once.
* OUTPUTS:
	* --output
	  * Prefix for output files
	* --changes
	  * If specified, a file listing changes in the <output>.fasta will be generated.
	* --vcf
	  * If specified, a vcf file will be generated
	* --tracks
	   * This options will cause many track files (*.bed, *.wig) suitable for viewing in
	   * a genome browser to be written.
* CONTROL:
	* --diploid
	  * Sample is from diploid organism; will eventually affect calling of heterozygous SNPs
	* --fix fixlist
	  * A comma-separated list of categories of issues to try to fix:
	  * "bases": try to fix individual bases and small indels;


I'll also output all files to a separate directory on my ISCI mount on Blade14. 

**NOTE:** I am not using the deg contigs in this correction. We're likely to get issues with the repetitive regions, but due to time constraints, I'm willing to get a "good" assembly in lieu of a "almost perfect" assembly.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/pilon

```bash
# first, to align the Papadum illumina reads to the Lachesis data to generate the fragment bam
bwa mem -R '@RG\tID:ilmn250\tLB:ilmn250papadum\tSM:papadum' /mnt/nfs/nfs2/GoatData/Lachesis/Lachesis_assembly.fasta /mnt/nfs/nfs2/GoatData/Ilmn/250bp_S1_L001_R1_001.fastq.gz /mnt/nfs/nfs2/GoatData/Ilmn/250bp_S1_L001_R2_001.fastq.gz > lachesis_ilm_250bp.sam &
bwa mem -R '@RG\tID:ilmn400\tLB:ilmn400papadum\tSM:papadum' /mnt/nfs/nfs2/GoatData/Lachesis/Lachesis_assembly.fasta /mnt/nfs/nfs2/GoatData/Ilmn/400bp_S2_L001_R1_001.fastq.gz /mnt/nfs/nfs2/GoatData/Ilmn/400bp_S2_L001_R2_001.fastq.gz > lachesis_ilm_400bp.sam

# convert to bam and sort
samtools view -bS lachesis_ilm_250bp.sam | samtools sort -T lachesis_ilm_250 -o lachesis_ilm_250bp.sorted.bam -
samtools view -bS lachesis_ilm_400bp.sam | samtools sort -T lachesis_ilm_400 -o lachesis_ilm_400bp.sorted.bam -

samtools index lachesis_ilm_250bp.sorted.bam & samtools index lachesis_ilm_400bp.sorted.bam

# merging
samtools merge -@ 10 lachesis_ilm_merged.sorted.bam lachesis_ilm_250bp.sorted.bam lachesis_ilm_400bp.sorted.bam

```

We discovered that the Lachesis fasta did not contain super-scaffolds, so we ended up aborting the error correction. Steve has done error correction on the deg + super-scaffold + unscaffolded contigs -- I just need to calculate the stats and then split the fasta.

<a name="penultimate"></a>
## Penultimate assembly correction and splitting
*10/1/2015*

Steve's fasta is located in this directory:

> Blade14: /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Papadum-v4BNG/

```bash
# Calling the gaps first 
~/jdk1.8.0_05/bin/java -jar ~/GetMaskBedFasta/store/GetMaskBedFasta.jar -f /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Papadum-v4BNG/papadum-v4bng-pilon.fa -o /mnt/iscsi/vnx_gliu_7/goat_assembly/papadum-v4bng-pilon.gaps.bed -s /mnt/iscsi/vnx_gliu_7/goat_assembly/papadum-v4bng-pilon.gaps.stats

# I'm concerned that Steve may have done several batches, and may not have included the deg contigs in this fasta
# I'll test this by fasta length
head papadum-*.fai
	==> papadum-v4bng-pilon.fa.fai <==
	Scaffold_1      32985612        12      80      81
	Scaffold_2      40642650        33397957        80      81
	Scaffold_4      20284375        74548653        80      81

	==> papadum-v4bng-pilon.full.fa.fai <==
	Scaffold_1      32985612        12      80      81
	Scaffold_2      40642650        33397957        80      81
	Scaffold_4      20284375        74548653        80      81

# Hmmm... by bp count, they look the same. Let's assume that they are and ask Steve later.
```

I've now got to remove gaps under 2kb from the bed file, and then I should be able to split the fasta file correctly.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly

```bash
# Getting all gaps above 2kb
perl -lane 'if($F[2] - $F[1] > 2000){print $_;}' < papadum-v4bng-pilon.gaps.bed > papadum-v4bng-pilon.gaps.gt2kb.bed
wc -l papadum-v4bng-pilon.gaps*.bed
  8593 papadum-v4bng-pilon.gaps.bed
  7491 papadum-v4bng-pilon.gaps.gt2kb.bed
 16084 total

# splitting >=2kb gaps, with 2kb minimum post-split contig size, with < 50% N content per contig
perl ~/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Papadum-v4BNG/papadum-v4bng-pilon.fa -o papadum-v4bng-pilon-split2kbgap.fa -b papadum-v4bng-pilon.gaps.gt2kb.bed -s papadum-v4bng-pilon-split2kbgap.original.coords.bed -m 2000 -n 0.5

# Checking the quality
samtools faidx papadum-v4bng-pilon-split2kbgap.fa

# The stats hold up so far. Let's work on the RH map now.
```

#### Remapping RH probes onto the corrected assembly



```bash
# indexing the genome
bwa index papadum-v4bng-pilon-split2kbgap.fa

# Creating the bam
bwa mem papadum-v4bng-pilon-split2kbgap.fa /mnt/nfs/nfs2/GoatData/RH_map/RHmap_probe_sequences.fasta | samtools view -bS - | samtools sort -T papadum.tmp -o rh_map/papadum-v4bng-pilon-split2kbgap.bam -
samtools index rh_map/papadum-v4bng-pilon-split2kbgap.bam

# Now to start the ordering
perl ~/perl_toolchain/assembly_scripts/probeMappingRHOrdering.pl rh_map/RHmap_to_PacBio-v3_contigs.txt rh_map/papadum-v4bng-pilon-split2kbgap.bam rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out

# I wrote a one-shot script to remove much of the tedium of editing out the spurious probe alignments
perl remove_spurious_probe_rh.pl < rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out > rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit.out

wc -l rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit.out
 1316 rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out
 1002 rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit.out

# Another one-shot script to condense down the headings
perl condense_rh_remappings_order.pl < rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit.out > rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit2.out

# I still need to edit several specific areas where the RH map has inversions compared to the Irys scaffolds. I'll label each one:

# Area 1
4       Scaffold_81.1   12999   1975874 +
4       Scaffold_81.2   124622  296935  -
4       Scaffold_81.1   2267514 2327076 -
4       Scaffold_81.2   473386  5106527 +
4       Scaffold_81.3   237540  10449679        +
4       Scaffold_99.1   8399    13467629        -
4       Scaffold_1403   17805   29726120        -

# Area 2
5       Scaffold_199.1  57520   2802112 +
5       Scaffold_199.2  47349   170676  -
5       Scaffold_199.1  2861132 2969615 -
5       Scaffold_205.1  1552    79311   -
5       utg9423 12294   12294   ?
5       Scaffold_199.2  253034  1021784 -

# Area 3
21      Scaffold_1771.1 159885  963969  -
21      Scaffold_1965   1151128 1395957 +
21      Scaffold_1771.1 21495   73684   +
21      Scaffold_1965   13686   1078812 -

# Area 4
29      Scaffold_59.2   4429    9932553 -
29      Scaffold_59.1   867713  3388273 -
29      Scaffold_7.2    17688252        17756202        +
29      Scaffold_59.1   46596   837928  +
29      Scaffold_7.2    795790  17648713        -
29      Scaffold_7.1    3147    628237  -

# Just as a side-note, the X chromosome is completely bloated with probes/mappings
```

Here is are the two one-shot scripts:

> remove_spurious_probe_rh.pl

```perl
#!/usr/bin/perl

@store;
while(<>){
        chomp;
        my @s = split(/\t/, $_);
        push(@store, \@s);
}
print join("\t", @{$store[1]});
print "\n";
for($x = 1; $x < scalar(@store) - 1; $x++){
        if($store[$x]->[4] eq "\?"){
                ($b1) = $store[$x-1]->[1] =~ /(Scaffold.+)\.\d+/;
                ($b2) = $store[$x+1]->[1] =~ /(Scaffold.+)\.\d+/;
                if($b2 eq $b1){next;}
        }
        print join("\t", @{$store[$x]});
        print "\n";
}
print join("\t", @{$store[-1]});
print "\n";
```

> condense_rh_remappings_order.pl

```perl
#!/usr/bin/perl

my @store;
while(<>){
        chomp;
        my @s = split(/\t/, $_);
        push(@store, \@s);
}

my $prevvalue = "NA"; my $prevchr;
my @sign;
my @values;
for(my $i = 0; $i < scalar(@store); $i++){
        if($prevvalue eq "NA"){
                $prevvalue = $store[$i]->[1];
                $prevchr = $store[$i]->[0];
                push(@sign, $store[$i]->[4]);
                push(@values, ($store[$i]->[2], $store[$i]->[3]));
                next;
		}elsif($prevvalue ne $store[$i]->[1]){
                my ($min, $max) = getMinandMax(@values);
                my $sign = getConsensusSign(@sign);
                print "$prevchr\t$prevvalue\t$min\t$max\t$sign\n";
                @values = ();
                @sign = ();
        }
        $prevvalue = $store[$i]->[1];
        $prevchr = $store[$i]->[0];
        push(@sign, $store[$i]->[4]);
        push(@values, ($store[$i]->[2], $store[$i]->[3]));
}
my ($min, $max) = getMinandMax(@values);
my $sign = getConsensusSign(@sign);
print "$prevchr\t$prevvalue\t$min\t$max\t$sign\n";

exit;

sub getConsensusSign{
        my (@s) = @_;
        my $sign = "NA";
        foreach my $b (@s){
                if($b eq "+" || $b eq "-"){
                        $sign = $b;
                }elsif($sign eq "NA" && $b eq "\?"){
                        $sign = "?";
                }
        }
        return $sign;
}

sub getMinandMax{
        my (@s) = @_;
        @s = sort{$a <=> $b} @s;
        return $s[0], $s[-1];
}
```

#### Testing if there are gap regions that are masked by nickase sites

```bash
perl -e '$prevchr = "NA"; $prevlen = 0; $prevstart = 0; while(<>){chomp; @s = split(/\t/); $len = $s[2] - $s[1]; if($len < 2000){if($len + $prevlen < 2000 && $prevchr eq $s[0] && $s[1] - $prevstart < 5000){print "$prevchr\t$prevstart\t$s[0]\t$s[1]\t$s[2]\t$len\n";}} $prevchr = $s[0]; $prevstart = $s[1]; $prevlen = $len;}' < papadum-v4bng-pilon.gaps.bed | wc -l
30

# OK, so there are 30 cases of this happening. Let's see if this impacts the output fasta:
perl -e '$prevchr = "NA"; $prevlen = 0; $prevstart = 0; while(<>){chomp; @s = split(/\t/); $len = $s[2] - $s[1]; if($len < 2000){if($len + $prevlen < 2000 && $prevchr eq $s[0] && $s[1] - $prevstart < 5000){print "$prevchr\t$prevstart\t$s[0]\t$s[1]\t$s[2]\t$len\n";}} $prevchr = $s[0]; $prevstart = $s[1]; $prevlen = $len;}' < papadum-v4bng-pilon.gaps.bed > papadum-v4bng-pilon.gaps.nickase.problems

grep Scaffold_348 papadum-v4bng-pilon.gaps.nickase.problems
	Scaffold_348    117431  Scaffold_348    117611  119354  1743
	Scaffold_348    2501404 Scaffold_348    2501509 2502751 1242

grep Scaffold_348 papadum-v4bng-pilon-split2kbgap.original.coords.bed
	Scaffold_348.1  1       119362
	Scaffold_348.2  160777  238774
	Scaffold_348.3  893773  1002703
	Scaffold_348.4  1723544 1989144
	Scaffold_348.5  2147326 2860068

# Unfortunately both are contained within the split contigs
```
--
*10/2/2015*

OK, time to fix these smaller gap regions. I can do this quickly with the gap bed file.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly

```bash
# Concatenating gap regions that are separated by only a few bases
perl concatenate_smallgaps_gapbed.pl < papadum-v4bng-pilon.gaps.bed > papadum-v4bng-pilon.gaps.concat.bed

# Removing gaps less than 2kb
perl -lane 'if($F[2] - $F[1] > 2000){print $_;}' < papadum-v4bng-pilon.gaps.concat.bed > papadum-v4bng-pilon.gaps.concat.gt2kb.bed

wc -l papadum-v4bng-pilon.gaps.bed papadum-v4bng-pilon.gaps.concat.bed papadum-v4bng-pilon.gaps.concat.gt2kb.bed
  8593 papadum-v4bng-pilon.gaps.bed
  1170 papadum-v4bng-pilon.gaps.concat.bed
   897 papadum-v4bng-pilon.gaps.concat.gt2kb.bed

# A fair reduction in gap intervals! Let's check against the prior gt2kb gap file
perl ~/bin/table_bed_length_sum.pl papadum-v4bng-pilon.gaps.concat.gt2kb.bed papadum-v4bng-pilon.gaps.gt2kb.bed papadum-v4bng-pilon.gaps.bed
FName   IntNum  TotLen  LenAvg  LenStdev        LenMedian       SmallestL       LargestL
papadum-v4bng-pilon.gaps.concat.gt2kb.bed       897     78,854,636        87909.2931995541        161683.087789662        48157   2015    1985589
papadum-v4bng-pilon.gaps.gt2kb.bed      7491    77,786,831        10384.0383126418        21937.0894821001        6175    2001    720841
papadum-v4bng-pilon.gaps.bed    8593    79,000,302        9193.56476201559        20717.7018212084        5348    0       720841

# OK, so we've got an increase of 1mb of gaps compared to the prior entry, but not that large of a difference overall

# Beginning work on preparing the fasta
perl ~/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Papadum-v4BNG/papadum-v4bng-pilon.fa -o papadum-v5bng-pilon-split2kbgap.fa -b papadum-v4bng-pilon.gaps.concat.gt2kb.bed -s papadum-v5bng-pilon-split2kbgap.original.coords.bed -m 2000 -n 0.5

# Checking scaffold counts
samtools faidx papadum-v5bng-pilon-split2kbgap.fa
wc -l *.fai
   2084 papadum-v4bng-pilon-split2kbgap.fa.fai
   2091 papadum-v5bng-pilon-split2kbgap.fa.fai	<- about 7 more split contigs
  33767 USDA_V3.fasta.fai

# BWA alignment
bwa index papadum-v5bng-pilon-split2kbgap.fa
bwa mem papadum-v5bng-pilon-split2kbgap.fa /mnt/nfs/nfs2/GoatData/RH_map/RHmap_probe_sequences.fasta | samtools view -bS - | samtools sort -T papadum.tmp -o rh_map/papadum-v5bng-pilon-split2kbgap.bam -

samtools index rh_map/papadum-v5bng-pilon-split2kbgap.bam

# Now the RH map probe ordering
perl ~/perl_toolchain/assembly_scripts/probeMappingRHOrdering.pl rh_map/RHmap_to_PacBio-v3_contigs.txt rh_map/papadum-v5bng-pilon-split2kbgap.bam rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.out

perl remove_spurious_probe_rh.pl < rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.out > rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.edit.out
perl condense_rh_remappings_order.pl < rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.edit.out > rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.edit2.out

wc -l rh_map/*out
   765 rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit2.out
  1000 rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.edit.out
  1316 rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out
   783 rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.edit2.out
  1009 rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.edit.out
  1325 rh_map/papadum-v5bng-pilon-split2kbgap.rhorder.out

# Now the manual editing
# area 1
4       Scaffold_81

# area 2
5       Scaffold_199
5       utg9423
5       Scaffold_205

# area 3
21      Scaffold_1771
21      Scaffold_1965

# area 4
29      Scaffold_59
29      Scaffold_7
```

<a name="rderrorcorr"></a>
## Lachesis read depth error correction

I need to calculate the read depth of the Lachesis scaffolds and subtract obvious deletion sites from gap regions.

Let's use JaRMS to try to find these deletion regions. Here are the files I need:

* /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Lachesis-2015-10-22-min/papadum-v5lachesis.full.fa
* /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat250-PBv4lachesis.bam
* /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.bam

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar call -i /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat250-PBv4lachesis.bam -i /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.bam -f /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Lachesis-2015-10-22-min/papadum-v5lachesis.full.fa -o papadum_pbv4_lachesis_jarms -t 5 -w 100

# Hopefully things don't blow up!
# OK, things blew up. I'm going to have to use the "interpret" function
~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar interpret -i papadum_pbv4_lachesis_jarms.gccorr.tmp -o papadum_pbv4_gccorr_windows.bed

# Now to eliminate the degenerate contigs
grep -v 'dtg' papadum_pbv4_gccorr_windows.bed > papadum_pbv4_gccorr_nodeg.bed

# Correcting the start coordinate errors from the gccorr module (gotta fix this later)
perl -lane '$F[1] -= 99; print join("\t", @F);' < papadum_pbv4_gccorr_nodeg.bed > corrected
mv corrected papadum_pbv4_gccorr_nodeg.bed

# getting just the Lachesis clusters
grep 'Lachesis' papadum_pbv4_gccorr_nodeg.bed > papadum_pbv4_gccorr_only_lachesis.bed

# Just like with the bird data, I'm going to pass them through Alkan's segmentation algorithm
cat papadum_pbv4_gccorr_only_lachesis.bed | cut -f4 | statStd.pl
	total   25880541
	Minimum 0.0
	Maximum 56241.49243645622		<- wow! There is one big window then!
	Average 29.557580
	Median  29.809064995042757
	Standard Deviation      17.575781
	Mode(Highest Distributed Value) 29.186489089417787

# We'll search for windows under the average - 1.5 stdevs (4 reads)
# Also, 4 windows with at least 3 windows under the cutoff value
perl ~/wssd-package/wssd_picker.pl -f papadum_pbv4_gccorr_only_lachesis.bed -w 4 -s 3 -c 4 -b 3 -n 5 -i 1 -m -o papadum_pbv4_gccorr_only_lachesis.dels.bed

wc -l papadum_pbv4_gccorr_only_lachesis.dels.bed
	6606 papadum_pbv4_gccorr_only_lachesis.dels.bed

# Quite a few events! Let's remove the gaps
sed 1d papadum_pbv4_gccorr_only_lachesis.dels.bed > temp
mv temp papadum_pbv4_gccorr_only_lachesis.dels.bed
mergeBed -i papadum_pbv4_gccorr_only_lachesis.dels.bed > papadum_pbv4_gccorr_only_lachesis.dels.merged.bed

~/jdk1.8.0_05/bin/java -jar ~/GetMaskBedFasta/store/GetMaskBedFasta.jar -f papadum_v5lachesis.full.fa -o papadum_v5lachesis.gaps.bed -s papadum_v5lachesis.gaps.stats
intersectBed -a papadum_pbv4_gccorr_only_lachesis.dels.merged.bed -b papadum_v5lachesis.gaps.bed -v > papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed

wc -l papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed
	4879 papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed

# Some! Were removed!
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/bed_length_sum.pl papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed
        Interval Numbers:       4879
        Total Length:           3905121
        Length Average:         800.393728222997
        Length Median:          499
        Length Stdev:           755.241372796992
        Smallest Length:        299
        Largest Length:         8799

# Also most were small. Let's check this against Serge's data
```

Serge's files are located in these locations:
> /mnt/nfs/nfs2/GoatData/Sergey201511

* out.rg.bam
* out.rg.bam.vcf
* out.splitters.rg.bam
* runLumpy.sh
* run.sh
* sort.sh

Let's check to see if the file stats match up to Serge's email (sometimes the files might be different!).

```bash
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -e '#' -f /mnt/nfs/nfs2/GoatData/Sergey201511/out.rg.bam.vcf -c 4 -m | head -n 25
```

|Entry                                                      | Count|
|:----------------------------------------------------------|-----:|
|DEL                                                      |  1152|
|DUP                                                      |    54|
|INV                                                      |     4|
|N[Lachesis_group0__33_contigs__length_157380104:100058933[ |     1|
|N[Lachesis_group0__33_contigs__length_157380104:116548985[ |     1|
|N[Lachesis_group0__33_contigs__length_157380104:116549078[ |     1|
|N[Lachesis_group0__33_contigs__length_157380104:137582245[ |     1|
|N[Lachesis_group0__33_contigs__length_157380104:59946576[  |     1|
|N[Lachesis_group0__33_contigs__length_157380104:65261217[  |     1|
|N[Lachesis_group10__21_contigs__length_94630891:62519055[  |     1|
|N[Lachesis_group10__21_contigs__length_94630891:81453433[  |     1|
|N[Lachesis_group10__21_contigs__length_94630891:88355619[  |     1|
|N[Lachesis_group11__32_contigs__length_91726360:3755997[   |     1|
|N[Lachesis_group11__32_contigs__length_91726360:84181818[  |     1|
|N[Lachesis_group12__41_contigs__length_87347178:42284630[  |     1|
|N[Lachesis_group12__41_contigs__length_87347178:73180301[  |     1|
|N[Lachesis_group12__41_contigs__length_87347178:73548433[  |     1|
|N[Lachesis_group12__41_contigs__length_87347178:73548460[  |     1|
|N[Lachesis_group12__41_contigs__length_87347178:73548684[  |     1|
|N[Lachesis_group13__19_contigs__length_83084112:22889562[  |     1|
|N[Lachesis_group13__19_contigs__length_83084112:31075263[  |     1|
|N[Lachesis_group13__19_contigs__length_83084112:31754806[  |     1|
|N[Lachesis_group13__19_contigs__length_83084112:70714264[  |     1|
...

The rest were all of the "BND" events, which appear to be "paired" complex rearrangements or transchromosomal events. Let's use a Lumpy script in order to get the BEDPE readout of these events.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
python ~/lumpy/scripts/vcfToBedpe -i /mnt/nfs/nfs2/GoatData/Sergey201511/out.rg.bam.vcf -o out.rg.bam.bedpe
# Gave a traceback and error:
	Traceback (most recent call last):
  File "/home/dbickhart/lumpy/scripts/vcfToBedpe", line 448, in <module>
    sys.exit(main())
  File "/home/dbickhart/lumpy/scripts/vcfToBedpe", line 443, in main
    vcfToBedpe(args.input, args.output)
  File "/home/dbickhart/lumpy/scripts/vcfToBedpe", line 315, in vcfToBedpe
    if int(var.info[ev]) > 0:
	KeyError: 'PE'
```

So, the VCF does not have a "PE" key and the script assumes that it does! Time to change it.

Modified code:

> lines 313 - 318
```python
ev_list = list()
            for ev in ['PE', 'SR']:
                if ev in var.info:
                        if int(var.info[ev]) > 0:
                                ev_list.append(ev)
            evtype = ','.join(ev_list)
```

Also

> lines 326 - 329
```python
format_dict = dict(zip(format_list, gt.split(':')))
                format_dict['PE'] = int(format_dict.get('PE', 0))
                format_dict['SR'] = int(format_dict.get('SR',0))
                format_dict['SU'] = int(format_dict.get('SU',0))
```

OK, now the script worked.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
python ~/lumpy/scripts/vcfToBedpe -i /mnt/nfs/nfs2/GoatData/Sergey201511/out.rg.bam.vcf -o out.rg.bam.bedpe

# Retrieving deletions
perl -lane 'if($F[10] eq "DEL"){print "$F[0]\t$F[1]\t$F[5]";}' < out.rg.bam.bedpe > serge_lumpy_dels.bed
# Retrieving dups
perl -lane 'if($F[10] eq "DUP"){print "$F[0]\t$F[1]\t$F[5]";}' < out.rg.bam.bedpe > serge_lumpy_dups.bed

# Getting only the BND events
perl -lane 'if($F[10] eq "BND"){print $_;}' < out.rg.bam.bedpe > serge_lumpy_bnds.bedpe

# Getting only the same scaffold BND events (inversions)
perl -lane 'if($F[0] eq $F[3]){print $_;}' < serge_lumpy_bnds.bedpe > serge_lumpy_bnds.same.bedpe
# Getting all other BNDs
perl -lane 'if($F[0] ne $F[3]){print $_;}' < serge_lumpy_bnds.bedpe > serge_lumpy_bnds.trans.bedpe

wc -l *.bed*
	   944 serge_lumpy_bnds.bedpe
       136 serge_lumpy_bnds.same.bedpe
       808 serge_lumpy_bnds.trans.bedpe
      1152 serge_lumpy_dels.bed
        54 serge_lumpy_dups.bed

```

Let's gather some summary stats. Deletions first (they're easier).

```bash
# Deletions -- JaRMs vs Lumpy SR
intersectBed -a papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -b serge_lumpy_dels.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | mergeBed -i stdin | wc -l
	1318

# Deletions -- Lumpy SR vs JaRMs
intersectBed -a serge_lumpy_dels.bed -b papadum_v5lachesis.gaps.bed -v | intersectBed -a stdin -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | mergeBed -i stdin | wc -l
	185

intersectBed -a serge_lumpy_dels.bed -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | mergeBed -i stdin | wc -l
	1318

# Hmmm... the gaps are acting very strangely here
intersectBed -a serge_lumpy_dels.bed -b papadum_v5lachesis.gaps.bed -v | wc -l
	944

# Checking the size of deletions that were not detected by JaRMs
intersectBed -a serge_lumpy_dels.bed -b papadum_v5lachesis.gaps.bed -v | intersectBed -a stdin -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -v | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/bed_length_sum.pl
        Interval Numbers:       762
        Total Length:           1539129
        Length Average:         2019.85433070866
        Length Median:          1201
        Length Stdev:           3632.39573243275
        Smallest Length:        6
        Largest Length:         47826

# Some are very small INDELs, but allot are within our range of detection (ie > 400 bp).
# Could be repetitive mappings, but let's see
# The first interval was repetitive, but the second had some discrepencies:

perl -lane 'if($F[0] eq "Lachesis_group0__33_contigs__length_157380104" && $F[2] >= 41479008 && $F[1] <= 41482513){print $_;}' < papadum_pbv4_gccorr_windows.bed | less
Lachesis_group0   41479099        41479099        2.051157503712246
Lachesis_group0   41479199        41479199        10.354122099489176
Lachesis_group0   41479299        41479299        0.9728829696472595
Lachesis_group0   41479399        41479399        6.008097420602061

# We'll accept that my read depth signal was very good and move on from there.
```

I really want to test to see if the BNDs align with Shawn's RH map inversions. Let's write a script to identify the regions that we need to check.

The [script](https://github.com/njdbickhart/perl_toolchain/blob/master/assembly_scripts/identifyLachesisProbPoints.pl) should process the lachesis orderings and provide context for each scaffold in the overall clusters. Clusters are only denoted based on their number, so I need to convert them to their actual names later. I also made sure that the coordinates account for the 5 lowercase "n's" in between the scaffolds.

> Blade 14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
perl ~/perl_toolchain/assembly_scripts/identifyLachesisProbPoints.pl -i lachesis_ordering.txt -f ../../papadum-v5bng-pilon-split2kbgap.fa.fai -o lachesis_problem_regions.bed -l lachesis_cluster_contig_coords.bed

# OK, I've got the problem regions sorted out, but let's get the windows that I want to search for defects
# We'll start with 20kb windows near the breakpoints (and the secondary chr points?)
perl -lane '$s1 = $F[1] - 20000; $e1 = $F[1] + 20000; if($s1 < 0){$s1 = 0;} $s2 = $F[2] - 20000; $e2 = $F[2] + 20000; print "$F[0]\t$s1\t$e1\n$F[0]\t$s2\t$e2";' < lachesis_problem_regions.bed > lachesis_problem_regions_2kb_windows.clusters.bed

# Now to add the actual cluster names to the file
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_problem_regions_2kb_windows.clusters.bed > lachesis_problem_regions_2kb_windows.fixed.bed

# Alright, it's intersection time!
wc -l lachesis_problem_regions_2kb_windows.fixed.bed
	170 lachesis_problem_regions_2kb_windows.fixed.bed	<- 170 total windows

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -wa | uniq | wc -l
	100	<- JaRMs identified problems in read depth

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_bnds.same.bedpe -wa | uniq | wc -l
	4	<- Lumpy SV calls on same Lachesis cluster

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_bnds.trans.bedpe -wa | uniq | wc -l
	15	<- Lumpy SV calls on different Lachesis clusters

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_bnds.bedpe -wa | uniq | wc -l    
	19	<- Lumpy SV BND calls regardless of cluster orientation (sum of above numbers)

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_dels.bed -wa | uniq | wc -l
	70	<- Lumpy Del calls
intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_dups.bed -wa | uniq | wc -l
	66	<- Lumpy Dup calls
cat serge_lumpy_dels.bed serge_lumpy_dups.bed | perl ~/bin/sortBedFileSTDIN.pl | mergeBed -i stdin | intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b stdin -wa | uniq | wc -l
	94	<- Combined Lumpy Dup and Dels

intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b serge_lumpy_bnds.bedpe -wa | uniq > lumpy_bnds_overlap_20kb_wins.bed
cat serge_lumpy_dels.bed serge_lumpy_dups.bed | perl ~/bin/sortBedFileSTDIN.pl | mergeBed -i stdin | intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b stdin -wa | uniq | intersectBed -a stdin -b lumpy_bnds_overlap_20kb_wins.bed -wa | uniq | wc -l
	8	<- BND + Del + Dup identified breakpoints
```

I'm skeptical about this -- let's see if there is an enrichment of read depth loss/BND events near the ends of each contig as well!

```bash
perl -lane '$s1 = $F[1] - 20000; $e1 = $F[1] + 20000; if($s1 < 0){$s1 = 0;} $s2 = $F[2] - 20000; $e2 = $F[2] + 20000; print "$F[0]\t$s1\t$e1\n$F[0]\t$s2\t$e2";' < lachesis_cluster_contig_coords.bed > lachesis_cluster_contig_coords_20kb_wins.bed
wc -l lachesis_cluster_contig_coords_20kb_wins.bed
	3052 lachesis_cluster_contig_coords_20kb_wins.bed

perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_cluster_contig_coords_20kb_wins.bed > lachesis_cluster_contig_coords_20kb_wins.fixed.bed

# OK! Intersection time here too!
intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -wa | uniq | wc -l
	1636 <- JaRMs

intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b serge_lumpy_bnds.same.bedpe -wa | uniq | wc -l
	175	<- Lumpy bnd same
intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b serge_lumpy_bnds.trans.bedpe -wa | uniq | wc -l
	198	<- Lumpy bnd trans
intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b serge_lumpy_bnds.bedpe -wa | uniq | wc -l
	363	<- Lumpy bnd total

intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b serge_lumpy_dels.bed -wa | uniq | wc -l  
	1484
intersectBed -a lachesis_cluster_contig_coords_20kb_wins.fixed.bed -b serge_lumpy_dups.bed -wa | uniq | wc -l
	1257

# Hmmm... not helping! Let's try within 10kb to see if that changes things
# Also, I'm going to make a shell script instead of typing this all out at once

# 2kb wins
perl -lane '$s1 = $F[1] - 2000; $e1 = $F[1] + 2000; if($s1 < 0){$s1 = 0;} $s2 = $F[2] - 2000; $e2 = $F[2] + 2000; print "$F[0]\t$s1\t$e1\n$F[0]\t$s2\t$e2";' < lachesis_cluster_contig_coords.bed > lachesis_cluster_contig_coords_2kb_wins.bed
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_cluster_contig_coords_2kb_wins.bed > lachesis_cluster_contig_coords_2kb_wins.fixed.bed

sh bed_intersection_script.sh lachesis_cluster_contig_coords_2kb_wins.fixed.bed
JaRMs
1030
lumpy bnd same
145
lumpy bnd trans
162
lumpy bnd all
303
lumpy dels
1444
lumpy dups
1248
combined lumpy dups dels
1904
BND + Del + Dup shared calls
203

perl -lane '$s1 = $F[1] - 2000; $e1 = $F[1] + 2000; if($s1 < 0){$s1 = 0;} $s2 = $F[2] - 2000; $e2 = $F[2] + 2000; print "$F[0]\t$s1\t$e1\n$F[0]\t$s2\t$e2";' < lachesis_problem_regions.bed > lachesis_problem_regions_actual_2kb_windows.bed
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_problem_regions_actual_2kb_windows.bed > lachesis_problem_regions_actual_2kb_windows.fixed.bed

sh bed_intersection_script.sh lachesis_problem_regions_actual_2kb_windows.fixed.bed
JaRMs
68
lumpy bnd same
3
lumpy bnd trans
12
lumpy bnd all
15
lumpy dels
65
lumpy dups
63
combined lumpy dups dels
86
BND + Del + Dup shared calls
7

# I'm still getting the same issues!
# How about a 2kb window with the first 500 bases occluded?
perl -lane '$s1 = $F[1] + 500; $e1 = $F[1] + 2000; if($s1 < 0){$s1 = 0;} if($e1 < 500){$e1 = 500;} $s2 = $F[2] - 2000; $e2 = $F[2] - 500; print "$F[0]\t$s1\t$e1\n$F[0]\t$s2\t$e2";' <lachesis_cluster_contig_coords.bed > lachesis_cluster_contig_coords_2kb_occlude.bed
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_cluster_contig_coords_2kb_occlude.bed > lachesis_cluster_contig_coords_2kb_occlude.fixed.bed

sh bed_intersection_script.sh lachesis_cluster_contig_coords_2kb_occlude.fixed.bed                             JaRMs
590
lumpy bnd same
0
lumpy bnd trans
2
lumpy bnd all
2
lumpy dels
1349
lumpy dups
1230
combined lumpy dups dels
1832
BND + Del + Dup shared calls
1
```

No clear pattern, and the occlusion strategy didn't really improve things.

Let's try to show the match up with Lumpy and JaRMs using a VENN diagram

```python
import pybedtools
lumpy = pybedtools.BedTool('serge_lumpy_dels.bed')
jarms = pybedtools.BedTool('papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed')
(lumpy + jarms).count()
	228
(lumpy - jarms).count()
	924
(jarms - lumpy).count()
	3561
```

Not the best overlap, but it's because of the different resolution sizes of the techniques and the different biases they have (ie. JaRMs works only on aligned reads and has GC bias, split read alignment can have misalignments giving false signal)

<a name="pbjelly"></a>
## Filling gaps with pbJelly

Serge has sent me the coordinates of the pbJelly filled gaps. PbJelly could provide the context that allows us to filter away the spurious SV calls and focus on the real events. Let's check the overlap of these filled gaps with the problem regions.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
# I've gotta convert the format here so that my bed files are comparable
grep -v 'na' Lachesis_assembly.gapInfo.bed | perl -lane '$F[0] =~ s/(.+)\|.+/$1/; print "$F[0]\t$F[1]\t$F[2]\t$F[3]";' > lachesis_pbjelly_corrected_gaps.bed

# Now the intersections
intersectBed -a lachesis_pbjelly_corrected_gaps.bed -b lachesis_problem_regions_actual_2kb_windows.fixed.bed | wc -l
	145 out of 175 (82%)

# Hmm... that's a huge proportion!
intersectBed -a lachesis_pbjelly_corrected_gaps.bed -b lachesis_cluster_contig_coords_2kb_wins.fixed.bed | wc -l
	2990 out of 3052 (97.9%)

# Now against the other calls
sh bed_intersection_script.sh lachesis_pbjelly_corrected_gaps.bed
JaRMs
0
lumpy bnd same
62
lumpy bnd trans
68
lumpy bnd all
128
lumpy dels
865
lumpy dups
708
combined lumpy dups dels
1119
BND + Del + Dup shared calls
86

# OK, that removes allot, but there are no JaRMs call overlaps... interesting
```

Serge sent me a category tab file with the types of gaps that pbJelly filled (or couldn't fill!). Let's take a look at the gap classes:

```bash
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f gap_fill_status.txt -c 1 -m
```
#### pbJelly gap types

|Entry         | Count|
|:-------------|-----:|
|doubleextend  |     5|
|filled        |   840|
|minreadfail   |   383|
|nofillmetrics |    16|
|overfilled    |   827|
|singleextend  |   622|

The following classes are safe and probably won't reveal flaws: **filled** and **overfilled.** However, **doublextend,** **nofillmetrics** and **singleextend** are likely targets. The **minreadfail** is a definite problem region.

Let's break apart the bed file and prepare it for intersection.

```bash
# I need to also prepare the contig names for association
perl -lane 'if($F[0] =~ /(ref.+)\.(\d+)e\d+\_ref.+\.(\d+)e\d+/){$d = "$1\_$2\_$3"; print "$d\t$F[1]";}' < gap_fill_status.txt > gap_fill_converted_status.txt
wc -l gap_fill_converted_status.txt
	1746 gap_fill_converted_status.txt

perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f gap_fill_converted_status.txt -c 1 -m
```

|Entry        | Count|
|:------------|-----:|
|doubleextend |     5|
|filled       |   765|
|minreadfail  |   148|
|overfilled   |   827|
|singleextend |     1|

```bash
# Now, splitting the gapfilled bed file
perl -e 'chomp(@ARGV); %err; open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[1] eq "minreadfail" || $s[1] eq "singleextend" || $s[1] eq "doubleextend"){$err{$s[0]} = 1;}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if(exists($err{$s[3]})){ next;}else{print "$_\n";}}' gap_fill_converted_status.txt lachesis_pbjelly_corrected_gaps.bed > lachesis_pbjelly_true_corrected_gaps.bed
perl -e 'chomp(@ARGV); %err; open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[1] eq "minreadfail" || $s[1] eq "singleextend" || $s[1] eq "doubleextend"){$err{$s[0]} = 1;}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if(exists($err{$s[3]})){ next;}else{print "$_\n";}}' gap_fill_converted_status.txt lachesis_pbjelly_corrected_gaps.bed | wc -l
	1601

perl -e 'chomp(@ARGV); %err; open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[1] eq "minreadfail" || $s[1] eq "singleextend" || $s[1] eq "doubleextend"){$err{$s[0]} = 1;}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if(exists($err{$s[3]})){ print "$_\n";}}' gap_fill_converted_status.txt lachesis_pbjelly_corrected_gaps.bed > lachesis_pbjelly_uncorrected_gaps.bed
perl -e 'chomp(@ARGV); %err; open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[1] eq "minreadfail" || $s[1] eq "singleextend" || $s[1] eq "doubleextend"){$err{$s[0]} = 1;}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if(exists($err{$s[3]})){ print "$_\n";}}' gap_fill_converted_status.txt lachesis_pbjelly_corrected_gaps.bed | wc -l         
	154

# I realized that these are gap regions, and I filtered them out from JaRMs output! Let's add them back in
intersectBed -a papadum_pbv4_gccorr_only_lachesis.dels.merged.bed -b papadum_v5lachesis.gaps.bed -wa | uniq > papadum_pbv4_gccorr_only_lachesis.dels.gaps.bed

# OK, let's see the overlaps
sh bed_gap_intersection_script.sh lachesis_pbjelly_uncorrected_gaps.bed
JaRMs
130
lumpy bnd same
2
lumpy bnd trans
3
lumpy bnd all
5
lumpy dels
62
lumpy dups
29
combined lumpy dups dels
76
BND + Del + Dup shared calls
2

sh bed_gap_intersection_script.sh lachesis_pbjelly_true_corrected_gaps.bed
JaRMs
681
lumpy bnd same
60
lumpy bnd trans
65
lumpy bnd all
123
lumpy dels
803
lumpy dups
679
combined lumpy dups dels
1043
BND + Del + Dup shared calls
84

# JaRMs hits almost all of them, but let's see what ones it misses
intersectBed -a lachesis_pbjelly_uncorrected_gaps.bed -b papadum_pbv4_gccorr_only_lachesis.dels.gaps.bed -v | head
	Lachesis_group10__21_contigs__length_94630891   5007925 5007931 ref0000018_3_4

cluster_10      4979950 5007925 utg41548
cluster_10      5007930 62519049        Scaffold_91.3

# There are some fluctuations in read depth, but they aren't 4 consecutive windows.
# Let's bite the bullet and run a combined Lumpy run
samtools view -b -F 1294 /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.bam > Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.discpe.bam
samtools view -h /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.bam | ~/lumpy/scripts/extractSplitReads_BwaMem -i stdin | samtools view -Sb - > Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.sr.bam

# Making the histo
samtools view /mnt/nfs/nfs2/GoatData/Ilmn/papadum-v5lachesis-full/bwa-out/Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.bam | head -n 500000 | ~/lumpy/scripts/pairend_distro.py -r 101 -X 4 -N 10000 -o Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.histo
Removed 6 outliers with isize >= 636
mean:408.08304983       stdev:54.122420943

~/lumpy/bin/lumpy -mw 4 -tt 0 -sr id:papadum,bam_file:Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.sr.bam,back_distance:10,weight:1,min_mapping_threshold:20 -pe id:papadum,bam_file:Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.discpe.bam,histo_file:Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.histo,mean:408,stdev:54,read_length:101,min_non_overlap:101,discordant_z:5,back_distance:10,weight:1,min_mapping_threshold:20 > Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf

# It turns out that the output is NOT a vcf! It's a Bedpe!
# Counting the number of events
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf -c 10 -m
```

|Entry            | Count|
|:----------------|-----:|
|TYPE:DELETION    |  1115|
|TYPE:DUPLICATION |   477|
|TYPE:INTERCHROM  |  2514|
|TYPE:INVERSION   |   505|

This is pretty close to Serge's data. Let's see the intersection.

```bash
# Flat intersection (not caring about SV type)
intersectBed -a Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf -b out.rg.bam.bedpe | wc -l
547	
# Removal of pbjelly ID'd gap fills
intersectBed -a Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf -b lachesis_pbjelly_true_corrected_gaps.bed -v | intersectBed -a stdin -b out.rg.bam.bedpe | wc -l
374
```

So there is a difference -- the signatures of read pairing are not necessarily correlated with misassemblies here. Let's try to see if corrected gaps that were "overfilled" match up more frequently with "deletions" because of the "overfilling."

```bash
# Creating overfilled-only gap file
perl -e 'chomp(@ARGV); %err; open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[1] eq "minreadfail" || $s[1] eq "singleextend" || $s[1] eq "doubleextend" || $s[1] eq "filled"){$err{$s[0]} = 1;}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if(exists($err{$s[3]})){ next;}else{print "$_\n";}}' gap_fill_converted_status.txt lachesis_pbjelly_corrected_gaps.bed > lachesis_pbjelly_overfilled_corrected_gaps.bed

# Testing the intersection again -- should be 374 if perfect
intersectBed -a Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf -b lachesis_pbjelly_overfilled_corrected_gaps.bed -v | intersectBed -a stdin -b out.rg.bam.bedpe | wc -l
398
```
<a name="pbjellysv"></a>
### Associating pbJelly filled gaps with SV signal

That's most of them! Great! So only a few "filled" gaps (24) had intersected with Illumina-based RP variant signals. I want to take the unfilled gaps and identify how often they intersect. From there, we'll break the signal and "eyeball" whether or not it coincides with large inversions or "hitchhiking" contigs.

```bash
# First the bed intersection
cat serge_lumpy_dels.lt20kb.bed serge_lumpy_dups.bed papadum_pbv4_gccorr_only_lachesis.dels.gaps.bed | perl ~/bin/sortBedFileSTDIN.pl | intersectBed -a lachesis_pbjelly_uncorrected_gaps.bed -b stdin -c > filter_pbjelly_uncorrected_bed_intersect_count.bed

# Now the bedpe intersection
cat Goat-Ilmn-HiSeq-Goat400-PBv4lachesis.lumpy.vcf serge_lumpy_bnds.bedpe | intersectBed -a lachesis_pbjelly_uncorrected_gaps.bed -b stdin -c > filter_pbjelly_uncorrected_bedpe_intersect_count.bed

# Just a NOTE! Combined, the illumina and pacbio split reads had 15 intersections (10 for illumina, 5 for pacbio)
# The two intersection sets did not overlap

# Combining the two count columns
cat filter_pbjelly_uncorrected_bed* | perl -e '%h; while(<STDIN>){chomp; @s = split(/\t/); $b = "$s[0]-$s[1]-$s[2]-$s[3]"; $h{$b} += $s[4];} foreach $k (sort {$a cmp $b} keys(%h)){@n = split(/\-/, $k); print join("\t", @n) . "\t$h{$k}\n";}' > filter_pbjelly_uncorrected_combined_intersect_count.bed

# Now, the total number of uncorrected gaps that have some evidence that they need to be split
perl -lane 'if($F[4]){print $_;}' < filter_pbjelly_uncorrected_combined_intersect_count.bed | wc -l
136
# Total count of uncorrected gaps:
wc -l filter_pbjelly_uncorrected_combined_intersect_count.bed
154 filter_pbjelly_uncorrected_combined_intersect_count.bed

# Intersections of the calls against known problem regions:
perl -lane 'if($F[4]){print $_;}' < filter_pbjelly_uncorrected_combined_intersect_count.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b stdin | wc -l
12 out of 85 <- intersections are low here

# Intersections of the unfilled gaps with no support from SV callers
perl -lane 'if(!$F[4]){print $_;}' < filter_pbjelly_uncorrected_combined_intersect_count.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | intersectBed -a lachesis_problem_regions_2kb_windows.fixed.bed -b stdin | wc -l
1 out of 85 <- ok, so the SV support is pretty reliable 

# Preparing the intersection data that will allow me to do "eyeball" comparisons
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^Lachesis.+/){$s[0] =~ /Lachesis_group(\d+)__.+/; $h{$1} = $s[0];}} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $name = $h{$b[1]}; $s[0] = $name; print join("\t", @s); print "\n";}' papadum_v5lachesis.full.fa.fai lachesis_cluster_contig_coords.bed > lachesis_cluster_contig_coords.fixed.bed

perl -lane 'if($F[4]){print $_;}' < filter_pbjelly_uncorrected_combined_intersect_count.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b stdin | wc -l
91	<- thats less than I expected
# I understand why -- the gap region is not included in the contig coords file that I have. 
# I need to extend the end of each contig by 5
# Also, the coordinates on the problem gap regions are likely based on the pbJelly final fasta

cat lachesis_cluster_contig_coords.fixed.bed | perl -lane '$F[2] += 3; print join("\t", @F);' > lachesis_cluster_contig_coords.fixed.extend.bed
cat filter_pbjelly_uncorrected_combined_intersect_count.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | intersectBed -a lachesis_cluster_contig_coords.fixed.extend.bed -b stdin -wb | uniq > filtered_problem_region_evidence.bed  <- 185 events (from cross overlap)
```

I think I'm going about this incorrectly. Let's get a gap bed file and determine intersections iteratively from that. I'll use the following datasets:

* BND calls for inversion detection
* Unfilled gaps for problem sites
* filled gaps for confirmation

I wrote a [script](https://github.com/njdbickhart/perl_toolchain/blob/master/sequence_data_scripts/LachesisGapDecisionCheck.pl) to automate these associations and to print them out to a hierarchical bed file.

```bash
perl ~/perl_toolchain/sequence_data_scripts/LachesisGapDecisionCheck.pl -c lachesis_cluster_contig_coords.fixed.bed -g lachesis_pbjelly_corrected_gaps.bed -b serge_lumpy_bnds.bedpe -f lachesis_pbjelly_true_corrected_gaps.bed -u lachesis_pbjelly_uncorrected_gaps.bed -o perl_associated_named_gap_file

# Now to sort and interleave the gaps with the original coordinates!
cat perl_associated_named_gap_file.gap.tab lachesis_cluster_contig_coords.fixed.bed | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl > perl_interleaved_sorted_anmed_gap_file.gap.tab
```

Criteria for visual inspection
* BNG scaffolds flanking unfilled gap
	* If scaffolds are portions of the same (ie. Scaffold175.1 + Scaffold175.2) the gap between them is unresolved
	* If scaffolds are different, then break at the scaffold side
		* If it matches the RH map order, place an "uknown size" gap in between
* Unfilled gaps flanking contig
	* remove from chromosome order
* Unfilled gaps near contigs
	* break based on position of gap
* Noteable inversion near 

Ultimately, use the RH map to guide assembly. 

I'm going to associate the sv information as well, and then do a quick screen.

```bash
perl ~/perl_toolchain/sequence_data_scripts/LachesisGapDecisionCheck.pl -c lachesis_cluster_contig_coords.fixed.bed -g lachesis_pbjelly_corrected_gaps.bed -b serge_lumpy_bnds.bedpe -f lachesis_pbjelly_true_corrected_gaps.bed -u lachesis_pbjelly_uncorrected_gaps.bed -o perl_associated_named_gap_file -e filter_pbjelly_uncorrected_combined_intersect_count.bed

cat perl_associated_named_gap_file.gap.tab | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl | intersectBed -a lachesis_problem_regions_actual_2kb_windows.fixed.bed -b stdin -wb | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -c 7 -f stdin -m

cat perl_associated_named_gap_file.gap.tab | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl > perl_associated_named_gap_file.gap.sorted.bed

```

<a name="probsitegapfill"></a>
### Known problem site gap fill status
|Entry                                                                                                  | Count|
|:------------------------------------------------------------------------------------------------------|-----:|
|FILLED                                                                                                 |   120|
|FILLED/Lachesis_group0__33_contigs__length_157380104;Lachesis_group7__23_contigs__length_108482926-+;- |     1|
|FILLED/Lachesis_group10__21_contigs__length_94630891;utg6725__unclustered--;-                          |     1|
|FILLED/Lachesis_group17__24_contigs__length_71147168;utg153339__unclustered--;+                        |     1|
|FILLED/Lachesis_group18__52_contigs__length_69822432--;-                                               |     1|
|FILLED/Lachesis_group19__387_contigs__length_68073856;Scaffold_1373.1__unclustered-+;-                 |     1|
|FILLED/Lachesis_group19__387_contigs__length_68073856;utg9894__unclustered--;-                         |     1|
|FILLED/Lachesis_group1__31_contigs__length_136678973--;-                                               |     1|
|FILLED/Lachesis_group1__31_contigs__length_136678973;Lachesis_group2__24_contigs__length_121104543--;- |     1|
|FILLED/Lachesis_group21__23_contigs__length_65245013;Lachesis_group22__26_contigs__length_62424687-+;+ |     1|
|FILLED/Lachesis_group24__27_contigs__length_60160405;utg9304__unclustered-+;-                          |     1|
|FILLED/Lachesis_group26__289_contigs__length_50366933;utg9373__unclustered-+;+                         |     1|
|FILLED/Lachesis_group6__33_contigs__length_112773130;Lachesis_group7__23_contigs__length_108482926--;+ |     1|
|FILLED/Lachesis_group8__26_contigs__length_106128401--;-                                               |     1|
|FILLED/Lachesis_group8__26_contigs__length_106128401;utg9532__unclustered-+;+                          |     1|
|UNFILLED                                                                                               |    11|

<a name="gapfilecols"></a>
### Here are the columns of the perl_associated_named_gap_file.gap.sorted.bed file:

1. Lachesis cluster
2. Start coord
3. End coord
4. pbJelly gap ID
5. Status
	1. "FILLED" <- pbjelly filled
	2. "FILLED/Lachesis...*" <- pbjelly filled, but has Lumpy-sv - PacBio split read discordant signal
	3. "UNFILLED" <- pbjelly couldn't fill
4. Read depth deletions, or lumpy-sv Illumina calls that are within 2kb of this region

<a name="prefinal"></a>
## Creating pre-final draft

I think that the ultimate goal is to follow the RH map order, but to keep PacBio contigs that were properly associated within the RH map. Shawn talked about a "orientation q score" value that allows us to gauge whether or not the Lachesis orientation is correct. The score is located on the "ordering" files within the "min" directory on his google drive folder. I've downloaded the ordering files and concatenated them into a single, combined file:

> Laptop: /cygdrive/c/SharedFolder/goat_assembly_paper/lachesis_ordering

```bash
for i in *.ordering; do echo $i; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); $ARGV[0] =~ /group(\d+)\.ordering/; $num = $1 * 1; while(<IN>){chomp; $_ =~ s/\r//g; if($_ =~ /^#/){next;} @s = split(/\t/); print "$num\t$s[1]\t$s[2]\t$s[3]\n";} close IN;' $i >> lachesis_combined_orient.tab; done

cat lachesis_combined_orient.tab | cut -f4 | perl ../../programming/perl_toolchain/bed_cnv_fig_table_pipeline/statStd.pl
total   1526
Minimum 0
Maximum 9706.41
Average 211.702023
Median  9.283815
Standard Deviation      787.307223
Mode(Highest Distributed Value) 0
```

The columns correspond to the following scheme:

1. Cluster number
2. Scaffold/contig ID
3. Contig is reverse complement (1 for true, 0 for false)
4. Orientation q score

The q score is a very broad distribution, with allot of zeroes apparently! Let's see if that matches up to the gaps that are part of the problem regions. I will do all of this on the server so that I can match things quickly.

> Blade14:

```bash
# Converting things for the association
cat lachesis_pbjelly_corrected_gaps.bed | perl -lane 'if($F[0] =~ /Lachesis_group(\d+)__.+/){ print "$1\t$F[1]\t$F[2]\t$F[3]";}' | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl > lachesis_pbjelly_cord_ids_gaps.bed

perl -lane '$F[0] =~ /cluster_(\d+)/; print "$1\t$F[1]\t$F[2]";' < lachesis_problem_regions_actual_2kb_windows.bed > lachesis_problem_regions_clustnum_2kb_windows.bed

# The file that contains the pbjelly gap ids within problem regions
intersectBed -a lachesis_pbjelly_cord_ids_gaps.bed -b lachesis_problem_regions_clustnum_2kb_windows.bed > lachesis_problem_regions_pbjelly_gap_names.bed

# The file that contains the putatively correct associations
intersectBed -a lachesis_pbjelly_cord_ids_gaps.bed -b lachesis_problem_regions_clustnum_2kb_windows.bed -v > lachesis_ok_regions_pbjelly_gap_names.bed

wc -l lachesis_problem_regions_pbjelly_gap_names.bed lachesis_ok_regions_pbjelly_gap_names.bed
  145 lachesis_problem_regions_pbjelly_gap_names.bed
 1604 lachesis_ok_regions_pbjelly_gap_names.bed
 1749 total <- so, not bad! 8% of the total gaps are known problems with respect to the RH map

### Generating the orientation q score beds
# Starting out by associating the pbjelly coord ids with the gap junctions
perl -e '$last = "NA"; $prev = 0; $ctg = "NA"; while(<>){chomp; @s = split(/\t/); if($last eq "NA" || $last ne $s[0]){$last = $s[0]; $prev = $s[2]; $ctg = $s[3];}else{$s[0] =~ /cluster_(\d+)/; print "$1\t$prev\t$s[1]\t$ctg;$s[3]\n"; $last = $s[0]; $prev = $s[2]; $ctg = $s[3];}}' < lachesis_cluster_contig_coords.bed > lachesis_cluster_contig_gap_coords.bed

# main association
intersectBed -a lachesis_pbjelly_cord_ids_gaps.bed -b lachesis_cluster_contig_gap_coords.bed -wb > lachesis_cluster_contig_gap_coords.pbjelly_assoc.tab

# problem regions
intersectBed -a lachesis_problem_regions_pbjelly_gap_names.bed -b lachesis_cluster_contig_gap_coords.bed -wb > lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.tab

# non-problem regions
intersectBed -a lachesis_ok_regions_pbjelly_gap_names.bed -b lachesis_cluster_contig_gap_coords.bed -wb > lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.tab

# Now to get the orientation q scores by substituting the scaffold/ctg names in the tab file I just generated!
perl -e 'chomp(@ARGV); %h; open(IN, "< $ARGV[0]"); while(<IN>){chomp; $_ =~ s/\r//g; @s = split(/\t/); $h{$s[1]} = $s[3];} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/;/, $s[-1]); @g; foreach $j (@b){push(@g, $h{$j});} print join("\t", @s) . "\t" . join(";", @g) . "\n"; @g =();}' lachesis_combined_orient.tab lachesis_cluster_contig_gap_coords.pbjelly_assoc.tab > lachesis_cluster_contig_gap_coords.pbjelly_assoc.qscore_assoc.tab

# Now for the problem regions
perl -e 'chomp(@ARGV); %h; open(IN, "< $ARGV[0]"); while(<IN>){chomp; $_ =~ s/\r//g; @s = split(/\t/); $h{$s[1]} = $s[3];} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/;/, $s[-1]); @g; foreach $j (@b){push(@g, $h{$j});} print join("\t", @s) . "\t" . join(";", @g) . "\n"; @g =();}' lachesis_combined_orient.tab lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.tab > lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.qscore_assoc.tab

perl -e 'chomp(@ARGV); %h; open(IN, "< $ARGV[0]"); while(<IN>){chomp; $_ =~ s/\r//g; @s = split(/\t/); $h{$s[1]} = $s[3];} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); @b = split(/;/, $s[-1]); @g; foreach $j (@b){push(@g, $h{$j});} print join("\t", @s) . "\t" . join(";", @g) . "\n"; @g =();}' lachesis_combined_orient.tab lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.tab > lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.qscore_assoc.tab

### Now to generate some statistics here
# I'm going to calculate the junction average and calculate summary stats from that
perl -lane '@b = split(/;/, $F[-1]); $avg = ($b[1] + $b[0]) / 2; print $avg;' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.qscore_assoc.tab | statStd.pl
total   1495
Minimum 0
Maximum 6897.385
Average 210.910559
Median  14.77334
Standard Deviation      663.234991
Mode(Highest Distributed Value) 0

perl -lane '@b = split(/;/, $F[-1]); $avg = ($b[1] + $b[0]) / 2; print $avg;' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.qscore_assoc.tab | statStd.pl
total   145
Minimum 0.149269
Maximum 5477.37
Average 269.955504
Median  31.1143875
Standard Deviation      711.407380
Mode(Highest Distributed Value) 11.00563

perl -lane '@b = split(/;/, $F[-1]); $avg = ($b[1] + $b[0]) / 2; print $avg;' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.qscore_assoc.tab | statStd.pl
total   1350
Minimum 0
Maximum 6897.385
Average 204.568695
Median  13.662974
Standard Deviation      657.816896
Mode(Highest Distributed Value) 0

perl -lane '@b = split(/;/, $F[-1]); $avg = ($b[1] + $b[0]) / 2; print $avg;' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.qscore_assoc.tab > ok_junction_vector.tab
perl -lane '@b = split(/;/, $F[-1]); $avg = ($b[1] + $b[0]) / 2; print $avg;' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.qscore_assoc.tab > problem_junction_vector.tab
```

Eyeballing the distributions, it seems like they are nearly the same. Let's test the Kolmogorov-Smirnof statistic to see if they are.

```R
ok <- read.delim("ok_junction_vector.tab", header=FALSE)
problem <- read.delim("problem_junction_vector.tab", header=FALSE)

ks.test(problem, as.numeric(ok$V1))
	#Two-sample Kolmogorov-Smirnov test

	#data:  problem and as.numeric(ok$V1)
	#D = 0.20802, p-value = 2.397e-05
	#alternative hypothesis: two-sided

ks.test(problem, as.numeric(ok$V1), alternative="less")

    #Two-sample Kolmogorov-Smirnov test

	#data:  problem and as.numeric(ok$V1)
	#D^- = 0.20802, p-value = 1.198e-05
	#alternative hypothesis: the CDF of x lies below that of y

# Let's plot this
problem$prob <- "problem"
ok$prob <- "ok"
dist <- rbind(problem, ok)
pdf(file="qorientscore.pdf", useDingbats=FALSE)

# Ran into a problem with ggplot. Gotta do it locally
ggplot(dist, aes(V1, fill=prob)) + geom_density(alpha = 0.2) + xlim(0,750)
```
One last test before I do what I promised: let's get the scores for the contigs vs the scaffolds.

```bash
# Unscaffolded unitigs
perl -lane '@b = split(/;/, $F[7]); @s = split(/;/, $F[8]); for($x = 0; $x < scalar(@b); $x++){if($b[$x] =~ /^utg/){print $s[$x];}}' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.qscore_assoc.tab | statStd.pl
	total   1470
	Minimum 0
	Maximum 53.0113
	Average 5.704568
	Median  2.86578
	Standard Deviation      7.442565
	Mode(Highest Distributed Value) 0

# scaffolds
perl -lane '@b = split(/;/, $F[7]); @s = split(/;/, $F[8]); for($x = 0; $x < scalar(@b); $x++){unless($b[$x] =~ /^utg/){print $s[$x];}}' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.qscore_assoc.tab | statStd.pl
	total   1520
	Minimum 0.00741432
	Maximum 9706.41
	Average 409.366353
	Median  35.66175
	Standard Deviation      1072.243166
	Mode(Highest Distributed Value) 11.2972

# As I thought, the pacbio unitigs are much lower. Let's see how they stack up in problem regions.
# Within problem regions
perl -lane '@b = split(/;/, $F[7]); @s = split(/;/, $F[8]); for($x = 0; $x < scalar(@b); $x++){if($b[$x] =~ /^utg/){print $s[$x];}}' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.qscore_assoc.tab | statStd.pl
	total   91
	Minimum 0
	Maximum 46.8204
	Average 4.272981
	Median  1.84092
	Standard Deviation      6.655081
	Mode(Highest Distributed Value) 2.15545

# Outside problem regions
perl -lane '@b = split(/;/, $F[7]); @s = split(/;/, $F[8]); for($x = 0; $x < scalar(@b); $x++){if($b[$x] =~ /^utg/){print $s[$x];}}' < lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.qscore_assoc.tab | statStd.pl
	total   1379
	Minimum 0
	Maximum 53.0113
	Average 5.799039
	Median  2.98478
	Standard Deviation      7.484168
	Mode(Highest Distributed Value) 0
```

OK, again, the distribution suggests that lower value contigs are likely in problem regions, but I still need manual editing. Let's do this: all problem contigs are removed in problem regions and all contigs with q scores lower than 1 are removed. Making an interleaved file with the data I will use for the separation.

```bash
cat lachesis_cluster_contig_gap_coords.pbjelly_assoc.problems.qscore_assoc.tab | perl -lane 'chomp; print "$_\tproblem";' > problem.temp
cat lachesis_cluster_contig_gap_coords.pbjelly_assoc.ok.qscore_assoc.tab | perl -lane 'chomp; print "$_\tproblem";' > ok.temp
cat lachesis_cluster_contig_coords.bed | perl -lane '$F[0] =~ /cluster_(\d+)/; print "$1\t$F[1]\t$F[2]\t$F[3]";' > scaff.temp

cat problem.temp ok.temp scaff.temp | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/sortBedFileSTDIN.pl > lachesis_interleaved_qscore_info.tab

# Turning it into one line for easier manual editing
perl -e '$last = "NA"; while($first = <>){$next = <>; chomp $first; chomp $next; @s = split(/\t/, $first); @n = split(/\t/, $next); if($n[0] ne $last && $s[0] ne $n[0]){$third = <>; chomp $third; print "$first\n$next\t$third\n"; $last = $n[0];}elsif($s[0] ne $last){print "$first\t$next\n"; $last = $s[0];}else{print "$first\t$next\n"; $last = $s[0];}}' < lachesis_interleaved_qscore_info.tab > lachesis_interleaved_qscore_info.oneline.tab

# I created a script that should process things and generate the final files
perl ~/perl_toolchain/assembly_scripts/reorderLachesisDecisTree.pl -t /mnt/nfs/nfs2/dbickhart/transfer/lachesis_ordering_decisions.txt -f ../../papadum-v5bng-pilon-split2kbgap.fa -i ../../papadum-v5bng-pilon-split2kbgap.fa.fai -o papadum-v6lach-bng-reorder

# Doing a quick check
samtools faidx papadum-v6lach-bng-reorder.fa
wc -l papadum-v6lach-bng-reorder.fa.fai
	31 papadum-v6lach-bng-reorder.fa.fai

tail papadum-v6lach-bng-reorder.fa.fai
cluster_30      35510596        2549820794      60      61
```

Lost quite a bit, but it's a very good start! Packaging the fasta and getting ready to ship it to the group.

Here are the conditions for the Lachesis decision file:

* A part is "removed" if
	* It is not a BNG scaffold
	* It has no RH map position
	* It has a q score alignment of < 1
* A part is a "problem" if
	* It has a different RH map order
	* Note that "problem" entries are included in the cluster to be reordered

Just a few things that bother me:

* There are some BNG scaffolds that were not incorporated by Lachesis
* We're missing about 300 mb

Let's try to clear up the first point.

```bash
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[5] =~ /^(Scaffold_\d+)\..+/){$h{$1} = 1; }} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^(Scaffold_\d+)\..+/){if(exists($h{$1})){ print join("\t", @s) . "\n";}}}' papadum-v6lach-bng-reorder.agp papadum_v5lachesis.full.fa.fai |wc -l
25 <- 25 scaffold segments that are missing

perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; @s = split(/\t/); if($s[5] =~ /^(Scaffold_\d+)\..+/){$h{$1} = 1; }} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^(Scaffold_\d+)\..+/){if(exists($h{$1})){ print join("\t", @s) . "\n";}}}' papadum-v6lach-bng-reorder.agp papadum_v5lachesis.full.fa.fai | cut -f2 | perl -e '$c = 0; while(<STDIN>){chomp; $c += $_;} print "$c\n";'
4,759,053 <- it's a start!
```

These are small enough that I can add them into the decision text.

```bash
perl ~/perl_toolchain/assembly_scripts/reorderLachesisDecisTree.pl -t /mnt/nfs/nfs2/dbickhart/transfer/lachesis_ordering_decisions.txt -f ../../papadum-v5bng-pilon-split2kbgap.fa -i ../../papadum-v5bng-pilon-split2kbgap.fa.fai -o papadum-v6lach-bng-reorder.norm
perl ~/perl_toolchain/assembly_scripts/reorderLachesisDecisTree.pl -t lachesis_ordering_decisions.txt -f ../../papadum-v5bng-pilon-split2kbgap.fa -i ../../papadum-v5bng-pilon-split2kbgap.fa.fai -o papadum-v6lach-bng-reorder.extra

## NOTE: the lachesis_ordering_decision.txt file in the directory is modified to have extra 
# elements that were BNG mapped but not Lachesis mapped
```

#### Checking one last problem site

There were a few scaffolds that appear to cause some inconsistencies in the nucmer plots. I'm going to test them for problems using a few of the SV metrics. For instance: count of times that they overlap with problem sites.

> Blade14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage

```bash
# first, the overall statistics of overlap of read depth and lumpy
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b pav4_gccorr_only_lachesis.dels.nogaps.bed -c | cut -f5 | statStd.pl
total   1526
Minimum 0
Maximum 73
Average 3.197248
Median  1
Standard Deviation      5.571333
Mode(Highest Distributed Value) 0

# Now, for lumpy
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_dels.lt20kb.bed -c | cut -f5 | statStd.pl
total   1526
Minimum 0
Maximum 27
Average 0.722149
Median  0
Standard Deviation      2.445497
Mode(Highest Distributed Value) 0

intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_bnds.bedpe -c | cut -f5 | statStd.pl
total   1526
Minimum 0
Maximum 86
Average 0.385321
Median  0
Standard Deviation      2.474186
Mode(Highest Distributed Value) 0

# So most contigs/scaffolds have few, if any, SV profiles
# Let's test it on specific cases
# Read depth
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -c | grep 'Scaffold_240'
Lachesis_group21__23_contigs__length_65245013   22837860        25311128        Scaffold_240.2  2
Lachesis_group21__23_contigs__length_65245013   25367924        30134733        Scaffold_240.3  3
Lachesis_group21__23_contigs__length_65245013   33174969        65245008        Scaffold_240.1  32

# SV dels
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_dels.lt20kb.bed -c | grep 'Scaffold_240'
Lachesis_group21__23_contigs__length_65245013   22837860        25311128        Scaffold_240.2  0
Lachesis_group21__23_contigs__length_65245013   25367924        30134733        Scaffold_240.3  2
Lachesis_group21__23_contigs__length_65245013   33174969        65245008        Scaffold_240.1  15

# SV BND
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_bnds.bedpe -c | grep 'Scaffold_240'
Lachesis_group21__23_contigs__length_65245013   22837860        25311128        Scaffold_240.2  0
Lachesis_group21__23_contigs__length_65245013   25367924        30134733        Scaffold_240.3  0
Lachesis_group21__23_contigs__length_65245013   33174969        65245008        Scaffold_240.1  4

# Not looking good!
# Let's segregate out all of the contigs/scaffolds that have > 2 stdevs + the average events in all three cases
intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b papadum_pbv4_gccorr_only_lachesis.dels.nogaps.bed -c | perl -lane 'if($F[4] > 13){print "$F[0]\t$F[1]\t$F[2]\t$F[3]";}' > misassembly_test_jarms_dels_problems.bed

intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_dels.lt20kb.bed -c | perl -lane 'if($F[4] > 5){print "$F[0]\t$F[1]\t$F[2]\t$F[3]";}' > misassembly_test_lumpy_dels_problems.bed

intersectBed -a lachesis_cluster_contig_coords.fixed.bed -b serge_lumpy_bnds.bedpe -c | perl -lane 'if($F[4] > 4){print "$F[0]\t$F[1]\t$F[2]\t$F[3]";}' > misassembly_test_lumpy_bnd_problems.bed

wc -l misassembly_test_*                                             
   79 misassembly_test_jarms_dels_problems.bed
   13 misassembly_test_lumpy_bnd_problems.bed
   52 misassembly_test_lumpy_dels_problems.bed
  144 total

# Converting them all to named lists
cat misassembly_test_jarms_dels_problems.bed | cut -f4 > misassembly_test_jarms_dels_problems.list
cat misassembly_test_lumpy_bnd_problems.bed | cut -f4 > misassembly_test_lumpy_bnd_problems.list
cat misassembly_test_lumpy_dels_problems.bed | cut -f4 > misassembly_test_lumpy_dels_problems.list

perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl  -o misassembly_test_jarms_dels_problems.list misassembly_test_lumpy_bnd_problems.list misassembly_test_lumpy_dels_problems.list
	File Number 1: misassembly_test_jarms_dels_problems.list
	File Number 2: misassembly_test_lumpy_bnd_problems.list
	File Number 3: misassembly_test_lumpy_dels_problems.list
	Set     Count
	1       49
	1;2     1
	1;2;3   6
	1;3     23
	2       5
	2;3     1
	3       22

grep 'Scaffold_240.1' group*.txt
	group_1_3.txt:Scaffold_240.1
```

I have a bigger problem with the RH map orientation on the agp files!

```bash
tail cluster6_lachesis_order.list
utg51695
Scaffold_1893.1
Scaffold_1849.1
Scaffold_1829.1
Scaffold_120
utg56448
utg56409
Scaffold_1849.2
Scaffold_1849.3
Scaffold_427.1

perl -lane 'if($F[0] == 6){print "$F[1]\t$F[2]\t$F[3]\t$F[4]\t$F[5]";}' <  /mnt/nfs/nfs2/dbickhart/transfer/lachesis_ordering_decisions.txt | tail
utg56409        23      .       .       +
utg56448        24      .       .       -
utg56447        25      .       .       -
Scaffold_1893.2 26      8       8       -
utg29159        27      .       .       +
utg51695        28      .       .       +
Scaffold_1893.1 29      8       9       -
Scaffold_1849.1 30      8       10      +
Scaffold_1829.1 31      8       15      +
Scaffold_120    32      8       14      +

# 1829.1 should be the final lachesis cluster but it's in the middle of the cluster6 order!
```

## Final preps and AGP file creation

Scaffold_240.1 is giving us problems, and chr3 and chr5 have some issues with the whole genome alignments.

I'm going to split Scaffold_240.1 into two subsections (Scaffold_240.1.1 and Scaffold_240.1.2) and create a new input fasta file. I'm also going to try to fix my agp output generation.

> Blade 14: /mnt/iscsi/vnx_gliu_7/goat_assembly/lachesis/test_coverage


```bash
# Checking the RH map for guidance
grep 'Scaffold_240.1' ../../rh_map/papadum-v4bng-pilon-split2kbgap.rhorder.out
8       Scaffold_240.1  18877424        18877424        ?
20      Scaffold_240.1  19726450        19726450        ?
23      Scaffold_240.1  33537   13295580        -
26      Scaffold_240.1  31865478        31951360        -
26      Scaffold_240.1  31598246        31762758        -
26      Scaffold_240.1  26714379        31381416        -
26      Scaffold_240.1  13867734        26665154        -
```

So, we should split at the 13295580 - 13867734 coordinate breakpoints

```bash
mkdir redo_240
echo -e "Scaffold_240.1\t13295580\t13867734" > redo_240/scaffold_240_breaks.bed

perl ~/perl_toolchain/sequence_data_scripts/splitFastaWBreakpointBed.pl -f ../../papadum-v5bng-pilon-split2kbgap.fa -o redo_240/papadum_v8bng-pilon_scaffolds.fa -b redo_240/scaffold_240_breaks.bed -s redo_240/scaffold_240_samtoolscoords.txt -m 5000 -n 0.95

samtools faidx redo_240/papadum_v8bng-pilon_scaffolds.fa
perl -lane 'print $F[1];' < redo_240/papadum_v8bng-pilon_scaffolds.fa.fai | perl -e '$c = 0; while(<>){chomp; $c += $_;} print "$c\n";'
2,620,600,033

# Looking good so far!
# I manually editted the decision file to remove Scaffold_240.1 and replace it with the two segments

perl ~/perl_toolchain/assembly_scripts/reorderLachesisDecisTree.pl -t /mnt/nfs/nfs2/dbickhart/transfer/lachesis_ordering_decisions_scaffold_240.txt -f redo_240/papadum_v8bng-pilon_scaffolds.fa -i redo_240/papadum_v8bng-pilon_scaffolds.fa.fai -o redo_240/papadum_v8lach-bng-reorder.norm

perl -lane 'print "$F[0]\t$F[1]\t$F[2]\t$F[5]\t$F[8]";' < papadum-v7lach-bng-reorder.norm.fa.agp > version_7.order
perl -lane 'print "$F[0]\t$F[1]\t$F[2]\t$F[5]\t$F[8]";' < redo_240/papadum_v8lach-bng-reorder.norm.agp > version_8.order
```

## Polishing final assembly

Sergey took us 99.9% of the distance. I'm just going to rename the fasta file headers just so that everyone has an easier time dealing with the file.

Also I'm going to run some (brief) tests. Brief, only because I lack access to anything resembling a server and utility data!

First, some preliminary checks.

> pwd: /home/dbickhart/share/goat_assembly_paper/papadum10/quiver

```bash
# checking the names of the fasta entries
samtools faidx goat.quivered.fasta
# Retrieving the largest lachesis clusters and sorting them
grep 'Contig' goat.quivered.fasta.fai | perl -e '@g; while(<>){chomp; @s = split(/\t/); push(@g, [$s[0], $s[1]]);} @g = sort {$b->[1] <=> $a->[1]} @g; for($x = 0; $x < 31; $x++){$c = $g[$x]; print $c->[0] . "\t" . $c->[1] . "\n";}' 

# Checking the number of "unitigs" and "contigs" in the fasta header names
perl -e '%h; while(<>){chomp; @s = split(/\t/); @b = split(//, $s[0]); $h{$b[0]} += 1;} foreach my $k (keys(%h)){print "$k\t$h{$k}\n";}' < goat.quivered.fasta.fai
C       711
u       29615

# The pipes are killing everything. I've gotta change them.
perl -ne '$_ =~ s/\|/_/g; print $_;' < goat.quivered.fasta > goat.quivered.fasta.tmp
samtools faidx goat.quivered.fasta.tmp
```

OK, so the naming scheme I'll use goes something like this:

* First 31 clusters are named: "clusterX"
* The remaining non-degenerate clusters/scaffolds are size sorted and named: "scaffoldX" (starting at 1)
* The rest are just the unitig numbers that Serge used before.

#### The Perl Script

```perl
#!/usr/bin/perl
# This is a one-shot script designed to replace the "contig" names from the pilon/pbjelly output 
# into more user-friendly names

use strict;
my $usage = "perl $0 <input, indexed fasta> <output fasta>\n";

chomp @ARGV;
unless(scalar(@ARGV) == 2){
	print $usage;
	exit;
}

# Generate contig name and size listing
my @reorg;
open(my $IN, "< $ARGV[0].fai")  || die "Could not open fai file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	push(@reorg, [$segs[0], $segs[1]]);
}
close $IN;
@reorg = sort {$b->[1] <=> $a->[1]} @reorg;

print "Done reorganizing names\n";

open(my $OUT, "> $ARGV[1]");
for(my $x = 0; $x < scalar(@reorg); $x++){
	my $num = $x + 1;
	my $ctgname = $reorg[$x]->[0];
	my $ctgsize = $reorg[$x]->[1];
	my $name;
	if($x < 31){
		$name = "cluster$num";
	}elsif($ctgname =~ /^Contig/){
		$num -= 31;
		$name = "scaffold$num";
	}elsif($ctgname =~ /^utg/){
		my @segs = split(/_/, $ctgname);
		$name = $segs[0];
	}else{
		print STDERR "Error! encountered name not expected! $ctgname\n";
		next;
	}
	
	open(my $IN, "samtools faidx $ARGV[0] $ctgname:1-$ctgsize |");
	my $head = <$IN>;
	print {$OUT} ">$name\n";
	while(my $line = <$IN>){
		print {$OUT} $line;
	}
	close $IN;
}

close $OUT;
```

Applying the script to the data

```bash
perl convert_goat_fasta_names.pl goat.quivered.fasta.tmp papadum.v10.quivered.polished.fasta

```

Now I'm going to remap the RH map probes to the data and check the order/orientation. If there are significant differences, then we're in trouble!

> 3850: /seq1/bickhart/side_projects/rh_map 

```bash
bwa mem papadum.v10.quivered.polished.fasta RHmap_probe_sequences.fasta > papadum_10_rh_mappings.sam

# Now to count unmapped probes
# Old mappings
perl -e '<>; $c = 0; $v = 0; while(<>){chomp; @s = split(/\t/); if($s[5] eq "*"){$v++;} $c++;} print "$c\t$v\n";'
45953   285

# New mappings
cat papadum_10_rh_mappings.sam | perl -e '$c = 0; $v = 0; $t = 0; while(<>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;} if($s[2] eq "*"){$v++;}elsif($s[2] =~ /^cluster/){$t++;} $c++;} print "$c\t$v\t$t\n";'
45632   2905    42305

#(total lines, unmapped, mapped to clusters)
# Hmm, that's less than expected. Let's see how the probe names match up

perl -e 'chomp(@ARGV); %h; open(DAT, "< $ARGV[0]"); while(<DAT>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;}else{$h{$s[0]} = [$s[2], $s[3]];}} close DAT; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); $d = $h{$s[0]}; $n = $s[5]; if($s[5] =~ /^utg/){@i = split(/_/, $s[5]); $n = $i[0];} print "$s[0]\t" . $d->[0] . "\t" . $d->[1] . "\t$n\t$s[6]\n";} close DAT;' papadum_10_rh_mappings.sam RHmap_to_PacBio-v3_contigs.txt > ordered_rh_mappings.tab

# "N/A" probes were not included in the fasta file, so lets get rid of those
grep -v '#N/A' ordered_rh_mappings.tab > ordered_rh_mappings.nona.tab

# Interestingly, some of these mappings just show probes that no longer adequately align to existing clusters
grep 'utg1957' ordered_rh_mappings.nona.tab
	OAR1_238375598.1        cluster1        102619982       utg1957 5494269
	OAR1_238449743.1        cluster1        102678183       utg1957 5436139
	BTB-00045786    *       0       utg1957 5403122
	OAR1_238532462.1        cluster1        102767990       utg1957 5346907
	BTB-01648341    cluster1        104103766       utg1957 4014925
	BTB-00044939    cluster1        104069097       utg1957 4049457
	BTB-00045108    cluster1        103997337       utg1957 4120908

# Let's test how often this occurs
perl -e '%h; while(<>){chomp; @s = split(/\t/); $h{$s[3]}->{$s[1]} += 1;} my($b, $n, $l, $m, $r); foreach $k (keys(%h)){ if($k eq "*"){ foreach $j (keys(%{$h{$k}})){ if($j ne "*"){$r++;}} next;} $ha = exists($h{$k}->{"*"}); $num = scalar(keys(%{$h{$k}})); if($num > 1 && !$ha){$m++;}elsif($num == 2 && $ha){$l++;}elsif($num == 1 && !$ha){$b++;}elsif($num == 1 && $ha){$n++;}} print "NoProb \| NoMaps \| PartialMap \| MultMaps \| Resolve\n:---\|:---\|:---\|:---\|:---\n"; print "$b\|$n\|$l\|$m\|$r\n";' < ordered_rh_mappings.nona.tab
```
#### Results from mappings listing 

NoProb | NoMaps | PartialMap | MultMaps | Resolve
:---|:---|:---|:---|:---
915|10|459|54|88

The "MultMaps" are the biggest concern here. 

```bash
# I'm comparing this to the previous checkpoint assembly: papadum v8
gunzip papadum_v8bng-pilon_scaffolds.fa.gz
bwa index papadum_v8bng-pilon_scaffolds.fa

bwa mem papadum_v8bng-pilon_scaffolds.fa RHmap_probe_sequences.fasta > papadum_8_rh_mappings.sam
cat papadum_8_rh_mappings.sam | perl -e '$c = 0; $v = 0; $t = 0; while(<>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;} if($s[2] eq "*"){$v++;}elsif($s[2] =~ /^cluster/){$t++;} $c++;} print "$c\t$v\t$t\n";'
45632   2503    0

# Pretty similar to the previous mappings
# Going to try this before the Pilon correction
bwa index goat_split_36ctg_assembly.fa
bwa mem goat_split_36ctg_assembly.fa RHmap_probe_sequences.fasta > papadum_4_rh_mappings.sam
cat papadum_4_rh_mappings.sam | perl -e '$c = 0; $v = 0; $t = 0; while(<>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;} if($s[2] eq "*"){$v++;}elsif($s[2] =~ /^cluster/){$t++;} $c++;} print "$c\t$v\t$t\n";'
45632   2634    0

# Pretty damn similar again!
# Let's try to associate these maps with the rest to see if there's a trend

```

The important thing is cluster order. Let's check that quickly.

```bash
perl -e '$last = "start"; $c = 0; while(<>){chomp; @s = split(/\t/); if($s[1] ne $last && $s[1] ne "*"){print "$last\t$c\n"; $last = $s[1]; $c = 0;} $c++;}' < ordered_rh_mappings.nona.tab | wc -l
903

# That looks bad, but there are lots of "singletons" where there was one probe mapping to an alternative contig/scaffold
perl -e '$last = "start"; $c = 0; while(<>){chomp; @s = split(/\t/); if($s[1] ne $last && $s[1] ne "*" && $c > 2){print "$last\t$c\n"; $last = $s[1]; $c = 0;} $c++;}' < ordered_rh_mappings.nona.tab | wc -l
824

# Still pretty high. Let's try to reformat the tab file I generated before and deal with the chromosomes
perl -e 'chomp(@ARGV); %h; open(DAT, "< $ARGV[0]"); while(<DAT>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;}else{$h{$s[0]} = [$s[2], $s[3]];}} close DAT; open(IN, "< $ARGV[1]"); <IN>; while(<IN>){chomp; @s = split(/\t/); $d = $h{$s[0]}; $n = $s[5]; if($s[5] =~ /^utg/){@i = split(/_/, $s[5]); $n = $i[0];} print "$s[0]\t$s[1]\t" . $d->[0] . "\t" . $d->[1] . "\t$n\t$s[6]\n";} close DAT;' papadum_10_rh_mappings.sam RHmap_to_PacBio-v3_contigs.txt > ordered_rh_mappings.tab
grep -v '#N/A' ordered_rh_mappings.tab > ordered_rh_mappings.nona.tab

# Prints the chromosome ordering with contig/scaffold contribution:
perl -e '$last = "start"; $lastc = -1; $c = 0; while(<>){chomp; @s = split(/\t/); if($s[2] ne $last && $s[2] ne "*" && $c > 2){print "$last\t$lastc\t$c\n"; $last = $s[2]; $lastc = $s[1]; $c = 0;} $c++;}' < ordered_rh_mappings.nona.tab | perl -e '%h; while(<>){chomp; @s = split(/\t/); $h{$s[1]}->{$s[0]} += $s[2];} foreach my $k (sort {$a <=> $b} keys(%h)){print "[$k]"; @js = sort{$h{$k}->{$b} <=> $h{$k}->{$a}} keys(%{$h{$k}}); foreach $j (@js){print "\t$j\t" . $h{$k}->{$j};} print "\n";}'

# Prints only the highest contributing contigs/scaffolds to each RH chromosome
perl -e '$last = "start"; $lastc = -1; $c = 0; while(<>){chomp; @s = split(/\t/); if($s[2] eq "*"){next;} if($s[2] ne $last && $c > 2){print "$last\t$lastc\t$c\n"; $last = $s[2]; $lastc = $s[1]; $c = 0;} $c++;}' < ordered_rh_mappings.nona.tab | perl -e '%h; while(<>){chomp; @s = split(/\t/); $h{$s[1]}->{$s[0]} += $s[2];} foreach my $k (sort {$a <=> $b} keys(%h)){print "[$k]"; @js = sort{$h{$k}->{$b} <=> $h{$k}->{$a}} keys(%{$h{$k}}); foreach $j (@js){if($h{$k}->{$j} > 10){print "\t$j\t" . $h{$k}->{$j};}} print "\n";}'
```

I've identified three mistakes that can be corrected.

chr23 (Scaffold_1637.1 ... not sure what it's doing there!):
**OAR20_34988140.1        23      cluster28       31879933        utg56619        242597**
ARS-BFGL-NGS-39980      23      cluster19       37132   utg1471 37049
OAR20_35185147.1        23      cluster19       44754   utg1471 44601
s65224.1        23      cluster19       86150   utg1471 85838
s67035.1        23      cluster19       116059  utg1471 115673
s62346.1        23      cluster19       216218  utg1471 215416
s36143.1        23      cluster19       228189  utg1471 227304
OAR20_35394096.1        23      cluster19       240856  utg1471 239999
BTA-56332-no-rs 23      *       0       utg1471 276984
s00007.1        23      cluster19       304932  utg1471 303947
s13273.1        23      cluster28       31717145        utg56619        80208
OAR20_35687386.1        23      cluster19       445509  utg1471 443953
BTA-56342-no-rs 23      cluster19       476001  utg1471 474420
OAR20_35863023.1        23      cluster19       521144  utg1471 519397
OAR20_35961027.1        23      cluster19       621645  utg1471 619517
s30412.1        23      cluster19       634367  utg1471 632202
OAR20_36071622.1        23      cluster19       725792  utg1471 723386
ARS-BFGL-BAC-3611       23      cluster19       769882  utg1471 767342
s69641.1        23      cluster19       815637  utg1471 812995
s33429.1        23      cluster19       835835  utg1471 833204
OAR20_36346864.1        23      cluster19       1046875 utg1044 155937
BTA-92720-no-rs 23      cluster19       1065604 utg1044 174586
OAR20_36383975.1        23      cluster19       1081680 utg1044 190642
OAR20_36396618.1        23      cluster19       1096172 utg1044 205104
ARS-BFGL-NGS-107881     23      cluster19       1103602 utg1044 212516
OAR20_36457784.1        23      cluster19       1154832 utg1044 263723
ARS-BFGL-NGS-82538      23      cluster19       1161456 utg1044 270342
OAR20_36481077.1        23      cluster19       1181627 utg1044 290452
OAR20_36542486.1        23      cluster19       1235181 utg1044 343812
OAR20_36646239.1        23      cluster19       1328781 utg1044 437211
ARS-BFGL-NGS-112063     23      cluster19       1378913 utg23190        1735562
OAR20_36737629.1        23      cluster19       1411359 utg1044 519670
OAR20_36774341_X.1      23      cluster19       1450046 utg1044 558246
ARS-BFGL-NGS-110889     23      *       0       utg1044 599303
**ARS-BFGL-NGS-104089     23      cluster28       33954938        utg523  829685
OAR20_37626453.1        23      cluster28       33979337        utg523  850990**

chr11: (Was this a quiver/pilon overcorrection? BNG Scaffold_22.1)
OAR3_50509171.1 11      cluster15       58939039        utg32509        5985124
OAR3_50474257.1 11      *       0       utg32511        14272
**utg32511 +
utg43837 +
utg1519 +
utg32563 +
utg45429 +
utg386 +
utg509 - 6,800,668 to 2,642,887**
OAR3_23520928.1 11      cluster15       58991810        utg509  2642887
OAR3_23366586.1 11      cluster15       59135635        utg509  2499167

chr21:
BTB-00811262    21      cluster20       41659038        utg28093        188358
BTA-51901-no-rs 21      scaffold1       68374   utg717  562953
**scaffold1 +**
s14592.1        21      scaffold1       3320181 utg23187        1339100
BTB-00800263    21      cluster20       41438419        utg23187        1286224

Testing the chr11 problem junctions

```bash
samtools faidx papadum.v10.quivered.polished.fasta cluster15:58939039-58979039 > chr11_cluster15_probsite.fa
bwa mem papadum_v8bng-pilon_scaffolds.fa chr11_cluster15_probsite.fa > chr11_cluster15_probsite.sam

perl -lane 'print "$F[0]\t$F[1]\t$F[2]\t$F[3]\t$F[4]";' < chr11_cluster15_probsite.sam
cluster15:58939039-58979039     16      Scaffold_22.2   2       60
cluster15:58939039-58979039     2064    Scaffold_22.1   2811767 60
cluster15:58939039-58979039     2064    Scaffold_22.1   2826288 60
cluster15:58939039-58979039     2064    Scaffold_22.1   28261785        60

#Scaffold_22.1 is on that side! Let's check the other
samtools faidx papadum.v10.quivered.polished.fasta cluster15:58991810-59135635 > chr11_cluster15_probsite2.fa
bwa mem papadum_v8bng-pilon_scaffolds.fa chr11_cluster15_probsite2.fa > chr11_cluster15_probsite2.sam
cluster15:58991810-59135635     16      Scaffold_22.1   2655186 60

# A little more upstream to see if we get away from Scaffold_22.1
samtools faidx papadum.v10.quivered.polished.fasta cluster15:58930039-58939039 > chr11_cluster15_probsite3.fa
bwa mem papadum_v8bng-pilon_scaffolds.fa chr11_cluster15_probsite3.fa > chr11_cluster15_probsite3.sam

perl -lane 'print "$F[0]\t$F[1]\t$F[2]\t$F[3]\t$F[4]";' < chr11_cluster15_probsite3.sam | tail -n 20
cluster15:58930039-58939039     16      Scaffold_22.2   23426   60

```

I need to use the proper cluster names to ask Sergey to fix these errors. Let's reverse engineer which clusters these were.

My name | Length | Serge's cluster
:--- | :--- | :---
cluster1   |     157459805   |    Contig9
cluster2    |    136571093    |   Contig16
cluster3    |    120757200    |   Contig22
cluster4    |    120098609    |   Contig5
cluster5    |    119051289    |   Contig3
cluster6     |   117708699    |   Contig12
cluster7     |   112708337     |  Contig15
cluster8     |   108463126     |  Contig14
cluster9     |   101128593     |  Contig19
cluster10    |   94703394      |  Contig24
cluster11    |   91592673     |   Contig13
cluster12    |   87320585      |  Contig6
cluster13    |   83050203      |  Contig25
cluster14    |   81929103      |  Contig18
cluster15    |   80796137      |  Contig11
cluster16    |   79400982      |  Contig17
cluster17    |   71795819     |   Contig27
cluster18    |   71178665      |  Contig20
cluster19    |   67351606      |  Contig2
cluster20    |   66116439      |  Contig7
cluster21    |   66077574      |  Contig0
cluster22    |   65196113     |   Contig8
cluster23    |   62349873      |  Contig28
cluster24    |   60336389      |  Contig21
cluster25    |   51441506      |  Contig23
cluster26    |   51337999      |  Contig10
cluster27    |   49920226      |  Contig1
cluster28    |   48900931      |  Contig26
cluster29    |   44739069      |  Contig29
cluster30     |  44685470      |  Contig31
cluster31      | 42877099      |  Contig4


Sergey confirmed that the AGP file he used accidentally truncated Scaffold_22.1. I'm going to check that one, then add in the missing Scaffold and check the final error and modify his AGP file as needed.

> 3850: /seq1/bickhart/side_projects

```bash
# Checking chr11's truncation
grep Scaffold_22.1 papadum_v10lach-bng-reorder.norm.agp
8       58902394        87165315        5       F       Scaffold_22.1   1       28262922        -

# That matches my data on the size of the scaffold fragment
# Let's check Scaffold_1637.1 now
grep Scaffold_1637.1 papadum_v10lach-bng-reorder.norm.agp
20      1       7847968 0       F       Scaffold_1637.1 1       7847968 +

# That's in the right place (at the start of the chromosome)
# Now let's see if we can incorporate "Scaffold1" from above. 
# First we need to figure out just what "Scaffold1" really is!
cd rh_map/
samtools faidx papadum.v10.quivered.polished.fasta scaffold1 > scaffold1.section.fa
bwa mem papadum_v8bng-pilon_scaffolds.fa scaffold1.section.fa > scaffold1.section.sam
samtools view -bS scaffold1.section.sam > scaffold1.section.bam
~/bedtools2/bin/bamToBed -i scaffold1.section.bam
	Scaffold_61.2   632203  3341330 scaffold1       60      +
	Scaffold_61.2   1       632143  scaffold1       60      +

# OK, we've got the name. Let's see if it is in the agp file
cd ..
grep Scaffold_61.2 papadum_v10lach-bng-reorder.norm.agp
Scaffold_61.2   1       3341330 1       F       Scaffold_61.2   1       3341330 +

# Unplaced, as we expected. Let's see if its constitutive components are placed
grep Scaffold_61 papadum_v10lach-bng-reorder.norm.agp
18      22869765        24605794        1       F       Scaffold_61.1   1       1736030 +
Scaffold_61.2   1       3341330 1       F       Scaffold_61.2   1       3341330 +

# I'm going to have to align the constitutive component of Scaffold_61 to the other assembly to see if it is the same cluster
cd rh_map/
samtools faidx papadum_v8bng-pilon_scaffolds.fa Scaffold_61.1 > Scaffold_61.1.section.fa
bwa mem papadum.v10.quivered.polished.fasta Scaffold_61.1.section.fa > Scaffold_61.1.section.sam
samtools view -bS Scaffold_61.1.section.sam > Scaffold_61.1.section.bam
~/bedtools2/bin/bamToBed -i Scaffold_61.1.section.bam
	cluster20       41469935        43206254        Scaffold_61.1   60      -

# This matches the same cluster and coordinates, but I'm going to have to be careful of how to orient it!
# Let's pull more flanking sequence and align that to be sure!
samtools faidx papadum_v8bng-pilon_scaffolds.fa Scaffold_1.7:1-1000000 > Scaffold_1.7.section.fa
bwa mem papadum.v10.quivered.polished.fasta Scaffold_1.7.section.fa > Scaffold_1.7.section.sam
samtools view -bS Scaffold_1.7.section.sam > Scaffold_1.7.section.bam
~/bedtools2/bin/bamToBed -i Scaffold_1.7.section.bam
cluster20       18628475        19276723        Scaffold_1.7:1-1000000  60      +
cluster20       18273514        18625837        Scaffold_1.7:1-1000000  60      +

samtools faidx papadum_v8bng-pilon_scaffolds.fa Scaffold_1.7:22186583-23186583 > Scaffold_1.7.end.fa
bwa mem papadum.v10.quivered.polished.fasta Scaffold_1.7.end.fa > Scaffold_1.7.end.sam
samtools view -bS Scaffold_1.7.end.sam > Scaffold_1.7.end.bam
~/bedtools2/bin/bamToBed -i Scaffold_1.7.end.bam
cluster20       40469583        41469935        Scaffold_1.7:22186583-23186583  60      +

# OK, so the scaffold goes after Scaffold_1.7 and before Scaffold_61.1
grep Scaffold_61.2 papadum_v8bng-pilon_scaffolds.fa.fai
Scaffold_61.2   3341330 734118399       60      61

perl -e '$start = 0; $last = 0; while(<>){chomp; @s = split(/\t/); if($s[0] == 18 && $s[5] eq "Scaffold_61.1"){$start = 1; $last = $s[2];}elsif($s[0] != 18){$start = 0;}elsif($start){if($s[7] ne "map"){$s[1] = $last + 1; $s[2] = $last + $s[7];}else{$s[1] = $last + 1; $s[2] = $last + 5;} $last = $s[2];} print join("\t", @s); print "\n";}' < papadum_v11lach-bng-reorder.norm.agp > papadum_v11lach-bng-reorder.norm.agp.temp
mv papadum_v11lach-bng-reorder.norm.agp.temp papadum_v11lach-bng-reorder.norm.fixed.agp
```

## Finalizing the assembly

First, performing the RH map tests to see if everything matches up.

> pwd: /home/dbickhart/share/goat_assembly_paper/quiver

```bash
bwa index goat.quivered.fasta
bwa mem goat.quivered.fasta ../rh_map/RHmap_probe_sequences.fasta > rh_map_probes_quiver.sam

# Mapping check
cat rh_map_probes_quiver.sam | perl -e '$c = 0; $v = 0; $t = 0; while(<>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;} if($s[2] eq "*"){$v++;}elsif($s[2] =~ /^cluster/){$t++;} $c++;} print "$c\t$v\t$t\n";'
45632   2400    0

# 100 more maps! Good deal!

# RH map probe check
# Converting to tab format
perl -e 'chomp(@ARGV); %h; open(DAT, "< $ARGV[0]"); while(<DAT>){chomp; @s = split(/\t/); if($s[0] =~ /^@/){next;}else{$h{$s[0]} = [$s[2], $s[3]];}} close DAT; open(IN, "< $ARGV[1]"); <IN>; while(<IN>){chomp; @s = split(/\t/); if($s[-1] eq "#N\A"){next;} $d = $h{$s[0]}; $n = $s[5]; if($s[5] =~ /^utg/){@i = split(/_/, $s[5]); $n = $i[0];} print "$s[0]\t$s[1]\t" . $d->[0] . "\t" . $d->[1] . "\t$n\t$s[6]\n";} close DAT;' rh_map_probes_quiver.sam ../rh_map/RHmap_to_PacBio-v3_contigs.txt > rh_map_probes_quiver.nona.tab

# Cluster 23 has a slight merger with cluster 18, but this is a very small addition that is just over a megabase.
# We'll ignore it for now.
```

OK, now I need to concatenate the list of contigs in the Pilon folder, add the other contigs that were not incorporated and then relabel them into a more machine-readable format.

> pwd: /home/dbickhart/share/goat_assembly_paper/pilon

```bash
for i in *.fasta; do echo $i; perl -e '$h = <>; chomp $h; $h =~ s/^>//g; @b = split(/\|/, $h); print ">$b[0]\n"; while(<>){print $_;}' < $i >> pilon_merged_output.fasta; done

# This should be my list of finished, error corrected contigs/scaffolds. 
# Let's write a script that takes the quivered fastas and adds them as separate entries as well.

# First, to get rid of the damn pipes in the quiver fasta
perl -ne 'if($_ =~ /^>/){$_ =~ s/\|/_/g;} print $_;' < ../quiver/goat.quivered.fasta > ../quiver/goat.quivered.nopipe.fasta
samtools faidx ../quiver/goat.quivered.nopipe.fasta

# now to run the script
perl ../final_contig_naming_script.pl pilon_merged_output.fasta ../quiver/goat.quivered.nopipe.fasta goat_v12_corrected.fa
```

Here is the script (it's an adaptation of my prior perl script):

```perl
#!/usr/bin/perl
# This is a one-shot script designed to incorporate all of the finished, corrected data into a 
# coherent, usable fasta file

use strict;
my $usage = "perl $0 <input pilon, indexed fasta> <input quiver, indexed fasta> <output fasta>\n";

chomp @ARGV;
unless(scalar(@ARGV) == 3){
	print $usage;
	exit;
}

# Generate contig name and size listing
my @reorg;
my %found;
my @unplaced;
open(my $IN, "< $ARGV[0].fai")  || die "Could not open fai file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	$found{$segs[0]} = 1;
	push(@reorg, [$segs[0], $segs[1]]);
}
close $IN;
@reorg = sort {$b->[1] <=> $a->[1]} @reorg;

open(my $IN, "< $ARGV[1].fai") || die "Could not open fai file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	my($short) = $segs[0] =~ m/(.+)_quiver/;
	if(!exists($found{$short})){
		push(@unplaced, [$segs[0], $segs[1]]);
	}
}
close $IN;
@unplaced = sort {$b->[1] <=> $a->[1]} @unplaced;
	

print "Done reorganizing names\n";

open(my $OUT, "> $ARGV[2]");
for(my $x = 0; $x < scalar(@reorg); $x++){
	my $num = $x + 1;
	my $ctgname = $reorg[$x]->[0];
	my $ctgsize = $reorg[$x]->[1];
	my $name;
	if($x < 31){
		$name = "cluster_$num";
	}elsif($ctgname =~ /^Contig/){
		$num -= 31;
		$name = "scaffold_$num";
	}elsif($ctgname =~ /^utg/){
		my @segs = split(/_/, $ctgname);
		$name = $segs[0];
	}else{
		print STDERR "Error! encountered name not expected! $ctgname\n";
		next;
	}
	
	open(my $IN, "samtools faidx $ARGV[0] $ctgname:1-$ctgsize |");
	my $head = <$IN>;
	print {$OUT} ">$name\n";
	while(my $line = <$IN>){
		print {$OUT} $line;
	}
	close $IN;
}

for(my $x = 0; $x < scalar(@unplaced); $x++){
	my $num = $x + 1;
	my $ctgname = $unplaced[$x]->[0];
	my $ctgsize = $unplaced[$x]->[1];
	
	my $name = "unplaced_$num";
	open(my $IN, "samtools faidx $ARGV[1] $ctgname:1-$ctgsize |");
	my $head = <$IN>;
	print {$OUT} ">$name\n";
	while(my $line = <$IN>){
		print {$OUT} $line;
	}
	close $IN;
}

close $OUT;
```

OK, so I made a mistake with the script in the original version (changed version is above) due to the addition of the "quiver" handle on the end of each "Contig" name. 

```bash
perl ../final_contig_naming_script.pl pilon_merged_output.fasta ../quiver/goat.quivered.nopipe.fasta goat_v13_corrected.fa

samtools faidx goat_v13_corrected.fa
wc -l *.fai
  30681 goat_v12_corrected.fa.fai
  29998 goat_v13_corrected.fa.fai
    683 pilon_merged_output.fasta.fai
  61362 total

# Looks good! I think that the number of entries matches what I'd expect

mv goat_v13_corrected.fa goat_v13_corrected.full.fa
samtools faidx goat_v13_corrected.full.fa

# Now to pull only the 683 pilon-corrected contigs
perl -lane 'if(!($F[0] =~ /unplaced_.+/)){print STDERR "$F[0]"; open(IN, "samtools faidx goat_v13_corrected.full.fa $F[0]:1-$F[1] |"); $h = <IN>; print ">$F[0]"; while(<IN>){chomp; print "$_";} close IN;}' < goat_v13_corrected.full.fa.fai > goat_v13_corrected.ctg.fa

# Finally, to create a BWA index and map the RH probes and SNP chip probes
bwa index goat_v13_corrected.full.fa
bwa mem goat_v13_corrected.full.fa ../snp_probes/my_temp.fq > goat_v13_corrected.full.snps.sam
bwa mem goat_v13_corrected.full.fa ../rh_map/RHmap_probe_sequences.fasta > goat_v13_corrected.full.rhmap.sam
```

Now to make the sam files into tab delimited entries. I'm going to accomplish this using a dedicated script this time.

```perl
#!/usr/bin/perl
# this is a one-shot script designed to associate RH map coordinates with aligned entries from a sam file
# Includes all prior mapping information from teh RH map file

use strict;
use Getopt::Std;

my %opts;
my $usage = "perl $0 -r <rh map> -s <sam file> -o <output tab file>\n";
getopt('rso', \%opts);

unless(defined($opts{'r'}) && defined($opts{'s'}) && defined($opts{'o'})){
	print $usage;
	exit;
}

# Get the sam file coordinates mapped into memory
my %coords; #= {snp probe} -> [chr, pos, orient]
open(my $IN, "< $opts{s}") || die "Could not open sam file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[0] =~ /^@/){next;}
	my $orient = "+";
	if($segs[1] & 0x10){
		$orient = "-";
	}
	
	push(@{$coords{$segs[0]}}, ($segs[2], $segs[3], $orient));
}
close $IN;

# Open the RH map file and associate sam entries with it
open(my $IN, "< $opts{r}") || die "Could not open RH map file!\n$usage";
open(my $OUT, "> $opts{o}");
my $h = <$IN>;
print {$OUT} "Probe\tChr(RH)\tPos(RH)\tChr(goat13)\tPos(goat13)\tOrient(goat13)\tChr(goat3)\tPos(goat3)\tChr(CHI1)\tPos(CHI1)\n";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	my ($g13chr, $g13pos, $g13orient);
	if(!(exists($coords{$segs[0]}))){
		print STDERR "Error with missing snp probe: $segs[0]!\n";
		($g13chr, $g13pos, $g13orient) = ("MISS", "MISS", "MISS");
	}else{
		($g13chr, $g13pos, $g13orient) = @{$coords{$segs[0]}};
		#$g13chr = $coords{$segs[0]}->[0]
		if($g13chr eq "*"){
			$g13chr = "UNMAP";
			$g13pos = "UNMAP";
			$g13orient = "UNMAP";
		}
	}
	
	# Deal with goatv3 contig naming convention
	my ($g3ctg) = $segs[5] =~ /(utg\d+)_len.*/;
	my $g3pos = $segs[6];
	if($g3ctg eq ""){
		$g3ctg = "UNMAP";
		$g3pos = "UNMAP";
	}
	if($segs[5] eq "\#N/A"){
		$g3ctg = "MISS";
		$g3pos = "MISS";
	}
	
	my ($bgichr, $bgipos, $rhchr, $rhpos) = ($segs[3], $segs[4], $segs[1], $segs[2]);
	
	print {$OUT} "$segs[0]\t$rhchr\t$rhpos\t$g13chr\t$g13pos\t$g13orient\t$g3ctg\t$g3pos\t$bgichr\t$bgipos\n";
	
}

close $IN;
close $OUT;

exit;
```

Associating SNP probes should be easier. Let's do it with a one-liner.

Nope! On second thought, the information on the probes needs more details added as separate columns. I'll repurpose my script above into another one-liner.

```perl
#!/usr/bin/perl
# this is a one-shot script designed to associate snp probe coordinates with aligned entries from a sam file
# Includes all prior mapping information from the snp probe file

use strict;
use Getopt::Std;

my %opts;
my $usage = "perl $0 -r <snp file> -s <sam file> -o <output tab file>\n";
getopt('rso', \%opts);

unless(defined($opts{'r'}) && defined($opts{'s'}) && defined($opts{'o'})){
	print $usage;
	exit;
}

our %refseq = (
	'NC_022293.1' => 'chr1',
	'NC_022294.1' => 'chr2',
	'NC_022295.1' => 'chr3',
	'NC_022296.1' => 'chr4',
	'NC_022297.1' => 'chr5',
	'NC_022298.1' => 'chr6',
	'NC_022299.1' => 'chr7',
	'NC_022300.1' => 'chr8',
	'NC_022301.1' => 'chr9',
	'NC_022302.1' => 'chr10',
	'NC_022303.1' => 'chr11',
	'NC_022304.1' => 'chr12',
	'NC_022305.1' => 'chr13',
	'NC_022306.1' => 'chr14',
	'NC_022307.1' => 'chr15',
	'NC_022308.1' => 'chr16',
	'NC_022309.1' => 'chr17',
	'NC_022310.1' => 'chr18',
	'NC_022311.1' => 'chr19',
	'NC_022312.1' => 'chr20',
	'NC_022313.1' => 'chr21',
	'NC_022314.1' => 'chr22',
	'NC_022315.1' => 'chr23',
	'NC_022316.1' => 'chr24',
	'NC_022317.1' => 'chr25',
	'NC_022318.1' => 'chr26',
	'NC_022319.1' => 'chr27',
	'NC_022320.1' => 'chr28',
	'NC_022321.1' => 'chr29',
	'NC_022322.1' => 'chrX',
	'NC_005044.2' => 'chrMT',
);

# Get the sam file coordinates mapped into memory
my %coords; #= {snp probe} -> [chr, pos, orient]
open(my $IN, "< $opts{s}") || die "Could not open sam file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[0] =~ /^@/){next;}
	my $orient = "+";
	if($segs[1] & 0x10){
		$orient = "-";
	}
	
	push(@{$coords{$segs[0]}}, ($segs[2], $segs[3]));
}
close $IN;

# Open the SNP map file and associate sam entries with it
open(my $IN, "< $opts{r}") || die "Could not open SNP map file!\n$usage";
open(my $OUT, "> $opts{o}");
my $h = <$IN>;
print {$OUT} "Probe\tSynthesized\tSuccessful\tMAF\tChr(goat13)\tPos(goat13)\tChr(Chi1)\tPos(Chi1)\tChr(UMD3)\tPos(UMD3)\tSequence\n";
while(my $line = <$IN>){
	chomp $line;
	$line =~ s/\r//g;
	my @segs = split(/\t/, $line);
	my ($g13chr, $g13pos);
	if(!(exists($coords{$segs[0]}))){
		print STDERR "Error with missing snp probe: $segs[0]!\n";
		($g13chr, $g13pos) = ("MISS", "MISS");
	}else{
		($g13chr, $g13pos) = @{$coords{$segs[0]}};
		#$g13chr = $coords{$segs[0]}->[0]
		if($g13chr eq "*"){
			$g13chr = "UNMAP";
			$g13pos = "UNMAP";
		}
	}
	my $bginame = (exists($refseq{$segs[10]}))? $refseq{$segs[10]} : $segs[10];
	
	my ($synth, $succ, $bgichr, $bgipos, $umdchr, $umdpos, $seq) = ($segs[2], $segs[3], $bginame, $segs[11], $segs[14], $segs[15], $segs[1]);
	
	print {$OUT} "$segs[0]\t$synth\t$succ\t$segs[4]\t$g13chr\t$g13pos\t$bgichr\t$bgipos\t$umdchr\t$umdpos\t$seq\n";
	
}

close $IN;
close $OUT;

exit;
```

Now I just need to reorder the SNP probes so that they are able to be used in association studies.

> pwd: /home/dbickhart/share/goat_assembly_paper/pilon

```bash
perl -e '$h = <>; print $h; %data; while(<>){chomp; @segs = split(/\t/); my @temp = @segs; push(@{$data{$segs[4]}}, \@temp);} @k = keys(%data); @sort = @k[map { unpack "N", substr($_,-4) } sort map{ my $key = $k[$_]; $key =~ s[(\d+)][ pack "N", $1 ]ge; $key . pack "N", $_ } 0..$#k]; foreach my $s (@sort){@d = @{$data{$s}}; @j = sort{$a->[5] <=> $b->[5]} @d; foreach my $t (@j){print join("\t", @{$t}); print "\n";}}' < goat_v13_corrected.full.snps.tab > goat_v13_corrected.full.snps.sorted.tab

# Checking the number of unplaced and scaffold chromosomes that have SNP probe mappings
perl ../../programming/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f goat_v13_corrected.full.snps.sorted.tab -c 4 | perl -lane '@b = split(/_/, $F[0]); print $b[0];' | perl ../../programming/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f stdin -c 0 -m
```

|Entry       | Count|
|:-----------|-----:|
|Chr(goat13) |     1|
|Entry       |     1|
|MISS        |     1|
|UNMAP       |     1|
|cluster     |    31|
|scaffold    |   215|
|unplaced    |   288|

Now going to count the number of entries per cluster/scaffold.

```bash
perl ../../programming/perl_toolchain/bed_cnv_fig_table_pipeline/tabFileColumnCounter.pl -f goat_v13_corrected.full.snps.sorted.tab -c 4 | perl -e '%h; while(<>){chomp; @s = split(/\t/); @b = split(/_/, $s[0]); $h{$b[0]} += $s[1];} foreach my $k (sort {$a cmp $b} keys(%h)){print "$k\t$h{$k}\n";}'
```
|Entry | Count| Perc |
|:--- | ---:| ---: |
MISS  |  1 | < 0.01%
UNMAP |  64 | 0.1%
cluster |59,082 | 98.47%
scaffold|        556 | 0.9%
unplaced |       297 | 0.5%

On the CHI_1.0 assembly, 1600 probes aligned to unplaced scaffolds (2.6%). 


## Updating the RH map probe lists

Adding the other chromosome scaffolds to the lists was requested by Shawn and a few other groups. I think it will be a good comparison to send to everyone.

First, let's index the two fastas so that we can map the probes.

> pwd: /home/dbickhart/share/goat_assembly_paper/pilon

```bash
bwa index Lachesis_assembly.fasta & bwa index papadum-v5bng-pilon-split2kbgap.fa

bwa mem Lachesis_assembly.fasta ../rh_map/RHmap_probe_sequences.fasta > lachesis_assembly_rh.sam
bwa mem papadum-v5bng-pilon-split2kbgap.fa ../rh_map/RHmap_probe_sequences.fasta > papadum_v5bng_rh.sam

perl change_and_associate_rh_map_extra.pl -r ../rh_map/RHmap_to_PacBio-v3_contigs.txt -s goat_v13_corrected.full.rhmap.sam -b papadum_v5bng_rh.sam -l lachesis_assembly_rh.sam -o goat_v13_corrected.full.rhmap.extend.tab
```

And here is the updated script code:

```perl
#!/usr/bin/perl
# this is a one-shot script designed to associate RH map coordinates with aligned entries from a sam file
# Includes all prior mapping information from teh RH map file
# Updated to add other assembly versions

use strict;
use Getopt::Std;

my %opts;
my $usage = "perl $0 -r <rh map> -s <sam file> -o <output tab file> -b bionano -l lachesis\n";
getopt('rsobl', \%opts);

unless(defined($opts{'r'}) && defined($opts{'s'}) && defined($opts{'o'})){
	print $usage;
	exit;
}

# Get the sam file coordinates mapped into memory
my %coords; #= {snp probe} -> [chr, pos, orient]
open(my $IN, "< $opts{s}") || die "Could not open sam file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[0] =~ /^@/){next;}
	my $orient = "+";
	if($segs[1] & 0x10){
		$orient = "-";
	}
	
	push(@{$coords{$segs[0]}}, ($segs[2], $segs[3], $orient));
}
close $IN;

# Other two sams
my %lach;
open(my $IN, "< $opts{l}") || die "Could not open lachesis file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[0] =~ /^@/){next;}
	my $orient = "+";
	if($segs[1] & 0x10){
		$orient = "-";
	}
	
	push(@{$lach{$segs[0]}}, ($segs[2], $segs[3]));
}
close $IN;

my %bng;
open(my $IN, "< $opts{b}") || die "Could not open bng file!\n$usage";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	if($segs[0] =~ /^@/){next;}
	my $orient = "+";
	if($segs[1] & 0x10){
		$orient = "-";
	}
	
	push(@{$bng{$segs[0]}}, ($segs[2], $segs[3]));
}
close $IN;

# Open the RH map file and associate sam entries with it
open(my $IN, "< $opts{r}") || die "Could not open RH map file!\n$usage";
open(my $OUT, "> $opts{o}");
my $h = <$IN>;
print {$OUT} "Probe\tChr(RH)\tPos(RH)\tChr(goat13)\tPos(goat13)\tOrient(goat13)\tChr(goat3)\tPos(goat3)\tChr(bng)\tPos(bng)\tChr(lach)\tPos(lach)\tChr(CHI1)\tPos(CHI1)\n";
while(my $line = <$IN>){
	chomp $line;
	my @segs = split(/\t/, $line);
	my ($g13chr, $g13pos, $g13orient, $bngchr, $bngpos, $lachchr, $lachpos);
	if(!(exists($coords{$segs[0]}))){
		print STDERR "Error with missing snp probe: $segs[0]!\n";
		($g13chr, $g13pos, $g13orient) = ("MISS", "MISS", "MISS");
		($bngchr, $bngpos, $lachchr, $lachpos) = ("MISS", "MISS", "MISS", "MISS");
	}else{
		($g13chr, $g13pos, $g13orient) = @{$coords{$segs[0]}};
		($bngchr, $bngpos) = @{$bng{$segs[0]}};
		($lachchr, $lachpos) = @{$lach{$segs[0]}};
		#$g13chr = $coords{$segs[0]}->[0]
		if($g13chr eq "*"){
			$g13chr = "UNMAP";
			$g13pos = "UNMAP";
			$g13orient = "UNMAP";
		}
		if($bngchr eq "*"){
			$bngchr = "UNMAP";
			$bngpos = "UNMAP";
		}
		if($lachchr eq "*"){
			$lachchr = "UNMAP";
			$lachpos = "UNMAP";
		}
	}
	
	# Deal with goatv3 contig naming convention
	my ($g3ctg) = $segs[5] =~ /(utg\d+)_len.*/;
	my $g3pos = $segs[6];
	if($g3ctg eq ""){
		$g3ctg = "UNMAP";
		$g3pos = "UNMAP";
	}
	if($segs[5] eq "\#N/A"){
		$g3ctg = "MISS";
		$g3pos = "MISS";
	}
	
	my ($bgichr, $bgipos, $rhchr, $rhpos) = ($segs[3], $segs[4], $segs[1], $segs[2]);
	
	print {$OUT} "$segs[0]\t$rhchr\t$rhpos\t$g13chr\t$g13pos\t$g13orient\t$g3ctg\t$g3pos\t$bngchr\t$bngpos\t$lachchr\t$lachpos\t$bgichr\t$bgipos\n";
	
}

close $IN;
close $OUT;

exit;
```
