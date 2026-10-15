---
License: Copyright © 2026 Durham University, SHAREing Project, MIT Licensed
Creator: Andrew Naden
Contributors: Andrew Naden; Ananya Gangopadhyay
Summary: Pre-assessment report template document
---

# <img src='./images/logo.svg' width=90 style="vertical-align:middle" alt="SHAREing logo"/> SHAREing: Pre-assessment report template

A external submission of [QuEST](https://quest.qtechtheory.org/) (Github [here](https://github.com/QuEST-Kit/QuEST/tree/main)), version `v.4.3.0` using the `qft` benchmark (specified in the GitHub issue [here](https://github.com/QuEST-Kit/QuEST/pull/726)) was made on the 20th of August 2026 to the SHAREing team at Durham.

This assessment is performed by Thomas Flynn of Durham University between the 13th of October 2026 and the 21st of September 2026.

## Disclaimers

1. This report is not a commentary on code quality, but an indicator of the quality of the current SHAREing testing
   methodology as of 14/10/26.
2. The pre-assessment is only a preliminary assessment of submission suitability and does not guarantee a full
   assessment. It will be provided to the submitter indicating if the full assessment will be undertaken or detail
   reasons for rejection.

## Table of contents

- [SHAREing: Pre-assessment report template](#-shareing-pre-assessment-report-template)
  - [Disclaimers](#disclaimers)
  - [Table of contents](#table-of-contents)
  - [2. Benchmark setup](#2-benchmark-setup)
    - [2.1 Fetch and build program](#21-fetch-and-build-program)
    - [2.2 Fetch and run benchmark](#22-fetch-and-run-benchmark)
    - [2.3 Reference architecture](#23-reference-architecture)
  - [3. Description of working environment](#3-description-of-working-environment)
    - [3.1 Hardware information](#31-hardware-information)
    - [3.2 Libraries and modules](#32-libraries-and-modules)
    - [3.3 Assessment tools](#33-assessment-tools)
  - [4. Compiler setup and optimisations](#4-compiler-setup-and-optimisations)
  - [5. Computational complexity and scaling](#5-computational-complexity-and-scaling)
  - [6. Memory, storage and I/O](#6-memory-storage-and-io)
  - [7. Additional comments from submitter](#7-additional-comments-from-submitter)
  - [8. Pre-assessment outcome](#8-pre-assessment-outcome)

## 2. Benchmark setup

### 2.1 Fetch and build program

The submitter has requested an assessment of **QuEST**, version **v4.3.0**. It can be fetched/installed as follows:

```bash
git clone https://github.com/QuEST-Kit/QuEST.git --branch v4.3.0
```
The CPU version can be built, starting in the `QuEST` directory, as follows
```bash
mkir build && cd build

# configure 
cmake .. -D CMAKE_BUILD_TYPE=Release -D QUEST_BUILD_TESTS=OFF

# build
cmake --build . --parallel
```
We can also add options to the configuration statement to configure MPI `-D QUEST_ENABLE_MPI=ON`, OpenMP `-D QUEST_ENABLE_OMP=ON`, and BMI2 `-D QUEST_ENABLE_BMI2=ON`. We note that we had to specify the MPI compilers in the configuration with the `-D CMAKE_C_COMPILER=mpicc -D CMAKE_CXX_COMPILER=mpicxx` flags, otherwise we are missing the `mpi.h` file. By default this will generate a `min_example` code which can be used to check the installation.

### 2.2 Fetch and run benchmark

The `qft.cpp` benchmark is hosted on a GitHub issue, linked [here](https://github.com/QuEST-Kit/QuEST/pull/726), and listed below
```cpp
#include <iostream>
#include <chrono>
#include "quest.h"

#if QUEST_COMPILE_MPI
#include <mpi.h>
#endif

int main (int argc, char * argv[])
{
  std::chrono::time_point<std::chrono::steady_clock> t_start, t_stop;
  std::chrono::duration<double> t_elapsed;
  int nqubits = 4;

  initQuESTEnv();

#if QUEST_COMPILE_MPI
  int rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &rank);
#endif

  // read in nqubits if provided
  if (argc < 2) {
#if QUEST_COMPILE_MPI
    if (!rank)
#endif
    std::cout << "nqubits argument not provided, running QFT on 4 qubits.\n" << std::endl;
  } else {
    nqubits = std::atoi(argv[1]);

    if (nqubits < 1 || nqubits > 44) {
#if QUEST_COMPILE_MPI
      if (!rank) {
#endif
      std::cout << "Invalid nqubits value provided: " << nqubits << "\n";
      std::cout << "Please provide a value of nqubits between 1 and 44." << std::endl;
#if QUEST_COMPILE_MPI
      }
#endif
      finalizeQuESTEnv();
      return EXIT_FAILURE;
    }

#if QUEST_COMPILE_MPI
    if (!rank)
#endif
    std::cout << "QFT will be run on " << nqubits << " qubits.\n" << std::endl;
  }

  reportQuESTEnv();

  Qureg qureg = createQureg(nqubits);

  initZeroState(qureg);

#if QUEST_COMPILE_MPI
  if (!rank)
#endif
  std::cout << "State initialised, running first QFT." << std::endl;

#if QUEST_COMPILE_MPI
  MPI_Barrier(MPI_COMM_WORLD);
#endif
  t_start = std::chrono::steady_clock::now();

  // apply QFT
  // every amplitude should equal 1 / sqrt(2^nqubits)
  // a.k.a. the Plus State (as in initPlusState)
  applyFullQuantumFourierTransform(qureg, false);

#if QUEST_COMPILE_MPI
  MPI_Barrier(MPI_COMM_WORLD);
#endif
  t_stop = std::chrono::steady_clock::now();

  reportQureg(qureg);

  t_elapsed = t_stop - t_start;

#if QUEST_COMPILE_MPI
  if (!rank) {
#endif
  std::cout << "First QFT runtime: " << t_elapsed.count() << "\n"  << std::endl;

  std::cout << "Running second QFT." << std::endl;
#if QUEST_COMPILE_MPI
  }
#endif

#if QUEST_COMPILE_MPI
  MPI_Barrier(MPI_COMM_WORLD);
#endif
  t_start = std::chrono::steady_clock::now();

  // applyQFT again
  // the result should be the all 0 state
  applyFullQuantumFourierTransform(qureg, false);

#if QUEST_COMPILE_MPI
  MPI_Barrier(MPI_COMM_WORLD);
#endif
  t_stop = std::chrono::steady_clock::now();

  reportQureg(qureg);

  t_elapsed = t_stop - t_start;

#if QUEST_COMPILE_MPI
  if (!rank)
#endif
  std::cout << "Second QFT runtime: " << t_elapsed.count() << std::endl;

  destroyQureg(qureg);

  finalizeQuESTEnv();

  return 0;
}
```
We place the `qft.cpp` file in the main `QuEST` directory, taking care when building to specify this file in our configuration with the giving an overall configuration state of
```bash
cmake .. -D CMAKE_BUILD_TYPE=Release -D QUEST_BUILD_TESTS=OFF -D QUEST_ENABLE_BMI2=ON -D QUEST_ENABLE_OMP=ON -D QUEST_ENABLE_MPI=ON -D CMAKE_C_COMPILER=mpicc -D CMAKE_CXX_COMPILER=mpicxx -D USER_SOURCE_NAMES=qft.cpp -D USER_OUTPUT_EXE_NAME=qft
```
which will give the `qft` executable in the `QuEST/build` directory.

The benchmark just takes in the number of qubits, i.e., a 20 qubit problem runs as
```bash
cd QuEST/build
./qft 20
```
Likewise, with hybrid MPI+OpenMP parallelism we have
```bash
cd QuEST/build
export OMP_NUM_THREADS=4
mpirun -n 2 ./qft 20
```
running on two MPI ranks with 4 threads each.

### 2.3 Reference architecture

## 3. Description of working environment

### 3.1 Hardware information

> [!IMPORTANT]
> Add the hardware information used, including where applicable the queue information if necessary. Comment on expected
normal limit for the hardware, e.g. size of the largest interconnected set of nodes, or memory limitations. Compare with
the reference architecture (if different), indicating any issues you may expect to see due to the differences.

The `<cluster_name>` system based at `<university/organisation>` was used for this assessment.

> [!IMPORTANT]
> Provide processor, memory and cache information as well as interconnect information (e.g. Infiniband, NVlink - if
across multiple nodes) of the system the assessment is to be performed on.

The hardware details for `<cluster_name>` are `[available at](<URL>)`. This information is corroborated by running
`cat /proc/cpuinfo` on one of the compute nodes via an interactive session. There are `<num_nodes>` compute nodes on the
system:

> [!IMPORTANT]
> The table below is an example and should be adapted for "non-standard" architectures, e.g. systems with Grace Hopper
nodes. A placeholder for hardware accelerators is provided. For most assessment projects, these are likely to be GPUs
(generally Nvidia or AMD). But some cases may also work with FPGAs, ASICs etc. Include any relevant specifications.

| Specification             | Per node                                                                     |
|---------------------------|------------------------------------------------------------------------------|
| Processors                | `<num_procs>` $\times$ `[<processor_name>](<link-to-online-details>)`        |
| `<Hardware accelerators>` | `<num_gpus/fpgas>` $\times$ `[<accelerator_name>](<link-to-online-details>)` |
| Clock speed per CPU       | `<freq>`MHz                                                                  |
| Sockets                   | `<num_sockets>`                                                              |
| Cores                     | `<num_cores>`                                                                |
| RAM                       | `<memory_size>`GB `<memory_type>`                                            |
| Local storage             | `<storage_size>`GB `<storage_type>`                                          |

### 3.2 Libraries and modules

In order to build the CPU version on the assessment system, the gi001 node of the Durham Hardware Lab, we need to load the following modules:

```bash
module load gnu_comp/14.1.0 openmpi/5.0.3
```
with CMake 3.31.8 for the build.

### 3.3 Assessment tools

1. likwid 5.5.1
2. GNU time (/usr/bin/time)

## 4. Compiler setup and optimisations

The CMake configuration and build instructions above give the following compiler flags for the CPU build
```
-O3 -DNDEBUG
```

## 5. Computational complexity and scaling

The number of qubits scales the problem and can be varied to increase the problem size for scaling tests. From the submission form we have: *Scaling is driven by volume of memory to hold a larger matrix that represents the quantum computer by increasing the number of qubits the size of the matrix doubles. Gate operations of a quantum circuit usually only operate on a tiny fraction of the circuit so floating point intensity is low. The capability to simulate large number of qubits not the number of cores available to operate on the matrix is the driving motivation. Exploring if weak scaling makes sense here would be interesting.*

Previous studies on scaling and performance are given in a GitHub issue linked [here](https://github.com/QuEST-Kit/QuEST/pull/726#issuecomment-4718718712) and a scaling analysis on ARCHER2 is included in the paper linked [here](https://arxiv.org/abs/2308.07402).https://github.com/QuEST-Kit/QuEST/blob/main/docs/cmake.md

## 6. Memory, storage and I/O

According to the submitter, the expected memory required for the benchmark is `<memory_size>`GB.

No I/O is considered here unless, as the submitter notes, check pointing and restart features are explored. Initially we leave this is as out-of-scope.

## 7. Additional comments from submitter

No further comments.

## 8. Pre-assessment outcome

The submitted program suitable to proceed to a high-level assessment.
