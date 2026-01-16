---
layout: post
title:  "Project Babylon basics"
date:   2025-11-27 23:30:10 +02
categories: java ai code_model babylon code_reflection
published: true
---
Before exploring Project Babylon's details, we will look into some of the fundamentals of the GPU programming model, using NVIDIA's CUDA as reference and whenever GPPU mentionedin this article, it implies NVIDIA's CUDA-enabled GPU.

These two excellent resources explain CUDA's model in detail if you would like to explore further: [1](https://people.montefiore.uliege.be/geuzaine/INFO0939/assets/slides/gpu-cuda-introduction.pdf) and [2](https://developer.nvidia.com/blog/cuda-refresher-cuda-programming-model).

### GPU Programming Basics

From a simplified pov, CUda's model involves two entities: a `kernel function` that can execute on multiple GPU threads, and a `compute function` that execute on the CPU which launches the kernel function while managing kernel's inputs and outputs.

CUDA-enabled GPUs follow Simple Instruction Mutiple Thread (SIMT) approach which means running identical instructions on many threads simultaneously. But each thread can access unique data via it's index, allowing all threads to operate on different parts of data at the same time and enabling massive parallelism. 

For instance, a kernel instruction like `c[idx] = a[idx] + b[idx]` —where `idx` is the thread ID- allows threads to process different array elements concurrently.

 Naturally, parallellism depends on type of instructions in the kernel function. If it contain many branches, not all threads may execute at the same time, and we won't achieve full parallelism as expected.

### Marking and Launching Kernels

In GPU programming, programmers explicitly annotate GPU-bound functions as kernels and specify the number of GPU threads for it's execution.

To make Java, GPU programming friendly, we need three bare minimum things:
   - A way to annotate methods as GPU-bound methods.
   - Access to the code of such methods in a way that can be easily converted to any other non-Java language.
   - Efficient management of data between Java and native.
 
Project Babylon addresses these points so that Java programs can access GPUs efficiently.

### Project Babylon Solution

A new concept called `code reflection` has been introduced to access a method's code. During compile time, a code model of functions marked for `code reflection` is created and made available (currently in a separate inner class). The code model acts as an IR, allowing libraries to convert it to any non-Java language.

As part of Project Babylon, converter libraries that transform Java kernel functions to CUDA, SPIR-V, OpenCL, etc., have been developed; these are referred to as the Heterogeneous Accelerator Toolkit (`HAT`). Methods annotated with `@Reflect` and having parameters like `ComputeContext` or `KernelContext` are identified as `compute functions` and `kernel functions`, respectively.

Since GPUs cannot access the Java heap, data must be off-heap. ByteBuffer could help, but it requires pinning memory to avoid GC, and accessible native memory is limited. Project Panama's Foreign Function & Memory (FFM) APIs enable efficient off-heap allocation on the Java side, and HAT uses FFM APIs heavily instead of JNI and ButeBuffer.

Thus, Project Babylon is trying to adress the `Java for AI` motto using `code reflection` in combination with new ClassFile API (introduced in Java 24) and FFM APIs.

I've added a beginner-friendly HAT101 sample using the `opencl` backend (GPU on MacBook Pro) to this [github project](https://github.com/deleSerna/ai-ex/tree/main/java/babylon/hat101/HAT101). The `MainWithHAT.java` class contain a `readme` to build and execute the particular sample.
   

**References**
1. [Project Babylon](https://openjdk.org/projects/babylon/)
2. [Article by Poonam Parhar](https://inside.java/2024/10/23/java-and-ai/)
3. [Artcle by Juan Fumero](https://jjfumero.github.io/posts/2025/02/07/babylon-and-tornadovm)
4. [SIMT vs SIMD](https://yosefk.com/blog/simd-simt-smt-parallelism-in-nvidia-gpus.html)
