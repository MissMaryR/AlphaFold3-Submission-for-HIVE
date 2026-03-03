# Instructions for AlphaFold3 Submission on HIVE

This guide walks through submitting AlphaFold3 (AF3) structure prediction jobs on the HIVE HPC cluster using a bulk submission script. This is useful when you need a predicted enzyme structure that isn't available in UniProt or RCSB PDB — the resulting model can feed directly into the [Rosetta docking pipeline](https://github.com/MissMaryR/Rosetta-Docking).

---

## Overview

| Step | What You're Doing |
|------|-------------------|
| 1 | Set up JSON input files for your sequence(s) |
| 2 | Upload files to HIVE and submit as a SLURM array job |
| 3 | Retrieve output, open in PyMOL, save as PDB, and relax for Rosetta |

---

## Files Included

| File | Description |
|------|-------------|
| `submit_af3_bulk.py` | Bulk submission script — finds all JSONs in a directory and submits as a SLURM array job |
| `monomer.json` | Template for a single-chain protein prediction |
| `trimer.json` | Template for a homotrimeric (3-chain) protein prediction |

---

## 1) Set Up Your JSON Files

AF3 takes JSON files as input. Two templates are provided — choose based on your system.

### Monomer (single chain)

```json
{
  "name": "monomer",
  "sequences": [
    {
      "protein": {
        "id": ["A"],
        "sequence": "YOUR_SEQUENCE_HERE"
      }
    }
  ],
  "modelSeeds": [1],
  "dialect": "alphafold3",
  "version": 1
}
```

### Multimer — example: homotrimer (3 identical chains)

```json
{
  "name": "trimer",
  "sequences": [
    {
      "protein": {
        "id": ["A", "B", "C"],
        "sequence": "YOUR_SEQUENCE_HERE"
      }
    }
  ],
  "modelSeeds": [1],
  "dialect": "alphafold3",
  "version": 1
}
```

> 💡 **Tips for setting up JSON files:**
> - Replace the `"sequence"` value with your protein's amino acid sequence
> - Change `"name"` to something descriptive — this becomes the output subfolder name
> - For a **monomer**, use `"id": ["A"]`
> - For a **multimer**, list all chain IDs: e.g., `["A", "B"]` for a dimer, `["A", "B", "C"]` for a trimer
> - If all chains are identical (homomultimer), you only need one sequence entry — just list all chain IDs
> - `"modelSeeds": [1]` sets the random seed for reproducibility; change this or add more seeds to generate additional models
> - Name each JSON file descriptively (e.g., `myenzyme_monomer.json`, `mycomplex_dimer.json`) — the filename is used to label the output

### Upload files to HIVE

Place all JSON files and `submit_af3_bulk.py` into the same folder, then upload to HIVE:

```bash
scp -r /path/to/local/folder username@hive.hpc.ucdavis.edu:/quobyte/jbsiegelgrp/username
```

---

## 2) Submit All JSONs as a SLURM Array Job

Navigate to the folder containing your files, then run:

```bash
python3 submit_af3_bulk.py /path/to/directory
```

The script will:
1. Find all `.json` files in the specified directory
2. Auto-generate a SLURM array job script (`af3_array_job.sbatch`)
3. Submit the job — each JSON gets its own array task running in parallel

### SLURM job settings (set automatically)

| Setting | Value |
|---------|-------|
| Partition | `low` |
| Time limit | 24 hours |
| CPUs | 8 per task |
| Memory | 64 GB |
| GPU | 1 per task |

### Monitor your job

After submission, the script prints your Job ID along with ready-to-use commands:

```bash
squeue -j <job_id>               # Check overall job status
squeue -j <job_id> -t all        # Check status of all array tasks
scancel <job_id>                 # Cancel the job
tail -f logs/af3_<job_id>_*.txt  # Watch live logs
```

### Output structure

After jobs complete, two items will appear inside your input directory:

| Output | Description |
|--------|-------------|
| `logs/` | SLURM log files per array task — includes runtime, peak VRAM, and GPU utilization |
| `<dirname>_output/` | Prediction results, one subfolder per JSON file |

For example, if your input folder is `myenzymes/`, results appear in `myenzymes/myenzymes_output/` with one subfolder per JSON.

> 💡 Each log file reports **peak VRAM usage**, **average GPU utilization**, and **total runtime** — handy for estimating resources for future jobs.

---

## 3) Retrieve and Use the Model

### Find your model file

Inside each job's output subfolder, look for:
```
monomer_model.cif
```
This `.cif` file is the predicted structure to use.

### Convert to PDB in PyMOL

1. Open `monomer_model.cif` in **PyMOL**
2. Save as PDB: **File → Export Structure → Export Molecule → Save as PDB**

### Next steps

Before using the model in Rosetta, run it through the relaxation pipeline:

➡️ [Relax PDBs for Rosetta](https://github.com/MissMaryR/Relax-pdbs-for-Rosetta)

> ⚠️ Relaxation is important — AF3 models can have clashes or non-ideal geometry that will cause problems during Rosetta docking if not resolved first.

Once relaxed, the PDB is ready to use as your enzyme input in the [Rosetta docking pipeline](https://github.com/MissMaryR/Rosetta-Docking) (Step 3).
