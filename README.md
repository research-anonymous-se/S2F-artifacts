# S2F: Efficient Hybrid Testing by Combining Fuzzing, Symbolic Execution, and Sampling

S2F is built on AFL and SymCC
## Docker Environment

#### Download the  docker image:

```bash
docker pull  S2F-artifacts-image
```

#### After pulling the image, verify it with:
```
$ docker images -a
REPOSITORY              TAG    IMAGE ID       CREATED          SIZE
S2F-artifacts-image                   v3     a0be2fab59d2   2 hours ago      13.08GB
```

#### Run the container in interactive mode:

```bash
sudo docker run -it S2F-artifacts-image bash
```
 CPU Binding for Fuzzing in Docker

Experiments can be executed in parallel, and the number of CPU cores used during parallel execution can be limited. However, each container must be allocated at least three CPU cores. We modified AFL_BUILD’s CPU-affinity mechanism such that when AFL selects a “free” core that lies outside the container’s permitted range, it automatically attempts to bind to the next available valid core.
```cpp
for each cpu in cpu_cores:
    if cpu is free and bind_to_cpu(cpu) succeeds:  // allowed by Docker
        selected_cpu = cpu
        break
```
---
## Experimental Environment

### Required Dependencies

The following are the dependencies needed by S2F, provided for reference by researchers interested in polyhedral sampling.
- **LLVM**: version 12.0.1, with projects `clang` and `compiler-rt` enabled.  
- **Z3**: version 4.12.0.  
- **gmp**: version 6.3.0, configured with `--enable-cxx` (required for `gmpCxx.h`).  
- **mprf**: version 4.2.0.  
- **Flint**: used in S2F to store polyhedral coefficients. Configure Flint with GMP and MPFR paths, for example:`--with-gmp=path_to/gmp_install --with-mpfr=path_to/mpfr_install`
- **GLPK**: used in S2F to find points inside polyhedra.
- **PPlite**:  used in S2F to construct polyhedra. Configure PPlite with Flint, MPFR, and GMP paths, for example: `--with-flint=path_to/flint_install/            --with-flint-include=path_to/flint_install/include \   --with-flint-lib=path_to/flint_install/lib \   --with-mpfr=path_to/mpfr_install  \ --with-gmp=path_to/gmp_install`
-  **pwalk**:   S2F uses the polyhedral sampling algorithm **John Walk**.   Build using `python setup.py build # No need to compile the Python extension. The libpwalk.so used by S2F does not need to link against pwalk_wrap.o.`

---
## Running S2F on Benchmark Programs
### S2F Structure and Build Configuration
Before running S2F, the target program must be compiled into `three distinct binaries`, each instrumented for a specific purpose: a fuzzing binary for AFL-based testing, a trace‑collection binary for gathering execution traces, and a symbolic‑execution binary for performing symbolic analysis.

The `build.sh` script can be used as a reference for compilation; the following provides additional explanations for the steps performed in the script.

```c
├── AFL_BUILD           # Builds binaries for fuzzing  
│                      cmake  C_COMPILER=/path_to/AFL_BUILD/afl-clang-fast    C_FLAGS+=" -fno-discard-value-names"
│               
├── CFG_BUILD           # Builds instrumented binaries for collecting traces  
│                      cmake  C_COMPILER=/path_to/CFG_BUILD/afl-clang-fast    C_FLAGS+=" -fno-discard-value-names"
│              
├── ceTools_sampler
│   ├── coordinate      # Coordinator
│   ├── s2f_tailoredSE  # Tailored symbolic execution engine 
```
#### Build S2F-tailoredSE
```bash
 #Enable required options and set dependency paths:
 cmake  -DZ3_DIR=/path_to/z3_install\
  -Dpplite_DIR=/path_to/PPLite_install\
  -Dflint_DIR=/path_to/flint_install\
  -Dgmp_DIR=/path_to/gmp_install\
  -Dpwalk_DIR=/path_to/vaidya-walk/code \
  -Dglpk_DIR=/path_to/glpk_install \
  -DCMAKE_BUILD_TYPE=Release -DQSYM_BACKEND=ON ..

cmake C_COMPILER=/path_to/s2f_tailoredSE/build/symcc \
      C_FLAGS+=" -fno-discard-value-names" ..
```

### Running S2F 
An example script for running S2F inside the Docker is provided below:
```bash
  run_libarchive.sh  
```
In our configuration, S2F is launched using two parallel AFL fuzzing instances and one symbolic‑execution instance
###  results data
All experiments are conducted on servers with 32 cores (2.40 GHz Intel Xeon Silver 4314), 128 GB of memory, and
Ubuntu 20.04 LTS.
- branch coverage
After running the experiments for 24 hours following the scripts described above, the edge-coverage results can be reproduced using:
```bash
python branch_coverage.py
```

- Reproducing Crashes
The data directory contains the zero‑day vulnerabilities discovered during the experiments, which can be reproduced using the following script:
```bash
python repo_crash.py
```



### Explanation of S2F Instrumentation
All three programs—used respectively for fuzzing, trace collection, and symbolic execution—must be instrumented at the same LLVM optimization phase. We therefore insert all instrumentation at EP_ModuleOptimizerEarly, ensuring that the analyses across these components remain consistent. `This explanation is provided for the reference of researchers interested in the S2F architecture.`
```cpp
static struct llvm::RegisterStandardPasses
    Y(llvm::PassManagerBuilder::EP_ModuleOptimizerEarly, addSymbolizePass);
```

  







