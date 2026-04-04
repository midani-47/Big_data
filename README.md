# Big_data
Last updated: 28/03/2026 at 15:30

## Study Summary: Longitudinal Transcriptome Analysis in Acute Lyme Disease
This project analyzes a longitudinal transcriptome study of 29 Lyme disease patients and 13 matched controls, collected at multiple time points (acute phase, 1 month, and 6 months post-treatment). The study investigates the molecular basis of acute Lyme disease and the development of post-treatment symptoms. 

### Their Analytical Pipeline (What They Did):
1. *Wet lab stuff*
2. *Alignment*: TopHat 2 (Bowtie 2) -> hg19 human genome (_maybe outdated ref?_)
3. *Quantification*: Cufflinks 2 -> FPKM values for 25,278 genes (_FPKM def. outdated, according to LLMs, and mathematically "flawed"_)
4. Differential expression: Voom transformation -> Limma linear modeling -> DEGs (fold change (FC) > 1.5, p < 0.05, false discovery rate (FDR) < 0.1%)
5. *Pathway analysis*: IPA (_which is an paid commercial tool_)
6. *Comparisons*: Cross-referenced with 15 other transcriptome datasets (microarray + RNA-seq) from other infections and chronic diseases

#### **Analysis Techniques:** 
Differential gene expression analysis, pathway enrichment analysis, and comparison with immune-mediated chronic diseases were performed to identify key molecular signatures and pathways associated with Lyme disease and its long-term effects.

### Study Key Findings:
- **Sustained Signature:** A differential gene expression signature of Lyme disease persists for at least 3 weeks following the acute phase.
- **Early Acute Phase:** Characterized by marked upregulation of Toll-like receptor (TLR) signaling but lacks activation of inflammatory T-cell apoptotic and B-cell developmental pathways seen in other acute infections.
- **Long-term Similarities:** Six months post-treatment, Lyme disease patients shared 31 to 60% of their pathways with immune-mediated chronic diseases (like rheumatoid arthritis).
- **Persistent Symptoms:** At 6 months post-treatment, no specific differential gene expression signature was observed between patients who recovered versus those with persistent symptoms.
- **Clinical Relevance:** The sustained signature suggests a potential panel of human host-based biomarkers could aid in sensitive clinical diagnosis prior to a detectable antibody response.


## Our thoughts and findings:
- Interesting citations:
   - “The average duration of acute illness, defined as the time from onset of EM rash and/or influenza-like symptoms to study enrollment and initiation of doxycycline therapy, was significantly longer in patients developing persistent symptoms (9.7 days for non-PTLDS and 19.3 days for PTLDS) than in patients with resolved illness (5.2 days)”

- 
- 


## Areas for improvement:

1. **Batch effects/Biological confounding variables (_source: Abed_):**
Not sure if this is a batch effect, or a _biological confounding variable_, but the authors said that there's no batch effect by PCA of the 13 controls alone and by finding zero DEGs between winter/spring controls. But isn't comparing only 8 vs 5 controls extremely  underpowered? For me, I remember seasons were a big factor in my response to the infection. The seasonal confound was not properly addressed in the study. Controls were sampled mostly in winter (61.5%) and patients were sampled mostly in summer (82.8%). Improvement could be that we include season as a covariate in the DESeq2 model to account for batch effects.

2. **Open-source pathway analysis (clusterProfiler + MSigDB) instead of IPA (_Source: Thomas_)**:
IPA is expensive. Thomas's pipeline uses clusterProfiler with MSigDB (Molecular Signatures Database) which is reproducible and free. we can do Gene Ontology (GO) enrichment, KEGG pathway analysis, and GSEA (Gene Set Enrichment Analysis) which tests ALL genes (not just the significant ones), without random thresholds like in the study.

3. **Small subgroup sizes for PTLDS comparison (_source: Abed_)**:
only 4 ptlds patients vs 15 resolved. does finding 0 DEGs "make sense" considering the low power (62% as they reported)? Should we just acknowledge this limitation clearly in the presentation?

4. **No gene set enrichment analysis (GSEA) (_source: Thomas)_**
the study only did over (/under)-representation analysis (ORA) via IPA, which only looks at genes above the significance cutoff. We should definitely use GSEA using clusterProfiler, beacuse it uses all genes detects more changes in pathways that ORA misses.

5. **Starting from raw counts, not FPKM (_source: LLMs_)**
The original used FPKM from Cufflinks, which normalizes by gene length and library size. But FPKM has known issues: it's not comparable across samples because it assumes the same total mRNA per cell. DESeq2's median-of-ratios normalization (size factors) is statistically superior for differential expression.

6. **Statistical Method: DESeq2 instead of Limma/Voom (_source: Thomas_)**
study used Limma/Voom which which was according to llms originally designed for microarrays.
DESeq2 directly models count data, with built-in shrinkage estimation of dispersion and log2 fold changes. Better suited for RNA-Seq, especially with small sample sizes (like n=4 for PTLDS). DESeq2 also has Cook's distance for automatic outlier detection per gene.


7. **Paired/longitudinal design not fully exploited (_source: LLM_)**
The same 29 patients were measured at 3 time points, creating a repeated measures design. The original study compared each time point vs. controls independently.
Improvement: Use DESeq2's ability to model paired samples by including patient as a blocking factor. This accounts for individual variation and increases statistical power.

8. **Arbitrary fold-change and FDR cutoffs _(sourece: LLM)_**
The original used FC > 1.5 and FDR < 0.1%. These are arbitrary.
Improvement: Use DESeq2's lfcShrink() for shrunken log2 fold change estimates (more reliable for genes with low counts), and explore results at multiple thresholds.

9. **Outdated alignment pipeline (_source: LLM, confirmed by Thomas_)**
TopHat/Cufflinks is now deprecated. Modern pipelines use STAR or HISAT2 for alignment and featureCounts or Salmon for quantification.
Thomas re-aligned the raw FASTQ data to **GRCh38.p13** (vs original hg19) using a modern pipeline and produced raw integer counts. This addresses both the outdated aligner and the FPKM issue. *(Ask Thomas which aligner/quantifier he used for the citation.)*

10. **Updated genome reference: GRCh38.p13 vs hg19 (_source: Thomas_)**
The original study used hg19 (GRCh37, released 2009). Thomas's re-alignment uses GRCh38.p13, the current human reference, with corrected sequences, better gene models, and hundreds of gap fixes.


## Our Data

| File | Description | Dimensions |
|------|-------------|------------|
| `Data/GSE63085_raw_counts_GRCh38.p13_NCBI.tsv.gz` | Raw integer counts (GRCh38.p13, NCBI Entrez Gene IDs) | 39,376 genes x 75 samples |
| `Data/Human.GRCh38.p13.annot.tsv.gz` | Gene annotation (Symbol, Description, GeneType, Ensembl, GO terms) | 39,376 genes x 17 columns |
| `sample_annotations.csv` | Sample metadata (Run, GSM, patient, disease state, time point) | 97 samples (75 in counts) |

22 samples from the annotation are not in the counts matrix (excluded during Thomas's quality filtering in the modern alignment pipeline).


## Our Improved Pipeline

```
Raw counts (GRCh38.p13, from Thomas)
  |-> DESeq2 (negative binomial model, size factor normalization)
      |-> Pre-filtering (remove low-count genes)
      |-> Differential expression (V1/V2/V5 vs Control)
      |-> lfcShrink (shrunken log2 fold changes)
  |-> Biology context (following Mohrkeg Ch. 8):
      |-> 1st gen: ORA via clusterProfiler + MSigDB
      |-> 2nd gen: GSEA via clusterProfiler (NEW vs original study)
      |-> 2nd gen: GSVA via GSVA package (NEW vs original study)
```

**Reference**: Mohrkeg T. *Beginner's Handbook of -Omics Data Analysis*,
Chapter 8: Downstream Analysis of Expression Data.


### Checked Steps:
- [x] Sample data annotation
- [x] Go through the paper and understand the methods used for data analysis
- [x] Get the data from Thomas (raw counts GRCh38.p13 + gene annotation)
- [x] Understand Thomas's Chapter 8 (three generations of pathway analysis)
- [x] Full EDA on the data (lyme_dge_analysis.Rmd)
- [x] DESeq2 differential expression (V1, V2, V5 vs Control)
- [ ] Downstream: ORA with clusterProfiler + MSigDB (1st generation)
- [ ] Downstream: GSEA with clusterProfiler (2nd generation)
- [ ] Downstream: GSVA transformation + analysis (2nd generation)
- [ ] Compare our DESeq2 results with the original Limma/Voom findings
- [ ] Final presentation / report