# Big_data

Last updated: 10/04/2026

## Study summary

Longitudinal transcriptome re-analysis of **Bouquet et al. 2016 (GSE63085)** — acute Lyme disease vs healthy controls at three visits (V1 acute, V2 post-treatment, V5 six months) — using **raw counts**, **DESeq2**, and **open-source pathway methods** (ORA, GSEA, GSVA, SPIA). See `lyme_dge_analysis.Rmd` / knitted `lyme_dge_analysis.html` for the full report.

---

## Project structure

| Path | Role |
|------|------|
| `lyme_dge_analysis.Rmd` | Main analysis notebook (source of truth for code and narrative) |
| `lyme_dge_analysis.html` | Knitted report (tables, figures, interpretation) |
| `sample_annotations.csv` | Sample metadata (GSM, patient, visit, group) |
| `Data/` | Input data: raw counts matrix (`GSE63085_raw_counts_GRCh38.p13_NCBI.tsv.gz`), gene annotation (`Human.GRCh38.p13.annot.tsv.gz`) |
| `results/` | Exported CSVs: DEG tables, ORA/GSEA/GSVA/SPIA outputs |
| `plots/` | Optional saved plots (if written from chunks) |
| `lyme_dge_analysis_cache/` | knitr chunk cache (speeds re-knitting; safe to delete to force full rerun) |

---

## Analysis steps (pipeline)

1. **Load data** — Align sample table to count matrix columns; engineer `group` / `visit` factors.
2. **QC & EDA** — Library sizes, RLE, densities, mean–variance, **MA (pre-normalisation)**.
3. **Pre-filtering** — Remove genes with consistently low counts (DESeq2-style filter).
4. **DESeq2** — `DESeqDataSet` with `design = ~ group`; **size factors** (median of ratios); **VST** for PCA/heatmaps; **MA after size factors** (normalized counts).
5. **Exploratory structure** — PCA (incl. patient trajectories), sample distance heatmap, top-variable genes.
6. **Differential expression** — `DESeq()`; contrasts V1/V2/V5 vs Control; **lfcShrink(type = "ashr")**; **DESeq2 `plotMA()`** for each contrast.
7. **Pathway analysis (generations)**  
   - **1st:** ORA (MSigDB Hallmark + GO:BP) via `clusterProfiler::enricher`  
   - **2nd:** GSEA (ranked genes); **GSVA** + limma on pathway scores  
   - **3rd:** **SPIA** on KEGG topology (perturbation + over-representation)  
8. **Theme comparison** — `compareCluster` across time points (Hallmark).
9. **Export** — CSVs under `results/`; interpretation section in the HTML ties methods to biology.

---

## Key outputs

| Output | Description |
|--------|-------------|
| `results/DEG_*_vs_Control.csv` | Per-contrast DESeq2 results with symbols and annotation |
| `results/ORA_Hallmark_*.csv` | Over-representation of Hallmark gene sets |
| `results/GSEA_Hallmark_*.csv` | GSEA statistics (NES, p values) per contrast |
| `results/GSVA_Hallmark_*.csv` / `GSVA_scores_per_sample.csv` | Pathway scores and limma tests on GSVA |
| `results/SPIA_KEGG_*_vs_Control.csv` | SPIA KEGG pathway table (pNDE, pPERT, pG, Status) |
| `lyme_dge_analysis.html` | Full narrative, all figures, session info |

---

## Study context (original paper)

### Their analytical pipeline (summary)

1. Alignment: TopHat 2 → hg19  
2. Quantification: Cufflinks 2 → FPKM  
3. Differential expression: Voom → Limma (FC > 1.5, FDR < 0.1%)  
4. Pathway analysis: IPA (commercial)  
5. Comparisons with other infection/chronic disease datasets  

### Study key findings (short)

- Sustained differential expression after the acute phase  
- Acute phase TLR-heavy signature; partial overlap with chronic immune disease pathways at six months  
- No simple residual DEG signature for PTLDS vs resolved at six months (with caveats about power)  

---

## Our notes and improvement themes

- Season / batch as potential confounders; original controls vs cases differ in sampling season.  
- Replace IPA with **clusterProfiler + MSigDB**; add **GSEA** and **GSVA**; use **DESeq2** on counts.  
- Raw counts and **GRCh38.p13** re-quantification (Thomas) vs original FPKM/hg19.  
- Paired/longitudinal design could be modeled more explicitly (e.g. patient effect) in a follow-up.  

---

## Data files

| File | Description |
|------|-------------|
| `Data/GSE63085_raw_counts_GRCh38.p13_NCBI.tsv.gz` | Raw integer counts (Entrez gene IDs × samples) |
| `Data/Human.GRCh38.p13.annot.tsv.gz` | Gene annotation (symbol, type, GO, etc.) |
| `sample_annotations.csv` | Sample metadata; **22** GSMs in the table are absent from the count matrix (Thomas’s QC / pipeline). |

---

## Checked steps (progress)

- [x] Sample annotation and alignment to counts  
- [x] EDA and DESeq2 DE (V1, V2, V5 vs Control)  
- [x] ORA + GSEA + GSVA  
- [x] SPIA (3rd generation, KEGG)  
- [ ] Final presentation (course deliverable)  

### Example headline results (from a full knit; re-run may differ slightly)

- Many thousands of DEGs at padj &lt; 0.05 (threshold-dependent for \|LFC\| &gt; 1)  
- Hallmark ORA/GSEA: strong inflammatory / IFN / TNF signatures at V1  
- GSVA: per-sample pathway scores + limma contrasts  
- SPIA: KEGG-level **Activated/Inhibited** calls with topology-aware **pG**  

---

## References

- Bouquet J et al. *mBio* 2016; 7(1):e00100-16.  
- Mohrke T. *Beginner’s Handbook of -Omics Data Analysis* (Ch. 7–8).  
- DESeq2, clusterProfiler, GSVA, SPIA documentation and vignettes.  
