# AlphaFold3 Submission on HIVE

Scripts for submitting AlphaFold3 (AF3) structure prediction jobs on the HIVE HPC cluster. Useful when you need a predicted enzyme structure that isn't available in UniProt or RCSB PDB — the resulting model feeds directly into the [Rosetta docking pipeline](https://github.com/MissMaryR/Rosetta-Docking).

---

## Repository Structure

```
├── submit_af3_bulk.py       # Bulk submission — finds all JSONs in a directory, submits as SLURM array job
├── af3_bulk_pae.py          # Same as above, but also generates PAE plots after each prediction
├── submit_af3_single.sh     # Single-job submission script
└── json/
    ├── monomer.json         # Template: single-chain protein prediction
    ├── trimer.json          # Template: homotrimeric (3-chain) prediction
    └── dock_CL3.json        # Example: protein–ligand docking with cellulose DP3
```

---

## Overview

| Step | What You're Doing |
|------|-------------------|
| 1 | Set up JSON input files for your sequence(s) and ligand(s) |
| 2 | Upload files to HIVE and submit as a SLURM array job |
| 3 | Retrieve output, open in PyMOL, save as PDB, and relax for Rosetta |

---

## 1) Set Up Your JSON Files

AF3 takes JSON files as input. Templates are provided in the `json/` folder — choose based on your system.

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

> 💡 For oligomers, use `af3_bulk_pae.py` to also generate PAE plots.

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

### Protein–Ligand Docking

Add a `ligand` entry with a SMILES string. See `json/dock_CL3.json` for a working example using cellulose DP3.

```json
{
  "name": "my_protein_ligand",
  "sequences": [
    {
      "protein": {
        "id": "A",
        "sequence": "YOUR_SEQUENCE_HERE"
      }
    },
    {
      "ligand": {
        "id": "B",
        "smiles": "YOUR_SMILES_HERE"
      }
    }
  ],
  "modelSeeds": [1],
  "dialect": "alphafold3",
  "version": 1
}
```

#### Oligosaccharide SMILES

| Ligand | Name | SMILES |
|--------|------|--------|
| CL3 | Cellulose DP3 | `OC[C@H]3O[C@@H](O[C@H]2[C@H](O)[C@@H](O)[C@H](O[C@H]1[C@H](O)[C@@H](O)[C@H](O)O[C@@H]1CO)O[C@@H]2CO)[C@H](O)[C@@H](O)[C@@H]3O` |
| CR3 | Curdlan DP3 | `OC[C@H]3O[C@@H](O[C@H]2[C@H](O)[C@@H](CO)O[C@@H](O[C@@H]1[C@@H](O)[C@H](O)O[C@H](CO)[C@H]1O)[C@@H]2O)[C@H](O)[C@@H](O)[C@@H]3O` |
| XY3 | Xylan DP3 | `O[C@@H]3CO[C@@H](O[C@@H]2CO[C@@H](O[C@@H]1CO[C@@H](O)[C@H](O)[C@H]1O)[C@H](O)[C@H]2O)[C@H](O)[C@H]3O` |

> 💡 **Tips for setting up JSON files:**
> - Replace `"sequence"` with your protein's amino acid sequence
> - Change `"name"` to something descriptive — this becomes the output subfolder name
> - For a **monomer**, use `"id": ["A"]`; for a **multimer**, list all chain IDs (e.g., `["A", "B", "C"]`)
> - If all chains are identical (homomultimer), you only need one sequence entry — just list all chain IDs
> - `"modelSeeds": [1]` sets the random seed; add more seeds to generate additional models
> - Name each JSON file descriptively (e.g., `myenzyme_monomer.json`) — the filename labels the output

### Upload to HIVE

Place all JSON files and submission scripts in the same folder, then upload:

```bash
scp -r /path/to/local/folder username@hive.hpc.ucdavis.edu:/quobyte/jbsiegelgrp/username
```

---

## 2) Submit Jobs

### Bulk submission (recommended)

```bash
python submit_af3_bulk.py /path/to/directory
```

The script will find all `.json` files in the directory, auto-generate a SLURM array job script, and submit — each JSON gets its own array task running in parallel.

### Bulk submission with PAE plots

For multimers/oligomers where you want to inspect inter-chain contacts:

```bash
python af3_bulk_pae.py /path/to/directory
```

This runs identically to `submit_af3_bulk.py` but also generates a PAE heatmap PNG for each prediction after it completes. Chain boundaries are marked with dashed lines to make inter-chain PAE blocks easy to read.

### Single job

```bash
sbatch submit_af3_single.sh /path/to/json/file.json
```

> 💡 For large complexes (>500 residues), uncomment the `--constraint` line in the generated `.sbatch` script to prioritize A100 or Blackwell GPUs.

### Monitor your job

After submission, the script prints your job ID along with ready-to-use commands:

```bash
squeue -j <job_id>                          # Check overall job status
squeue -j <job_id> -t all                   # Check all array tasks
scancel <job_id>                            # Cancel the job
tail -f logs/af3_<job_id>_<task>.out        # Watch live logs
sacct -j <job_id> --format=JobID,Elapsed,CPUTime,MaxRSS  # Check efficiency
```

### Output structure

After jobs complete, two items appear inside your input directory:

| Output | Description |
|--------|-------------|
| `logs/` | SLURM log files per task — includes runtime, peak VRAM, and GPU utilization |
| `<dirname>_output/` | Prediction results, one subfolder per JSON file |

For example, if your input folder is `myenzymes/`, results appear in `myenzymes/myenzymes_output/`.

Each log reports **peak VRAM**, **average GPU utilization**, and **total runtime** — handy for estimating resources for future jobs.

---

## 3) Retrieve and Use the Model

### Find your model file

Inside each job's output subfolder, look for:
```
<jobname>_model.cif
```

### Convert to PDB in PyMOL

1. Open `*_model.cif` in **PyMOL**
2. **File → Export Structure → Export Molecule → Save as PDB**

### Relax for Rosetta

Before using in Rosetta, run through the relaxation pipeline:

➡️ [Relax PDBs for Rosetta](https://github.com/MissMaryR/Relax-pdbs-for-Rosetta)

> ⚠️ Relaxation is important — AF3 models are formatted differently than what Rosetta expects.

Once relaxed, the PDB is ready for the [Rosetta docking pipeline](https://github.com/MissMaryR/Rosetta-Docking).
