# BUSCO (NeSI) — scripts, checking output, and summary interpretation

This file contains:
- Slurm scripts used for BUSCO runs (Hybrid, L, N)
- Commands to check job outputs and BUSCO summaries
- The BUSCO results (L, N, Hybrid) and a reviewer-ready interpretation paragraph
- A brief note on typical major-revision deadlines

---

## 1) Set working directory (recommended)

```bash
cd /nesi/project/uoo03946/transrate_compare
pwd
mkdir -p slurm
```

---

## 2) BUSCO Slurm scripts (Hybrid, L, N)

These scripts match the same settings:
- BUSCO v5.8.2
- lineage: `actinopterygii_odb10`
- mode: `transcriptome`
- CPUs: 8
- MEM: 32G
- outputs written to: `/nesi/project/uoo03946/busco/`

### 2.1 Hybrid BUSCO script

Create the script:

```bash
cat > slurm/busco_hybrid.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --time=06:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --job-name=busco_hybrid
#SBATCH --output=busco_hybrid.%j.out
#SBATCH --error=busco_hybrid.%j.err

source /etc/profile

module --force purge
module purge

module load NeSI/zen3
module load BUSCO/5.8.2-gimkl-2022a

WORK=/nesi/project/uoo03946/busco
mkdir -p "$WORK"
cd "$WORK"

busco \
  -i /nesi/project/uoo03946/transrate_compare/assemblies/Hybrid.fasta \
  -l actinopterygii_odb10 \
  -o hybrid_busco \
  -m transcriptome \
  -c 8
EOF
```

Submit:

```bash
sbatch slurm/busco_hybrid.sl
```

### 2.2 L BUSCO script

Create the script:

```bash
cat > slurm/busco_L.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --time=06:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --job-name=busco_L
#SBATCH --output=busco_L.%j.out
#SBATCH --error=busco_L.%j.err

source /etc/profile

module --force purge
module purge

module load NeSI/zen3
module load BUSCO/5.8.2-gimkl-2022a

WORK=/nesi/project/uoo03946/busco
mkdir -p "$WORK"
cd "$WORK"

busco \
  -i /nesi/project/uoo03946/transrate_compare/assemblies/L.fasta \
  -l actinopterygii_odb10 \
  -o L_busco \
  -m transcriptome \
  -c 8
EOF
```

Submit:

```bash
sbatch slurm/busco_L.sl
```

### 2.3 N BUSCO script

Create the script:

```bash
cat > slurm/busco_N.sl <<'EOF'
#!/bin/bash -e
#SBATCH --account=uoo03946
#SBATCH --time=06:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --job-name=busco_N
#SBATCH --output=busco_N.%j.out
#SBATCH --error=busco_N.%j.err

source /etc/profile

module --force purge
module purge

module load NeSI/zen3
module load BUSCO/5.8.2-gimkl-2022a

WORK=/nesi/project/uoo03946/busco
mkdir -p "$WORK"
cd "$WORK"

busco \
  -i /nesi/project/uoo03946/transrate_compare/assemblies/N.fasta \
  -l actinopterygii_odb10 \
  -o N_busco \
  -m transcriptome \
  -c 8
EOF
```

Submit:

```bash
sbatch slurm/busco_N.sl
```

---

## 3) Checking BUSCO output and summaries

### 3.1 Confirm folders exist

```bash
cd /nesi/project/uoo03946/busco
ls -lh
```

You should see something like:
- `L_busco/`
- `N_busco/`
- `hybrid_busco/`

### 3.2 Print the key BUSCO summary lines

```bash
cd /nesi/project/uoo03946/busco

echo "=== L ==="
grep -H "C:" L_busco/short_summary*.txt

echo "=== N ==="
grep -H "C:" N_busco/short_summary*.txt

echo "=== Hybrid ==="
grep -H "C:" hybrid_busco/short_summary*.txt
```

### 3.3 If `short_summary*.txt` isn’t found

```bash
find L_busco -maxdepth 3 -name "short_summary*.txt" -type f
find N_busco -maxdepth 3 -name "short_summary*.txt" -type f
find hybrid_busco -maxdepth 3 -name "short_summary*.txt" -type f
```

### 3.4 Quick “did it finish?” check (logs)

```bash
tail -n 20 L_busco/logs/busco.log
tail -n 20 N_busco/logs/busco.log
tail -n 20 hybrid_busco/logs/busco.log
```

---

## 4) BUSCO results (from grep output)

```text
L:      C:80.1%[S:34.6%,D:45.4%],F:5.7%,M:14.2%,n:3640
N:      C:79.8%[S:39.0%,D:40.8%],F:5.2%,M:15.0%,n:3640
Hybrid: C:83.0%[S:26.4%,D:56.6%],F:5.7%,M:11.2%,n:3640
```

### 4.1 Interpretation

- **Hybrid is more complete**: **+2.9%** vs L (83.0 − 80.1) and **+3.2%** vs N (83.0 − 79.8).
- The improvement is mainly from **fewer missing BUSCOs**:
  - Missing drops from **14.2–15.0%** down to **11.2%** (a ~3–4% absolute reduction).
- **Duplicated BUSCOs (D) are higher in Hybrid** (56.6%), which is common for transcriptomes and especially combined/hybrid assemblies (more isoforms/allelic variants retained).

