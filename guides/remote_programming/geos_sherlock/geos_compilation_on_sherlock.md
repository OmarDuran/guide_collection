# GEOS Compilation on Sherlock

## Overview
This guide outlines the necessary steps to compile the GEOS simulator on the Stanford Sherlock cluster. It covers the compilation of Third-Party Libraries (TPLs) as well as the GEOS simulator itself, utilizing sbatch scripts.

The compilation process for GEOS can be efficiently simplified into a single command line instruction, such as `sbatch compile_geos.sbatch`. This guide will walk you through the step-by-step construction of the `compile_geos.sbatch` script. For other build types, you can achieve compilation by manually modifying this script to accommodate the desired release build type.

### Remark
Note that `GROUP_HOME` is a shared storage device; therefore, each `<SUID>` should create a folder named after its corresponding SUID to maintain user-specific storage and make it easy to identify the folder's owner.


# Compilation of GEOS

### Step 0: Loading the Necessary Modules

Before starting the compilation, load the necessary modules:

```bash
module load system devel math
module load git/2.45.1 git-lfs/2.4.0 cmake/3.24.2 ninja/1.9.0 gcc/12.4.0 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1
```

For more information on modules, see [Sherlock Modules Documentation](https://www.sherlock.stanford.edu/docs/software/modules/).

### Step 1: Cloning the Sources

Clone the required repositories and initialize the submodules:

```bash
GIT_CLONE_PROTECTION_ACTIVE=false git clone https://github.com/GEOS-DEV/thirdPartyLibs.git
cd thirdPartyLibs
git lfs install
git pull
git submodule init
git submodule update
cd ..

GIT_CLONE_PROTECTION_ACTIVE=false git clone https://github.com/GEOS-DEV/GEOS.git
cd GEOS
git lfs install
git submodule init
git submodule update
cd ..
```

### Step 2: Configure TPLs

The following is an example CMake configuration file named `sherlock-custom.cmake`. This file maps some of the loaded modules to configure TPLs (Third-Party Libraries) and GEOS.

```cmake
# Custom Configuration
set(CONFIG_NAME "sherlock-custom" CACHE PATH "")
set(GCC_ROOT "/share/software/user/open/gcc/12.4.0" CACHE PATH "")
set(MPI_ROOT "/share/software/user/open/openmpi/5.0.5" CACHE PATH "")
set(BLAS_LIBRARIES "/share/software/user/open/openblas/0.3.28/lib/libblas.so" CACHE STRING "")
set(LAPACK_LIBRARIES "/share/software/user/open/openblas/0.3.28/lib/liblapack.so" CACHE STRING "")

# Base Configuration
site_name(HOST_NAME)

# Compiler Settings
set(CMAKE_C_COMPILER       "${GCC_ROOT}/bin/gcc"      CACHE PATH "")
set(CMAKE_CXX_COMPILER     "${GCC_ROOT}/bin/g++"      CACHE PATH "")
set(CMAKE_Fortran_COMPILER "${GCC_ROOT}/bin/gfortran" CACHE PATH "")

# MPI Options
set(ENABLE_MPI ON CACHE PATH "" FORCE)
set(MPI_C_COMPILER       "${MPI_ROOT}/bin/mpicc"   CACHE PATH "")
set(MPI_CXX_COMPILER     "${MPI_ROOT}/bin/mpic++"  CACHE PATH "")
set(MPI_Fortran_COMPILER "${MPI_ROOT}/bin/mpifort" CACHE PATH "")
set(MPIEXEC              "${MPI_ROOT}/bin/mpirun"  CACHE PATH "")
set(MPIEXEC_NUMPROC_FLAG "-n" CACHE STRING "")
set(ENABLE_WRAP_ALL_TESTS_WITH_MPIEXEC ON CACHE BOOL "")

# CUDA Options
if(ENABLE_CUDA)
  set(CMAKE_CUDA_HOST_COMPILER ${MPI_CXX_COMPILER} CACHE STRING "")
  set(CMAKE_CUDA_COMPILER ${CUDA_TOOLKIT_ROOT_DIR}/bin/nvcc CACHE STRING "")
  set(CMAKE_CUDA_FLAGS "-restrict -arch ${CUDA_ARCH} --expt-extended-lambda --expt-relaxed-constexpr -Werror cross-execution-space-call,reorder,deprecated-declarations" CACHE STRING "")
  set(CMAKE_CUDA_FLAGS_RELEASE "-O3 -DNDEBUG -Xcompiler -DNDEBUG -Xcompiler -O3" CACHE STRING "")
  set(CMAKE_CUDA_FLAGS_RELWITHDEBINFO "-g -lineinfo ${CMAKE_CUDA_FLAGS_RELEASE}" CACHE STRING "")
  set(CMAKE_CUDA_FLAGS_DEBUG "-g -G -O0 -Xcompiler -O0" CACHE STRING "")
endif()

# Valgrind Options
set(ENABLE_VALGRIND OFF CACHE BOOL "")

# Caliper Options
set(ENABLE_CALIPER ON CACHE BOOL "")

# Hypre Options
if(ENABLE_HYPRE_CUDA)
  set(ENABLE_PETSC OFF CACHE BOOL "")
  set(ENABLE_TRILINOS OFF CACHE BOOL "")
  set(GEOS_LA_INTERFACE "Hypre" CACHE STRING "")
endif()

# Include TPL Configuration
include(${CMAKE_CURRENT_LIST_DIR}/../tpls.cmake)
```

Copy the custom configuration file and configure TPLs for both Debug and Release builds:

```bash
cp build_utils/sherlock-custom.cmake GEOS/host-configs/Stanford/.
cd thirdPartyLibs/
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Debug
cd ..
```

### Step 3: Compile TPLs in Debug mode

```bash
cd build-sherlock-custom-debug/
make
cd ..
```

### Step 4: Configure GEOS

```
cd GEOS/ || { echo "Failed to enter GEOS directory"; exit 1; }

# Get absolute path for TPls installations
tpls_path=$(realpath ../thirdPartyLibs/install-sherlock-custom-debug/)

python3 scripts/config-build.py -hc host-configs/Stanford/sherlock-custom.cmake -bt Debug -n --ninja -D GEOS_TPL_DIR="$tpls_path"

```

### Step 5: Compile GEOS
```
# get number of cpu
cpu_count=$(lscpu | grep "^CPU(s):" | awk '{print $2}')

cd build-sherlock-custom-debug/ || { echo "Failed to enter build-sherlock-custom-debug directory"; exit 1; }
make -j "$cpu_count"
cd ../..
```

## Creating an SBATCH Script

This procedure can be combined into a single `compile_geos.sh` script to request resources and execute the steps above sequentially. While having a single script is convenient since it compile steps 0-5. One can only clone ocassionally and compiling TPLs oncascionally as well, while building GEOS, specially for develpers. Because that in this guide the process is splitted in three major steps

* Cloning the source code (Steps 1)
* Building TPLs (Steps 2 and 3);
* Building GEOS (Steps 4 and 5).



Below is an example of how the scripts should look:

### Cloning Script

```
#!/bin/bash
# Load necessary modules
module load system 
module load git/2.45.1 git-lfs/2.4.0

# Clone the sources
GIT_CLONE_PROTECTION_ACTIVE=false git clone https://github.com/GEOS-DEV/thirdPartyLibs.git
cd thirdPartyLibs || { echo "Failed to enter thirdPartyLibs directory"; exit 1; }
git lfs install
git pull
git submodule init
git submodule update
cd ..

GIT_CLONE_PROTECTION_ACTIVE=false git clone https://github.com/GEOS-DEV/GEOS.git
cd GEOS || { echo "Failed to enter GEOS directory"; exit 1; }
git lfs install
git submodule init
git submodule update
cd ..
```

### Building TPLs

```
#!/bin/bash
#SBATCH --job-name=tpls_build        # Name of the job
#SBATCH --output=tpls_output.log  # Output log file 
#SBATCH --error=tpls_error.log    # Error log file 
#SBATCH --nodes=1                         # Use one node
#SBATCH --ntasks=1                        # Number of tasks (usually for MPI, set to 1 for non-MPI)
#SBATCH --cpus-per-task=4                 # Request 4 CPU cores
#SBATCH --mem=16G                          # Request 16 GB of memory
#SBATCH --time=02:00:00                   # Set a time limit of 2.0 hours
#SBATCH --partition=dev                # Specify the partition

# Email notifications
#SBATCH --mail-type=END,FAIL               # Email notifications for job completion and failure
#SBATCH --mail-user=suid@stanford.edu    # Replace with your email address

# Load necessary modules
module load system devel math
module load cmake/3.24.2 gcc/12.4.0 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1

# Step 2: Configure TPLs
cp sherlock-custom.cmake ../GEOS/host-configs/Stanford/.
cd ../thirdPartyLibs/ || { echo "Failed to enter thirdPartyLibs directory"; exit 1; }
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Debug -DNUM_PROC=4

# Step 3: Compile TPLs Debug
cd build-sherlock-custom-debug/ || { echo "Failed to enter build-sherlock-custom-debug directory"; exit 1; }
make
cd ../..
```

### Building GEOS

```
#!/bin/bash
#SBATCH --job-name= geos_build         # Name of the job
#SBATCH --output=geos_output.log  # Output log file 
#SBATCH --error=geos_error.log    # Error log file 
#SBATCH --nodes=1                         # Use one node
#SBATCH --ntasks=1                        # Number of tasks (usually for MPI, set to 1 for non-MPI)
#SBATCH --cpus-per-task=4                 # Request 4 CPU cores
#SBATCH --mem=16G                          # Request 16 GB of memory
#SBATCH --time=02:00:00                   # Set a time limit of 2.0 hours
#SBATCH --partition=dev                # Specify the partition

# Email notifications
#SBATCH --mail-type=END,FAIL               # Email notifications for job completion and failure
#SBATCH --mail-user=suid@stanford.edu    # Replace with your email address

# Load necessary modules
module load system devel math
module load cmake/3.24.2 gcc/12.4.0 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1

# Step 4: Configure GEOS
cd GEOS/ || { echo "Failed to enter GEOS directory"; exit 1; }

# Get absolute path for TPls installation
tpls_path=$(realpath ../thirdPartyLibs/install-sherlock-custom-debug/)
python3 scripts/config-build.py -hc host-configs/Stanford/sherlock-custom.cmake -bt Debug -D GEOS_TPL_DIR="$tpls_path"

# Step 5: Compile GEOS Debug

cd build-sherlock-custom-debug/ || { echo "Failed to enter build-sherlock-custom-debug directory"; exit 1; }
make -j 4
cd ../..
```

## Compiling GEOS with a SBATCH Script


The `compile_geos.sh` file automates the build process and use the concept of dependency in SLURM to excecute each job only after it is completed. 

```
# Clone sources
source build_utils/clone.sh
# Submit the first job
tpls_id=$(sbatch build_utils/tpls.sh | awk '{print $4}')
# Submit the second job with a dependency on the first job
sbatch --dependency=afterok:$tpls_id build_utils/geos.sh
```

Before running it, create in the same directory a `build_utils` folder that contains `sherlock-custom.cmake` and create the files `clone.sh`, `tpls.sh` and `geos.sh` with the suggested content. To execute the script, run:

```bash
source compile_geos.sh
```

It will create a unique identifiers for the processes (for instance, 58367115) for further reference.

You will receive an email confirmation upon the completion or failure for each job. Below is an example of a typical email notification:

```bash
Job ID: 58367115
Cluster: sherlock
User/Group: suid/tchelepi
State: COMPLETED (exit code 0)
Nodes: 1
Cores per node: 4
CPU Utilized: 03:59:10
CPU Efficiency: 67.18% of 05:56:00 core-walltime
Job Wall-clock time: 01:29:00
Memory Utilized: 3.32 GB
Memory Efficiency: 41.52% of 8.00 GB
```

To monitor the output of the process, if it is still active, you may connect to Sherlock at any time and execute the following command:

```bash
tail -f geos_output.log
```


## Summary GEOS compilation 
At this point, you have successfully compiled the GEOS on Sherlock cluster, by employing command line tools in conjunction with various concepts related to SLURM (Simple Linux Utility for Resource Management). Note that this guide does not cover the installation of different versions of dependencies or GPU-based compilation. However, those processes would involve similar operations as described along the sections of this document.


## Bibliography 

- [GEOSX Documentation](https://geosx-geosx.readthedocs-hosted.com/en/latest/#)  
  https://geosx-geosx.readthedocs-hosted.com/en/latest/#

- [Sherlock Documentation](https://www.sherlock.stanford.edu/docs/)  
  https://www.sherlock.stanford.edu/docs/