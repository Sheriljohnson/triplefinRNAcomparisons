# De novo Transcriptome Annotation on NeSI

**TransDecoder → BLAST → Trinotate (L & N species)**

This document describes the complete, working NeSI pipeline for annotating two species-specific Trinity transcriptomes using **TransDecoder**, **BLAST**, and **Trinotate**, with correct use of **login nodes, Slurm, and project storage**.

---

## 1. Location of New Trinity Assemblies

The new species-specific Trinity assemblies are located in the `revisions/` directory:

```bash
/nesi/nobackup/uoo03946/revisions/RF_trinity_outputL.Trinity.fasta
/nesi/nobackup/uoo03946/revisions/RF_trinity_outputN.Trinity.fasta
```

Optional sanity check (number of transcripts):

```bash
grep -c "^>" /nesi/nobackup/uoo03946/revisions/RF_trinity_outputL.Trinity.fasta
grep -c "^>" /nesi/nobackup/uoo03946/revisions/RF_trinity_outputN.Trinity.fasta
```

---

## 2. Create a TransDecoder Working Directory (HOME)

TransDecoder can safely be run in `$HOME`.

```bash
mkdir -p ~/transdecoder_new
cd ~/transdecoder_new
pwd
```

Expected output:

```
/home/USERNAME/transdecoder_new
```

---

## 3. Copy Assemblies and Gene Maps

```bash
cp /nesi/nobackup/uoo03946/revisions/RF_trinity_outputL.Trinity.fasta .
cp /nesi/nobackup/uoo03946/revisions/RF_trinity_outputL.Trinity.fasta.gene_trans_map .

cp /nesi/nobackup/uoo03946/revisions/RF_trinity_outputN.Trinity.fasta .
cp /nesi/nobackup/uoo03946/revisions/RF_trinity_outputN.Trinity.fasta.gene_trans_map .
```

Verify files:

```bash
ls -lh
```

Expected sizes (approximate):

* L fasta ~244 MB
* N fasta ~216 MB

---

## 4. Run TransDecoder (L then N)

```bash
TransDecoder.LongOrfs -t RF_trinity_outputL.Trinity.fasta
TransDecoder.Predict  -t RF_trinity_outputL.Trinity.fasta

TransDecoder.LongOrfs -t RF_trinity_outputN.Trinity.fasta
TransDecoder.Predict  -t RF_trinity_outputN.Trinity.fasta
```

These steps may take minutes to ~1 hour depending on assembly size.

---

## 5. Confirm TransDecoder Outputs

```bash
ls -lh *.transdecoder.pep
ls -ld *.transdecoder_dir
```

Optional sanity check:

```bash
grep -c "^>" RF_trinity_outputL.Trinity.fasta.transdecoder.pep
grep -c "^>" RF_trinity_outputN.Trinity.fasta.transdecoder.pep
```

Counts in the **thousands to tens of thousands** are normal.

---

## 6. Summary of Files at This Stage

You should now have:

### Assemblies

* `RF_trinity_outputL.Trinity.fasta`
* `RF_trinity_outputN.Trinity.fasta`

### Gene maps

* `RF_trinity_outputL.Trinity.fasta.gene_trans_map`
* `RF_trinity_outputN.Trinity.fasta.gene_trans_map`

### TransDecoder proteins

* `RF_trinity_outputL.Trinity.fasta.transdecoder.pep`
* `RF_trinity_outputN.Trinity.fasta.transdecoder.pep`

---

## 7. Move to Project Space for Annotation

BLAST and Trinotate **must not** be run from `$HOME`.

```bash
mkdir -p /nesi/project/uoo03946/johsh68p_trinotate
cd /nesi/project/uoo03946/johsh68p_trinotate
```

Copy inputs:

```bash
cp ~/RF_trinity_outputL.Trinity.fasta .
cp ~/RF_trinity_outputN.Trinity.fasta .

cp ~/RF_trinity_outputL.Trinity.fasta.gene_trans_map .
cp ~/RF_trinity_outputN.Trinity.fasta.gene_trans_map .

cp ~/RF_trinity_output*.transdecoder.pep .
cp ~/uniprot_sprot.pep .
```

Sanity check:

```bash
ls -lh
```

You should see ~1 GB of data.

---

## 8. Build BLAST Database

```bash
module load BLAST
makeblastdb -in uniprot_sprot.pep -dbtype prot
```

Confirm:

```bash
ls uniprot_sprot.pep*
```

Expected:

```
.pdb .phr .pin .pot .psq .pto .ptf
```

---

## 9. IMPORTANT: Where Slurm Jobs Must Be Submitted

Slurm jobs **must be submitted from a NeSI login node**, not from `lander`.

Correct login:

```bash
ssh USERNAME@login.hpc.nesi.org.nz
```

Confirm:

```bash
hostname
which sbatch
```

You should see `loginXX` and `/usr/bin/sbatch`.

---

## 10. BLASTX (L species)

Create script:

```bash
cat > blastx_L.sl <<'EOF'
#!/bin/bash -e
#SBATCH --job-name=blastx_L
#SBATCH --account=uoo03946
#SBATCH --time=7-00:00:00
#SBATCH --cpus-per-task=32
#SBATCH --mem=50G
#SBATCH --output=blastx_L.%j.out
#SBATCH --error=blastx_L.%j.err

module purge
module load BLAST

blastx -query RF_trinity_outputL.Trinity.fasta \
  -db uniprot_sprot.pep \
  -num_threads 32 \
  -max_target_seqs 1 \
  -outfmt 6 \
  -evalue 1e-3 \
  > blastx_L.outfmt6
EOF
```

Submit:

```bash
sbatch blastx_L.sl
```

---

## 11. BLASTP (L species)

```bash
cat > blastp_L.sl <<'EOF'
#!/bin/bash -e
#SBATCH --job-name=blastp_L
#SBATCH --account=uoo03946
#SBATCH --time=5-00:00:00
#SBATCH --cpus-per-task=32
#SBATCH --mem=50G
#SBATCH --output=blastp_L.%j.out
#SBATCH --error=blastp_L.%j.err

module purge
module load BLAST

blastp -query RF_trinity_outputL.Trinity.fasta.transdecoder.pep \
  -db uniprot_sprot.pep \
  -num_threads 32 \
  -max_target_seqs 1 \
  -outfmt 6 \
  -evalue 1e-3 \
  > blastp_L.outfmt6
EOF

sbatch blastp_L.sl
```

---

## 12. BLASTX (N species)

```bash
cat > blastx_N.sl <<'EOF'
#!/bin/bash -e
#SBATCH --job-name=blastx_N
#SBATCH --account=uoo03946
#SBATCH --time=7-00:00:00
#SBATCH --cpus-per-task=32
#SBATCH --mem=50G
#SBATCH --output=blastx_N.%j.out
#SBATCH --error=blastx_N.%j.err

module purge
module load BLAST

blastx -query RF_trinity_outputN.Trinity.fasta \
  -db uniprot_sprot.pep \
  -num_threads 32 \
  -max_target_seqs 1 \
  -outfmt 6 \
  -evalue 1e-3 \
  > blastx_N.outfmt6
EOF

sbatch blastx_N.sl
```

---

## 13. BLASTP (N species)

```bash
cat > blastp_N.sl <<'EOF'
#!/bin/bash -e
#SBATCH --job-name=blastp_N
#SBATCH --account=uoo03946
#SBATCH --time=5-00:00:00
#SBATCH --cpus-per-task=32
#SBATCH --mem=50G
#SBATCH --output=blastp_N.%j.out
#SBATCH --error=blastp_N.%j.err

module purge
module load BLAST

blastp -query RF_trinity_outputN.Trinity.fasta.transdecoder.pep \
  -db uniprot_sprot.pep \
  -num_threads 32 \
  -max_target_seqs 1 \
  -outfmt 6 \
  -evalue 1e-3 \
  > blastp_N.outfmt6
EOF

sbatch blastp_N.sl
```

---

## 14. Monitor Jobs

```bash
squeue -u $USER
```

Typical state:

* 3 jobs running
* 1 job pending due to `(Priority)` — normal

---

## 15. After Jobs Finish

```bash
ls -lh blastx_*.outfmt6 blastp_*.outfmt6
```

At this point, BLAST is complete and you are ready for:

* Trinotate database loading (L and N separately)
* Trinotate reports
* GO extraction
* Downstream DE / GOseq

---

## Key Take-Home Points

* `lander` is **authentication only**, not Slurm
* `sbatch` must be run from **login nodes**
* BLAST must be run in **project space**
* TransDecoder outputs are reused cleanly
* L and N are annotated **independently**, satisfying reviewer concerns

---


---

# Checking progress + Trinotate loading (post-BLAST)

## How to check progress later

When you come back:

```bash
cd /nesi/project/uoo03946/johsh68p_trinotate
squeue -u $USER
```

And you can watch log files:

```bash
tail -n 20 blastx_L.3830329.out
tail -n 20 blastp_L.3830362.out
tail -n 20 blastx_N.3830383.out
```

*(For the pending one, the `.out` won’t exist until it starts.)*

## Quick sanity checks

### 1) Count how many hits are in each file

```bash
wc -l blastp_L.outfmt6 blastp_N.outfmt6 blastx_L.outfmt6 blastx_N.outfmt6
```

### 2) Spot-check the format looks correct

```bash
head -n 3 blastx_L.outfmt6
head -n 3 blastp_L.outfmt6
```

You should see 12-column, tab-delimited rows.

---

## Trinotate notes

- Your BLAST outputs look totally sane (nice hit counts; format is correct). ✅
- The Trinotate error is also simple: `Trinotate … init` expects the SQLite file to already exist. You need a boilerplate DB first, then run `init`.
- If you asked for `Trinotate_L.sqlite` but it doesn’t exist yet, Trinotate will refuse.

---

# Trinotate Slurm scripts (drop-in)

Run from:

```bash
cd /nesi/project/uoo03946/johsh68p_trinotate
```

## 1) Lapillum (L) — `trinotate_L_clean.sl`

```bash
cat > trinotate_L_clean.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --job-name=trinotate_L_clean
#SBATCH --time=08:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=48G
#SBATCH --output=trinotate_L_clean.%j.out
#SBATCH --error=trinotate_L_clean.%j.err

module --force purge
module load NeSI
module load Trinotate/3.2.2-GCC-9.2.0

WORKDIR=/nesi/project/uoo03946/johsh68p_trinotate
cd "$WORKDIR"

echo "Hostname: $(hostname)"
echo "Date: $(date)"

echo "Checking required inputs:"
ls -lh \
  Trinotate_L.sqlite \
  RF_trinity_outputL.Trinity.fasta \
  RF_trinity_outputL.Trinity.fasta.gene_trans_map \
  RF_trinity_outputL.Trinity.fasta.transdecoder.pep \
  blastx_L.outfmt6 \
  blastp_L.outfmt6

echo "INIT"
Trinotate Trinotate_L.sqlite init \
  --gene_trans_map RF_trinity_outputL.Trinity.fasta.gene_trans_map \
  --transcript_fasta RF_trinity_outputL.Trinity.fasta \
  --transdecoder_pep RF_trinity_outputL.Trinity.fasta.transdecoder.pep

echo "LOAD BLASTX"
Trinotate Trinotate_L.sqlite LOAD_swissprot_blastx blastx_L.outfmt6

echo "LOAD BLASTP"
Trinotate Trinotate_L.sqlite LOAD_swissprot_blastp blastp_L.outfmt6

echo "REPORT"
Trinotate Trinotate_L.sqlite report > Trinotate_L_clean.xls

echo "DONE"
ls -lh Trinotate_L_clean.xls
EOF
```

Submit + check:

```bash
sbatch trinotate_L_clean.sl
squeue -u $USER
```

If it exits quickly:

```bash
ls -lt trinotate_L_clean.*.err | head
tail -n 80 trinotate_L_clean.*.err
```

## 2) Nigripenne (N) — `trinotate_N_clean.sl`

```bash
cat > trinotate_N_clean.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --job-name=trinotate_N_clean
#SBATCH --time=08:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=48G
#SBATCH --output=trinotate_N_clean.%j.out
#SBATCH --error=trinotate_N_clean.%j.err

module --force purge
module load NeSI
module load Trinotate/3.2.2-GCC-9.2.0

WORKDIR=/nesi/project/uoo03946/johsh68p_trinotate
cd "$WORKDIR"

echo "Hostname: $(hostname)"
echo "Date: $(date)"

echo "Checking required inputs:"
ls -lh \
  Trinotate_N.sqlite \
  RF_trinity_outputN.Trinity.fasta \
  RF_trinity_outputN.Trinity.fasta.gene_trans_map \
  RF_trinity_outputN.Trinity.fasta.transdecoder.pep \
  blastx_N.outfmt6 \
  blastp_N.outfmt6

echo "INIT"
Trinotate Trinotate_N.sqlite init \
  --gene_trans_map RF_trinity_outputN.Trinity.fasta.gene_trans_map \
  --transcript_fasta RF_trinity_outputN.Trinity.fasta \
  --transdecoder_pep RF_trinity_outputN.Trinity.fasta.transdecoder.pep

echo "LOAD BLASTX"
Trinotate Trinotate_N.sqlite LOAD_swissprot_blastx blastx_N.outfmt6

echo "LOAD BLASTP"
Trinotate Trinotate_N.sqlite LOAD_swissprot_blastp blastp_N.outfmt6

echo "REPORT"
Trinotate Trinotate_N.sqlite report > Trinotate_N_clean.xls

echo "DONE"
ls -lh Trinotate_N_clean.xls
EOF
```

Submit + check:

```bash
sbatch trinotate_N_clean.sl
squeue -u $USER
```

---

## A3) Generating the report interactively (what worked)

If you want to generate the `.xls` manually (not Slurm), this is the working sequence:

```bash
cd /nesi/project/uoo03946/johsh68p_trinotate
module load Trinotate/3.2.2-GCC-9.2.0

Trinotate Trinotate_L.sqlite report > Trinotate_L_clean.xls
Trinotate Trinotate_N.sqlite report > Trinotate_N_clean.xls

mkdir -p final_annotations
mv Trinotate_L_clean.xls Trinotate_N_clean.xls final_annotations/
ls -lh final_annotations

cp Trinotate_L.sqlite Trinotate_N.sqlite final_annotations/
```

---

# Getting accession numbers

## 1) Confirm which columns contain BLAST accessions (N species)

```bash
cd /nesi/project/uoo03946/johsh68p_trinotate/final_annotations
head -n 1 Trinotate_N_clean.xls | tr '\t' '\n' | nl
```

You should see something like (column numbers may vary):

- `BLASTX_hit`
- `BLASTP_hit`

These fields usually look like:

- `X1WHY6`
- `Q9XYZ3^Q9XYZ3_HUMAN^...`

## 2) Extract primary BLAST accession for N (BLASTX + BLASTP)

This matches what you did for L: take only the accession, strip everything after `^`.

Example (adjust column numbers if needed).

If:

- `BLASTX_hit` = column 3
- `BLASTP_hit` = column 7

```bash
awk -F'\t' '
BEGIN{OFS="\t"}
NR==1{
  print $0, "BLASTX_ACC", "BLASTP_ACC"
  next
}
{
  bx = ($3!="." && $3!="") ? $3 : "."
  bp = ($7!="." && $7!="") ? $7 : "."

  sub(/\^.*/, "", bx)
  sub(/\^.*/, "", bp)

  print $0, bx, bp
}
' Trinotate_N_clean.xls > Trinotate_N_with_accessions.tsv
```

This gives you clean accession IDs only (e.g. `Q9XYZ3`) for N.

## 3) Optional: Create a gene-level accession file for N

If you previously collapsed transcript-level hits to gene level for L, do the same here.

Assuming column 1 is `gene_id`:

```bash
awk -F'\t' '
BEGIN{OFS="\t"}
NR>1{
  g=$1
  if ($NF!=".") acc[g]=$NF
}
END{
  for (g in acc) print g, acc[g]
}
' Trinotate_N_with_accessions.tsv > N_gene_BLAST_accessions.tsv
```

## 4) Quick QC checks

How many genes have accessions?

```bash
cut -f2 N_gene_BLAST_accessions.tsv | sort | uniq | wc -l
```

Any weird formatting left?

```bash
grep '\^' N_gene_BLAST_accessions.tsv | head
```

*(Should return nothing.)*
