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

