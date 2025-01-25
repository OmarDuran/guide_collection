# GEOS Compilation on Sherlock

## Overview
This guide provides a step-by-step process for compiling the GEOS simulator on the Stanford Sherlock cluster. The compilation involves both the Third-Party Libraries (TPLs) and the GEOS simulator itself. These steps can be executed using a script for submitting two jobs, one for the compilation of tpls and a second for compilation of GEOS.

### Important Note
Ensure that the `cmake` file and the shell scripts (`clone.sh`, `tpls.sh`, `geos.sh`) are placed in the same folder named `build_utils`. This organization is crucial for the successful execution of the compilation process.

## Compilation Steps

### Main Step: Execute the Compile Script
Begin the compilation process by executing the `compile_geos.sh` script:

```bash
source compile_geos.sh
```

This script orchestrates the compilation process by calling other scripts in a defined order.

### Step 1: Clone the Sources
The **`clone.sh`** script is responsible for fetching the required repositories and initializing the necessary submodules. The script performs the following actions:

- It loads the appropriate modules for Git.
- It clones the Third-Party Libraries (`thirdPartyLibs`) and the GEOS source code.
- It initializes the Git Large File Storage (LFS) and updates the submodules.

**Content of `clone.sh`:**

```bash
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

### Step 2: Configure TPLs
The **`tpls.sh`** script configures and compiles the Third-Party Libraries. This involves copying a custom configuration file and executing the build commands:

- It loads the necessary modules.
- It copies the CMake configuration file (`sherlock-custom.cmake`) into the appropriate directory for GEOS.
- It executes the `config-build.py` script to configure TPLs for Debug builds before running `make` to compile them.

**Content of `tpls.sh`:**

```bash
#!/bin/bash
#SBATCH --job-name=tpls_job        # Name of the job
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
cp build_utils/sherlock-custom.cmake GEOS/host-configs/Stanford/.
cd thirdPartyLibs/ || { echo "Failed to enter thirdPartyLibs directory"; exit 1; }
python3 scripts/config-build.py -hc ../GEOS/host-configs/Stanford/sherlock-custom.cmake -bt Debug -DNUM_PROC=4

# Step 3: Compile TPLs Debug
cd build-sherlock-custom-debug/ || { echo "Failed to enter build-sherlock-custom-debug directory"; exit 1; }
make
cd ../..
```

### Step 3: Configure GEOS
The **`geos.sh`** script takes care of configuring and compiling the GEOS simulator itself. It performs these tasks:

- It loads the necessary modules.
- It retrieves the absolute path for the TPLs installation.
- It executes the `config-build.py` script for GEOS, linking it to the previously built TPLs.
- Finally, it compiles GEOS using `make`.

**Content of `geos.sh`:**

```bash
#!/bin/bash
#SBATCH --job-name=geos_job         # Name of the job
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

The `compile_geos.sh` file combines the above steps into a unified process. It handles the sequence and dependencies between the jobs:

```bash
# Clone sources
source build_utils/clone.sh

# Submit the first job for TPLs compilation
tpls_id=$(sbatch build_utils/tpls.sh | awk '{print $4}')

# Submit the GEOS compilation job with a dependency on the TPLs job
sbatch --dependency=afterok:$tpls_id build_utils/geos.sh
```

### Execution
To begin the entire process, simply run:

```bash
source compile_geos.sh
```

This will initiate the compilation while managing job dependencies. You will receive email notifications regarding job completion or failure.

### Monitoring Progress
To monitor the output while the compilation is in progress, use the following command:

```bash
tail -f geos_output.log
```

## Conclusion
You have successfully compiled the GEOS simulator on the Sherlock cluster using this guide. The process effectively employs SLURM's resource management capabilities to streamline job execution in sequence. For advanced usage, additional configurations and modifications may be required based on specific project needs.

## References
- [GEOSX Documentation](https://geosx-geosx.readthedocs-hosted.com/en/latest/#)
- [Sherlock Documentation](https://www.sherlock.stanford.edu/docs/)