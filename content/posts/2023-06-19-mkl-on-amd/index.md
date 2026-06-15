---
title: Optimizing MKL Performance on AMD CPUs
date: 2023-06-19
slug: 2023-06-19-mkl-on-amd
tags:
  - hpc
  - mkl
  - amd
  - linux
---

## The Problem

My lab has some AMD EPYC 7713 servers. We bought them because some people in the group run programs with very high CPU load (I don't know what kind of load it is, or why it can't run on the GPU, and I don't have the energy to help everyone solve it one by one). AMD processors with their many cores are a great fit for this kind of demand.

But as nice as AMD processors are, using them in a deep-learning lab brings an extra problem: the numpy and PyTorch installed by Anaconda both use MKL as their BLAS implementation by default, and MKL's library functions are also the hotspots of most high-CPU-load programs. However, **MKL checks whether it is running on an Intel CPU, and if not, the optimizations have no effect.**

Since this is a deep-learning lab, few people have enough HPC background to compile suitable versions of numpy and PyTorch themselves, and it's hard for them to break away from Anaconda, so the dependency on MKL is hard to remove. For this reason I needed a solution that is **transparent to ordinary users**.

## The Solution

A widely circulated solution can be found via search engines: set the environment variable `MKL_DEBUG_CPU_TYPE=5`. This used to work, but **it no longer works for MKL 2020 and later versions**.

In the end I found a more clever solution [here](https://documentation.sigma2.no/jobs/mkl.html).

MKL calls a function `mkl_serv_intel_cpu_true()` to check whether it is running on an Intel CPU. As long as we provide a fake `mkl_serv_intel_cpu_true()` that always returns `1`, we can trick MKL into thinking it is running on an Intel CPU.

To do this, we can use Linux's **`LD_PRELOAD` mechanism**. The dynamic library pointed to by `LD_PRELOAD` has the highest loading priority, so as long as we compile the desired `mkl_serv_intel_cpu_true()` function into an `so` file and point `LD_PRELOAD` at it, we can load this function ahead of everything else.

> I have often heard of the `LD_PRELOAD` mechanism being used for library-function hijacking attacks; here it counts as a clever use.

## Implementation

Create `mkl_trick.c`:

```c
int mkl_serv_intel_cpu_true() {
    return 1;
}
```

Compile it with `gcc -shared -fPIC -o libmkl_trick.so mkl_trick.c`, and copy the generated `libmkl_trick.so` to `/usr/local/lib`.

Add the following to the shell's global initialization file:

```bash
export MKL_DEBUG_CPU_TYPE=5  # compatibility with older MKL versions
export MKL_ENABLE_INSTRUCTIONS=AVX2  # optional, tells MKL it can use AVX2
export LD_PRELOAD=/usr/local/lib/libmkl_trick.so
```

Some of my labmates use Bash and some use ZSH, so both need to be modified:

- Bash: create the file `/etc/profile.d/mkl.sh` and add the above content
- ZSH: add it to `/etc/zsh/zshenv`

## References

- [https://documentation.sigma2.no/jobs/mkl.html](https://documentation.sigma2.no/jobs/mkl.html)
