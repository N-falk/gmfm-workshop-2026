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

<!-- Add later steps (new tools, tutorial stages) below in the same format. -->
