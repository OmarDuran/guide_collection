# GEOS Compilation on Sherlock

## Overview
This guide outlines the steps necessary to compile the GEOS simulator on the Stanford Sherlock cluster. It covers the configuration, installation, and execution of the simulator in debug mode using VSCode within interactive sessions on the cluster.

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

This procedure can be combined into a `compile_tpls.sbatch ` script to request resources and execute the steps above sequentially. Below is an example of how the sbatch script should look:

```bash
#!/bin/bash
#SBATCH --job-name=compile_tpls         # Name of the job
#SBATCH --output=job_tpls_output_%j.log  # Output log file (%j will be replaced by job ID)
#SBATCH --error=job_tpls_error_%j.log    # Error log file (%j will be replaced by job ID)
#SBATCH --nodes=1                         # Use one node
#SBATCH --ntasks=1                        # Number of tasks (usually for MPI, set to 1 for non-MPI)
#SBATCH --cpus-per-task=4                 # Request 4 CPU cores
#SBATCH --mem=8G                          # Request 8 GB of memory
#SBATCH --time=03:00:00                   # Set a time limit of 3.0 hours
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
cd ..

# Step 4: Compile TPLs Release
cd build-sherlock-custom-release/ || { echo "Failed to enter build-sherlock-custom-release directory"; exit 1; }
make
cd ..
```
## Compiling TPls with the SBATCH Script


The `compile_tpls.sbatch` file automates the build process. Before running it, create in the same directory a `build_utils` folder that contains `sherlock-custom.cmake`. To execute the script, run:

```bash
sbatch compile_tpls.sbatch
```
It will create a unique identifier for the process (for instance, 58367115) for further reference.

You will receive an email confirmation upon the completion or failure of the job. Below is an example of a typical email notification:

```bash
Job ID: 58367115
Cluster: sherlock
User/Group: oduran/tchelepi
State: COMPLETED (exit code 0)
Nodes: 1
Cores per node: 4
CPU Utilized: 04:34:52
CPU Efficiency: 45.71% of 10:01:20 core-walltime
Job Wall-clock time: 02:30:20
Memory Utilized: 3.31 GB
Memory Efficiency: 41.33% of 8.00 GB
```

If the process is still active, you may connect to Sherlock at any time and execute the following command:

```bash
tail -f job_tpls_output_58367115.log
```

to monitor the output of the process.



## Summary TPLs compilation 
Follow the steps above to successfully compile GEOS TPLs on the Sherlock environment. 


# Compiling GEOS in Visual Studio Code

This second part of the guide illustrates how to compile GEOS and run it as a simple application within Visual Studio Code (VSCode). While this method is specifically tailored for VSCode users, similar principles apply for compiling on local machines. Users may need to perform slightly different operations depending on their operating system.

### Using Sherlock OnDemand

Sherlock OnDemand ([link](https://ondemand.sherlock.stanford.edu/pun/sys/dashboard/apps/index)) offers an application called **code-server**, which launches a visual instance of VSCode that is accessible through your browser.

1. **Select code-server** by clicking on it. You will be prompted to allocate computational resources for your session.
2. **Fill out the request form with the following data**:

   - **Input**:
     ```
     system devel math git/2.45.1 git-lfs/2.4.0 gcc/12.4.0 cmake/3.24.2 python/3.12.1 openmpi/5.0.5 openblas/0.3.28 cuda/12.6.1
     ```
   - **Workspace**:
     ```
     /home/users/oduran
     ```

   - **Partition**: dev
   - **CPUs**: 4
   - **Memory (GB)**: 16
   - **Runtime (in hours)**: 2 

3. Finally, click **Launch**.

The system will notify you by email when your session is ready if you select the option "I would like to receive an email when the session starts." Once the session is ready, connect to the code-server application, and your interactive VSCode session will start in a new browser tab.

### Opening the GEOS Folder

1. In VSCode, select **File** > **Open Folder**.
2. Enter the path: `/home/groups/tchelepi/oduran/GEOS` and click **OK**.

### Configuring GEOS

1. Press `Ctrl + Shift + P` and select **Open Workspace Settings (JSON)**. This action will create an empty `settings.json` file.
2. Paste the following content, ensuring the TPLs absolute path is accurate:

```json
{
    "cmake": {
        "sourceDirectory": "${workspaceFolder}/src",
        "buildDirectory": "${workspaceFolder}/../geos-build/${buildType}",
        "installPrefix": "${workspaceFolder}/../geos-install/${buildType}",
        "cacheInit": "${workspaceFolder}/host-configs/Stanford/sherlock-custom.cmake", 
        "configureSettings": {
            "BLT_MPI_COMMAND_APPEND": "--use-hwthread-cpus;--oversubscribe",
            "ENABLE_OPENMP": "OFF",
            "ENABLE_HYPRE": "ON",
            "ENABLE_TRILINOS": "OFF",
            "ENABLE_CUDA": "OFF",
            "GEOS_TPL_DIR": "/home/groups/tchelepi/oduran/thirdPartyLibs/install-sherlock-custom-debug"
        },
        "generator": "Unix Makefiles",
        "useProjectStatusView": false
    }
}
```

3. Press `Ctrl + Shift + P` and select **CMake: Delete Cache and Reconfigure**. If the process is successful, at least the Debug and Release configurations will be available in the CMake icon located on the left toolbar of the VSCode interface.

### Building GEOS in Debug Mode

1. Ensure the **Debug** configuration is selected in the CMake icon.
2. Determine the number of jobs to pass to `make -j`. Since you requested 4 physical CPUs, it is important to check the CPU configuration using the `lscpu` command for accurate job assignment.
3. Open the terminal via **View** > **Terminal** and execute:
   ```bash
   lscpu
   ```

   The output will resemble this:
   ```
   Architecture:          x86_64
   CPU op-mode(s):        32-bit, 64-bit
   Byte Order:            Little Endian
   CPU(s):                20
   On-line CPU(s) list:   0-19
   Thread(s) per core:    1
   Core(s) per socket:    10
   Socket(s):             2
   ...
   ```

   In this case, `CPU(s) = 20`.

4. Press `Ctrl + Shift + P` and type **CMake: Open CMake Tools Extension**. Locate **CMake: Parallel Jobs** and set the number of jobs to **20**. Then, press `Ctrl + Shift + P` again and select **CMake: Configure**.

5. Build files will be generated in: `/home/groups/tchelepi/oduran/geos-build/Debug`.

### Building GEOS

1. Press `Ctrl + Shift + P` and select **CMake: Build**.
2. Wait for the compilation process to finish (approximately 20 minutes, depending on the requested resources).

### Running GEOS in Debug Mode

1. To run a GEOS example input file in debug mode, press `Ctrl + Shift + P` and select **Debug: Add Configuration**. Choose the **lldb** debugger.
2. This action will create a `launch.json` file. Replace its contents with the following configuration:

```json
{
    "version": "0.2.0",
    "configurations": 
    [
        { 
            "name": "Debug GEOS",
            "type": "lldb",
            "request": "launch",
            "program": "${userHome}/geos-build/Debug/bin/geosx",
            "args": ["-i", "${workspaceFolder}/inputFiles/singlePhaseFlow/incompressible_pebi3d.xml", "-o", "../${workspaceFolder}/geos-output"],
            "cwd": "${workspaceFolder}"
        }
    ]    
}
```

3. Click the play button on the left toolbar. You should see **Debug GEOS** listed among the configurations. Run the debug mode. It should stop at the first execution line in `main.cpp`. From there, you can set breakpoints and debug as needed.

## Summary GEOS compilation 
At this point, you have successfully compiled and executed GEOS on the Stanford cluster, Sherlock, utilizing command line tools, various concepts from SLURM, and the VSCode IDE. This guide has covered the compilation and execution of GEOS on Sherlock. Note that different dependency versions and GPU execution are not within the scope of this guide. However, those processes would involve similar operations as described along the sections of this document.


## Bibliography 

- [GEOSX Documentation](https://geosx-geosx.readthedocs-hosted.com/en/latest/#)  
  https://geosx-geosx.readthedocs-hosted.com/en/latest/#

- [Sherlock Documentation](https://www.sherlock.stanford.edu/docs/)  
  https://www.sherlock.stanford.edu/docs/

- [Visual Studio Code Documentation](https://code.visualstudio.com/docs)  
  https://code.visualstudio.com/docs