S2F-artifacts


## Docker Environment

#### Download the  docker image:

```bash
docker pull  S2F-artifacts-image
```

#### After pulling the image, verify it with:
```
$ docker images -a
REPOSITORY              TAG    IMAGE ID       CREATED          SIZE
S2F-artifacts-image     v3     a0be2fab59d2   2 hours ago      13.08GB
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
## Running S2F on Program
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
Run S2F using a script from within Docker, for example:
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

  

###0-day vulnerability 
Figure 1 shows a code snippet from **cyclonedds** where a stack buffer overflow occurs in `strlcpy`. When `identifier` has exactly `IDMAX = 1024` characters, `sizeof(macroname)` is 1025 while `strlen(identifier)` returns 1024. As `size > len` evaluates to true, `size` becomes `len + 1 = 1025`. The call to `memcpy(dest, src, size)` thus copies 1025 bytes. The subsequent write `dest[size] = '\0'` attempts to access `dest[1025]`, which exceeds the allocated buffer and triggers a stack buffer overflow. This vulnerability can result in memory corruption and unpredictable program behavior. Both sampling and solving RB branches are able to directly trigger this crash.

```c
char macroname[IDMAX + 1]; // IDMAX = 1024   
strlcpy(macroname, identifier, sizeof(macroname));  

size_t strlcpy(char* dest, char* src, size_t size)
{
    size_t len = strlen(src);
    if (size == 0)
        return len;
    size = (size > len) ? len + 1 : size - 1;
    memcpy(dest, src, size);
    dest[size] = '\0'; // crash
    return len;
} // system.c:4989
```





### Vulnerability by solving RB

#### 1. Pointer Definitions
```c
/* The next unread compilation unit within the .debug_info section.
   Zero indicates that the .debug_info section has not been loaded
   into a buffer yet. */
bfd_byte *info_ptr;
/* Pointer to the end of the .debug_info section memory buffer. */
bfd_byte *info_ptr_end;
```

#### 2. `unit_length` Source
```c
/* Read each remaining comp. units checking each as they are read. */
while (stash->info_ptr < stash->info_ptr_end)
{
    bfd_vma length;
    unsigned int offset_size = addr_size;
    bfd_byte *info_ptr_unit = stash->info_ptr;
    length = read_4_bytes(stash->bfd_ptr, stash->info_ptr, stash->info_ptr_end);
}
```

#### 3. Initialization of `info_ptr` (buf) and `end_ptr` (end)
```c
static struct comp_unit *parse_comp_unit(struct dwarf2_debug *stash,
                                         bfd_vma unit_length,
                                         bfd_byte *info_ptr_unit,
                                         unsigned int offset_size)
{
    bfd_byte *info_ptr = stash->info_ptr;
    bfd_byte *end_ptr = info_ptr + unit_length;
}
```

Debug information:
```
(gdb) p/x stash->info_ptr    # Initial value
$4 = 0x606000000c84
(gdb) p/x info_ptr          # Value when error occurs
$5 = 0x60606444162c
(gdb) p/x end_ptr           # end pointer remains unchanged
$6 = 0x606067bc7a13
```

#### 4. Modification of `info_ptr` Value
```c
/* Read and fill in the value of attribute ATTR as described by FORM.
   Read data starting from INFO_PTR, but never at or beyond INFO_PTR_END.
   Returns an updated INFO_PTR taking into account the amount of data read. */

static bfd_byte *read_attribute_value {
    ......
    blk->size = safe_read_leb128(abfd, info_ptr, &bytes_read, FALSE, info_ptr_end);
    info_ptr += bytes_read;
    blk->data = read_n_bytes(abfd, info_ptr, info_ptr_end, blk->size);
    info_ptr += blk->size;     # info_ptr is incremented by a very large value
    attr->u.blk = blk;
}
```

Debug information:
```
(gdb) p blk->size
$4 = 1682180503
(gdb) p info_ptr
$5 = (bfd_byte *) 0x60606444162c ""
```

#### 5. Vulnerability Trigger Location
```c
static unsigned int read_4_bytes(bfd *abfd, bfd_byte *buf, bfd_byte *end)
{
    if (buf + 4 > end)
        return 0;
    return bfd_getl32(abfd, buf);
}

bfd_getl32(const void *p)
{
    const bfd_byte *addr = (const bfd_byte *) p;
    unsigned long v;
    v = (unsigned long) addr[0]; # Attempt to read offset from buf field
}
```


## 7. Binary File Contents

### Crash Binary:
```
[ 4] .debug_info PROGBITS 0000000000000000 00000040
```
`.debug_info` section starts at offset 0x40, length is 0x3a (58 bytes)

Hex dump of `.debug_info`:
```
xxd -s 0x40 -l 0x3a id:000107,scr:004554
00000040: 8f6d bc67 0400 0000 0000 0801 0000 0010  .m.g............
00000050: 0097 0944 6400 0000 0000 0000 0002 0000  ...Dd...........
00000060: 0000 0116 3200 0000 0903 0000 0000 0008  ....2...........
00000070: 0100 0304 0569 6e74 0000                 .....int..
```

Key values:
- `8f 6d bc 67` = `0x67bc6d8f` = 1740402063 
- `97 09 44 64` = `0x64440997` = 1682180503

### Parent Binary:
```

[ 4] .debug_info PROGBITS 0000000000000000 00000040

xxd -s 0x40 -l 0x3a  id:004554
00000040: 1100 0000 0400 0000 0000 0801 0000 0010  ................
00000050: 0000 0000 0000 0000 0000 0000 0002 0000  ................
00000060: 0000 0116 3200 0000 0903 0000 0000 0008  ....2...........
00000070: 0100 0304 0569 6e74 0000                 .....int..
```


The vulnerability occurs due to integer overflow in `blk->size` (1682180503), which causes `info_ptr` to be incremented beyond valid memory bounds. When `read_4_bytes()` is subsequently called, the `buf` pointer points to an invalid memory location (0x60606444162c), leading to a segmentation fault when attempting to dereference it in `bfd_getl32()`.

The root cause is insufficient bounds checking when reading LEB128-encoded size values, allowing an attacker to craft a malicious `.debug_info` section with an artificially large `blk->size` value that causes pointer arithmetic overflow.


