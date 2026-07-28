# AS-in-Small-cell-lung-cancer
Analysis of Alternative splicing in Tumor vs. Normal cells for Small cell lung cancer<br>

**Keywords**
Alternative Splicing, Small Cell Lung Cancer, Translational oncology<br>

# Introduction

In this study, I investigated Alternative Splicing in Small Cell Lung Cancer (AS in SCLC). Data I used was taken from the following study [1].  <br>
SCLC is notoriously aggressive form of cancer. Aberrant alternative splicing plays a huge role in cell differentiation and phenotype.<br>

# Analysis method

For this analysis, I used RNA STAR (in 2-pass mode for enhanced splice junction detection) coupled with IsoformSwitchAnalyzeR. IsoformSwitchAnalyzeR focuses on Isoform Switch Identification (ISI), allowing to observe changes in alternative splicing.<br>

# Workflow

Complete workflow is provided on Figure 1.

![Processing flow](Images/Complete_WF.png)

**Figure 1: Complete Processing Workflow**

Workflow steps are described in the following sections<br>

## Reads preprocessing

Initial QC shows negligible adapter content, but for an assay on alternative splicing, requirements for sequence precision are much higher than they would be for a standard gene expression or variant calling.
Because of that, I did not rely on the aligner's soft-clipping, I used fastp for trimming adapter content.<br>

**Initial QC**: [MultiQC raw reads](https://droslj.github.io/AS-in-Small-cell-lung-cancer/MultiQC_pre_trim.html)

**Post trimming QC**: [MultiQC post trimming](https://droslj.github.io/AS-in-Small-cell-lung-cancer/MultiQC_post_trim.html)

## 2-Pass STAR mode for Novel Junctions discovery

RNA STAR supports 2-pass mode for improved junction discovery. In the first pass, STAR aligns the reads and discovers splice junctions de novo. In the second pass, it uses those discovered junctions for more accurate splice junction detection.<br>

## Transcript abundance 

Salmon was used to quantify transcript abundances per sample. Salmon used output from RNA STAR as input (unsorted bam files).<br>

## DESeq2 analysis

In an Isoform switching assay, performing DESeq2 analysis in paralel offers several benefits. DESeq2 analysis was performed using transcript quantifications from Salmon. PCA plot, shown on Figure 2, exposed two major, distinct quality control issues that needed to be resolved before trusting any downstream differential expression statistics from DESeq2:<br>

Issue #1:  Severe Outlier (SRR38500642)<br>
Sample SRR38500642 (Treated) is pulled completely away from everything else along the Y-axis (PC2 explains 23% of the variance).<br>

Issue #2: A Likely Sample Swap / Mix-up (SRR38500645)<br>
Sample SRR38500645 is labeled as Normal, but it groups far closer to the Treated samples (SRR38500641 and SRR38500643) than it does to other normal samples.<br>

![PCA plot run# 1](/Images/DESeq2_PC_plot_run1.png)

**Figure 2: PCA plot (DESeq2 run1)**

Closer look on provided data revealed following:
 - Issue 1. The Outlier sample (SRR38500642) has a Sequencing Depth Issue with roughly 30% less sequencing data than its counterparts <br>
 - Issue 2. The Misclustered Control (SRR38500645) is due to Unmatched Biological Conditions.<br>

According to the abstract, the authors performed multi-omics profiling on treatment-naïve human SCLC tumors along with paired adjacent tissues (NAT, n=12)—which they modeled here in mice (Mus musculus):<br>
- Cancer samples: C4-1, C3-1 and C1-1<br>
- Adjacent tissue: T2-2, T2-1, and T1-1.<br>

I proceeded by droping the low-depth outlier sample SRR38500642 from the count matrix entirely to stabilize the variance and repeated the DESeq2 analysis.<br> 

On second run, the PCA plot (Figure 3) again revealed irregularities.<br>

![PCA plot run #2](/Images/DESeq2_PC_plot_run2.png)

**Figure 3: PCA plot (DESeq2 run2)**

SRR38500645 (Tumor / C3-1) sample is not clustering on the far right with other tumor replicates (SRR38500644 and SRR38500646), it is pulled heavily to the left along PC1, sitting much closer to the healthy NormalX samples.
In oncology datasets, this intermediate positioning typically points to one of two biological or technical realities:<br>
 1. High Normal Tissue Contamination (Low Tumor Purity)<br>
 2. A Partial Sample Mix-up or Subtype Difference<br>

By keeping this sample, software in subsequent steps would assume that tumors are naturally highly variable, which will blow out the dispersion estimates and dramatically shrink your final list of statistically significant differentially expressed genes (DEGs), so I eliminated this sample from further analysis and continued to Isoform analysis with only four samples.<br>

Repeated DESeq2 analysis revealed that samples are now matched (Figure 4) and it was OK to proceed to next step.<br>

![PCA plot run #3](/Images/DESeq2_PC_plot_run3.png)

**Figure 4: PCA plot (DESeq2 run3)**

## Analysis of Isoform switcing (IsoformSwitchAnalyzeR)

Analysis of Isoform switching was performed in following steps:
 - Step 1: Import data into R using the appropriate import function to build initial design and transcript structure database<br>
 - Step 2: Run Part I of the IsoformSwitchAnalyzeR pipeline. This step filters out lowly expressed transcripts, identifies isoform switches, predicts the Open Reading Frames (ORFs), and outputs amino acid sequences<br>
 - Step 3: Prediction of protein domains with PfamScan using the amino acid FASTA file generated in Step 1 and required files from PFAM scan database<br>
 - Step 4: PartRun part II of the IsoformSwitchAnalyzeR pipeline - Full Analysis (R). This step integrates results from previous steps to map the exact exon-altering events, and correctly identifies which specific protein domains were gained or lost during isoform switches.  <br>

# Results of Isoform switching

The following sections contain description of some of the more representative gene isoform switching events.<br>


## Hip1r

Description
Hip1r is a critical component of clathrin-mediated endocytosis and vesicle trafficking. It physically links the plasma membrane, clathrin coats, and the actin cytoskeleton using different domains.

![Hip1r](/Images/Hip1r.png)

**Figure 5: Hip1r Isoforms**

Gene Expression Profile
Total Hip1r expression exhibits an unchanged / flat trend in the tumor compared to normal tissue.


Isoforms detected:
Normal condition: ENSMUST00000032549.13 (Full-length ANTH domain-containing transcript)
Tumor condition: ENSMUST00000115383.8 (Truncated / ANTH domain-less isoform)

Event type:
PFAM Domain Loss (ANTH domain truncation) / Alternative Transcription Event


Explanation
Hip1r is a vital clathrin-binding partner required for clathrin-mediated endocytosis (CME) and the internalization/degradation of cell-surface growth factor receptors. By shifting expression to an isoform lacking the lipid-binding ANTH domain (PF07659), the tumor cell impairs CME and endosomal recycling kinetics, preventing proper receptor degradation and sustaining growth-factor signaling cascades.


## Coro2b

Description
Coro2b is an actin-binding protein that regulates cell motility, lamellipodia formation, and cytoskeletal remodeling via autoinhibitory WD40 repeat interactions.

![Coro2b](/Images/Coro2b.png)

**Figure 6: Coro2b Isoforms**

Gene Expression Profile
Total Coro2b expression exhibits an unchanged / minimal change profile in the tumor compared to normal tissue.


Isoforms detected:
Normal condition: ENSMUST00000030588.13 (Full-length autoinhibited isoform)
Tumor condition: ENSMUST00000109591.9 (WD40 domain-truncated isoform)

Event type:
PFAM Domain Loss / Truncation (WD40 repeat loss)

Explanation
Coro2b autoinhibition depends on intact WD40 repeat domains (PF00400 / PF12894). The tumor isoform truncates these regulatory repeats, releasing autoinhibition and generating a constitutively active actin-binding protein that accelerates cytoskeletal reorganization, cell motility, and invasive growth.


## Txlng

Description
Txlng (Taxilin Gamma) is involved in intracellular vesicle trafficking, syntaxin binding, and nucleocytoplasmic transport pathways.

![Txlng](/Images/Txlng.png)

**Figure 7: Txlng Isoforms**

Gene Expression Profile
Total Txlng gene expression is non-significant / masked in tumor versus normal tissue.

Isoforms detected:
Normal condition: ENSMUST00000032997.14 (Long $3'$ UTR / NMD-sensitive isoform)
Tumor condition: ENSMUST00000168322.8 (Short $3'$ UTR / NMD-insensitive isoform)

Event type:
Alternative Transcription Termination Site (ATTS) / NMD Escape

Explanation
In normal cells, Txlng transcripts carry a long 3' UTR containing premature termination codons (PTCs) that mark the mRNA for nonsense-mediated decay (NMD). In the tumor, an upstream termination shift (ATTS) shortens the 3' UTR, allowing the transcript to escape NMD surveillance and produce functional taxilin protein.


 

## Ero1a

Description
Ero1a is an essential endoplasmic reticulum (ER) oxidoreductase that catalyzes disulfide bond formation during protein folding and oxidative stress response.

![Ero1a](/Images/Ero1a.png)

**Figure 8: Ero1a Isoforms**

Gene Expression Profile
Total Ero1a gene expression exhibits an apparent downregulation in tumor tissue.

Isoforms detected:
Normal condition: ENSMUST00000022838.13 (Functional protein-coding transcript)
Tumor condition: ENSMUST00000111818.8 (PTC-containing / NMD-sensitive isoform)

Event type:
Exon Inclusion / NMD Trapping


Explanation
By including an alternative exon containing a premature stop codon, the tumor shunts the Ero1a transcript pool into NMD-mediated degradation (PF04137 domain disruption). This selective trapping lowers functional Ero1a protein levels, dampening excessive ER stress signaling and preventing stress-induced apoptosis in the tumor microenvironment.



 ## Uqcc5 
 
Description
Uqcc5 is a nuclear-encoded assembly factor required for the proper biogenesis and stabilization of Mitochondrial Complex III (cytochrome $bc_1$ complex).

![Uqcc5](/Images/Uqcc5.png)

**Figure 9: Uqcc5**

Gene Expression Profile
Total Uqcc5 expression exhibits an unchanged (flat) profile in tumor compared to normal tissue.

Isoforms detected:
Normal condition: ENSMUST00000203261.3 (Full-length UPF0640 domain isoform)
Tumor condition: ENSMUST00000090205.5 ($C$-terminal truncated isoform)

Event type:
PFAM Domain Truncation (C-terminal UPF0640 truncation)

Explanation
The full-length C-terminal UPF0640 domain (PF03729) is required to stabilize nascent cytochrome b during Complex III assembly. Partial truncation of this domain alters Complex III assembly kinetics, allowing tumor cells to tune oxidative phosphorylation and metabolic flux under hypoxic conditions.

## Rras

Description
Rras is a small GTPase of the Ras superfamily that regulates cell adhesion, integrin activation, and blood vessel architecture.

![Rras](/Images/Rras.png)

**Figure 10: Rras**

Gene Expression Profile
Total Rras expression exhibits a completely flat / unchanged trend in the tumor compared to normal tissue.


Isoforms detected:
Normal condition: ENSMUST00000024843.13 (Canonical GTPase isoform)
Tumor condition: ENSMUST00000165832.8 (Remodeled GTPase domain isoform)

Event type:
Quantitative Shift / GTPase Domain Remodeling

Explanation
The switch remodels the core Ras GTPase domain (PF00071), modifying GTP/GDP binding dynamics and effector interactions. This structural modification alters cell adhesion and integrin-mediated signal transduction without requiring changes in total Ras transcriptional output.

## Fxyd

Description
Fxyd3 (Mat8) is an auxiliary transmembrane subunit that regulates the affinity and turnover kinetics of the Na+/K-ATPase ion pump in epithelial membranes.

![Fxyd3](/Images/Fxyd3.png)

**Figure 11: Fxyd3**

Gene Expression Profile
Total Fxyd3 expression exhibits a modest / non-significant change in the tumor compared to normal tissue.

Isoforms detected:
Normal condition: ENSMUST00000020121.12 (Canonical promoter transcript)
Tumor condition: ENSMUST00000109088.8 ($5'$ N-terminal shifted transcript)

Event type:
Alternative Transcription Start Site (ATSS) / 5' UTR Shift

Explanation
The switch to an alternative upstream promoter alters the 5' UTR and N-terminal translation initiation context while retaining the main FXYD domain (PF00388). This enhances transcript stability and translational efficiency under stress, enabling tumor cells to tune ion homeostasis, maintain electrochemical gradients, and survive membrane hyperpolarization.

## Hsd17b14

Description
Hsd17b14 encodes a short-chain dehydrogenase/reductase (SDR) family enzyme responsible for the $NAD(P)-dependent oxidoreduction of steroid hormones and lipid substrates.

![Hsd17b14](/Images/Hsdb17b14.png)

**Figure 12: Hsd17b14**

Gene Expression Profile
Total Hsd17b14 expression remains unchanged in the tumor compared to normal tissue.

Isoforms detected:
Normal condition: ENSMUST00000021678.13 (Catalytically active SDR isoform)
Tumor condition: ENSMUST00000112445.8 (SDR domain-deleted isoform)

Event type:
PFAM Domain Loss (SDR catalytic domain loss)

Explanation
The tumor switches away from the functional enzyme isoform to a variant that completely lacks the catalytic short-chain dehydrogenase/reductase domain (PF00106). This functional inactivation decouples steroid and lipid turnover without altering overall gene expression, rewiring intracellular lipid metabolism to support rapid cellular proliferation.

## Commd9

Description
Commd9 is a component of the COMMD protein family involved in endosomal protein sorting, copper homeostasis, and negative regulation of NF-kB transcriptional activity.

![Commd9](/Images/Commd9.png)

**Figure 12: Commd9**

Gene Expression Profile
Total Commd9 expression exhibits an unchanged trend in the tumor compared to normal tissue.

Isoforms detected:
Normal condition: ENSMUST00000021381.12 (Intact COMM domain isoform)
Tumor condition: ENSMUST00000122396.8 (COMM domain-truncated isoform)

Event type:
PFAM Domain Truncation (COMM domain deletion)

Explanation
The COMM domain (PF04433) is essential for mediating protein-protein interactions within endosomal recycling complexes and inhibiting NF-kB nuclear translocation. Truncation of this domain in the tumor variant impairs endosomal sorting pathways and relieves NF-kB inhibition, favoring pro-survival inflammatory signaling.

# Summary of Isoform analysis

Isoform switching of top 10 genes detected in this assay is summarized in the Table 1:

![Summary of Isoform switching events](/Images/Summary_switching.png)

**Table 1: Summary of Isoform analysis**

**References**
 [1] PRJNA1464579
