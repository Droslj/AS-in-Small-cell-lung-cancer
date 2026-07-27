# AS-in-Small-cell-lung-cancer
Analysis of Alternative splicing in Tumor vs. Normal cells for Small cell lung cancer<br>

**Keywords**
Alternative Splicing, Small Cell Lung Cancer, Translational oncology<br>

# Introduction

In this study, I investigated Alternative Splicing in Small Cell Lung Cancer (AS in SCLC). Data I used was taken from the following study [1].  <br>
SCLC is notoriously aggressive form of cancer. Aberrant alternative splicing plays a huge role in cell differentiation and phenotype.<br>

# Analysis method

For this analysis, I used RNA STAR in 2-pass mode combined with IsoformSwitchAnalyzeR. IsoformSwitchAnalyzeR focuses on Isoform Switch Identification (ISI), allowing to observe changes in alternative splicing.<br>

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

Salmon was used to obtain transcript abundances per sample. Salmon used output from RNA STAR as input (unsorted bam files).<br>

## DESeq2 analysis

In an Isoform switching assay, DESeq2 analysis is interesting because it enables comparison of DE genes with DE isoforms. DESeq2 analysis was performed using transcript quantifications from Salmon. PCA plot, shown on Figure 2, exposed two major, distinct quality control issues that needed to be resolved before trusting any downstream differential expression statistics from DESeq2:<br>

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

Hip1r is a critical component of clathrin-mediated endocytosis and vesicle trafficking. It physically links the plasma membrane, clathrin coats, and the actin cytoskeleton using different domains.<br>

Gene Expression Profile: Total Hip1r expression exhibits a downward trend in the tumor compared to normal tissue.<br>

The Isoform Switch:
○	Normal: Dominated by the stable, full-length ENSMUST00000020165.13 transcript (~61% usage), which codes for the fully functional endocytic accessory protein.<br>
○	Tumor: Switches dominance to ENSMUST00000140224.2 (~54% usage). This variant is structurally truncated at the 3' end and is marked as NMD Sensitive (targeted for nonsense-mediated decay).<br>

![Hip1r](/Images/Hip1r.png)

**Figure 5: Hip1r Isoforms**

Hip1r is a vital clathrin-binding partner required for clathrin-mediated endocytosis (CME) and the internalization/degradation of activated receptor tyrosine kinases (RTKs) like EGFR. By diverting the transcript pool into an NMD-sensitive "trap" transcript, the tumor effectively depletes functional HIP1R protein. This dismantles CME, preventing the internalization and degradation of cell-surface growth receptors, thereby driving sustained, runaway extracellular growth-factor signaling.<br>

## Coro2b

Coro2b is a member of Coronin family of proteins, which are crucial regulators of actin cytoskeleton, cell motility, and focal adhesions. <br>

Gene Expression Profile: Unchanged overall, with overlapping error bars indicating identical transcription levels between conditions.<br>

The Isoform Switch:
○	Normal: Strictly utilizes the long, functional ENSMUST00000026208.10 isoform (~81% usage).<br>
○	Tumor: Reallocates splicing to over 60% usage of ENSMUST00000130006.2, a highly truncated, stable 3'-end variant consisting of only two exons.<br>

![Coro2b](/Images/Coro2b.png)

**Figure 6: Coro2b Isoforms**

Coronin 2B is a type-II coronin that regulates F-actin remodeling and focal adhesion turnover via interaction with the Arp2/3 complex and cofilin. The tumor-induced short variant lacks the essential N-terminal β-propeller WD40 domain required for normal F-actin binding. This truncated isoform acts as a dominant-negative competitive buffer, structurally disrupting actin networks to lower rigid tissue anchoring, which promotes cell detachment, plasticity, and mesenchymal migration.<br>

## Txlng

Taxilins (especially gamma-taxilin) play critical roles in intracellular vesicle trafficking and interact with nascent polypeptide-associated complex (NAC) subunits during protein translation at the ribosome.<br>

Gene Expression Profile: Global Txlng transcription is moderately down-regulated in the tumor.<br>

The Isoform Switch<br>
○	Normal: Relies on the unstable, truncated ENSMUST00000141648.2 transcript (~51% usage), which is classified as NMD Sensitive.<br>
○	Tumor: Rescues expression by switching to ENSMUST00000063229.14 (~78% usage), a fully complete, long, NMD Insensitive transcript.<br>

![Txlng](/Images/Txlng.png)

**Figure 7: Txlng Isoforms**
 
Taxilin gamma is a crucial structural protein involved in intracellular vesicle trafficking, centrosomal dynamics, and mitotic spindle coordination. In normal cells, its expression is tightly controlled and buffered via NMD decay. The tumor bypasses this regulatory constraint via an "NMD rescue switch," ensuring a stable, highly translated pool of full-length TXLNG to support accelerated intracellular cargo routing and rapid mitotic cell divisions.

## Ero1a

ERO1A encodes an endoplasmic reticulum oxidoreductase that promotes disulfide bond formation and supports protein folding in the secretory pathway. <br>

Gene Expression Profile: Total expression is highly variable but trends lower in the tumor group.<br>

The Isoform Switch:<br>
○	Normal: Exclusively utilizes the stable, full-length ENSMUST00000022416.7 transcript (100% usage).<br>
○	Tumor: Slashes usage of the full-length transcript down to ~41%, rerouting nearly 60% of the transcript pool into the novel, NMD Sensitive truncated fragment ENSMUST00000211158.2.<br>

![Ero1a](/Images/Ero1a.png)

**Figure 8: Ero1a Isoforms**

Ero1a is a hypoxia-induced enzyme that catalyzes disulfide bond formation in the ER, a process that generates hydrogen peroxide (H2O2) as a byproduct. Because hypoxia in tumors forces transcription of protein-folding machinery, unchecked ERO1A activity would generate toxic, fatal levels of oxidative stress. The tumor solves this by using splicing as a post-transcriptional "release valve," trapping excess transcripts in the NMD decay pathway to keep ER oxidative stress in a safe, sub-lethal signaling zone.<br>


 ## Uqcc5 
 
Uqcc5 (Ubiquinol-Cytochrome C Reductase Complex Assembly Factor 5) is a mitochondrial factor that supports mitochondrial respiratory chain assembly and mitochondrial ribosome binding activity.<br>

Gene Expression Profile: Completely flat and statistically unchanged between normal and tumor tissues.<br>

The Isoform Switch:<br>
○	Normal: Expresses longer transcripts (...64032.10 and ...203261.3) that code for the full-length assembly factor.<br>
○	Tumor: Selectively activates a dormant, heavily truncated, 2-exon 5'-end variant (ENSMUST00000090205.5), taking over 25% of the total transcript pool.<br>

![Uqcc5](/Images/Uqcc5.png)

**Figure 9: Uqcc5**

Uqcc5 is an essential assembly factor for Complex III of the mitochondrial respiratory chain. The tumor-specific activation of this truncated, NMD-insensitive variant produces a non-functional protein fragment. By forcing a quarter of the transcript pool into this non-functional state, the tumor structurally impairs Complex III assembly, shifting metabolic dependency away from standard respiration toward alternative pathways while generating moderate ROS signaling to drive proliferation.<br>

## Rras

Rras (Related RAS Viral Oncogene Homolog) encodes a small GTPase in the Ras family that integrates signals controlling angiogenesis, vascular homeostasis and regeneration, cell adhesion, and neuronal axon guidance. The encoded Ras-related protein R-Ras functions as a molecular switch, and its activity is linked to Ras protein signal transduction together with regulation of the ERK1/2 cascade and PI3K/AKT signaling.<br>

Gene Expression Profile: Significantly down-regulated, dropping from ~8 units in normal tissue to ~2.5 units in the tumor.<br>

The Isoform Switch:<br>
○	Normal: Splices the pool into a nearly 50/50 split between full-length and truncated variants.<br>
○	Tumor: Completely extinguishes the truncated ENSMUST00000210895.2 variant (0% usage), consolidating 100% of remaining transcription into the full-length, NMD Insensitive ENSMUST00000044111.10 transcript.<br>

![Rras](/Images/Rras.png)

**Figure 10: Rras**

R-Ras is a small GTPase that coordinates integrin activation, focal adhesion dynamics, and cell migration. The truncated normal variant lacks the critical membrane-anchoring CAAX box and GTPase domains, serving as a biological sink. Even at a lower global transcription baseline, the tumor completely eliminates this inhibitory buffer, ensuring that 100% of the translated proteins are fully membrane-anchorable and active, facilitating cell motility and matrix invasion.<br>

## Fxyd

Fxyd3 (FXYD Domain Containing Ion Transport Regulator 3) encodes the FXYD domain-containing ion transport regulator 3, a membrane protein that regulates the function of Na+/K+-ATPases and other ion pumps and channels.<br>

Gene Expression Profile: Displays a downward trend in the tumor, marked by high inter-tumor variance.<br>

The Isoform Switch:<br>
○	Normal: Over 51% of transcripts are routed into the short, 2-exon 3'-end fragment ENSMUST00000165465.8.<br>
○	Tumor: Completely eliminates the short fragment (0% usage), shifting over 61% of total transcript dominance to the full-length, multi-exon ENSMUST00000167369.8 transcript.<br>

![Fxyd3](/Images/Fxyd3.png)

**Figure 11: Fxyd3**

FXYD3 is a crucial transmembrane regulator of the Na+/K+-ATPase (sodium-potassium pump). The short normal variant lacks the necessary domains to interact with the pump, acting as a brake. The tumor eliminates this regulatory brake, prioritizing the full-length FXYD3 isoform. This structurally stabilizes the sodium-potassium pump, enabling it to maintain osmotic cell volume and drive pro-survival signaling within the highly acidic, low-pH extracellular niche of the tumor microenvironment.<br>

## Hsd17b14

Hsd17b14 (Hydroxysteroid 17-Beta Dehydrogenase 14) gene encodes hydroxysteroid 17-beta dehydrogenase that participates in steroid metabolism at the C17 position and also acts on other substrates. It belongs to the short-chain dehydrogenase/reductase superfamily and is part of the SDR family, placing it among enzymes with broad metabolic roles. The protein is found in the cytosol and cytoplasm, and its activity is linked to lipid metabolism and steroid metabolic process. As a catalytic enzyme, HSD17B14 is associated with binding and catalytic activity and is implicated in the metabolism of steroids, fatty acids, prostaglandins, xenobiotics, and fucose-related pathways. Its biological context includes L-fucose catabolic process and lipid metabolic process, together with steroid metabolism<br>

Gene Expression Profile: Moderately down-regulated in tumor tissue, dropping from ~4 units to ~2.3 units.<br>

The Isoform Switch:<br>
○	Normal: Relies exclusively on the stable, full-length ENSMUST00000107752.12 transcript (100% usage).<br>
○	Tumor: Slashes usage of the full-length isoform to ~54%, shifting ~46% of the remaining pool into the novel, truncated internal 3-exon fragment ENSMUST00000211029.2.<br>

![Hsd17b14](/Images/Hsdb17b14.png)

**Figure 12: Hsd17b14**

HSD17B14 is a lipid- and steroid-metabolizing enzyme that oxidizes active estrogens/androgens and metabolizes fatty acid substrates to maintain tissue homeostasis. The tumor-induced internal fragment lacks catalytic domain exons but escapes NMD. By forcing nearly half of the transcript pool to translate into a non-functional enzyme, the tumor acts as a metabolic throttle, reducing the clearance of active lipids or growth-promoting steroids to sustain proliferative signaling.<br>

## Commd9

Commd9 (COMM Domain-Containing Protein 9) encodes a COMM domain-containing protein that is predicted to participate in sodium ion transport and act upstream of cholesterol homeostasis.<br>

Gene Expression Profile: Total expression is moderately down-regulated, dropping from ~6 units to ~3.5 units.<br>

The Isoform Switch:<br>
○	Normal: Divides transcripts between full-length and truncated variants (~41% usage for the short ...133576.2).<br>
○	Tumor: Completely abolishes the truncated variant (0% usage), consolidating 100% of remaining expression into the full-length, NMD Insensitive ENSMUST00000028584.8 transcript.<br>

![Commd9](/Images/Commd9.png)

**Figure 12: Commd9**

COMMD9 plays a critical role in endosomal recycling of cell-surface cargo and directly interacts in the nucleus with the transcription factor TFDP1 via its conserved C-terminal COMM domain to drive cell-cycle progression (G1 to S phase). The normal truncated variant lacks the COMM domain, acting as a regulatory buffer. The tumor eliminates this buffer completely, ensuring that 100% of translated COMMD9 is structurally equipped to bind TFDP1, maximizing E2F1-mediated cell-cycle transition and cargo recycling.


**References**
 [1] PRJNA1464579
