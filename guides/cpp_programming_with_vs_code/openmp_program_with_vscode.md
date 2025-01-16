# Guide to Compile and Debug a simple OpenMP program in VSCode on macOS

In this guide, we will create a C++ program that uses OpenMP for parallel processing, compile it, and run it in debug mode using Visual Studio Code (VSCode) on macOS.

## Step 1: Create the Project Folder

1. **Create a New Folder**:
   Create a folder named `openmp_playground` where you will store your C++ file.

2. **Create the C++ Source File**:
   Inside the `openmp_playground` folder, create a file named `par_sum.cpp` and copy the following code into it:

   ```cpp
   #include <stdio.h>
   #include <stdlib.h>
   #include <omp.h>

   #define ARRAY_SIZE 10000000 // Define the size of the array

   int main() {
       int *array = (int *)malloc(ARRAY_SIZE * sizeof(int));
       if (array == NULL) {
           fprintf(stderr, "Memory allocation failed\n");
           return 1;
       }

       // Initialize the array with random integers
       for (int i = 0; i < ARRAY_SIZE; i++) {
           array[i] = rand() % 100; // Random integers between 0 and 99
       }

       long long sum = 0; // Variable to hold the sum
       double start_time, end_time;

       // Start timing
       start_time = omp_get_wtime();

       // Use OpenMP to parallelize the sum computation
       #pragma omp parallel
       {
           long long local_sum = 0; // Each thread's local sum
           #pragma omp for // Distribute the workload
           for (int i = 0; i < ARRAY_SIZE; i++) {
               local_sum += array[i];
           }

           // Combine the local sums into the global sum (critical section)
           #pragma omp atomic
           sum += local_sum;
       }

       // End timing
       end_time = omp_get_wtime();

       // Output the results
       printf("Total sum: %lld\n", sum);
       printf("Time taken: %f seconds\n", end_time - start_time);

       // Clean up
       free(array);
       return 0;
   }
   ```

## Step 2: Open the Folder in Visual Studio Code

1. **Launch VSCode**.
2. **Open the Folder**:
   - Go to `File` > `Open Folder...` and select the `openmp_playground` folder you just created.

## Step 3: Create the Build Task

1. **Create `tasks.json`**:
   - Press `Cmd+Shift+B` to open the Build Task menu.
   - Select `Configure Default Build Task`.
   - Choose `Others`, which will create a `tasks.json` file.

2. **Edit `tasks.json`**:
   Replace the content of the `tasks.json` file with the following configuration to compile your C++ program with OpenMP support:

   ```json
   {
    "version": "2.0.0",
    "tasks": [
      {
        "label": "build par_sum",
        "type": "shell",
        "command": "g++",
        "args": [
          "-Xclang",
          "-fopenmp", 			// Enable OpenMP support
          "-g", 				// Generate debug symbols
          "-L/opt/homebrew/opt/libomp/lib",
          "-I/opt/homebrew/opt/libomp/include",
          "-lomp",
          "par_sum.cpp",		// Source file
          "-o",
          "par_sum" 			// Output executable name
        ],
        "group": {
          "kind": "build",
          "isDefault": true
        },
        "problemMatcher": [
          "$gcc"
        ]
      }
    ]
   }
   ```

## Step 4: Create the Launch Configuration for Debugging

1. **Create `launch.json`**:
   - Click on the Debug icon in the Activity Bar on the side of the window or use `Cmd+Shift+D` to open the Debug view.
   - Click on the gear icon (⚙️) at the top of the Debug panel and select `C++ (GDB/LLDB)`.

2. **Edit `launch.json`**:
   Replace the contents of the `launch.json` file with the following configuration:

   ```json
   {
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug par_sum",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/par_sum", // Path to your compiled binary
            "args": [], // Any command line arguments to pass to your program (empty in this case)
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}", // Working directory
            "environment": [],
            "externalConsole": false, // Set to true if you want to use an external console
            "MIMode": "lldb", // Use LLDB for macOS
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "build par_sum", // Name of the build task
            "logging": {
                "exception": true,
                "trace": true,
                "traceResponse": true
            }
        }
    ]
    }
   ```

## Step 5: Build the Program

1. **Compile the Code**:
   - To compile the code, press `Cmd+Shift+B`.
   - Make sure there are no errors in the build process.

## Step 6: Run the Debugger

1. **Start Debugging**:
   - Go back to the Debug view, select `Debug par_sum` from the dropdown, and click the green play button (▶️) to start debugging.

2. **Set Breakpoints**:
   - You can set breakpoints in your code by clicking in the left gutter of the code editor. Once your program runs in debug mode, it will pause execution at these breakpoints, allowing you to inspect variables and control the flow of the program.

## Summary 

You have successfully created, compiled, and set up debugging for an OpenMP-enabled C++ program using Visual Studio Code on macOS.
```
