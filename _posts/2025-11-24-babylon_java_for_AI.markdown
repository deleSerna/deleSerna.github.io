---
layout: post
title:  "Project Babylon basics"
date:   2025-11-24 22:450:10 +02
categories: java ai code_model babylon code_reflection
published: false
---
Before digging into the details of project babylon, it is best to understand some basics of GPU programming model we will use NVIDIA's CUDA model for that. These are two good very resources to understand NVIDA's CUDA GPU programming model [1](https://people.montefiore.uliege.be/geuzaine/INFO0939/assets/slides/gpu-cuda-introduction.pdf) and [2](https://developer.nvidia.com/blog/cuda-refresher-cuda-programming-model).

From a simplistic pov, GPU programming model contains 2 entities `kernel function` and `compute function`. Kernel function will actually execute on mutiple threads of GPU. Compute function executes on CPU which apart from launching the kernel function, handle the input and output of the kernel function.

GPU follows Simple Instruction Mutiple Thread(SIMT) approach. That means, it will run the same instruction on multiple threads. But each thread can access the data based on it's index and thus all the threads can be operate on different parts of data at the same time. Therefore, it can achieve massive parallellism without any race conditions. 

For eg: the instruction to GPU could be `c[idx] = a[idx] + b[idx];` where `idx` can be the thread id. If we execute that instruction on mutiple threads each will be accessing different indices of the array and thus achieving massive parallelism by operating on different data parts at the same time.

 Naturally, parallellism depends on type of instructions in the kernel function. If it contain many branches then all threads may not be exceuting at the same time and we won't be getting full parallelism as expected.

In GPU programming, programmer should mark the functions that need to be excuted on GPU (aka `kernel function`) explicitly and also need to specify the number of threads that should execute the kernel function.

Therefore, in java program, if we can similarly mark function that need to be executed on GPU as `kernel function` and get access to the kernel function code, then with the help of java libraries,
we can convert that code to CUDA or any othe programming model as you can imagine and run on GPU like a normal CUDA program. Project babylon is trying to adress these concerns. 

How is it solving?

For that purpose, a new concept called 'code reflection' has been introduced. During compile time, a code model of functions that are marked for 'code reflection' is made and avialble in some form (currently in a separate inner class). GPU converter libraries use those code models and convert them to GPU specific language. These converrter libraries are currently placed under Heteerrogeneous Accelarator Toolkit libraries (HAT).

As you can imagine, since GPU can not access java heap, we have to place our data that needs to be executed on GPU ofheap. ByteBuffer could help here but that meant we need to pin the memory to avoid GCing that sapce. FFM APIs provided by project panama can be used for this purpose so that we can create ofheap memory on java side and we do not need to create a copy of it to pass it to GPU.

 $JDK_26/bin/javac -cp /Users/nadeesh.tv/repos/personal/openjdk/babylon/hat/build/hat-core-1.0.jar --add-modules jdk.incubator.code --enable-preview  --source 26  -sourcepath src/main/java   src/main/java/org/example/Main.java -d build

> $JDK_26/bin/java --add-modules jdk.incubator.code --enable-preview -cp build:/Users/nadeesh.tv/repos/personal/openjdk/babylon/hat/build/hat-core-1.0.jar org.example.Main


 $JDK_26/bin/javac -cp $HAT_JAR/hat-core-1.0.jar:$HAT_JAR/hat-backend-java-mt-1.0.jar --add-modules jdk.incubator.code --enable-preview  --source 26  -sourcepath src/main/java   src/main/java/org/example/MainWithHAT.java -d build
nadeesh.tv@GS-Nadeesh-G6DT6HJH7K ~/r/p/g/a/j/b/h/HAT101 (main)> $JDK_26/bin/java -cp $HAT_JAR/hat-core-1.0.jar:$HAT_JAR/hat-backend-java-mt-1.0.jar:build org.example.MainWithHAT
   

**References**
1. [Project babylon](https://openjdk.org/projects/babylon/)
2. [Article by Poonam Parhar](https://inside.java/2024/10/23/java-and-ai/)
3. [Artcle by 
Juan Fumero](https://jjfumero.github.io/posts/2025/02/07/babylon-and-tornadovm)
4. [SIMT vs SIMD](https://yosefk.com/blog/simd-simt-smt-parallelism-in-nvidia-gpus.html)
