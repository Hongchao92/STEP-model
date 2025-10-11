# STEP model

STEP ($s_T$ with enhanced propagation) model is an extended G-equation model with a modified formulation of the turbulent flame speed to consider the effects of thermodiffusive instabilities in hydrogen flames and their interactions with turbulence. The model is developed in the commercial software CONVERGE using user-defined functions (UDF). The compiled library of the UDF, containing the STEP model, is provided in this repository to facilitate the use of this model. 


## Citations

Please cite the following publication when using this model

H. Chu, V. Varatharajan, B. Pehlivanlar, T. Lehmann, L. Berger, D. Golc, S. Esposito, T. Howarth, E. Porcelli, D. Laera, M. Günther, M. Gauding, J. Beeckmann, S. Pischinger, and H. Pitsch. An extended G-equation formulation for simulating thermodiffusively unstable hydrogen flames. Proceedings of the Combustion Institute, 2025.

## Files

This repository contains two files:
1. `libconverge_udf.so` is the compiled CONVERGE UDF library.
2. `flame_data.csv` is the table containing the flame speed, length, and the parameter $\omega_2$.

## Installation

The UDF provided here is for CONVERGE Version 5.0.2. Please ensure that you have installed this version of CONVERGE. 

1. Before using the UDF, you must first create a folder to store your UDF files. You can create it anywhere — either in your working directory or alongside your CONVERGE case folder.

2. Open a terminal, `cd` to the folder you just created, and run `cvg_udf_init`. This command will automatically generate a `CMakeLists.txt` file and create the subdirectories `examples/`, `src/`, `include/`, and `build/`. If you encounter the error `zsh: command not found: cvg_udf_init`, you can go to this path to find the `cvg_udf_init` file: `your-installation-path.../Convergent_Science/CONVERGE_CFD/5.0.2/CONVERGE/x64/bin`. 

3. Download and copy the compiled library `libconverge_udf.so` provided in this repository to the `build` folder.

4. Link `build` folder in your job submission script:
   
   `export LD_LIBRARY_PATH= your-UDF-path/build:$LD_LIBRARY_PATH`
   
5. Activate UDF usage in `inputs.in`. Set `inputs.in` → `feature_control` → `udf_flag = 1`.

6. Activate the following User-specified subroutines in `udf.in`:
   
        user_laminar_flamespeed_flag:1
        user_g_eqn_flag:1
   
   You can find a more detailed introduction in the CONVERGE UDF Manual. Steps 5 and 6 can also be done in Converge Studio.

7. Add the `flame_data.csv` with the $\omega_2$ values into your case folder, where the input files are located. This table will be used during the simulation. It is worth noting that this table needs to be made based on your own cases. The file `flame_data.csv` is only for providing an example of the table format. In flame_data.csv, a constant amount of EGR in the air is assumed. 
