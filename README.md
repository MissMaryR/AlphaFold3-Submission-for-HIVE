# Instructions for AlphaFold3 Submission on HIVE

## 1) Depending on if you have a monomer or a multimer - set up your json files
   * upload all json files and submit_af3_bulk.py to a folder in HIVE

## 2) Submit all jsons in the directory/folder
   run with 
  ```
  python3 submit_af3_bulk.py /path/to/directory
  ```

starts a script that submits all json files in the specified directory into an array job

will produce a logs folder
and an /path/to/directory_output folder

example:
input folder will have a folder in it called input_output with folders for each json file

file named monomer_model.cif is the model to put into PyMOL and save as a pdb

then probably run through [relaxation](https://github.com/MissMaryR/Relax-pdbs-for-Rosetta) before running on Rosetta
