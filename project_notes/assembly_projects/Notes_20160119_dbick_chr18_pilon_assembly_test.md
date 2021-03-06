# Creating accurate scaffolds from BAC-PACBIO sequencing
---
*1/19/2016*

These are my notes on taking existing BAC sequences from the chr18 bac clones, polishing them and assembling them into larger contiguous segments (or at least scaffolding them!).

## Table of Contents
* [Protocol](#protocol)
* [Assembling contigs into larger contigs](#assemble)
* [Getting rid of the vector and trying again](#vector)
* [Testing the pipeline on un-corrected fastas](#uncorrected)
* [Holstein BACs](#holstein)
* [Aligning with larger cattle scaffolds](#align)

<a name="protocol"></a>
## Protocol

The quivered BAC clone contigs are unrefined, and likely need pilon correction. I'm going to create a dummy fasta file (reference genome + extra segments to be polished) and Pilon-correct it. Then I'm going to use Velvet to try to do a quick graph-based assembly. If that fails, I'll go pure consensus-overlap (Celera?).

#### Here are the pipeline steps:
1. Identify regions on the original reference genome that are covered by the BAC sequences and remove them
2. Place BAC fasta entries at the end of the reference genome as extra "chrs"
3. Align reads from an individual to the segments
4. Run Pilon on the BAM file (or a subset of the BAM with only the clones we're interested in?)
5. Extract the corrected entries
6. Perform OLC or graph-based assembly

Let's start by getting things ready and proceeding in a step-wise fashion with the existing BAC sequences from John's data.

Files:
* LIB14363_unitig_3_oriented_vector_trimmed.fasta  
* LIB14414_unitig_273.fasta  
* LIB14435_unitig_94_vector_trim.fasta

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs

```bash
# I'm going to test to see if I can identify the regions of the chromosome that need to be removed from a simple alignment
cat *.fasta > pilon_testrun/combined_chr18_fastas.fa
bwa mem ../reference/umd3_kary_unmask_ngap.fa pilon_testrun/combined_chr18_fastas.fa > pilon_testrun/combined_chr18_fastas.sam

perl -lane 'if($F[0] =~ /^@/ || $F[1] > 16){next;}else{print "$F[1]\t$F[2]\t$F[3]\t$F[4]\t" . (length($F[9]) + $F[3]);}' < pilon_testrun/combined_chr18_fastas.sam
	16      chr18   57435857        60      57626394
	0       chr18   57565716        60      57716083
	16      chr18   57676772        60      57835648

# This is obviously not the full picture because the alignments aren't complete, but it's a good start!
# Creating the mask bed file so that we can exclude this region from alignment
echo -e "chr18\t57435857\t57835648" > pilon_testrun/chr18_mask_location.bed
~/bedtools-2.17.0/bin/maskFastaFromBed -fi ../reference/umd3_kary_unmask_ngap.fa -bed pilon_testrun/chr18_mask_location.bed -fo pilon_testrun/umd3_chr18_region_mask.fa

# Now to make and finalize the reference file
cat pilon_testrun/umd3_chr18_region_mask.fa pilon_testrun/combined_chr18_fastas.fa > pilon_testrun/umd3_chr18_masked_combined_unitigs.fa
```

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/pilon_testrun

```bash
bwa index umd3_chr18_masked_combined_unitigs.fa
samtools faidx umd3_chr18_masked_combined_unitigs.fa
perl ~/perl_toolchain/sequence_data_pipeline/runMergedBamPipeline.pl --fastqs ../Arlinda-Chief.10x.spreadsheet.tab --reference umd3_chr18_masked_combined_unitigs.fa --config test_pipeline.cnfg --threads 15 --output arlinda_chr18

# Oops! My config file had an improper ParallelGCThreads argument!
perl ~/perl_toolchain/sequence_data_scripts/bamMergeUtility.pl -d Arlinda-Chief -o arlinda_chr18_merged.bam -n 10

samtools view -H arlinda_chr18/arlinda_chr18_merged.bam | grep '@SQ' | tail
# Supposedly, I can pull space separated chromosomes using samtools
# Let's try

samtools view arlinda_chr18/arlinda_chr18_merged.bam LIB14363_unitig_3 LIB14414_unitig_273 LIB14435_unitig_94 | tail
# Looks like it worked! Generating a bam

samtools view -hb arlinda_chr18/arlinda_chr18_merged.bam LIB14363_unitig_3 LIB14414_unitig_273 LIB14435_unitig_94 > arlinda_chr18/arlinda_chr18_merged_subsectioned.bam
samtools index arlinda_chr18/arlinda_chr18_merged_subsectioned.bam
```

Here's where things get dicey. I don't want Pilon to correct the entire reference (it'll probably take all day!) but I also don't want it to crap out immediately when it checks the BAM header against the fastas that I'm trying to correct. I'm going to demo it with the subset of contigs that I want to correct, and if that fails, I'll try to reformat the bam header in order to trick the program into correcting only the reads that aligned to the specific contigs.

```bash
## Demo 1: subset reference fasta
samtools faidx combined_chr18_fastas.fa

~/jdk1.8.0_05/bin/java -jar ~/bin/pilon-1.13.jar --genome combined_chr18_fastas.fa --frags arlinda_chr18/arlinda_chr18_merged_subsectioned.bam --output combined_chr18_fixed --changes --vcf --diploid

# It worked! And it took only a few minutes!
perl -e '%h; while(<>){chomp; @s = split(/\s+/); if(length($s[2]) == 1 && length($s[3]) == 1 && $s[2] ne "\." && $s[3] ne "\."){$h{"SNP"} += 1;}else{ $h{"INDEL"} += 1;}} print "SNP\t" . $h{"SNP"} . "\n"; print "INDEL\t" . $h{"INDEL"} . "\n"; print "\n";' < combined_chr18_fixed.changes
```

There were several files that were generated by the above commands:
* combined_chr18_fixed.changes  <- This was a log file with all of the changes made
* combined_chr18_fixed.fasta    <- This was the corrected sequence
* combined_chr18_fixed.vcf      <- This is a vcf format of the changes file

#### Data summary

| Dataset | Metric | Value |
| :--- | :--- | ---: |
Original | Length | 499,780 bp
Pilon | Length | 499,711 bp 
Pilon | SNPs | 609
Pilon | INDELs | 115

<a name="assemble"></a>
## Assembling contigs into larger contigs

Now I'm going to take the error-corrected (and uncorrected!) fastas and attempt to merge them into larger contigs. We'll try velvet first, and then Celera if needed. We'll see what works!

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/pilon_testrun

```bash
# Note: had to recompile with "make 'LONGSEQUENCES=1'"
~/velvet_1.2.10/velveth pilon_velvet 11 -fasta -long combined_chr18_fixed.fasta
~/velvet_1.2.10/velvetg pilon_velvet

# Damn, it make a ton of contigs, Let's try a larger hash distance
rm pilon_velvet/*
~/velvet_1.2.10/velveth pilon_velvet 31 -fasta -long combined_chr18_fixed.fasta
~/velvet_1.2.10/velvetg pilon_velvet

# Still 1977 nodes. Not worth it.
```

OK, now I'm going to try AMOS (specifically minimus from that pipeline).

```bash
~/amos-3.1.0/bin/toAmos -s combined_chr18_fixed.fasta -o combined_chr18_fixed.afg
~/amos-3.1.0/bin/minimus2 combined_chr18_fixed -D REFCOUNT=1
	 The log file is: combined_chr18_fixed.runAmos.log
	Doing step 10:  Building AMOS bank & Dumping reads
	Doing step 11
	Doing step 12
	Doing step 13
	Doing step 20:  Getting overlaps
	Doing step 21
	Command: /usr/local/bin/show-coords -H -c -l -o -r -I 94 combined_chr18_fixed.delta | /home/dbickhart/amos-3.1.0/bin/nucmerAnnotate | egrep 'BEGIN|END|CONTAIN|IDENTITY' > combined_chr18_fixed.coords  exited with status: 1
```

Damn thing needs everything installed to specific directories! I edited the minimus2 executable (it was a shell script) to remove the hardlinks to nucmer entries.

```bash
~/amos-3.1.0/bin/minimus2 combined_chr18_fixed -D REFCOUNT=1

# Oops! It turns out that I should rename the original fasta file because it is overwritten by minimus2!
samtools faidx combined_chr18_fixed.fasta
head combined_chr18_fixed.fasta.fai
1       316083  3       60      61
mv combined_chr18_fixed.fasta combined_chr18_fixed.mini_refcount1.fasta

# Hmm, the refcount field may be limiting. I'm setting it to the default to do an all-vs-all alignment
~/amos-3.1.0/bin/minimus2 combined_chr18_fixed -D REFCOUNT=0
head combined_chr18_fixed.fasta.fai
1       316083  3       60      61

# Same result!
diff combined_chr18_fixed.fasta combined_chr18_fixed.mini_refcount1.fasta

# No differences!
```

The singular contig represents a 183,628 bp reduction in the naieve concatenated length of the three BAC contigs. 

Let's align this to Chr18 to see how this will work out. I'm interested to see the nucmer lineup and to see if we're hitting the right target region.

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/pilon_testrun

```bash
# getting the chr18 fasta ready
samtools faidx ../../reference/umd3_kary_unmask_ngap.fa chr18 > umd3_chr18_unmasked.fa

# This was taken from my notes file: Notes_20150831_dbick_goat_igc_region_annotation_scaffolding.md
nucmer -mumref -l 100 -c 1000 -p chr18_contig_align_test umd3_chr18_unmasked.fa combined_chr18_fixed.fasta
delta-filter -i 95 -o 95 chr18_contig_align_test.delta > chr18_contig_align_test.best.delta
dnadiff -p chr18_contig_align_test.dna -d chr18_contig_align_test.best.delta

mummerplot -p chr18_contig_align_test.mum --large --fat --png -Q combined_chr18_fixed.fasta chr18_contig_align_test.dna.1delta
cat chr18_contig_align_test.mum.gp | awk '{if (match($0, "len=")) { print substr($1, 1, index($1, "_")-1)"\" "$2" "$3} else print $0}'  > chr18_contig_align_test.mum.fixed.gp

# Then I had to change the terminal to emf and comment out mouse settings
gnuplot -geometry 900x900+0+0 -title john_contig chr18_contig_align_test.mum.fixed.gp
```

Hmm, close, but the reference sequence really crowds out the plot here. I need to extract only the reference sequence I need.

```bash
mkdir full_chr_18_fixed
mv chr18_contig_align_test.* full_chr_18_fixed/

samtools faidx ../../reference/umd3_kary_unmask_ngap.fa chr18:57300000-58000000 > umd3_chr18_573_580_unmasked.fa

nucmer -mumref -l 100 -c 1000 -p chr18_subsection_align_test umd3_chr18_573_580_unmasked.fa combined_chr18_fixed.fasta
delta-filter -i 95 -o 95 chr18_subsection_align_test.delta > chr18_subsection_align_test.best.delta
dnadiff -p chr18_subsection_align_test.dna -d chr18_subsection_align_test.best.delta

mummerplot -p chr18_subsection_align_test.mum --large --fat --png -Q combined_chr18_fixed.fasta chr18_subsection_align_test.dna.1delta
cat chr18_subsection_align_test.mum.gp | awk '{if (match($0, "len=")) { print substr($1, 1, index($1, "_")-1)"\" "$2" "$3} else print $0}' > chr18_subsection_align_test.mum.fixed.gp

gnuplot -geometry 900x900+0+0 -title john_contig chr18_subsection_align_test.mum.fixed.gp

mkdir full_chr_18_subsection
mv chr18_subsection_align_test.* full_chr_18_subsection/
```

It looks like John's contig has ~20kb more sequence near the 150kb mark than the reference, and there is one inversion and several translocations! Ah, damn, bad news is that it is the BAC vector! It turns out that Tim did not vector trim Lib14414_unitig_273.fasta, at the least. I found the vector location from NCBI's vecscreen.

<a name="vector"></a>
## Getting rid of the vector and trying again.

It's a big waste of time, but I've gotta get rid of the vector to be sure that my assembly is correct.


> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/pilon_testrun

```bash
# Starting from the beginning
# PILON
~/jdk1.8.0_05/bin/java -jar ~/bin/pilon-1.13.jar --genome combined_chr18_fastas.fa --frags arlinda_chr18/arlinda_chr18_merged_subsectioned.bam --output combined_chr18_pilon --changes --vcf --diploid

# Removing vector from corrected sequence
samtools faidx combined_chr18_pilon.fasta
samtools faidx combined_chr18_pilon.fasta LIB14414_unitig_273_pilon > /mnt/nfs/nfs2/dbickhart/transfer/lib14414_unitig_273.pilon.fa

samtools faidx combined_chr18_pilon.fasta LIB14363_unitig_3_pilon LIB14414_unitig_273_pilon:1-141764 LIB14435_unitig_94_pilon > combined_chr18_pilon.novec.fasta
# This was after NCBI vec screening

# AMOS and Minimus2
~/amos-3.1.0/bin/toAmos -s combined_chr18_pilon.novec.fasta -o combined_chr18_pilon.novec.fixed.afg
mv combined_chr18_pilon.novec.fasta combined_chr18_pilon.novec.fasta.orig
rm combined_chr18_pilon.novec.fasta.fai
~/amos-3.1.0/bin/minimus2 combined_chr18_pilon.novec.fixed -D REFCOUNT=0

samtools faidx combined_chr18_pilon.novec.fixed.fasta
head combined_chr18_pilon.novec.fixed.fasta.fai
1       307495  3       60      61

# Interesting... isn't this the approximate size of the vector?
# I'll do a nucmer plot of this against the previous merged fasta to see how that processed as well

## nucmer against chr18 subsection
nucmer -mumref -l 100 -c 1000 -p chr18_subsection_align_novec umd3_chr18_573_580_unmasked.fa combined_chr18_pilon.novec.fixed.fasta
delta-filter -i 95 -o 95 chr18_subsection_align_novec.delta > chr18_subsection_align_novec.best.delta
dnadiff -p chr18_subsection_align_novec.dna -d chr18_subsection_align_novec.best.delta
mummerplot -p chr18_subsection_align_novec.mum --large --fat --png -Q combined_chr18_pilon.novec.fixed.fasta chr18_subsection_align_novec.dna.1delta
cat chr18_subsection_align_novec.mum.gp | awk '{if (match($0, "len=")) { print substr($1, 1, index($1, "_")-1)"\" "$2" "$3} else print $0}' > chr18_subsection_align_novec.mum.fixed.gp

gnuplot -geometry 900x900+0+0 -title john_contig chr18_subsection_align_novec.mum.fixed.gp
mv chr18_subsection_align_novec.mum.png chr18_subsection_align_novec.mum.emf
```

There's still a section of the genome that doesn't match the UMD3.1 reference. Let's try to run my subsection alignment program on the master, corrected fasta to see what pops out.

```bash
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f combined_chr18_pilon.novec.fixed.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o combined_chr18_pilon.novec.align.tab

perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f combined_chr18_pilon.novec.fixed.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o combined_chr18_pilon.novec.align.tab
loaded fasta file!
[M::main_mem] read 308 sequences (307494 bp)...
[M::mem_process_seqs] Processed 308 reads in 3.477 CPU sec, 3.392 real sec
[main] Version: 0.7.10-r789
[main] CMD: bwa mem ../../reference/umd3_kary_unmask_ngap.fa temp.fq
[main] Real time: 56.742 sec; CPU: 9.354 sec
Longest aligments:      chr     start   end     length
                        chr1    20951590        153283269       132331679
                        chr2    16512761        125716377       109203616
                        chr11   48330260        107023187       58692927
                        chr5    29133071        85219301        56086230
                        chr16   5146193 55925864        50779671
                        chr23   27321674        52492537        25170863
                        chr13   3668010 27796611        24128601
                        chrX    74404074        96914001        22509927
                        chr18   40333783        62181617        21847834
                        chr17   66030015        73910427        7880412
                        chr3    2051723 2052723 1000
                        chr26   32458140        32459140        1000
                        chr8    12875966        12876966        1000
                        chr29   41546141        41547141        1000
                        chr14   56806974        56807974        1000
                        chr24   13511406        13511520        114
                        chr27   22840110        22840160        50
                        chr28   6264630 6264677 47
                        chr7    25614570        25614615        45
                        chr10   87992202        87992246        44
Output in: combined_chr18_pilon.novec.align.tab
```
The data really just confirmed the nucmer aligns, but with the added bonus of BWA mem mappings of the start and end of the contig to chr18.

<a name="uncorrected"></a>
## Testing the pipeline on un-corrected fastas 

I'm not happy to just try the pilon data, let's try assembling the uncorrected fastas and plotting that.

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/pilon_testrun

```bash
# AMOS
~/amos-3.1.0/bin/toAmos -s combined_chr18_fasta_novec.fa -o combined_chr18_fasta_novec.assemble.afg
~/amos-3.1.0/bin/minimus2 combined_chr18_fasta_novec.assemble -D REFCOUNT=0

samtools faidx combined_chr18_fasta_novec.assemble.fasta
head combined_chr18_fasta_novec.assemble.fasta.fai
1       307535  3       60      61

# So there is a difference in size at least!
# I made an automation script to run the nucmer plot
sh ../run_nucmer_plot_automation_script.sh umd3_chr18_573_580_unmasked.fa combined_chr18_fasta_novec.assemble.fasta

gnuplot -geometry 900x900+0+0 -title john_nopilon_contig combined_chr18_fasta_novec.mum.fixed.gp
```

There were just some minor changes with the sequence towards the 3' end of the assembly, notably the position of a large mum around the 57.65 Mb in the Pilon corrected version that is expanded, and a region in the uncorrected contig that maps towards the end of the contig. I suspect that these are due to poor read alignments after the correction.

<a name="holstein"></a>
## Holstein BACs

OK, the first assembly showed little difference between UMD3 apart from an inversion and some expanded repeats. I'm going to see if the RPCI-42 library (Holstein) has any differences from the Domino BAC sequence.

Here are the files that will be used:
* LIB14397_unitig_1_vector_trimmed.fasta        
* LIB14414_unitig_273.fasta 
* LIB14398_unitig_305_quiver.Vector.Trim.fasta  
* LIB14435_unitig_94_vector_trim.fasta

**NOTE:** the lib14414 unitig is not vector trimmed! Let's do that now.

NCBI's [vecscreen](http://www.ncbi.nlm.nih.gov/tools/vecscreen/) shows that the last 10kb are the vector. Specifically these bases: 141780-150367.

Trimming now...

> pwd: /home/dbickhart/share/grants/immune_gene_cluster_grant/assemblies/chr18_bacs/rpci_42

```bash
samtools faidx LIB14414_unitig_273.fasta
samtools faidx LIB14414_unitig_273.fasta LIB14414_unitig_273:1-141780 | perl -e '<>; print ">LIB14414_unitig_273_trimmed\n"; while(<>){print $_;}' > LIB14414_unitig_273.trimmed.fasta
```

From my notes, 14414 did not assemble well, and may overlap the same region as 14398. Here is the appropriate table:

Library |	Region Name	| Clone Name	| End Coords	| Notes	| Library	| Results
:--- | :--- | :--- | :--- | :--- | :--- | :---
RPCI-42	|chr18|	RPCI42_154M1|	chr18:57,371,161-57,531,510|	Flanking 5' region|	14397|	finished
RPCI-42	|chr18|	RPCI42_118F24|	chr18:57,548,063-57,704,285|	Overlaps SNP|	14414|	Did not assemble well
RPCI-42	|chr18|	RPCI42_118B22|	chr18:57,548,064-57,704,269|	Perhaps a clone of the previous region?|	14398|	finished
RPCI-42	|chr18|	RPCI42_3D15|	chr18:57,657,380-57,806,510|	Flanking 3' region|	14435|	finished

My thoughts: three assemblies might be best here. Assemble with 14414, with 14398 and with both.  

Lets start with the entirety of the pipeline and get Pilon-corrected assemblies before Amos overlap.

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/holstein_bacs

```bash
# Removing target chr18 regions
cat ../LIB14397_unitig_1_vector_trimmed.fasta ../LIB14398_unitig_305_quiver.Vector.Trim.fasta ../LIB14414_unitig_273.vectortrim.fasta ../LIB14435_unitig_94_vector_trim.fasta > rpci42_bac.fasta
bwa mem ../../reference/umd3_kary_unmask_ngap.fa rpci42_bac.fasta > rpci42_bacs_uncorrected_align.sam

perl -lane 'if($F[0] =~ /^@/ || $F[1] > 16){next;}else{print "$F[0]\t$F[1]\t$F[2]\t$F[3]\t$F[4]\t" . (length($F[9]) + $F[3]);}' < rpci42_bacs_uncorrected_align.sam
	LIB14397_unitig_1        0       chr18   57381124        60      57541326
	LIB14398_unitig_305     16      chr18   57565716        60      57729084
	LIB14414_unitig_273     0       chr18   57565716        60      57707495
	LIB14435_unitig_94     16      chr18   57676772        60      57835648

# Looks like the region is chr18:57381124-57835648
# NOTE: the two "clones" that Tim mentioned differ in end sequence alignments by ~ 20kb
# Masking the fasta
echo -e "chr18\t57381124\t57835648" > rpci42_bac_mask.bed
~/bedtools-2.17.0/bin/maskFastaFromBed -fi ../../reference/umd3_kary_unmask_ngap.fa -bed rpci42_bac_mask.bed -fo chr18_initial_masked.fa

# Making the pilon fasta
cat chr18_initial_masked.fa rpci42_bac.fasta > chr18_combined_unitigs_masked.fa
bwa index chr18_combined_unitigs_masked.fa
samtools faidx chr18_combined_unitigs_masked.fa

# Alignment
perl ~/perl_toolchain/sequence_data_pipeline/runMergedBamPipeline.pl --fastqs ../Arlinda-Chief.10x.spreadsheet.tab --reference chr18_combined_unitigs_masked.fa --config ../pilon_testrun/test_pipeline.cnfg --threads 15 --output arlinda_rpci42

# Extracting contigs
samtools view -hb arlinda_rpci42/Arlinda-Chief/Arlinda-Chief.merged.bam 'LIB14397_unitig_1_vector_trimmed' 'LIB14398_unitig_305|quiver_Vector_Trim' LIB14414_unitig_273 LIB14435_unitig_94_vector_trim > arlinda_rpci42/Arlinda-Chief.unitigs.bam

# Pilon correction
samtools index /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/holstein_bacs/arlinda_rpci42/Arlinda-Chief.unitigs.bam
~/jdk1.8.0_05/bin/java -jar ~/bin/pilon-1.13.jar --genome chr18_combined_unitigs_masked.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.bam --output chr18_rpci42_unitigs --changes --vcf --diploid

# Pilon crashed -- I suspect its due to the presence of a pipe in the fasta names
perl -ne 'if($_ =~ /^>/){ $_ =~ s/\|/_/g; print $_;}else{print $_;}' < chr18_combined_unitigs_masked.fa > chr18_combined_unitigs_masked.nopipe.fa
perl -ne 'if($_ =~ /^>/){ $_ =~ s/\|/_/g; print $_;}else{print $_;}' < rpci42_bac.fasta > rpci42_bac.corrected.fasta

samtools view -h arlinda_rpci42/Arlinda-Chief.unitigs.bam | perl -ne '$_ =~ s/\|/_/g; print $_;' | samtools view -bS - > arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam
samtools index arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam

~/jdk1.8.0_05/bin/java -Xmx75g -jar ~/bin/pilon-1.13.jar --genome chr18_combined_unitigs_masked.nopipe.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam --output chr18_rpci42_unitigs --changes --vcf --diploid
# OK... still crashing. Let's try this without the pilon correction

# Amos assembly of non-polished contigs
cp rpci42_bac.corrected.fasta rpci42_bac.corrected.fasta.bak
~/amos-3.1.0/bin/toAmos -s rpci42_bac.corrected.fasta -o rpci42_bac.corrected.afg
~/amos-3.1.0/bin/minimus2 rpci42_bac.corrected -D REFCOUNT=0

head rpci42_bac.corrected.fasta.fai
	1       163390  3       60      61

# That's... really weird. Let's see how the alignments turn out
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f rpci42_bac.corrected.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o rpci42_bac.corrected.fasta.align.tab
# Nope! Doesn't look good.

# Let's try individual pilon targets
samtools faidx rpci42_bac.fasta
# Lib14397_1 worked
~/jdk1.8.0_05/bin/java -Xmx70g -jar ~/bin/pilon-1.16.jar --genome chr18_combined_unitigs_masked.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam --output lib14397_1_pilon --changes --vcf --diploid --targets LIB14397_unitig_1_vector_trimmed

# Lib14398_305 failed
~/jdk1.8.0_05/bin/java -Xmx70g -jar ~/bin/pilon-1.16.jar --genome chr18_combined_unitigs_masked.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam --output lib14398_305_pilon --changes --vcf --diploid --targets LIB14398_unitig_305_quiver_Vector_Trim

# Lib14414_273 worked
~/jdk1.8.0_05/bin/java -Xmx70g -jar ~/bin/pilon-1.16.jar --genome chr18_combined_unitigs_masked.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam --output lib14414_273_pilon --changes --vcf --diploid --targets LIB14414_unitig_273

# Lib14435_94 worked
~/jdk1.8.0_05/bin/java -Xmx70g -jar ~/bin/pilon-1.16.jar --genome chr18_combined_unitigs_masked.fa --frags arlinda_rpci42/Arlinda-Chief.unitigs.corrected.bam --output lib14435_94_pilon --changes --vcf --diploid --targets LIB14435_unitig_94_vector_trim

# Taking the error corrected fasta entries for further analysis
cat lib14397_1_pilon.fasta lib14414_273_pilon.fasta lib14435_94_pilon.fasta > pilon_corrected_rpci14_bac.fa
cp pilon_corrected_rpci14_bac.fa pilon_corrected_rpci14_bac.fa.bak

# Amos assembly of polished contigs
~/amos-3.1.0/bin/toAmos -s pilon_corrected_rpci14_bac.fa -o pilon_corrected_rpci14_bac.afg
~/amos-3.1.0/bin/minimus2 pilon_corrected_rpci14_bac -D REFCOUNT=0
samtools faidx pilon_corrected_rpci14_bac.fasta

# I didn't get a contig, trying to figure out why
```

OK! The problem is in the nucmer coords file, which shows no overlap among the three contigs. Also, in my prior assembly, the third contig didn't assemble into the final product at all! The information is in the "singletons" file. 

First assembly singletons:
* LIB14435_unitig_94_vector_trim

Second assembly singletons:
* LIB14397_unitig_1_vector_trimmed_pilon
* LIB14414_unitig_273_pilon
* LIB14435_unitig_94_vector_trim_pilon

Let's try an assembly alignment to try to sort this out.

```bash
# Lib 14397
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f lib14397_1_pilon.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o lib14397_1_pilon.fasta.align.tab
Longest aligments:      chr     start   end     length
                        chr18   57371117        57532624        161507
                        *       0       1000    1000
                        chr27   22286043        22286293        250
                        chrX    44952057        44952154        97
                        chr6    113122076       113122147       71
                        chr4    3045190 3045260 70

# Lib 14414
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f lib14414_273_pilon.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o lib14414_273_pilon.fasta.align.tab
Longest aligments:      chr     start   end     length
                        chr1    20951590        153283269       132331679
                        chr24   13511406        55366739        41855333
                        chrX    101793354       142730599       40937245
                        chr16   5146193 36768842        31622649
                        chr18   40333924        62955483        22621559
                        chr23   27321674        42251667        14929993
                        chr13   3668010 3669010 1000
                        chr15   25930890        25931890        1000
                        chr11   6304540 6305540 1000
                        chr26   32458140        32459140        1000
                        chr6    48897110        48898110        1000
                        chr2    16512761        16513761        1000
						... not so good!
# Lib 14435
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f lib14435_94_pilon.fasta -r ../../reference/umd3_kary_unmask_ngap.fa -o lib14435_94_pilon.fasta.align.tab
Longest aligments:      chr     start   end     length
                        chr1    20951590        121426804       100475214
                        chr10   15174811        87992246        72817435
                        chrX    74404074        134697683       60293609
                        chr16   5146193 55925864        50779671
                        chr18   15236467        62961880        47725413
                        chr5    22425571        22426571        1000
                        chr26   32458140        32459140        1000
                        chr20   7720770 7721770 1000
                        chr13   78628163        78629163        1000
                        chr4    69969728        69970728        1000
                        chr11   74810900        74811900        1000
                        chr6    67547644        67547816        172
                        chr24   13511406        13511551        145
                        chr23   27321674        27321789        115
                        chr19   53681295        53681355        60
```

OK, the problem is that 14414 has issues and so does 14435. I suspect that they were not properly assembled.

<a name="align"></a>
## Aligning with larger cattle scaffolds

Tim sent me his current (unpolished) version of the assembly. My hope is to extract the Chr18 regions that we need, polish them with pilon, collect the polishing data, and then compare the regions back to UMD3 and BTAU4. This should be the final piece of the puzzle that John needs.

My first step is to identify the regions that each clone aligns to on the new scaffolds.

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs

```bash
perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f LIB14397_unitig_1_vector_trimmed.fasta -r cattle_31Mar2016_bfJmS.fasta -o scaffold_comp/LIB14397_unitig_1_juan.tab
Longest aligments:      chr     start   end     length
                        ScbfJmS_318     56875450        57035827        160377
                        *       0       1000    1000

# Holy crapoli! That's one solid, contiguous block???

perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f LIB14398_unitig_305_quiver.Vector.Trim.fasta -r cattle_31Mar2016_bfJmS.fasta -o scaffold_comp/LIB14398_unitig_305_juan.tab
Longest aligments:      chr     start   end     length
                        ScbfJmS_1701    86193152        99873250        13680098
                        ScbfJmS_2343    2755967 5740448 2984481
                        ScbfJmS_318     57051564        57213021        161457 <- this is the alignment
                        ScbfJmS_1121    30478491        30479491        1000
                        ScbfJmS_692     9632368 9633368 1000
                        ScbfJmS_528     34542801        34542962        161
                        ScbfJmS_989     40131499        40131597        98
                        ScbfJmS_1283    27416047        27416122        75
                        ScbfJmS_296     34434764        34434825        61

perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f LIB14414_unitig_273.vectortrim.fasta -r cattle_31Mar2016_bfJmS.fasta -o scaffold_comp/LIB14414_unitig_273_juan.tab
Longest aligments:      chr     start   end     length
                        ScbfJmS_318     57079699        57861777        782078	<- over embellished
                        ScbfJmS_2343    2756044 2757044 1000
                        ScbfJmS_1121    30478491        30478744        253
                        ScbfJmS_528     34542801        34542962        161
                        ScbfJmS_217     53536990        53537118        128
                        ScbfJmS_2171    7927026 7927133 107
                        ScbfJmS_989     40131499        40131597        98
                        ScbfJmS_1864    19772502        19772579        77
                        ScbfJmS_1283    27416047        27416122        75
                        ScbfJmS_1701    86193107        86193152        45

perl ~/perl_toolchain/assembly_scripts/alignUnitigSectionsToRef.pl -f LIB14435_unitig_94_vector_trim.fasta -r cattle_31Mar2016_bfJmS.fasta -o scaffold_comp/LIB14435_unitig_94_juan.tab
Longest aligments:      chr     start   end     length
                        ScbfJmS_318     57127035        57366337        239302

# Another one segment run!
```

Let's tabulate the regions that we're interested in:

* ScbfJmS_318	56875450	57366337
* chr18			57,371,161	57,806,510

Just for my notes:

> tag SNP is ARS-BFGL-NGS-109285 at 57,589,121 bp on BTA 18

OK, now just to extract and compare to the same region in UMD3.1. Our SNP should be somewhere in the 57.100 Mb region of this assembly.

```bash
samtools faidx cattle_31Mar2016_bfJmS.fasta ScbfJmS_318:56875450-57366337 > scaffold_comp/ScbfJmS_318-56875450_57366337.fa
samtools faidx /mnt/iscsi/vnx_gliu_7/reference/umd3_kary_unmask_ngap.fa chr18:57371161-57806510 > scaffold_comp/chr18-57371161_57806510.fa

cd scaffold_comp
sh ../run_nucmer_plot_automation_script.sh chr18-57371161_57806510.fa ScbfJmS_318-56875450_57366337.fa
```

OK, the region looks good! There's a good amount of inversion after the SNP coordinate as well. Let's Pilon polish this and then ship it off to Ben for annotation.

> Blade14: /mnt/iscsi/vnx_gliu_7/john_assembled_contigs/scaffold_comp

```bash
perl ~/perl_toolchain/sequence_data_pipeline/runMergedBamPipeline.pl --fastqs ../Arlinda-Chief.10x.spreadsheet.tab --output arlinda_chief_newassembly --reference ../cattle_31Mar2016_bfJmS.fasta --config test_pipeline.cnfg  --threads 10

~/jdk1.8.0_05/bin/java -Xmx75g -jar ~/pilon-1.16.jar --genome ../cattle_31Mar2016_bfJmS.fasta --frags arlinda_chief_newassembly/Arlinda-Chief/Arlinda-Chief.merged.bam > cattle_31Mar2016_bfJmS.pilon.fasta

# No matter what I do, Pilon runs out of heap space and crashes
# I suspect that it's due to the number of chromosome scaffolds here
# Let's reduce the number

cat ../cattle_31Mar2016_bfJmS.fasta.fai | sort -k2rn | perl -ne '@s = split(/\t/); if($s[1] > 100000){print "$s[0] ";}'; echo
samtools view -b arlinda_chief_newassembly/Arlinda-Chief/Arlinda-Chief.merged.bam ScbfJmS_2191 ScbfJmS_994 ScbfJmS_1738 ScbfJmS_1688 ScbfJmS_1701 ScbfJmS_1121 ScbfJmS_989 ScbfJmS_801 ScbfJmS_1407 ScbfJmS_1238 ScbfJmS_1568 ScbfJmS_217 ScbfJmS_2073 ScbfJmS_393 ScbfJmS_296 ScbfJmS_318 ScbfJmS_1864 ScbfJmS_1518 ScbfJmS_433 ScbfJmS_1805 ScbfJmS_2163 ScbfJmS_1283 ScbfJmS_2027 ScbfJmS_553 ScbfJmS_621 ScbfJmS_528 ScbfJmS_1659 ScbfJmS_1990 ScbfJmS_1789 ScbfJmS_436 ScbfJmS_1017 ScbfJmS_1552 ScbfJmS_775 ScbfJmS_2171 ScbfJmS_1380 ScbfJmS_61 ScbfJmS_505 ScbfJmS_2419 ScbfJmS_402 ScbfJmS_2331 ScbfJmS_2067 ScbfJmS_2346 ScbfJmS_692 ScbfJmS_879 ScbfJmS_1250 ScbfJmS_1950 ScbfJmS_1059 ScbfJmS_1154 ScbfJmS_2149 ScbfJmS_2064 ScbfJmS_2330 ScbfJmS_469 ScbfJmS_1723 ScbfJmS_155 ScbfJmS_1904 ScbfJmS_573 ScbfJmS_545 ScbfJmS_164 ScbfJmS_2243 ScbfJmS_2343 ScbfJmS_2323 ScbfJmS_1876 ScbfJmS_1490 ScbfJmS_2290 ScbfJmS_70 ScbfJmS_1502 ScbfJmS_364 ScbfJmS_246 ScbfJmS_1348 ScbfJmS_763 ScbfJmS_1852 ScbfJmS_1539 ScbfJmS_1945 ScbfJmS_2061 ScbfJmS_1138 ScbfJmS_158 ScbfJmS_1165 ScbfJmS_907 ScbfJmS_446 ScbfJmS_2252 ScbfJmS_1160 ScbfJmS_1026 ScbfJmS_2091 ScbfJmS_1645 ScbfJmS_1222 ScbfJmS_347 ScbfJmS_1899 ScbfJmS_124 ScbfJmS_1356 ScbfJmS_2257 ScbfJmS_474 ScbfJmS_1364 ScbfJmS_892 ScbfJmS_268 ScbfJmS_1704 ScbfJmS_1324 ScbfJmS_359 ScbfJmS_713 ScbfJmS_2055 ScbfJmS_2057 ScbfJmS_565 ScbfJmS_264 ScbfJmS_2118 ScbfJmS_2447 ScbfJmS_373 ScbfJmS_1771 ScbfJmS_1751 ScbfJmS_1344 ScbfJmS_516 ScbfJmS_432 ScbfJmS_1396 ScbfJmS_2180 ScbfJmS_282 ScbfJmS_511 ScbfJmS_1336 ScbfJmS_2108 ScbfJmS_1097 ScbfJmS_2286 ScbfJmS_2217 ScbfJmS_1335 ScbfJmS_1578 ScbfJmS_829 ScbfJmS_965 ScbfJmS_317 ScbfJmS_1727 ScbfJmS_2110 ScbfJmS_1934 ScbfJmS_1091 ScbfJmS_1598 ScbfJmS_139 ScbfJmS_142 ScbfJmS_1867 ScbfJmS_938 ScbfJmS_2009 ScbfJmS_1217 ScbfJmS_46 ScbfJmS_843 ScbfJmS_1042 ScbfJmS_2140 ScbfJmS_1131 ScbfJmS_1072 ScbfJmS_2452 ScbfJmS_360 ScbfJmS_1330 ScbfJmS_1776 ScbfJmS_779 ScbfJmS_1513 ScbfJmS_315 ScbfJmS_1636 ScbfJmS_571 ScbfJmS_1274 ScbfJmS_1378 ScbfJmS_547 ScbfJmS_350 ScbfJmS_59 ScbfJmS_1312 ScbfJmS_2104 ScbfJmS_1223 ScbfJmS_1438 ScbfJmS_752 ScbfJmS_1334 ScbfJmS_1172 ScbfJmS_2446 ScbfJmS_1498 ScbfJmS_1928 ScbfJmS_1285 ScbfJmS_40 ScbfJmS_2318 ScbfJmS_234 ScbfJmS_1782 ScbfJmS_1463 ScbfJmS_1368 ScbfJmS_955 ScbfJmS_1439 ScbfJmS_837 ScbfJmS_855 ScbfJmS_781 ScbfJmS_189 ScbfJmS_848 ScbfJmS_2017 ScbfJmS_1959 ScbfJmS_1618 ScbfJmS_178 ScbfJmS_2420 ScbfJmS_304 ScbfJmS_1257 ScbfJmS_1806 ScbfJmS_24 ScbfJmS_2130 ScbfJmS_563 ScbfJmS_1626 ScbfJmS_424 ScbfJmS_2022 ScbfJmS_728 ScbfJmS_1639 ScbfJmS_1957 ScbfJmS_1625 ScbfJmS_334 ScbfJmS_406 ScbfJmS_1534 ScbfJmS_297 ScbfJmS_2469 ScbfJmS_2203 ScbfJmS_212 ScbfJmS_759 ScbfJmS_307 ScbfJmS_792 ScbfJmS_2044 ScbfJmS_488 ScbfJmS_346 ScbfJmS_836 ScbfJmS_2269 ScbfJmS_1208 ScbfJmS_2085 ScbfJmS_1988 ScbfJmS_2341 ScbfJmS_935 ScbfJmS_555 ScbfJmS_2096 > arlinda_chief_newassembly/arlinda-chief.subsection.gt100k.bam

# I need to generate a reference fasta for just these segments as well!
samtools faidx ../cattle_31Mar2016_bfJmS.fasta ScbfJmS_2191 ScbfJmS_994 ScbfJmS_1738 ScbfJmS_1688 ScbfJmS_1701 ScbfJmS_1121 ScbfJmS_989 ScbfJmS_801 ScbfJmS_1407 ScbfJmS_1238 ScbfJmS_1568 ScbfJmS_217 ScbfJmS_2073 ScbfJmS_393 ScbfJmS_296 ScbfJmS_318 ScbfJmS_1864 ScbfJmS_1518 ScbfJmS_433 ScbfJmS_1805 ScbfJmS_2163 ScbfJmS_1283 ScbfJmS_2027 ScbfJmS_553 ScbfJmS_621 ScbfJmS_528 ScbfJmS_1659 ScbfJmS_1990 ScbfJmS_1789 ScbfJmS_436 ScbfJmS_1017 ScbfJmS_1552 ScbfJmS_775 ScbfJmS_2171 ScbfJmS_1380 ScbfJmS_61 ScbfJmS_505 ScbfJmS_2419 ScbfJmS_402 ScbfJmS_2331 ScbfJmS_2067 ScbfJmS_2346 ScbfJmS_692 ScbfJmS_879 ScbfJmS_1250 ScbfJmS_1950 ScbfJmS_1059 ScbfJmS_1154 ScbfJmS_2149 ScbfJmS_2064 ScbfJmS_2330 ScbfJmS_469 ScbfJmS_1723 ScbfJmS_155 ScbfJmS_1904 ScbfJmS_573 ScbfJmS_545 ScbfJmS_164 ScbfJmS_2243 ScbfJmS_2343 ScbfJmS_2323 ScbfJmS_1876 ScbfJmS_1490 ScbfJmS_2290 ScbfJmS_70 ScbfJmS_1502 ScbfJmS_364 ScbfJmS_246 ScbfJmS_1348 ScbfJmS_763 ScbfJmS_1852 ScbfJmS_1539 ScbfJmS_1945 ScbfJmS_2061 ScbfJmS_1138 ScbfJmS_158 ScbfJmS_1165 ScbfJmS_907 ScbfJmS_446 ScbfJmS_2252 ScbfJmS_1160 ScbfJmS_1026 ScbfJmS_2091 ScbfJmS_1645 ScbfJmS_1222 ScbfJmS_347 ScbfJmS_1899 ScbfJmS_124 ScbfJmS_1356 ScbfJmS_2257 ScbfJmS_474 ScbfJmS_1364 ScbfJmS_892 ScbfJmS_268 ScbfJmS_1704 ScbfJmS_1324 ScbfJmS_359 ScbfJmS_713 ScbfJmS_2055 ScbfJmS_2057 ScbfJmS_565 ScbfJmS_264 ScbfJmS_2118 ScbfJmS_2447 ScbfJmS_373 ScbfJmS_1771 ScbfJmS_1751 ScbfJmS_1344 ScbfJmS_516 ScbfJmS_432 ScbfJmS_1396 ScbfJmS_2180 ScbfJmS_282 ScbfJmS_511 ScbfJmS_1336 ScbfJmS_2108 ScbfJmS_1097 ScbfJmS_2286 ScbfJmS_2217 ScbfJmS_1335 ScbfJmS_1578 ScbfJmS_829 ScbfJmS_965 ScbfJmS_317 ScbfJmS_1727 ScbfJmS_2110 ScbfJmS_1934 ScbfJmS_1091 ScbfJmS_1598 ScbfJmS_139 ScbfJmS_142 ScbfJmS_1867 ScbfJmS_938 ScbfJmS_2009 ScbfJmS_1217 ScbfJmS_46 ScbfJmS_843 ScbfJmS_1042 ScbfJmS_2140 ScbfJmS_1131 ScbfJmS_1072 ScbfJmS_2452 ScbfJmS_360 ScbfJmS_1330 ScbfJmS_1776 ScbfJmS_779 ScbfJmS_1513 ScbfJmS_315 ScbfJmS_1636 ScbfJmS_571 ScbfJmS_1274 ScbfJmS_1378 ScbfJmS_547 ScbfJmS_350 ScbfJmS_59 ScbfJmS_1312 ScbfJmS_2104 ScbfJmS_1223 ScbfJmS_1438 ScbfJmS_752 ScbfJmS_1334 ScbfJmS_1172 ScbfJmS_2446 ScbfJmS_1498 ScbfJmS_1928 ScbfJmS_1285 ScbfJmS_40 ScbfJmS_2318 ScbfJmS_234 ScbfJmS_1782 ScbfJmS_1463 ScbfJmS_1368 ScbfJmS_955 ScbfJmS_1439 ScbfJmS_837 ScbfJmS_855 ScbfJmS_781 ScbfJmS_189 ScbfJmS_848 ScbfJmS_2017 ScbfJmS_1959 ScbfJmS_1618 ScbfJmS_178 ScbfJmS_2420 ScbfJmS_304 ScbfJmS_1257 ScbfJmS_1806 ScbfJmS_24 ScbfJmS_2130 ScbfJmS_563 ScbfJmS_1626 ScbfJmS_424 ScbfJmS_2022 ScbfJmS_728 ScbfJmS_1639 ScbfJmS_1957 ScbfJmS_1625 ScbfJmS_334 ScbfJmS_406 ScbfJmS_1534 ScbfJmS_297 ScbfJmS_2469 ScbfJmS_2203 ScbfJmS_212 ScbfJmS_759 ScbfJmS_307 ScbfJmS_792 ScbfJmS_2044 ScbfJmS_488 ScbfJmS_346 ScbfJmS_836 ScbfJmS_2269 ScbfJmS_1208 ScbfJmS_2085 ScbfJmS_1988 ScbfJmS_2341 ScbfJmS_935 ScbfJmS_555 ScbfJmS_2096 > cattle_31Mar2016_bfJmS.prepilon.fasta 

# Retrying pilon
~/jdk1.8.0_05/bin/java -Xmx80g -jar ~/pilon-1.16.jar --genome cattle_31Mar2016_bfJmS.prepilon.fasta --frags arlinda_chief_newassembly/arlinda-chief.subsection.gt100k.sort.bam

# OK! crashed again! Just Scaffold ScbfJmS_318 then!
samtools faidx ../cattle_31Mar2016_bfJmS.fasta ScbfJmS_318 > ScbfJmS_318.subsection.fa
samtools view -b arlinda_chief_newassembly/Arlinda-Chief/Arlinda-Chief.merged.bam ScbfJmS_318 > arlinda_chief_newassembly/ScbfJmS_318.arlinda_chief.subsection.bam
samtools index arlinda_chief_newassembly/ScbfJmS_318.arlinda_chief.subsection.bam


~/jdk1.8.0_05/bin/java -Xmx80g -jar ~/pilon-1.16.jar --genome ScbfJmS_318.subsection.fa --frags arlinda_chief_newassembly/ScbfJmS_318.arlinda_chief.subsection.bam --output ScbfJmS_318_pilon_corrected
```

Remapping snp probes onto the new assembly segment to check probe order continuity.

> 3850: /seq1/side_projects/john_chr18/snp_probe_remap

```bash
bwa index ScbfJmS_318_pilon_corrected.fasta.gz

# Generating SNP fastq and location position file
perl -e '<>; open(SNP, "> hd_snp_probes.fq"); open(LOC, "> hd_snp_locations.tab"); while(<>){chomp; $_ =~ s/\r//g; @s = split(/\t/); @b = split(/[\[\]]/, $s[4]); $b[2] =~ tr/ACGT/TGCA/; $b[2] = reverse($b[2]); my $q1 = "I" x length($b[0]); my $q2 = "I" x length($b[2]); print SNP "\@$s[0].f\n$b[0]\n+\n$q1\n\@$s[0].r\n$b[2]\n+\n$q2\n"; print LOC "$s[0]\t$s[2]\t$s[3]\n";}' < /work1/grw/chips/BovineHD_B_AlleleReport.txt

bwa mem ScbfJmS_318_pilon_corrected.fasta.gz hd_snp_probes.fq > ScbfJmS_318_hd_probe_align.sam
perl ~/perl_toolchain/snp_utilities/getSNPPositionFromSAM.pl ScbfJmS_318_hd_probe_align.sam ScbfJmS_318_hd_probe_align.tab
perl -lane 'if($F[0] =~ /ProbeName/){print $_;}elsif($F[1] ne "*"){print $_;}' < ScbfJmS_318_hd_probe_align.tab > ScbfJmS_318_hd_probe_align.filtered.tab

# Trying this again with the actual probes from the BovineHD
perl -lane '$q = "I" x length($F[1]); print "\@$F[0]\n$F[1]\n+\n$q";' < bovine_hd_snpprobes_illumina.txt > bovine_hd_snpprobes_illumina.fq

bwa mem ScbfJmS_318_pilon_corrected.fasta.gz bovine_hd_snpprobes_illumina.fq > bovine_hd_snpprobes_illumina.ScbfJmS.sam

perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl probe_seq_align_marker.list bovineHD_chr18_markers.txt
File Number 1: probe_seq_align_marker.list
File Number 2: bovineHD_chr18_markers.txt
Set     Count
1       199
1;2     17591
2       868

# Some of the "1" category were unreliable mismatches because I didn't align to the whole genome
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -o probe_seq_align_marker.list bovineHD_chr18_markers.txt

perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); %h; while(<IN>){chomp; $h{$_} = 1;} close IN; open(IN, "< $ARGV[1]"); while(<IN>){chomp; @s = split(/\t/); if($s[0] =~ /^@/ || $s[2] eq "*" || $s[1] & 2048){next;} if(exists($h{$s[0]})){ my $len = 0; while($s[5] =~ /(\d+)([MIDNSHPX])/g){ $c = $2; if($c eq "M" || $c eq "D" || $c eq "N" || $c eq "X" || $c eq "P"){$len += $1;}} $s[3] += $len - 1; print "$s[0]\t$s[2]\t$s[3]\n";}} close IN;' bovineHD_chr18_markers.txt bovine_hd_snpprobes_illumina.ScbfJmS.sam | sort -k3n > ScbfJmS_318_probe_locs.tab
```

Now, I need to devise a method for identifying inversions. I'll just rank them and do a plot to try to expose some inversions.

```bash
perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); my %h; my $c = 1; while(<IN>){chomp; @s = split(/\t/); $h{$s[0]} = $c; $c++;} close IN; open(IN, "< $ARGV[1]"); $c = 1;  while(<IN>){chomp; @s = split(/\t/); print "$c\t$h{$s[0]}\n"; $c++;} close IN;' bovine_HD_chr18_umd3_order.tab ScbfJmS_318_probe_locs.tab > probe_order.vecs
```

Plotting a simple diagram to visualize all of this

```R
data <- read.delim("probe_order_bp.vecs", sep="\t", header=FALSE)
plot(data, xlim=c(57000000, 58000000), ylim=c(56000000, 58000000))
# assembled region start
abline(v=57371161)
# assembled region end
abline(v=57806510)
# priority SNP
abline(v=57589121)
dev.copy2pdf(file="local_aligned_snp_tagbwsnp.pdf", useDingbats=FALSE)
```

There appears to be this large region (100kb) that is devoid of any markers. This corresponds to our large inverted/repetitive regions identified in the assembly.

```bash
# Allowing for direct probe position comparisons
perl -e 'print "SNPID\tUMD3\tNEW\n"; chomp(@ARGV); open(IN, "< $ARGV[0]"); my %h; my $c = 1; while(<IN>){chomp; @s = split(/\t/); $h{$s[0]} = $s[2]; $c++;} close IN; open(IN, "< $ARGV[1]"); $c = 1;  while(<IN>){chomp; @s = split(/\t/); $j = "NA"; if(exists($h{$s[0]})){$j = $h{$s[0]};} print "$s[0]\t$s[2]\t$j\n"; $c++;} close IN;' ScbfJmS_318_probe_locs.tab bovine_HD_chr18_umd3_order.tab > mapped_probe_umd3_new_order_comp.tab

# The last chunk of chr18 from 63,226,272 to 65,999,159 is missing on this contig
perl -lane 'if($F[2] eq "NA" && $F[1] < 63226272){print $_;}' < mapped_probe_umd3_new_order_comp.tab | wc -l
	52 <- probes unmapped in our new assembly compared to UMD3

# Let's try to pull the probes that aren't on the HD chip but are aligned in our data again to try to insert them in the middle
# Grabbing the new probes that we need to intercalate
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -l 1 probe_seq_align_marker.list bovineHD_chr18_markers.txt | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/grepMultipleExactMatches.pl -g stdin -c 0 -f bovine_hd_snpprobes_illumina.ScbfJmS.sam -e '@' -x 0,2,3,5 | sort -k 3n | perl -lane 'if($F[3] =~ /S$/){next;}else{print $_;}' > new_assembly_newlymapped_probes.tab

# There were 162 probes that need to be added to the list
perl intercalation_script.pl new_assembly_newlymapped_probes.tab mapped_probe_umd3_new_order_comp.tab > mapped_probe_umd3_new_order_comp.intercalate.tab
```

Here's the crude intercalation script:

```perl
chomp(@ARGV);
open(IN, "< $ARGV[0]");
my @n;
while(<IN>){
        chomp;
        my @s = split(/\t/);
        push(@n, \@s);
}
close IN;
open(IN, "< $ARGV[1]");
my $head = <IN>;
print $head;
while(<IN>){
        chomp;
        my @s = split(/\t/);
        my $i = 0;
        while($s[2] > $n[$i]->[2]){
                print $n[$i]->[0] . "\tNA\t" . $n[$i]->[2] . "\n";
                $i++;
        }
        if($i >= 1){
                for($x = 0; $x < $i; $x++){
                        shift(@n);
                }
        }
        print join("\t", @s) . "\n";
}
```

