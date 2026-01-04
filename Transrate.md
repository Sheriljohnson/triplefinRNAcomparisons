# Transrate comparisons (species-specific vs hybrid)

This document sets up a Transrate workspace and runs **reference-mode** comparisons between:
- species-specific assemblies (L, N)
- the hybrid assembly (Hybrid).

Hybrid file to use for Transrate:

```bash
/nesi/project/uoo03946/denovo_transcriptome/RF_trinity_output.Trinity.fasta
```

---

## A) Workspace setup

### A1) Create the Transrate workspace

```bash
WORK=/nesi/project/uoo03946/transrate_compare
mkdir -p $WORK/{assemblies,reads,results}
cd $WORK
```

### A2) Link the three assemblies (no copying)

```bash
ln -sf /nesi/project/uoo03946/johsh68p_trinotate/RF_trinity_outputL.Trinity.fasta assemblies/L.fasta
ln -sf /nesi/project/uoo03946/johsh68p_trinotate/RF_trinity_outputN.Trinity.fasta assemblies/N.fasta
ln -sf /nesi/project/uoo03946/denovo_transcriptome/RF_trinity_output.Trinity.fasta assemblies/Hybrid.fasta

ls -lh assemblies
```

You should see:
- `L.fasta`
- `N.fasta`
- `Hybrid.fasta`

### A3) Link reads (edit paths ONLY if needed)

> If you only want **reference-mode** comparisons (assembly vs reference), you can skip read links.
> If you want **read-mapping metrics** later, link reads here.

```bash
# L reads
ln -sf /path/to/L_R1.fastq.gz reads/L_R1.fq.gz
ln -sf /path/to/L_R2.fastq.gz reads/L_R2.fq.gz

# N reads
ln -sf /path/to/N_R1.fastq.gz reads/N_R1.fq.gz
ln -sf /path/to/N_R2.fastq.gz reads/N_R2.fq.gz

ls -lh reads
```

---

## B) Reference-mode Transrate comparisons (Slurm)

### B1) Folder setup (if not already present)

```bash
cd /nesi/project/uoo03946/transrate_compare
mkdir -p slurm results assemblies reads
ls -ld slurm results assemblies reads
```

### B2) Create the Transrate reference-mode Slurm template

```bash
cat > slurm/transrate_ref.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --time=08:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --output=%x.%j.out
#SBATCH --error=%x.%j.err

module purge
module load Miniconda3
conda activate transrate

WORK=/nesi/project/uoo03946/transrate_compare
cd $WORK

transrate \
  --assembly ASSEMBLY \
  --reference REF \
  --threads 8 \
  --output OUTDIR
EOF

ls -lh slurm/transrate_ref.sl
head -n 5 slurm/transrate_ref.sl
```

### B3) Confirm assemblies are linked (important)

```bash
cd /nesi/project/uoo03946/transrate_compare
ls -lh assemblies

# Optional: verify targets resolve
readlink -f assemblies/L.fasta
readlink -f assemblies/N.fasta
readlink -f assemblies/Hybrid.fasta
```

If not linked correctly, re-link them:

```bash
ln -sf /nesi/project/uoo03946/johsh68p_trinotate/RF_trinity_outputL.Trinity.fasta assemblies/L.fasta
ln -sf /nesi/project/uoo03946/johsh68p_trinotate/RF_trinity_outputN.Trinity.fasta assemblies/N.fasta
ln -sf /nesi/project/uoo03946/denovo_transcriptome/RF_trinity_output.Trinity.fasta assemblies/Hybrid.fasta
```

---

### B4) Submit **L vs Hybrid**

Species-specific as `--assembly`; hybrid as `--reference`.

```bash
cd /nesi/project/uoo03946/transrate_compare

sed -e 's|ASSEMBLY|assemblies/L.fasta|' \
    -e 's|REF|assemblies/Hybrid.fasta|' \
    -e 's|OUTDIR|results/ref_L_vs_Hybrid|' \
    slurm/transrate_ref.sl > slurm/ref_L_H.sl

sbatch --job-name=ref_L_H slurm/ref_L_H.sl
```

Monitor:

```bash
squeue -u $USER | head

# replace <JOBID> with the number Slurm prints on submission
tail -n 60 ref_L_H.<JOBID>.out
tail -n 60 ref_L_H.<JOBID>.err
```

---

### B5) The other three reference-mode jobs (single block)

```bash
cd /nesi/project/uoo03946/transrate_compare

# N vs Hybrid
sed -e 's|ASSEMBLY|assemblies/N.fasta|' \
    -e 's|REF|assemblies/Hybrid.fasta|' \
    -e 's|OUTDIR|results/ref_N_vs_Hybrid|' \
    slurm/transrate_ref.sl > slurm/ref_N_H.sl
sbatch --job-name=ref_N_H slurm/ref_N_H.sl

# Hybrid vs L
sed -e 's|ASSEMBLY|assemblies/Hybrid.fasta|' \
    -e 's|REF|assemblies/L.fasta|' \
    -e 's|OUTDIR|results/ref_Hybrid_vs_L|' \
    slurm/transrate_ref.sl > slurm/ref_H_L.sl
sbatch --job-name=ref_H_L slurm/ref_H_L.sl

# Hybrid vs N
sed -e 's|ASSEMBLY|assemblies/Hybrid.fasta|' \
    -e 's|REF|assemblies/N.fasta|' \
    -e 's|OUTDIR|results/ref_Hybrid_vs_N|' \
    slurm/transrate_ref.sl > slurm/ref_H_N.sl
sbatch --job-name=ref_H_N slurm/ref_H_N.sl
```

---

## Notes

- This workflow uses **Transrate reference-mode** (`--assembly` vs `--reference`).
- If you later want read-based metrics, ensure reads are linked in `reads/` and use the appropriate Transrate options.
- Transrate is older software; for long-term reproducibility, record the environment/module versions you use.
