# Submits all jsons from a specific directory

   run with 

  ```
  python3 submit_af3_bulk.py /path/to/directory
  ```

starts a script that submits all json files in the specified directory into an array job

will produce a logs folder
and an /path/to/directory_output folder

example:
input folder will have a folder in it called input_output with folders for each json file;

file named example_model.cif is the model to put into PyMOL and save as a pdb

then probably run through relaxation before running on Rosetta
