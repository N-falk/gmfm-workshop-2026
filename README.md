# gmfm-workshop-2026
Command Line Instructions for the Workshop

# GMFM Workshop 2026 - Command Log

Commands to reproduce the setup by hand on Pawsey Setonix.
Run them from a login node unless a step says otherwise. Each step lists what it
does, the commands, and how to check that it worked.

Paths used throughout:

```bash
export WORKDIR=/scratch/pawsey1018/$USER/gmfm_worksop_2026
export CONDA_ROOT=/scratch/pawsey1018/$USER/miniforge3
```

> `/scratch` is purged after 21 days without access. If conda or an environment
> stops working, repeat Steps 1â€“2. Environment recipes are kept in `~/conda-envs/`
> (in `$HOME`, which is not purged).

---

## Step 1 - Install Miniforge (conda + mamba) - 2026-11-23

```bash
cd /scratch/pawsey1018/$USER

# Download the installer (skip if Miniforge3-Linux-x86_64.sh is already here)
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"

# If an older/broken install exists, remove it first
# (a partly purged install fails to reinstall in place)
rm -rf /scratch/pawsey1018/$USER/miniforge3

# Install non-interactively (-b) into /scratch
bash Miniforge3-Linux-x86_64.sh -b -p /scratch/pawsey1018/$USER/miniforge3

# Only needed once: add conda/mamba to ~/.bashrc (already done on this account)
# /scratch/pawsey1018/$USER/miniforge3/bin/conda init bash
# /scratch/pawsey1018/$USER/miniforge3/bin/mamba shell init --shell bash --root-prefix /scratch/pawsey1018/$USER/miniforge3

# Reload the shell so conda is on PATH
source ~/.bashrc
```

Check that it worked:

```bash
conda --version    # conda 26.9.4
mamba --version    # 2.9.0
```

---

## Step 2 - Create a `fastqc` environment and install FastQC into it - 2026-11-23

One command creates a new environment named `fastqc` **and** installs the FastQC
tool into it:

```bash
mamba create -y -n fastqc -c conda-forge -c bioconda fastqc
```

- `create -n fastqc` - make a new environment called `fastqc`
- `-c conda-forge -c bioconda` - channels to download packages from (bioconda hosts most bioinformatics tools)
- `fastqc` (last word) - the package to install into the new environment
- `-y` - don't ask for confirmation

Equivalent two-step version (create an empty environment, then install FastQC into it).
Use `mamba install` this way to add more tools to an existing environment later:

```bash
mamba create -y -n fastqc
conda activate fastqc
mamba install -y -c conda-forge -c bioconda fastqc
conda deactivate
```

Save a recipe so the environment can be rebuilt after a purge:

```bash
mkdir -p ~/conda-envs
cat > ~/conda-envs/fastqc.yml <<'EOF'
name: fastqc
channels:
  - conda-forge
  - bioconda
dependencies:
  - fastqc
EOF

# To rebuild later:
# mamba env create -f ~/conda-envs/fastqc.yml
```

Check that it worked:

```bash
conda activate fastqc
fastqc --version   # FastQC v0.13.0
conda deactivate
```

If `conda activate` fails (e.g. "CondaError: Run 'conda init' before 'conda activate'",
or `conda: command not found`), the current shell hasn't loaded conda. Either open a new
terminal, or load it by hand:

```bash
source /scratch/pawsey1018/$USER/miniforge3/etc/profile.d/conda.sh
conda activate fastqc
```

The same `source .../conda.sh` line is needed at the top of Slurm job scripts,
because batch jobs don't read `~/.bashrc`.

---

## Step 3 - Download the raw reads - 2026-11-18

Source: Zenodo record 3977236 (URL also saved in the `file name` file).

```bash
mkdir -p $WORKDIR/raw_data
cd $WORKDIR/raw_data
wget https://zenodo.org/record/3977236/files/female_oral2.fastq-4143.gz
cd $WORKDIR
```

Check that it worked:

```bash
ls -lh raw_data/                                   # female_oral2.fastq-4143.gz, ~129K
md5sum raw_data/female_oral2.fastq-4143.gz         # d6031015b00efe22e9023191c017b5b8
zcat raw_data/female_oral2.fastq-4143.gz | head -4 # first read (4 lines)
echo $(( $(zcat raw_data/female_oral2.fastq-4143.gz | wc -l) / 4 ))   # 812 reads
```

---

## Step 4 - Run FastQC on the raw reads

Input: `raw_data/female_oral2.fastq-4143.gz`

```bash
cd $WORKDIR
mkdir -p qc/fastqc_raw
conda activate fastqc
fastqc -t 2 --outdir qc/fastqc_raw raw_data/*.gz
conda deactivate
```

Notes:
- FastQC v0.13.0 only accepts `--outdir`; the short `-o` used in older tutorials gives
  `fastqc: error: unrecognized arguments: -o`.
- The output directory must already exist (hence the `mkdir -p`), otherwise FastQC
  stops with `[ERROR] Output dir qc/fastqc_raw does not exist`.

Outputs: `qc/fastqc_raw/*_fastqc.html` (open in a browser) and `*_fastqc.zip`.

_Status: run 2026-11-23 - produced `qc/fastqc_raw/female_oral2.fastq-4143_fastqc.html` and `.zip`._

---

## Step 5 - Create a `cutadapt` environment and install Cutadapt into it - 2026-11-23

Cutadapt removes adapter sequences and low-quality ends from reads. It gets its own
environment, the same way as FastQC in Step 2:

```bash
mamba create -y -n cutadapt -c conda-forge -c bioconda cutadapt
```

Save the recipe for rebuilding after a purge:

```bash
cat > ~/conda-envs/cutadapt.yml <<'EOF'
name: cutadapt
channels:
  - conda-forge
  - bioconda
dependencies:
  - cutadapt
EOF

# To rebuild later:
# mamba env create -f ~/conda-envs/cutadapt.yml
```

Check that it worked:

```bash
conda activate cutadapt
cutadapt --version   # 5.2
conda deactivate
```

_Status: installed. Trimming not yet run (adapter sequence/settings to come from the tutorial)._

---

## Step 6 - Trim adapters and low-quality bases with Cutadapt - 2026-11-23

Parameters come from `cutadapt_parameters.txt`:

- `-a CTGTCTCTTATACACATCT` - remove the Nextera/Illumina adapter from the 3' end of reads
- `-q 20` - trim bases with quality below 20 from the 3' end
- `-m 20` - throw away reads shorter than 20 bp after trimming
- `--report minimal` - write a one-line, tab-separated summary instead of the long report

The template uses `raw_data/{sample}.fastq.gz`, but our file is named
`female_oral2.fastq-4143.gz`, so the input file is given directly and `female_oral2`
is used as the sample name for the outputs:

```bash
cd $WORKDIR
mkdir -p qc/trimmed
conda activate cutadapt

SAMPLE=female_oral2
cutadapt \
  -a CTGTCTCTTATACACATCT \
  -q 20 \
  -m 20 \
  -o qc/trimmed/${SAMPLE}_trimmed.fastq.gz \
  --report minimal \
  raw_data/female_oral2.fastq-4143.gz \
  > qc/trimmed/${SAMPLE}_cutadapt_report.txt

conda deactivate
```

Check that it worked:

```bash
ls -l qc/trimmed/
column -t qc/trimmed/female_oral2_cutadapt_report.txt
```

Result:

| status | in_reads | in_bp | too_short | out_reads | w/adapters | qualtrim_bp | out_bp |
|---|---|---|---|---|---|---|---|
| WARN | 812 | 240,352 | 0 | 812 | 461 | 84,277 | 116,483 |

All 812 reads kept; 461 (57%) had adapter removed; about half the bases were trimmed.
`WARN` is Cutadapt's "adapter may be incomplete at its 5' end" warning (the adapter is
almost always preceded by `G`). This is common for this Nextera adapter and the trimming
is still valid. To see the full message, run the same command without
`--report minimal` (and without the `>` redirect).

_Status: run 2026-11-23 - produced `qc/trimmed/female_oral2_trimmed.fastq.gz` and `female_oral2_cutadapt_report.txt`._

---

## Step 7 - Run FastQC on the trimmed reads - 2026-11-23

Same as Step 4, but on Cutadapt's output, into a separate folder so the raw and
trimmed reports can be compared:

```bash
cd $WORKDIR
mkdir -p qc/fastqc_trimmed
conda activate fastqc
fastqc -t 2 --outdir qc/fastqc_trimmed qc/trimmed/*_trimmed.fastq.gz
conda deactivate
```

Compare the PASS/WARN/FAIL summary of raw vs trimmed (reads it straight from the zip files):

```bash
for d in raw trimmed; do
  echo "== $d"
  unzip -p qc/fastqc_$d/*.zip '*/summary.txt' | cut -f1,2
done
```

Result (raw → trimmed):

| Module | Raw | Trimmed |
|---|---|---|
| Per base sequence quality | FAIL | **PASS** |
| Per sequence quality scores | WARN | **PASS** |
| Adapter Content | FAIL | **PASS** |
| Sequence Length Distribution | PASS | WARN (expected - reads are now 26â€“296 bp) |
| Per tile sequence quality | WARN | FAIL |
| Per base sequence content, Per sequence GC, Duplication, Overrepresented seqs | FAIL | FAIL |

Trimming removed the adapter and the low-quality read ends. The remaining FAILs are
normal for a small (812-read) amplicon-style/metagenomic test file and are not
something trimming is meant to fix.

_Status: run 2026-11-23 - produced `qc/fastqc_trimmed/female_oral2_trimmed_fastqc.html` and `.zip`._

---

## Step 8 - Install MultiQC and combine all QC results - 2026-11-23

_Extra step, not part of the Galaxy tutorial._ MultiQC collects the FastQC and
Cutadapt outputs into a single HTML report.

### 8a - Install MultiQC in its own environment

```bash
mamba create -y -n multiqc -c conda-forge -c bioconda multiqc

cat > ~/conda-envs/multiqc.yml <<'EOF'
name: multiqc
channels:
  - conda-forge
  - bioconda
dependencies:
  - multiqc
EOF
```

Check: `conda activate multiqc && multiqc --version` → `multiqc, version 1.35`

### 8b - Write a full Cutadapt log for MultiQC

MultiQC can't read the one-line `--report minimal` file from Step 6. Re-run Cutadapt
with the same parameters but the normal (full) report, sending the reads to
`/dev/null` so the trimmed file from Step 6 is not overwritten:

```bash
cd $WORKDIR
conda activate cutadapt
cutadapt -a CTGTCTCTTATACACATCT -q 20 -m 20 \
  -o /dev/null \
  raw_data/female_oral2.fastq-4143.gz \
  > qc/trimmed/female_oral2_cutadapt_full.log
conda deactivate
```

### 8c - Run MultiQC on the whole `qc/` folder

```bash
cd $WORKDIR
conda activate multiqc
mkdir -p qc/multiqc
multiqc qc/ -d -dd 1 -o qc/multiqc --force
conda deactivate
```

- `qc/` - folder to search for tool outputs
- `-d -dd 1` - prefix each sample name with its folder (`fastqc_raw | â€¦`, `fastqc_trimmed | â€¦`).
  **Needed here:** without it MultiQC trims `.fastq-4143` and `_trimmed` off both names,
  they both become `female_oral2`, and one FastQC report silently replaces the other.
- `-o qc/multiqc` - where to write the report
- `--force` - overwrite an earlier report instead of making `multiqc_report_1.html`

Check: the output should say `cutadapt | Found 1 reports` and `fastqc | Found 2 reports`.

Result (General Statistics table):

| Sample | % adapter-trimmed bp | % GC | Avg length | % duplicates |
|---|---|---|---|---|
| fastqc_raw \| female_oral2 | | 44 | 296 | 63.2 |
| fastqc_trimmed \| female_oral2 | | 54 | 140.7 | 65.9 |
| trimmed \| female_oral2 (Cutadapt) | 51.5 | | | |

_Status: run 2026-11-23 - produced `qc/multiqc/multiqc_report.html` and `qc/multiqc/multiqc_data/`._

---

## Step 9 - Install Kraken2 for taxonomic classification - 2026-11-23

### 9a - Install Kraken2 in its own environment

```bash
mamba create -y -n kraken2 -c conda-forge -c bioconda kraken2

cat > ~/conda-envs/kraken2.yml <<'EOF'
name: kraken2
channels:
  - conda-forge
  - bioconda
dependencies:
  - kraken2
EOF
```

Check: `conda activate kraken2 && kraken2 --version` → `Kraken version 2.17.2`

### 9b - Download the reads for taxonomy (2026-11-23)

Four files from Zenodo record 7871630 (URLs also in the `file name` file, under
"kraken taxonomy"). They are **paired-end**: two samples, `JC1A` and `JP4D`, each with an
R1 (forward) and R2 (reverse) file.

```bash
cd $WORKDIR/raw_data
for s in JC1A JP4D; do
  for r in R1 R2; do
    wget https://zenodo.org/record/7871630/files/${s}_${r}.fastqsanger.gz
  done
done
cd $WORKDIR
```

`.fastqsanger.gz` is Galaxy's name for an ordinary gzipped FASTQ (Sanger/Phred+33
qualities), so tools treat it like `.fastq.gz`.

Check that it worked: R1 and R2 of each sample must have the same number of reads.

```bash
for f in raw_data/J*.fastqsanger.gz; do
  gzip -t "$f" && echo "$f  reads=$(( $(zcat "$f" | wc -l) / 4 ))"
done
md5sum raw_data/J*.fastqsanger.gz
```

| File | Size | Reads | md5 |
|---|---|---|---|
| JC1A_R1.fastqsanger.gz | 21M | 137,139 | d81ab0af18820753e1d57eb3cd793485 |
| JC1A_R2.fastqsanger.gz | 20M | 137,139 | 8d0606ff0f73044eb9786a18c4272d99 |
| JP4D_R1.fastqsanger.gz | 175M | 1,116,049 | cad03d2dbdcc721c82362eb8fa812d17 |
| JP4D_R2.fastqsanger.gz | 191M | 1,116,049 | ed76606b2a30c4221f1a3c5265c7a2b8 |

### 9c - Databases

Pre-built Kraken2 databases are kept centrally on Setonix (no download needed):

| Path | Contents | Size |
|---|---|---|
| `/scratch/references/kraken2/pluspfp_20230605/` | PlusPFP: Standard (bacteria, archaea, viruses, human) + protozoa, fungi, plants | 158 GB |
| `/scratch/references/kraken2/nt_20230502/` | All of NCBI nt | 516 GB |

`--db` takes the **folder** (containing `hash.k2d`, `opts.k2d`, `taxo.k2d`), not a file.
Kraken2 loads the whole database into memory, so either database needs a
`highmem` Slurm job (nodes have ~1 TB RAM), not the login node.

### 9d - Classify both samples with Kraken2 as a Slurm job

Database: **PlusPFP** (`/scratch/references/kraken2/pluspfp_20230605`), the smaller of
the two (158 GB vs 516 GB for nt). Kraken2 loads the whole database into memory, so the
job goes to the `highmem` partition with 180 GB. `highmem` allows at most ~7.9 GB per
CPU, so 180 GB needs 24 CPUs.

Both samples run inside **one** job, so the 158 GB database is only read once.

**1. Create the folders** (Slurm won't create the `logs/` folder itself, and the job
fails with no log if it's missing):

```bash
cd $WORKDIR
mkdir -p slurm logs
```

**2. Save the script as `slurm/kraken2.slurm`.** Either paste the script part into a
new file with an editor (`nano slurm/kraken2.slurm`), or copy this whole block into the
terminal, which writes the file for you:

```bash
cat > slurm/kraken2.slurm <<'EOF'
#!/bin/bash
#SBATCH --job-name=kraken2
#SBATCH --account=pawsey1018
#SBATCH --partition=highmem
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=24
#SBATCH --mem=180G
#SBATCH --time=02:00:00
#SBATCH --output=logs/%x_%j.out
#SBATCH --error=logs/%x_%j.err

# Kraken2 taxonomic classification of the paired-end taxonomy reads (SETUP_LOG Step 9).
# Submit from the workshop directory:  sbatch slurm/kraken2.slurm

set -eo pipefail

KRAKEN_DB=/scratch/references/kraken2/pluspfp_20230605
SAMPLES="JC1A JP4D"

# Batch jobs don't read ~/.bashrc, so load conda here
source /scratch/pawsey1018/$USER/miniforge3/etc/profile.d/conda.sh
conda activate kraken2

cd "$SLURM_SUBMIT_DIR"
mkdir -p taxonomy

for SAMPLE in $SAMPLES; do
  echo "$(date '+%F %T')  Classifying $SAMPLE"
  kraken2 \
    --db "$KRAKEN_DB" \
    --threads "$SLURM_CPUS_PER_TASK" \
    --paired \
    --gzip-compressed \
    --report taxonomy/${SAMPLE}_kraken2_report.txt \
    --output taxonomy/${SAMPLE}_kraken2_output.txt \
    raw_data/${SAMPLE}_R1.fastqsanger.gz \
    raw_data/${SAMPLE}_R2.fastqsanger.gz
done

echo "$(date '+%F %T')  Done"
EOF
```

**3. Submit it** from the workshop directory:

```bash
cd $WORKDIR
sbatch slurm/kraken2.slurm
```

**4. Check on it:**

```bash
squeue -u $USER                                   # PD = waiting, R = running
sacct -u $USER -S today --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
cat logs/kraken2_<jobid>.out                      # progress messages
cat logs/kraken2_<jobid>.err                      # Kraken2's per-sample summary (% classified)
```

When it has finished (`COMPLETED`, exit code `0:0`), `taxonomy/` contains, per sample:

- `<sample>_kraken2_report.txt`: summary table, % and number of reads assigned to each taxon
- `<sample>_kraken2_output.txt`: one line per read pair with the taxon it was assigned to

What the options do:

- `--paired` - treat the R1 and R2 files as two ends of the same fragments (each pair gets one classification)
- `--threads "$SLURM_CPUS_PER_TASK"` - use exactly the 24 CPUs the job asked for
- `--gzip-compressed` - input is `.gz` (newer versions detect this automatically; harmless to include)
- To run other samples, change the `SAMPLES="..."` line; files must be named
  `raw_data/<sample>_R1.fastqsanger.gz` and `..._R2.fastqsanger.gz`.

Notes:

- Expect a queue wait. `highmem` has only 16 nodes and pawsey1018 is over its
  allocation (low priority). On 2026-11-23 Slurm estimated a start about a day away.
- Most of the run time is loading the database. Classifying the reads themselves takes
  minutes at most.
- The script was checked with `bash -n` and `sbatch --test-only` (Slurm accepts the
  request) but has not been run.

- `--paired` - treat the two files as R1/R2 of the same fragments (each pair gets one classification)
- `--report` - summary table: % and number of reads assigned to each taxon
- `--output` - one line per read pair with the taxon it was assigned to
- `--gzip-compressed` - input is `.gz` (newer versions detect this automatically; harmless to include)

_Status: run 2026-11-23 as job 53622360 (COMPLETED, exit 0:0, 6 min). Produced
`taxonomy/JC1A_kraken2_report.txt` and `taxonomy/JP4D_kraken2_report.txt`
(classified: JC1A 56.6%, JP4D 30.5%). The per-read `*_kraken2_output.txt` files were
afterwards moved to `taxonomy/output files/`. Peak memory was 180.0 GB, right at the
180G limit. For future runs consider `--mem=200G` with `--cpus-per-task=26`._

---

## Step 10 - Build a taxon-by-sample (OTU) table from the Kraken2 reports - 2026-11-23

Script: `kraken_taxa_tools/kraken2otu2.py` (own script). It reads every Kraken2 report in
a folder and writes `otu_table_<rank>.csv` into that same folder: one row per taxon, one
column per sample. Values are column 2 of the report (reads assigned to that taxon
*and everything below it*), so the genus table counts reads from all species in each genus.

Fixes made to the script on 2026-11-23:

- **Rank matching (bug that made the table empty):** Kraken2 reports write ranks as codes
  (`G`, `S`, `F`, plus sub-ranks like `G1`, `S1`), but the script looked them up by full
  name (`genus`). The file's codes are now converted with the same `RANK_ALIASES`
  table, and codes it doesn't know (`U`, `R`, `G1`, `S2`, ...) are skipped.
- **Column order:** report files are now `sorted()`, so samples always appear in the same order.
- Line 1 (a stray Markdown fence) was commented out by hand to stop a `SyntaxError`.

Before running: the per-read `*_kraken2_output.txt` files were moved into
`taxonomy/output files/`, so only the reports are in `taxonomy/`.

```bash
cd $WORKDIR
python3 kraken_taxa_tools/kraken2otu2.py \
  -i taxonomy \
  -l genus \
  -e _kraken2_report.txt
```

- `-i` - folder containing the reports (the table is also written here)
- `-l` - rank: `genus`/`G`, `species`/`S`, `family`/`F`, `order`/`O`, `class`/`C`, `phylum`/`P`, `superkingdom`/`D`
- `-e _kraken2_report.txt` - only read the report files (not `_output.txt`), and use
  what's left of the file name as the sample name (`JC1A`, `JP4D`)

The script only uses the Python standard library, so any `python3` works (the
conda base environment's Python 3.14 was used).

Check that it worked:

```bash
wc -l taxonomy/otu_table_genus.csv        # 2199 = header + 2198 genera
head -4 taxonomy/otu_table_genus.csv
```

Result (`taxonomy/otu_table_genus.csv`, first rows):

```
otu,JC1A,JP4D
Paracoccus,1521,18096
Cereibacter,391,3608
Rhodobacter,342,2669
```

All 2,198 genera were cross-checked against the reports: 0 mismatches.

_Status: run 2026-11-23 - produced `taxonomy/otu_table_genus.csv`._

---

## Step 11 - Assemble reads into contigs with MEGAHIT - 2026-11-24

### 11a - Install MEGAHIT in its own environment

```bash
mamba create -y -n megahit -c conda-forge -c bioconda megahit

cat > ~/conda-envs/megahit.yml <<'EOF'
name: megahit
channels:
  - conda-forge
  - bioconda
dependencies:
  - megahit
EOF
```

Check: `conda activate megahit && megahit --version` → `MEGAHIT v1.2.9`

### 11b - Download the reads for assembly

Zenodo record 7818827 (URLs in the `file name` file, under "MEGAHIT"). The tutorial lists
6 paired-end samples (ERR2231567 to ERR2231572). **Only ERR2231567 is used here**: Zenodo
throttled the downloads after a few files (one file took >20 min), so the other samples
were stopped and deleted.

```bash
mkdir -p $WORKDIR/raw_data
cd $WORKDIR/raw_data
for r in 1 2; do
  wget https://zenodo.org/record/7818827/files/ERR2231567_${r}.fastqsanger.gz
done
cd $WORKDIR

# All 6 samples, if you have time (~200 MB per file; can be slow):
# for s in ERR2231567 ERR2231568 ERR2231569 ERR2231570 ERR2231571 ERR2231572; do
#   for r in 1 2; do wget https://zenodo.org/record/7818827/files/${s}_${r}.fastqsanger.gz; done
# done
```

Check that it worked:

```bash
for f in raw_data/ERR2231567_*.fastqsanger.gz; do
  gzip -t "$f" && echo "$f  reads=$(( $(zcat "$f" | wc -l) / 4 ))"
done
md5sum raw_data/ERR2231567_*.fastqsanger.gz
```

| File | Size | Reads | md5 |
|---|---|---|---|
| ERR2231567_1.fastqsanger.gz | 203M | 1,017,705 | 27d2b2dc37ce9811dfb44aafd7ab1719 |
| ERR2231567_2.fastqsanger.gz | 220M | 1,017,705 | 28d3914f4517d210251b18ade4f28caa |

Reads are ~287 bp on average.

### 11c - Assemble with MEGAHIT as a Slurm job

Tutorial settings ("MEGAHIT INSTRUCTIONS" in `file name`):

- **Individual assembly**: each sample is assembled on its own (not pooled)
- **k-mer sizes**: min 21, max 91, step 12 → `--k-min 21 --k-max 91 --k-step 12`.
  MEGAHIT turns this into k = 21, 33, 45, 57, 69, 81, **91** (stepping by 12 stops at
  81, and MEGAHIT adds the max as the last step). Confirmed in a test run.

The script is an **array job**: array task N assembles the Nth sample in `SAMPLES`. It is
set up for the one sample; to do more, add names to `SAMPLES` and change
`--array=1` to e.g. `--array=1-6`, and the samples then run in parallel.

**1. Create the folders:**

```bash
cd $WORKDIR
mkdir -p slurm logs
```

**2. Save the script as `slurm/megahit.slurm`** (paste the whole block into the terminal,
or paste the script part into `nano slurm/megahit.slurm`):

```bash
cat > slurm/megahit.slurm <<'EOF'
#!/bin/bash
#SBATCH --job-name=megahit
#SBATCH --account=pawsey1018
#SBATCH --partition=work
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=28G
#SBATCH --time=04:00:00
#SBATCH --array=1
#SBATCH --output=logs/%x_%A_%a.out
#SBATCH --error=logs/%x_%A_%a.err

# MEGAHIT individual assembly: one array task per sample (SETUP_LOG Step 11).
# Submit from the workshop directory:  sbatch slurm/megahit.slurm

set -eo pipefail

# Array task N assembles the Nth sample in this list (add more names and widen --array)
SAMPLES=(ERR2231567)
SAMPLE=${SAMPLES[$((SLURM_ARRAY_TASK_ID - 1))]}

# Batch jobs don't read ~/.bashrc, so load conda here
source /scratch/pawsey1018/$USER/miniforge3/etc/profile.d/conda.sh
conda activate megahit

cd "$SLURM_SUBMIT_DIR"
mkdir -p assembly

# Give MEGAHIT 90% of the job's memory, in bytes (SLURM_MEM_PER_NODE is in MB)
MEM_BYTES=$(( SLURM_MEM_PER_NODE * 1024 * 1024 * 9 / 10 ))

echo "$(date '+%F %T')  Assembling $SAMPLE (task $SLURM_ARRAY_TASK_ID)"

# MEGAHIT refuses to write into an existing output folder; --force replaces it
megahit \
  -1 raw_data/${SAMPLE}_1.fastqsanger.gz \
  -2 raw_data/${SAMPLE}_2.fastqsanger.gz \
  --k-min 21 \
  --k-max 91 \
  --k-step 12 \
  --num-cpu-threads "$SLURM_CPUS_PER_TASK" \
  --memory "$MEM_BYTES" \
  --out-dir assembly/${SAMPLE} \
  --out-prefix ${SAMPLE} \
  --force

echo "$(date '+%F %T')  Done: assembly/${SAMPLE}/${SAMPLE}.contigs.fa"
EOF
```

**3. Submit and check:**

```bash
cd $WORKDIR
sbatch slurm/megahit.slurm
squeue -u $USER
sacct -u $USER -S today --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
tail logs/megahit_<jobid>_1.err      # MEGAHIT's progress and final contig summary
```

What the options do:

- `-1` / `-2` - forward and reverse reads of the pair
- `--num-cpu-threads` - use the 16 CPUs the job asked for
- `--memory` - a number above 1 is bytes; set to 90% of the 28 GB job so MEGAHIT stays
  inside it (MEGAHIT's default is 90% of the *whole node*, which would exceed the job's share)
- `--out-dir` / `--out-prefix` - results go to `assembly/<sample>/<sample>.contigs.fa`
- `--force` - overwrite an earlier run's output folder instead of stopping with an error
- Contigs shorter than 200 bp are dropped (MEGAHIT's default `--min-contig-len 200`)

Result for ERR2231567 (`assembly/ERR2231567/ERR2231567.contigs.fa`, 114 MB):

| Contigs | Total length | Longest | Mean | N50 |
|---|---|---|---|---|
| 220,147 | 109.8 Mbp | 28,907 bp | 498 bp | 506 bp |

N50 = half of the assembled bases are in contigs at least this long. Short contigs like
these are normal for a metagenome assembled from ~1 M read pairs.

The job peaked at 3.4 GB of memory and took 16 min. `--cpus-per-task=4 --mem=7G` would
probably be enough for one sample of this size, and would likely start sooner.

_Status: run 2026-11-24 as job 53670605 (COMPLETED, exit 0:0, 16 min). ERR2231568 not
downloaded yet: Zenodo throttled the download to ~180 bytes/s._

---

## Step 12 - Assembly quality control with QUAST - 2026-11-24

### 12a - Install QUAST in its own environment

A plain `mamba create -n quast ... quast` does **not** give a working install: the solver
either picks the old QUAST 4.6.3 (crashes with `DistributionNotFound`) or QUAST 5.2 with
Python 3.13 (crashes with `No module named 'distutils'`). Pin both versions:

```bash
mamba create -y -n quast -c conda-forge -c bioconda 'quast>=5.2' 'python=3.10'

cat > ~/conda-envs/quast.yml <<'EOF'
name: quast
channels:
  - conda-forge
  - bioconda
dependencies:
  - quast>=5.2
  - python=3.10
EOF
```

Check: `conda activate quast && quast.py --version` → `QUAST v5.3.0`

### 12b - Run QUAST on the MEGAHIT contigs

No reference genome (it's a metagenome), so QUAST reports size/contiguity statistics.
It only takes ~25 s, so the login node is fine.

```bash
cd $WORKDIR
conda activate quast
quast.py \
  -t 4 \
  -o assembly_qc/quast_ERR2231567 \
  assembly/ERR2231567/ERR2231567.contigs.fa
conda deactivate
```

- `-t 4` - threads
- `-o` - output folder
- QUAST's default `--min-contig 500`: most statistics only count contigs (≥) 500 bp (rows
  marked `>= 0 bp` include all contigs). MEGAHIT kept contigs (≥) 200 bp, so QUAST's N50
  (907) is higher than MEGAHIT's own (506).

Check that it worked:

```bash
cat assembly_qc/quast_ERR2231567/report.txt
```

Outputs: `report.txt`/`.tsv`/`.html`/`.pdf` (summary table) and `icarus.html` (contig viewer).

Result (contigs (≥) 500 bp unless noted):

| Metric | Value |
|---|---|
| # contigs (all, (≥) 0 bp) | 220,147 |
| # contigs ((≥) 500 bp) | 58,252 |
| # contigs (≥) 1 kb / (≥) 5 kb / (≥) 10 kb / (≥) 25 kb | 12,123 / 676 / 135 / 3 |
| Total length (all) | 109.8 Mbp |
| Total length ((≥) 500 bp) | 55.7 Mbp |
| Largest contig | 28,907 bp |
| N50 / N90 | 907 / 547 bp |
| L50 | 14,820 contigs |
| GC | 39.37 % |
| N's per 100 kbp | 0.00 |

N50 = half of the assembled bases are in contigs at least this long; L50 = how many
contigs that takes.

_Status: run 2026-11-24 - produced `assembly_qc/quast_ERR2231567/` (no warnings or errors)._

---

## Step 13 - Binning, part 1: map reads back to the contigs (Bowtie2 + SAMtools) - 2026-11-24

MetaBAT 2 groups contigs into bins using how deeply each contig is covered by reads, so
the reads first have to be mapped to the contigs. This step makes the sorted BAM file it
needs. All of it runs in seconds on these small tutorial files, so the login node is fine.

### 13a - Install Bowtie2 and SAMtools (one environment each)

```bash
mamba create -y -n bowtie2  -c conda-forge -c bioconda bowtie2
mamba create -y -n samtools -c conda-forge -c bioconda samtools

for t in bowtie2 samtools; do
  printf "name: $t\nchannels:\n  - conda-forge\n  - bioconda\ndependencies:\n  - $t\n" > ~/conda-envs/$t.yml
done
```

Check: `bowtie2 --version` → 2.5.5 (in env `bowtie2`); `samtools --version` → 1.25 (in env `samtools`)

### 13b - Download the binning input files

Zenodo record 17661262 (Galaxy tutorial "Binning of metagenomic sequencing data"; URLs in
`file name` under "BINNING"). These are small, pre-made tutorial files: the contigs and
reads here are **not** the ERR2231567 assembly from Step 11.

The `zenodo.org/record/...` links in `file name` were being throttled to under 30 bytes/s,
so the files were fetched through Zenodo's API links instead, which downloaded at normal
speed:

```bash
mkdir -p $WORKDIR/binning/input
cd $WORKDIR/binning/input
for f in MEGAHIT_contigs.fasta reads_forward.fastqsanger.gz reads_reverse.fastqsanger.gz; do
  curl -L -o $f https://zenodo.org/api/records/17661262/files/$f/content
done
cd $WORKDIR

# Equivalent normal links (use these if Zenodo isn't throttling):
# wget https://zenodo.org/record/17661262/files/MEGAHIT_contigs.fasta  (etc.)
```

Check (checksums match those published on Zenodo):

```bash
md5sum binning/input/*
```

| File | Size | Contents | md5 |
|---|---|---|---|
| MEGAHIT_contigs.fasta | 262 KB | 63 contigs | 5913c7983587626f84551e642b1469dd |
| reads_forward.fastqsanger.gz | 1.3 MB | 9,462 reads | 0f5988c910dda3bcb049d25785ac2d79 |
| reads_reverse.fastqsanger.gz | 1.3 MB | 9,462 reads | 8f265f43b1c853a981a71b03a5548bb0 |

### 13c - Index the contigs, map the reads, sort the alignments

Commands from `file name`, with the tutorial's placeholder file names (`contigs.fasta`,
`reads_R1.fastq.gz`, `reads_R2.fastq.gz`) replaced by the real ones. Run inside `binning/`:

```bash
cd $WORKDIR/binning

# 1. Build a Bowtie2 index of the contigs (writes contigs_index.*.bt2)
conda activate bowtie2
bowtie2-build input/MEGAHIT_contigs.fasta contigs_index > bowtie2_build.log 2>&1

# 2. Map the read pairs to the contigs
bowtie2 \
  -x contigs_index \
  -1 input/reads_forward.fastqsanger.gz \
  -2 input/reads_reverse.fastqsanger.gz \
  -S output.sam \
  --no-unal \
  2> mapping_stats.txt
conda deactivate

# 3. Sort the alignments by position into a BAM file
conda activate samtools
samtools sort -o output_sorted.bam output.sam
conda deactivate
```

- `-x contigs_index` - the index built in step 1
- `-1` / `-2` - forward and reverse reads
- `-S output.sam` - write alignments as SAM (plain text)
- `--no-unal` - leave reads that didn't map out of the SAM file
- `2> mapping_stats.txt` - Bowtie2 prints its alignment summary on stderr; save it
- `samtools sort -o ...` - sort by contig position and write compressed BAM

**Fix to the tutorial instructions:** they say `samtools sort -o output_sorted.bam output.bam`,
but Bowtie2 wrote `output.sam`, and no `output.bam` exists. `samtools sort` reads SAM
directly, so the input is `output.sam` (no separate SAM→BAM conversion is needed).

Check that it worked:

```bash
cat mapping_stats.txt                       # overall alignment rate
samtools quickcheck output_sorted.bam && echo "BAM ok"
samtools flagstat output_sorted.bam
```

Result: **99.51% overall alignment rate**. 9,462 read pairs; 98.80% aligned concordantly
exactly once, 0.25% more than once, and 93 single reads did not align. The sorted BAM holds
18,831 mapped reads.

Outputs in `binning/`: `output_sorted.bam` (input for MetaBAT 2), `mapping_stats.txt`,
`output.sam` (intermediate; can be deleted), `contigs_index.*.bt2`.

_Status: run 2026-11-24. MetaBAT 2 run in Step 14._

---

## Step 14 - Binning, part 2: group contigs into bins with MetaBAT 2 - 2026-11-24

Uses the contigs and the sorted BAM from Step 13. Default MetaBAT 2 settings.

### 14a - Install MetaBAT 2 in its own environment

```bash
mamba create -y -n metabat2 -c conda-forge -c bioconda metabat2

cat > ~/conda-envs/metabat2.yml <<'EOF'
name: metabat2
channels:
  - conda-forge
  - bioconda
dependencies:
  - metabat2
EOF
```

Check: `conda activate metabat2 && metabat2 --help | head -2` → version 2:2.18. The
environment also contains `jgi_summarize_bam_contig_depths`, used below.

### 14b - Calculate per-contig coverage, then bin

```bash
cd $WORKDIR/binning
conda activate metabat2

# 1. Average read depth of every contig, from the sorted BAM
jgi_summarize_bam_contig_depths --outputDepth depth.txt output_sorted.bam

# 2. Bin the contigs, using their sequence composition and depth
mkdir -p metabat2_bins
metabat2 \
  -i input/MEGAHIT_contigs.fasta \
  -a depth.txt \
  -o metabat2_bins/bin \
  -t 4

conda deactivate
```

- `--outputDepth depth.txt` - table of contig name, length and average depth
- `-i` - the contigs to bin
- `-a depth.txt` - the depth table from step 1
- `-o metabat2_bins/bin` - prefix for output: bins are written as `metabat2_bins/bin.1.fa`, `bin.2.fa`, ...
- `-t 4` - threads. **Use this on the login node.** Without it MetaBAT 2 uses every
  CPU it can see (the 2026-11-24 run used 247 threads without `-t`). It made no
  difference here because the run took under a second, but on larger data it would
  overload the shared login node.

Defaults used (printed at the top of the MetaBAT 2 output): contigs shorter than
**2,500 bp are not binned** (`minContig 2500`), and bins smaller than **200,000 bp are not
written** (`minClsSize 200000`).

Check that it worked:

```bash
ls metabat2_bins/
cat metabat2_bins/bin.BinInfo.txt
```

Result:

| Bin | Contigs | Total length | Avg coverage |
|---|---|---|---|
| bin.1.fa | 44 | 223,330 bp | 4.85Ã— |

Of the 63 contigs, 50 are (≥) 2,500 bp; MetaBAT 2 placed 44 of them (90.5% of the
long-contig bases) in a single bin. Only one bin is expected: the tutorial data only
totals ~266 kb, so at most one bin can pass the 200 kb minimum bin size. For comparison,
the Zenodo record also contains the tutorial's own MetaBAT output (`metabat_0.fasta`,
one bin).

Outputs in `binning/`: `depth.txt`, `metabat2_bins/bin.1.fa`, `bin.BinInfo.txt` (bin
summary), `bin.BinMembers.txt` (which contigs are in which bin).

_Status: run 2026-11-24 - 1 bin produced._

---

## Step 15 - Check bin quality with CheckM2 - 2026-11-24

CheckM2 estimates how **complete** a bin is (how much of a full genome it contains) and how
**contaminated** it is (how much comes from other organisms), using machine-learning
models and a protein database.

### 15a - Install CheckM2 in its own environment

Pin scikit-learn to 1.6.1. CheckM2 1.1.0's models were saved with that version. With the
newer scikit-learn that conda installs by default (1.9.1), CheckM2 prints
`InconsistentVersionWarning ... might lead to breaking code or invalid results`.

```bash
mamba create -y -n checkm2 -c conda-forge -c bioconda checkm2 'scikit-learn=1.6.1'

cat > ~/conda-envs/checkm2.yml <<'EOF'
name: checkm2
channels:
  - conda-forge
  - bioconda
dependencies:
  - checkm2
  - scikit-learn=1.6.1
EOF
```

Check:

```bash
conda activate checkm2
checkm2 --version                                       # 1.1.0
python -c 'import sklearn; print(sklearn.__version__)'  # 1.6.1
```

(On 2026-11-24 the environment was first made without the pin, then fixed with
`mamba install -n checkm2 'scikit-learn=1.6.1'`. Results were identical either way.)

### 15b - Database

CheckM2 needs its DIAMOND database (`uniref100.KO.1.dmnd`, 3.1 GB). **Don't download it**
(`checkm2 database --download` fetches it from Zenodo, which was throttling). Setonix
has a central copy, from the VEBA database set (v9, Aug 2025):

```
/scratch/references/VEBA-DB_v9/Classify/CheckM2/uniref100.KO.1.dmnd
```

### 15c - Run CheckM2 on the MetaBAT 2 bin

```bash
cd $WORKDIR/binning
conda activate checkm2
checkm2 predict \
  --threads 4 \
  --input metabat2_bins \
  --extension .fa \
  --output-directory checkm2 \
  --database_path /scratch/references/VEBA-DB_v9/Classify/CheckM2/uniref100.KO.1.dmnd
conda deactivate
```

- `--input metabat2_bins` with `--extension .fa` - check every `.fa` file in that folder (here just `bin.1.fa`)
- `--output-directory` - results folder (CheckM2 creates it)
- `--database_path` - the central database above
- `--threads 4` - keeps the shared login node usable; took ~1 min for one bin. For many
  bins, run it as a Slurm job.

Check that it worked: the last log line is `CheckM2 finished successfully.`, then:

```bash
column -t -s$'\t' checkm2/quality_report.tsv
```

Result (`binning/checkm2/quality_report.tsv`):

| Bin | Completeness | Contamination | Genome size | Contigs | Contig N50 | GC | Genes | Coding density |
|---|---|---|---|---|---|---|---|---|
| bin.1 | 13.62% | 0.0% | 223,330 bp | 44 | 5,126 bp | 37% | 244 | 0.92 |

The bin is clean (no contamination) but holds only ~14% of a genome. That's expected:
it's only 223 kb, and a bacterial genome is typically 2â€“6 Mb. The tutorial data is a small
subset. By the usual MIMAG standard this is a **low-quality** bin (high quality is (≥) 90%
complete and < 5% contaminated; medium is (≥) 50% and < 10%).

_Status: run 2026-11-24 - produced `binning/checkm2/quality_report.tsv`._

---

<!-- Add later steps (new tools, tutorial stages) below in the same format. -->

