# mythesis

So this is where I will document all the code I used from the start to the end of my fourth year thesis in Biochemistry. Wahoo.

First things first, after the nanopore sequencing run we are left with 3 files: fast5_fail, fast5_pass and fast5_skip. The below code is capable of basecalling all three of those files (recursive). You will need to which specify which nanopore flowcell and kit was used. 
```
guppy_basecaller --input_path ./ --save_path ./guppyoutputRS --recursive --flowcell FLO-MIN111 --kit SQK-LSK109 -x "cuda:0"
```

After Guppy basecalling, you will have a file named sequencing_summary.txt. With that, you can use Nanoplot to generate quality plots.
```
NanoPlot --summary ./guppyoutputRS/sequencing_summary.txt --loglength -o nanoplot
```
I used [NanoFilt](https://github.com/wdecoster/nanofilt) to remove any reads below an average q-score of 7 and length below 2000 bases. I also removed the first 50 bases from each read to ensure that no poor quality bases are included. There is absolutely no reason why the output file should be named "trimmed_plasmid_reads." This work was adapted from the great Dan Giguere ([his protocal](https://github.com/dgiguer/long-read-plasmid-assembly)) and this is what he used to name his file.
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

The end :)
