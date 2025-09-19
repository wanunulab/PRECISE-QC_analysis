# PRECISE-QC: Step‑by‑Step Pipeline Manual

*A reproducible guide to run the Yee et al. (2025) analysis from raw POD5 to error profiles.*

---

## 0) Prerequisites

### 0.1 Install required tools (tested versions)

* **Dorado** v1.0.0
* **samtools** ≥1.17
* **BWA-MEM** v0.7.15 (or 0.7.17)
* **Porechop** v0.2.4
* **Python** ≥3.9 with **pysamstats** installed (`pip install pysamstats`)
* **IGV** (optional, for visual inspection)

### 0.2 Prepare project folders

```bash
# adjust PROJECT to your desired location
export PROJECT=$HOME/projects/precise-qc
mkdir -p $PROJECT/{raw,ref,work,out}
export RAW=$PROJECT/raw
export REF=$PROJECT/ref
export WORK=$PROJECT/work
export OUT=$PROJECT/out
```

### 0.3 Get the reference and data

* Place your **POD5** (or TAR of POD5s) in `$RAW/`.
* Put the **sgRNA reference sequence FASTA** at `$REF/reference.fa`.
* (Optional, for reproducing the paper figures) Download the datasets:

  * **Labeled data:** link in the paper’s Figshare page
  * **Unlabeled data:** link in the paper’s Figshare page

> If you are reproducing exactly Yee et al. (2025), download from https://www.ncbi.nlm.nih.gov/sra/PRJNA1305499 and place POD5 files under `$RAW/`.

---

## 1) Basecalling (with modified bases + moves table)

### 1.1 Run Dorado basecalling

**Recommended Dorado Version (Dorado ≥0.8):**

```bash
# MODEL: rna004_130bps_sup@v5.2.0
dorado basecaller rna004_130bps_sup@v5.2.0 $RAW \
  --modified-bases pseU_2OmeU m5C_2OmeC 2OmeG inosine_m6A_2OmeA \
  --emit-moves \

```

**The list of modifications in the command is optional, adjust according to your sequence:**

```bash
# Equivalent to the command shown in the manuscript methods
# dorado --modified-bases ... --emit-moves > basecalled.bam
```

---

## 2) Align reads to the sgRNA reference

### 2.1 Index the reference

```bash
bwa index $REF/reference.fa
```

### 2.2 Run BWA-MEM with ONT presets (as used in the study)

```bash
bwa mem -t 1 -w 13 -k 6 -x ont2d $REF/reference.fa $WORK/basecalled.fastq > $WORK/alignment.sam
```

> Note: These parameters are tuned for short (\~100 nt) ONT reads against a small sgRNA reference.

### 2.3 Convert and sort alignments

```bash
samtools view -@ 4 -bS $WORK/alignment.sam | samtools sort -o $WORK/alignment.sorted.bam
samtools index $WORK/alignment.sorted.bam
```

---

## 3) Filter primary alignments and remove soft-clipped reads

### 3.1 Keep only primary alignments

```bash
samtools view -@ 4 -b -F 0x100 $WORK/alignment.sorted.bam -o $WORK/primary.bam
samtools index $WORK/primary.bam
```

### 3.2 Remove any read whose CIGAR contains soft clipping ("S")

```bash
samtools view -h $WORK/primary.bam \
| awk '($0 ~ /^@/ || $6 !~ /S/)' \
| samtools view -b -o $WORK/unclipped.bam
samtools index $WORK/unclipped.bam
```


---

## 4) Select full‑length reads (95–105 nt) and profile errors

### 4.1 Filter by read length on unclipped alignments

```bash
samtools view -h $WORK/unclipped.bam \
| awk 'substr($1,1,1)=="@" || (length($10)>=95 && length($10)<=105)' \
| samtools view -b -o $WORK/full_length.bam
samtools index $WORK/full_length.bam
```

### 4.2 Generate per‑nucleotide error profiles (pysamstats)

```bash
pysamstats --type variation --fasta $REF/reference.fa $WORK/full_length.bam > $OUT/only_variation.txt
```

> Output: `$OUT/only_variation.txt` (columns include ref pos, depth, mismatches, insertions, deletions, etc.).

---

## 5) Analyze truncated reads

### 5.1 Find reads with the 5′ adapter (Porechop)

```bash
porechop -i $WORK/basecalled.fastq -o $WORK/adapter_reads.fastq \
  --barcode_diff 1 --barcode_threshold 74 --verbosity 2
```

### 5.2 Align adapter‑containing truncated reads

```bash
bwa mem -t 1 -w 13 -k 6 -x ont2d $REF/reference.fa $WORK/adapter_reads.fastq > $WORK/trunc_alignment.sam
samtools view -bS $WORK/trunc_alignment.sam | samtools sort -o $WORK/trunc_alignment.bam
samtools index $WORK/trunc_alignment.bam
```


### 5.3 Inspect in IGV (optional)

* Open `$REF/reference.fa` and `$WORK/trunc_alignment.bam` in **IGV**.
* Examine coverage and CIGAR patterns near the 5′ end.

---

## 6) Outputs you should have

* `$WORK/basecalled.bam(.bai)` — basecalled reads with modified‑base and move metadata
* `$WORK/basecalled.fastq` — (optional) FASTQ export of the above
* `$WORK/alignment.sorted.bam(.bai)` — all alignments to sgRNA
* `$WORK/primary.bam(.bai)` — primary alignments only
* `$WORK/unclipped.bam(.bai)` — primary alignments with no soft clipping
* `$WORK/full_length.bam(.bai)` — 95–105 nt full‑length reads
* `$WORK/adapter_reads.fastq` — reads containing the 5′ adapter (truncated set)
* `$WORK/trunc_alignment.bam(.bai)` — alignments of adapter‑containing reads
* `$OUT/error_profile.tsv` — per‑nucleotide error profile for full‑length reads
* `$OUT/truncated_*lengths.tsv` — length distributions for truncated reads

---

## 7) Common pitfalls & troubleshooting

* **No MM/ML tags:** ensure Dorado was run with `--modified-bases` and the selected modifications.
* **No moves:** ensure `--emit-moves` was included; some tags are model/CLI dependent.
* **pysamstats fails:** make sure BAMs are **coordinate‑sorted** and **indexed**; use the same `reference.fa` you aligned to.
* **Soft‑clipped reads remain:** re‑run step 3.2; your filter must run on **SAM text** (`samtools view -h ... | awk ... | samtools view -b`).
* **Few full‑length reads:** adjust the length window in step 4.1 (e.g., `90–110 nt`) and re‑profile.
* **Adapter detection weak:** try relaxing/tightening `--barcode_threshold` in Porechop; verify the expected 5′ adapter sequence is included in Porechop’s database or provide a custom adapter file.

---

## (Optional) Alternative aligner if the reads are longer than 300 bases

For some ONT datasets, **minimap2** can be used instead of BWA‑MEM:

```bash
# install: conda install -c bioconda minimap2
minimap2 -ax map-ont $REF/reference.fa $WORK/basecalled.fastq | \
  samtools sort -o $WORK/alignment.bam
samtools index $WORK/alignment.bam
```


---

## Authors & Credits

Analysis by **Yvonne Yee** and **Dinara Boyko** (Northeastern University, Departments of Chemical Engineering & Physics).



---

## How to cite

If you use this pipeline, please cite:
TODO: link to bioarx



---
