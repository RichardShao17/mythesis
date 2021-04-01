# Naphthenic Acid thesis
## Author: Richard Shao

This is where I will document all the code I used from the start to the end of my fourth year thesis in Biochemistry

After the nanopore sequencing run we are left with 3 files: fast5_fail, fast5_pass and fast5_skip. The below code is capable of basecalling all three of those files (recursive). You will need to which specify which nanopore flowcell and kit was used. 
```
guppy_basecaller --input_path ./ --save_path ./guppyoutputRS --recursive --flowcell FLO-MIN111 --kit SQK-LSK109 -x "cuda:0"
```

After Guppy basecalling, you will have a file named sequencing_summary.txt. With that, you can use Nanoplot to generate quality plots.
```
NanoPlot --summary ./guppyoutputRS/sequencing_summary.txt --loglength -o nanoplot
```
I used [NanoFilt](https://github.com/wdecoster/nanofilt) to remove any reads below an average q-score of 7 and length below 2000 bases. I also removed the first 50 bases from each read to ensure that no poor quality bases are included. There is absolutely no reason why the output file should be named "trimmed_plasmid_reads." This work was adapted from the Dan Giguere ([the protocol is here](https://github.com/dgiguer/long-read-plasmid-assembly)) and this is what he used to name his file.
```
NanoFilt -l 2000 -q 7 --headcrop 50 > ./trimmed_plasmid_reads.fastq
```
After filtering the reads, we are ready for assemblies. We do assemblies through [Flye](https://github.com/fenderglass/Flye).
```
nohup flye --nano-raw trimmed_plasmid_reads.fastq -t 10 --meta -o flyeoutputRS
```
After that, we want to polish our assemblies using the long read polisher [Medaka](https://github.com/nanoporetech/medaka). It takes both the trimmed_plasmid_reads.fastq file from nanofilt as well as the assembly.fasta file following assembly with Flye. It is recommended for best results to specify which model you were using.
```
medaka_consensus -i trimmed_plasmid_reads.fastq -d /flyeoutputRS/assembly.fasta -o medakaoutput RS -t 10 -m -r103_min_high_g360
```
**Now comes time to validate our circular genomes**

First step is to pull out the specific fasta file for the circularized contig (ex. contig170). The numbers 6610, 8308 are found through another awk code which escapes me right now (ask Dan). But once we find which lines we are after, simply apply the code below and it will pull out only that sequence!
```
awk 'NR > 6610 && NR < 8308 {print $0}' assembly.fasta > contig_170.fasta
```
Then we use minimap2 to map our trimmed reads to the contig 170 fasta.
```
minimap2 -ax map-ont contig_170.fasta trimmed_reads.fa > minimap2contig170.sam
```
Now we use Gerenuq to do some filtering! 

```
gerenuq -i minimap2contig170.sam -o gfilteredminimap2contig170.sam -l 2000 -m 0.9
```
For the next step (minipolish) we need the sam file from gerenuq to become a fasta file.
```
samtools fasta gfilteredminimap2contig170.sam > gfilteredminimap2contig.fa
```
Next we use minipolish. Tip to future self: remember how iffy the gfa file is and how after you pull it out, you have to make it into a gfa2 file (gfakluge) first and then in front of every edge_170 you have to put the letter l (ex. edge_170l). You also have to delete that number on the top of the file indicating length of sequence. And then somehow it just works.

```
./gfatools view -l edge_170 -r assembly.gfa > contig_170.gfa
```
```
minipolish --skip_initial -t 8 gfilteredminimap2contig.fa g2contig170.gfa > minipolishedcontig170.gfa
```

Next we use medaka to get a consensus sequence. 

```
medaka_consensus -i gfilteredminimap2contig.fa -d contig_170.fa -o medakacontig170
```
Now we use minimap2 again!!!

```
minimap2 -a ./medakacontig170/consensus.fasta gfilteredminimap2contig.fa > Gfiltmedakaminimap2.sam
```
even though I mapped the "filtered reads" against the polished genome, there will be lots of secondary alignments produced that need to be filtered by gerenuq - so you need to filter the SAM file again by gerenuq before converting to bam and calculating the coverage

```
gerenuq -i Gfiltmedakaminimap2.sam -o test.sam -l 2000 -m 0.9
```
Now we need to convert to bam for mosdepth.

```
samtools view -bS test.sam > test.bam

```
Then we need to sort the bam file
```
samtools sort test.bam -o test_sorted.bam
```
Now we are ready to use mosdepth! (remember there are two dashes in --by 1000)
```
mosdepth --by 1000 mosdepthoutput testsorted.bam
```
After mosdepth, you are going to want to unzip the mosdepthoutput.regions.bed file

``` 
gunzip mosdepthoutput.regions.bed.gz
```
and then you can scp that file over to your local machine and use R to make a line graph. 


