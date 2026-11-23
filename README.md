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
> stops working, repeat Steps 1-2. Environment recipes are kept in `~/conda-envs/`
> (in `$HOME`, which is not purged).

---

## Step 1 - Install Miniforge (conda + mamba) -

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

## Step 2 - Create a `fastqc` environment and install FastQC into it

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

Result (raw â†’ trimmed):

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

Check: `conda activate multiqc && multiqc --version` â†’ `multiqc, version 1.35`

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

Check: `conda activate kraken2 && kraken2 --version` â†’ `Kraken version 2.17.2`

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

### 9d - Classify both samples with Kraken2 as a Slurm job (NOT RUN)

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

<!-- Add later steps (new tools, tutorial stages) below in the same format. -->
