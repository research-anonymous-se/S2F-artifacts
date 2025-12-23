# S2F: Efficient Hybrid Testing by Combining Fuzzing, Symbolic Execution, and Sampling

S2F is built on AFL and SymCC
## Docker Environment

#### Download the  docker image:

```bash
docker pull  S2F
```

#### After pulling the image, verify it with:
```
$ docker images -a
REPOSITORY              TAG    IMAGE ID       CREATED          SIZE
XXXX                     v3     a0be2fab59d2   2 hours ago      13.08GB
```

#### Run the container in interactive mode:

```bash
sudo docker run -it S2F bash
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
