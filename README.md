# gmfm-workshop-2026
Command Line Instructions for the Workshop

# GMFM Workshop 2026 â€” Command Log

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

## Step 1 â€” Install Miniforge (conda + mamba) â€” 2026-11-23

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

## Step 2 â€” Create a `fastqc` environment and install FastQC into it â€” 2026-11-23

One command creates a new environment named `fastqc` **and** installs the FastQC
tool into it:

```bash
mamba create -y -n fastqc -c conda-forge -c bioconda fastqc
```

- `create -n fastqc` â€” make a new environment called `fastqc`
- `-c conda-forge -c bioconda` â€” channels to download packages from (bioconda hosts most bioinformatics tools)
- `fastqc` (last word) â€” the package to install into the new environment
- `-y` â€” don't ask for confirmation

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

## Step 3 â€” Download the raw reads â€” 2026-11-18

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

## Step 4 â€” Run FastQC on the raw reads

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

_Status: run 2026-11-23 â€” produced `qc/fastqc_raw/female_oral2.fastq-4143_fastqc.html` and `.zip`._

---

## Step 5 â€” Create a `cutadapt` environment and install Cutadapt into it â€” 2026-11-23

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

## Step 6 â€” Trim adapters and low-quality bases with Cutadapt â€” 2026-11-23

Parameters come from `cutadapt_parameters.txt`:

- `-a CTGTCTCTTATACACATCT` â€” remove the Nextera/Illumina adapter from the 3' end of reads
- `-q 20` â€” trim bases with quality below 20 from the 3' end
- `-m 20` â€” throw away reads shorter than 20 bp after trimming
- `--report minimal` â€” write a one-line, tab-separated summary instead of the long report

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

_Status: run 2026-11-23 â€” produced `qc/trimmed/female_oral2_trimmed.fastq.gz` and `female_oral2_cutadapt_report.txt`._

---

## Step 7 â€” Run FastQC on the trimmed reads â€” 2026-11-23

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
| Sequence Length Distribution | PASS | WARN (expected â€” reads are now 26â€“296 bp) |
| Per tile sequence quality | WARN | FAIL |
| Per base sequence content, Per sequence GC, Duplication, Overrepresented seqs | FAIL | FAIL |

Trimming removed the adapter and the low-quality read ends. The remaining FAILs are
normal for a small (812-read) amplicon-style/metagenomic test file and are not
something trimming is meant to fix.

_Status: run 2026-11-23 â€” produced `qc/fastqc_trimmed/female_oral2_trimmed_fastqc.html` and `.zip`._

---

## Step 8 â€” Install MultiQC and combine all QC results â€” 2026-11-23

_Extra step, not part of the Galaxy tutorial._ MultiQC collects the FastQC and
Cutadapt outputs into a single HTML report.

### 8a â€” Install MultiQC in its own environment

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

### 8b â€” Write a full Cutadapt log for MultiQC

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

### 8c â€” Run MultiQC on the whole `qc/` folder

```bash
cd $WORKDIR
conda activate multiqc
mkdir -p qc/multiqc
multiqc qc/ -d -dd 1 -o qc/multiqc --force
conda deactivate
```

- `qc/` â€” folder to search for tool outputs
- `-d -dd 1` â€” prefix each sample name with its folder (`fastqc_raw | â€¦`, `fastqc_trimmed | â€¦`).
  **Needed here:** without it MultiQC trims `.fastq-4143` and `_trimmed` off both names,
  they both become `female_oral2`, and one FastQC report silently replaces the other.
- `-o qc/multiqc` â€” where to write the report
- `--force` â€” overwrite an earlier report instead of making `multiqc_report_1.html`

Check: the output should say `cutadapt | Found 1 reports` and `fastqc | Found 2 reports`.

Result (General Statistics table):

| Sample | % adapter-trimmed bp | % GC | Avg length | % duplicates |
|---|---|---|---|---|
| fastqc_raw \| female_oral2 | | 44 | 296 | 63.2 |
| fastqc_trimmed \| female_oral2 | | 54 | 140.7 | 65.9 |
| trimmed \| female_oral2 (Cutadapt) | 51.5 | | | |

_Status: run 2026-11-23 â€” produced `qc/multiqc/multiqc_report.html` and `qc/multiqc/multiqc_data/`._

---

<!-- Add later steps (new tools, tutorial stages) below in the same format. -->
