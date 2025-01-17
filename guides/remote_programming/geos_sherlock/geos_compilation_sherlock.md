# GEOS Compilation with Sherlock

## Overview
This guide details the steps required to compile the GEOS model on the Sherlock supercomputer, including terminal configuration, installation, execution, and debugging with VSCode in interactive sessions.

### Remark
Note that `GROUP_HOME` is a shared storage device; therefore, each `<SUID>` should create a folder named after its corresponding SUID to maintain user-specific storage and make it easy to identify the folder's owner.

## Steps

### Step 0: Loading the Necessary Modules

Before starting the compilation, load the necessary modules:

```bash
module load system devel math
module load git/2.45.1 git-lfs/2.4.0 gcc/12.4.0 cmake/3.24.2 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1
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

# OpenMP Options
set(ENABLE_OPENMP ON CACHE BOOL "")

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
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Release
cd ..
```

### Step 3: Compile TPLs in Debug mode

```bash
cd build-sherlock-custom-debug/
make
cd ..
```

### Step 4: Compile TPLs in Release mode

```bash
cd build-sherlock-custom-release/
make
cd ..
```

## Creating an SBATCH Script

This procedure can be combined into an `sbatch` script to request resources and execute the above steps sequentially. Below is an example of how the `sbatch` script should look:

```bash
#!/bin/bash
#SBATCH --job-name=compile_tpls         # Name of the job
#SBATCH --output=job_tpls_output_%j.log  # Output log file (%j will be replaced by job ID)
#SBATCH --error=job_tpls_error_%j.log    # Error log file (%j will be replaced by job ID)
#SBATCH --nodes=1                         # Use one node
#SBATCH --ntasks=1                        # Number of tasks (usually for MPI, set to 1 for non-MPI)
#SBATCH --cpus-per-task=4                 # Request 4 CPU cores
#SBATCH --mem=8G                          # Request 8 GB of memory
#SBATCH --time=02:30:00                   # Set a time limit of 2.5 hour
#SBATCH --partition=normal                # Specify the partition
# Email notifications
#SBATCH --mail-type=END,FAIL              # Email notifications for job completion and failure
#SBATCH --mail-user=oduran@stanford.edu   # Replace with your email address

# Step 0: Load the necessary modules
module load system devel math
module load git/2.45.1 git-lfs/2.4.0 gcc/12.4.0 cmake/3.24.2 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1

# Step 1: Clone the sources
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

# Step 2: Configure TPLs
cp build_utils/sherlock-custom.cmake GEOS/host-configs/Stanford/.
cd thirdPartyLibs/ || { echo "Failed to enter thirdPartyLibs directory"; exit 1; }
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Debug
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Release

# Step 3: Compile TPLs Debug
cd build-sherlock-custom-debug/ || { echo "Failed to enter build-sherlock-custom-debug directory"; exit 1; }
make

# Step 4: Compile TPLs Release
cd build-sherlock-custom-release/ || { echo "Failed to enter build-sherlock-custom-release directory"; exit 1; }
make
```

## Summary TPLs compilation 
Follow the steps above to successfully compile GEOS TPLs on the Sherlock environment. 