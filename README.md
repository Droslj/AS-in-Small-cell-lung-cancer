# AS-in-Small-cell-lung-cancer
Analysis of Alternative splicing in Tumor vs. Normal cells for Small cell lung cancer

**Keywords**
Alternative Splicing, Small Cell Lung Cancer, Translational oncology

# Introduction

In this study, I investigated Alternative Splicing in Small Cell Lung Cancer (AS in SCLC). Data I used was taken from the following study [1].  
SCLC is notoriously aggressive form of cancer. Aberrant alternative splicing plays a huge role in cell differentiation and phenotype.
For this analysis, I used RNA STAR in 2-pass mode combined with IsoformSwitchAnalyzeR. IsoformSwitchAnalyzeR focuses on Isoform Switch Identification (ISI), allowing to observe changes in splicingthat alter the functional architecture of the protein (e.g., losing a domain, shifting an open reading frame, or changing a cellular localization signal).

# Workflow

Complete workflow is provided on Figure 1.

<img width="1172" height="1696" alt="image" src="https://github.com/user-attachments/assets/f859df17-5203-40b5-95d9-33f6a30f84bd" />

**Figure 1: Complete Processing Workflow**

Workflow steps are described in the following sections

## Reads preprocessing

Initial QC shows negligible adapter content, but for an assay on alternative splicing, requirements for sequence precision are much higher than they would be for a standard gene expression or variant-calling workflow.
Because of that, I do not rely on the aligner's soft-clipping. Feed the trimmed version into your splice-aware aligner to ensure your junction counts are as accurate as possible.

[MultiQC pre](https://droslj.github.io/AS-in_Small-cell-lung-cancer/QC/MultiQC_pre_trim.html)

**Initial QC**

[MultiQC post](https://droslj.github.io/AS-in_Small-cell-lung-cancer/QC/MultiQC_post_trim.html)

**Post trimming QC**

## 2-Pass STAR mode for Novel Junctions discovery

RNA STAR supports 2-pass mode for improved junction discovery. In the first pass, STAR aligns the reads and discovers splice junctions de novo. In the second pass, it uses those discovered junctions to re-align all reads, which significantly improves the mapping of reads that cross novel or rare junctions.


Raw FastQ Reads⟶STAR 2-Pass (Aligns Splice Junctions)⟶Salmon/Kallisto (Quantifies Full Isoforms)
Then, in your R environment with IsoformSwitchAnalyzeR:
1.	Part 1 (isoformSwitchAnalysisPart1()): Imports the abundance matrix, runs the differential mapping statistics, identifies switches, and extracts the FASTA sequences of the switching transcripts.
2.	External Webservers/Tools: You feed those sequences to Pfam (for domains) and CPAT/CPC2 (for coding potential).
3.	Part 2 (isoformSwitchAnalysisPart2()): Integrates those results to map the exact exon-skipping events to their downstream proteomic damage and spits out publication-ready visual models of the transcripts.
Sources
1.	Selecting differential splicing methods: Practical considerations for short-read RNA sequencing - PMC
2.	Enabling Analysis of Isoform Switches with Functional Consequences and the Underlying Alternative Splicing
3.	IsoformSwitchAnalyzeR
4.	IsoformSwitchAnalyzeR: vignettes/IsoformSwitchAnalyzeR.Rmd - rdrr.io
________________________________________
User prompt: This is part of settings of RNA star
Response: Your current settings look great and are mostly dialed in perfectly for a standard RNA-seq run. However, because you are explicitly feeding this into IsoformSwitchAnalyzeR, there is one parameter in this screenshot you should change, and one hidden parameter you should verify.
Here is exactly what to modify:
1. The Critical Change: Per gene/transcript output
●	What you have selected: Per gene read counts (GeneCounts)
●	What you need to change it to: Look for an option that outputs TranscriptomeSAM (often phrased as Transcriptome coordinates or Per transcript alignments).
●	Why: As we discussed, IsoformSwitchAnalyzeR operates on full-length transcripts and isoforms, not just total gene-level counts. To get those isoform proportions, you will need downstream quantifiers like Salmon or RSEM. If you want to run Salmon in its alignment-based mode, it requires the BAM file to be mapped directly to transcriptome coordinates rather than standard genomic coordinates.
Note: If you instead plan to run Salmon in its standard, blindingly fast "Quasi-mapping/Selective Alignment" mode directly on your trimmed FastQ files, you don't technically need STAR to output transcript-level coordinates (or gene counts) at all. However, generating the TranscriptomeSAM output is a standard safety net.
2. The Great Choice: Use 2-pass mapping
●	What you have selected: Yes, perform single-sample 2-pass mapping of all reads
●	Verdict: Keep this exactly as it is. This is the --twopassMode Basic setting which will dynamically look for novel junctions in your SCLC data and re-align reads around them to ensure high-precision junction boundaries.
3. The Hidden Check: Length of the genomic sequence around annotated junctions
●	What you have typed: 100
●	Double check: STAR's ideal value here is ReadLength−1 (the tool flag --sjdbOverhang). Earlier, your MultiQC plots showed your read length maxing out at 150 bp.
●	Action: If your reads are indeed 150 bp, change this 100 to 149. This optimizes how STAR builds its internal splice junction splice database specifically tailored to your sequence length.
Everything else in your screenshot—including leaving the default BAM tags selected—is completely fine for your downstream workflow!
________________________________________
User prompt: OK, I'll give it a try with these settings
Response: Awesome! Give it a spin with those tweaks.
Just to recap your game plan before you hit Run Tool:
1.	Change Per gene/transcript output to include TranscriptomeSAM (if you plan to use alignment-based transcript quantification downstream).
2.	Change the Length of the genomic sequence parameter from 100 to 149 (since your reads go up to 150 bp).
3.	Keep that 2-pass mapping enabled to capture all those tumor-specific novel splice junctions.
Once STAR finishes running, you'll have a highly accurate alignment file ready for your isoform quantification. If you run into any errors or need help setting up Salmon/Kallisto on those outputs next, just let me know! Good luck with the run!
________________________________________
User prompt: I got the star to finish
Response: That is awesome news! STAR can be quite a resource hog, so getting a successful run finished on UseGalaxy is a great milestone.
Now that the alignment is complete, you are sitting on the precise splice-junction data needed for your alternative splicing analysis. Your next step depends slightly on how you want to handle the isoform quantification before moving into R with IsoformSwitchAnalyzeR.
What are you planning to use for the transcript quantification step—are you going to run Salmon on the TranscriptomeSAM BAM file, or did you decide to use another tool?
________________________________________
User prompt: Star finished the first step
Response: That’s fantastic! Having the first major hurdle cleared is a great feeling, especially since STAR handles the heavy lifting of mapping those complex splice junctions.
Now that your alignment is done, your workflow is moving into the quantification phase. Since you're targeting IsoformSwitchAnalyzeR, you have two main pathways to get the transcript-level abundances it needs:
●	Option A (The Standard Way): Use Salmon in Quasi-mapping mode directly on your trimmed FastQ collections, using a reference transcriptome (like GENCODE). This is extremely fast and highly accurate for isoform-level quantification.
●	Option B (The Alignment Way): Use Salmon (or RSEM) in Alignment mode, feeding it the TranscriptomeSAM BAM file that STAR just generated.
Which route are you planning to take for the quantification, or are you ready to jump into setting up Salmon in Galaxy?
________________________________________
User prompt: How about star second step?
Response: Because you selected "Yes, perform single-sample 2-pass mapping of all reads" in your settings, STAR has actually already completed both the first and second passes internally!
The "first step" you saw finish in Galaxy was the entire single-sample job. Here is what STAR just did automatically under the hood within that single run:
1.	Pass 1: It mapped your reads to the genome using your Gencode GTF file to discover any novel, unannotated splice junctions specific to your small cell lung cancer samples.
2.	Internal Bridge: It extracted those newly discovered junctions and dynamically injected them into its internal genomic index.
3.	Pass 2: It automatically re-aligned all your reads from scratch using this updated index, which significantly improved the alignment precision for reads crossing those novel splice boundaries.
What this means for you right now
You do not need to set up or launch a separate "second pass" job in Galaxy. Your output collection (the BAM files) is already fully optimized, multi-pass aligned, and ready to go.
Next Step
Since you configured the tool to output the transcript-level metadata, you are ready to move directly to transcript quantification. You can now open up the Salmon tool in Galaxy to calculate the transcript abundances (TPM/counts) needed for IsoformSwitchAnalyzeR.
Would you like help setting up the parameters for Salmon next?
Sources
1.	Proteomics / Neoantigen 1a: Fusion-Database-Generation / Hands-on - Galaxy Training!
2.	Mapping RNA-seq Reads with STAR - PMC - NIH
________________________________________
User prompt: Ok, which output of star should I use
Response: Now that STAR has wrapped up its 2-pass alignment, it will have generated a few different datasets in your Galaxy history. Because you are heading toward IsoformSwitchAnalyzeR via transcript quantification, here is exactly which output you need to use:
1. If you are using Salmon in "Alignment-Based" Mode
If you plan to run Salmon by feeding it an alignment file, look for the output labeled:
●	RNA STAR: transcript alignments (This corresponds to the TranscriptomeSAM file).
●	Why: This file contains reads mapped directly to transcript coordinates (isoforms) rather than genomic coordinates, which is exactly what alignment-based transcript quantifiers require.
2. If you are using Salmon in "Quasi-Mapping" Mode (Recommended)
If you choose this route, you actually don't use the STAR BAM outputs for quantification.
●	Why: You will feed your original trimmed FastQ collections directly into Salmon, alongside a reference transcriptome (like a GENCODE FASTA file).
●	What you use the STAR output for: In this scenario, you keep the standard RNA STAR: mapped reads (the genomic BAM file) as a backup for visualization in a genome browser (like IGV) to visually inspect your SCLC splicing events later.
Summary Checklist for your Next Step
Look at your Galaxy history and identify these files:

If your next tool is...	Use this STAR Output / Galaxy Dataset
Salmon (Alignment mode)	RNA STAR: transcript alignments (BAM)
Salmon (Quasi-mapping mode)	Skip STAR BAMs for now; use your trimmed FastQ datasets + reference transcriptome.
Visualizing junctions in IGV	RNA STAR: mapped reads (Genomic BAM)
Which mode are you leaning toward for your Salmon quantification run?
________________________________________
User prompt: I got these files
Response: First of all, take a look at that log file—93.00% uniquely mapped reads is an incredibly high-quality alignment result! For total RNA-seq data, anything above 80% is great, so 93% means your reads are cleanly mapping right where they should without a massive amount of multi-mapping noise (which is under 5%).
Looking at your updated Galaxy history panel on the right, you have exactly the files needed to move forward. Here is what they are and how you will use them:
1. 240: RNA STAR on collection 127: transcriptome-mapped.bam
●	What it is: This is your TranscriptomeSAM file. It contains your reads mapped directly to the coordinates of the known transcripts (isoforms) from your Gencode GTF, rather than coordinates on chromosomes.
●	How to use it: This is the exact file you want to feed into Salmon if you choose to run Salmon in its Alignment-Based Mode. It is perfectly tailored for transcript-level abundance estimation.
2. 239: RNA STAR on collection 127: mapped.bam
●	What it is: This is your standard genomic BAM file (reads mapped to chromosomes/contigs).
●	How to use it: You won't use this for Salmon transcript quantification. Instead, keep this file for downstream genomic steps—like looking at coverage in a genome browser (IGV) or feeding it to an exon-level tool if you ever decide to look at local splicing junctions visually.
3. 238: RNA STAR on collection 127: splice junctions.bed
●	What it is: A clean tabular list of all high-confidence splice junctions STAR identified across your samples, including both annotated ones and any novel de novo junctions it uncovered during the 2-pass run.
Your Next Step: Setting up Salmon
Since you have dataset 240 (transcriptome-mapped.bam), you are perfectly poised to use Salmon. Search for Salmon in the Galaxy tool panel.
When configuring it:
1.	Set the Type of execution or Quantification mode to Alignment Mode (instead of Quasi-mapping).
2.	For the Alignment File, select your collection 240 (RNA STAR on collection 127: transcriptome-mapped.bam).
3.	For the Reference Transcriptome, provide the exact same reference FASTA file (the transcript sequences matching your Gencode V38 annotation) that you used to construct or guide the alignment.
Are you ready to pull up the Salmon tool window, or would you like to double-check the reference transcriptome file first?
________________________________________
User prompt: The collection 240 is unsorted
Response: That is completely normal! In fact, it is exactly what Salmon requires.
When STAR outputs transcript-mapped reads (TranscriptomeSAM), it outputs them in the original order of the reads in your FastQ files (queryname/name-sorted) rather than sorted by genomic coordinates.
Here is why this is actually perfect:
●	Salmon Needs It Unsorted: Pair-end transcript quantifiers like Salmon require the forward and reverse reads to appear sequentially right next to each other in the file to properly evaluate the fragment length distribution. If you coordinate-sort the BAM file, the pairs get separated across different genomic locations, and Salmon will throw an error or fail to pair them up.
●	No Extra Processing Required: You don't need to run any sort tools or re-order anything.
You can feed collection 240 directly into Salmon exactly as it is! Do you have your reference transcriptome FASTA file ready to go for the Salmon step?
________________________________________
User prompt: I have this
Response: There is one critical issue with your current configuration that will cause Salmon to fail or give completely incorrect results.
Take a look at your Transcript file selection:
●	What you have selected: 208: GRCm39.primary_assembly.genome.fa.gz
The Problem
This file is the genomic FASTA file (the full DNA chromosomes of the mouse genome). Salmon cannot use a genome file here. Because you are quantifying transcripts, Salmon explicitly needs the transcriptome FASTA file (the cDNA sequences of all exons/transcripts spliced together).
If you use the genome file, Salmon won't be able to match the transcript IDs from your STAR BAM file to their actual nucleotide sequences, and it won't be able to calculate transcript lengths or biases properly.
The Fix
You need to change the Transcript file parameter to a transcript-specific reference file.
●	Look in your history or shared data for a file that contains rna, cdna, or transcripts in the name (for example, gencode.vM38.transcripts.fa.gz).
Do you have a transcript-specific FASTA file uploaded to your history that matches the gencode.vM38 annotation you used during the STAR step?
________________________________________
User prompt: No, I used genomic fasta file for star
Response: Ah, I see! If you used the genomic FASTA file during the STAR alignment step, that actually changes things.
Since STAR was only given the genomic sequences, it relied entirely on your GTF file (gencode.vM38.annotation.gtf.gz) to figure out where the exons and transcripts are. It then extracted those transcript coordinates to build your 240: transcriptome-mapped.bam file.
Because transcriptome-mapped.bam contains alignments mapped to transcript names (like ENSMUST...), Salmon still absolutely requires a nucleotide FASTA file of those exact transcript sequences to perform its bias corrections, index matching, and abundance math. You cannot pass Salmon the full genome sequence (GRCm39.primary_assembly.genome.fa.gz) because it won't find the matching transcript IDs inside it.
To move forward right now, you have two options:
Option A: Find and use the matching Transcriptome FASTA (Recommended)
Since you used Gencode vM38 for your GTF file, you just need the matching transcriptome FASTA file from Gencode.
●	Look in your history or UseGalaxy's shared data for a file named something like gencode.vM38.transcripts.fa.gz or gencode.vM38.pc_transcripts.fa.gz.
●	Select that file under Transcript file in Salmon.
Option B: Extract the transcripts from your Genome using your GTF
If UseGalaxy doesn't have the pre-computed Gencode vM38 transcriptome FASTA handy, you can generate it yourself directly in Galaxy using the two files you already have.
1.	Search for a tool named gffread or Extract transcript sequences in the left panel.
2.	Feed it your genomic file (Dataset 208) and your GTF file (Dataset 207).
3.	It will output a beautifully clean transcriptome FASTA file containing every spliced transcript sequence, which you can then plug straight into Salmon.
Do you see a Gencode transcript FASTA file anywhere in your available histories, or should we run gffread to make one real quick?

References:
 [1] PRJNA1464579 
