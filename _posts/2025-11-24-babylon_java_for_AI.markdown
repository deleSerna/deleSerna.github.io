---
layout: post
title:  "Project Babylon basics"
date:   2025-11-24 22:450:10 +02
categories: java ai code_model babyloncode_reflection
published: false
---
**Note**: Do not use this write up as a 101 source for virtual threads but instead read it once you have read other 101 write ups about it to clarify some of your lingering questions in your mind afterwards.

 Virtual thread is originally introduced as a preview feature in Java 19 as part of [JEP 425](https://openjdk.org/jeps/425) and has added as a stable feature in Java 21.
 Some key points to note down are - 
 - It is not inherentally fast compared to normal java thread (will call it as `platform thread` from here onwards).
 - Only useful, if tasks are mix of CPU and IO bound. If it is CPU bound then  it wont be better than platform threads.
 - It should not be pooled as it's only meant to use for some short lived synchronous tasks.

**Limiting factors of current threads in java are** 
  - The number of available threads are limited because the JDK implements threads as wrappers around operating system (OS)  
     threads. OS threads (and so platform threads)  are limited, so we cannot have too many of them, which makes the current platform threads ill-suited to the thread-per-request style. 
  - You have to  program in asynchronous style to increase the throughput and that is not easy for debugging because in stacktrace you won't always see the continuity between the callee and the caller. 
   
   A virtual thread is an instance of `java.lang.Thread` that is not tied to a particular OS thread unlike the platform thread.

   When a code running in a virtual thread has invoked a blocking I/O operation in the `java.*` API, the runtime performs a non-blocking OS call and automatically suspends the virtual thread until it can be resumed later.
   Number of java libraries are changed it's internal implementation to support this non blocking operations by using NIO (eg: [JEP-353](https://openjdk.org/jeps/353), [JEP-373](https://openjdk.org/jeps/373) etc.) 
   or using `ReentrantLock` instead of `synchronized block`. 
   
   [Here](https://github.com/deleSerna/javanewfeatures/blob/main/java23/virtualthread/src/IoEchoThreadPoolServer.java#L20), I have adapted a small NIO based client server program using Virtual Thread to understand the changes in the ServerSocket api for virtual thread.
   
   Steps to follow
   1. Start IoEchoThreadPoolServer.
   2. Start IoEchoClient.
   3. At this point, one of the worker theread has connected to the client and is waiting for an input from the client.
   4. Get `pid` of the IoEchoThreadPoolServer (eg: ps -ef | grep IoEchoThreadPoolServer)
   5. Dump the server thread log.

If we go through the logs of the server then we could see that in case of the virtual thread, the thread is going into a parked state (yielding the platform thread and be suspended) when it is waiting for an input from the client.
   ```
   #30  virtual
      java.base/java.lang.VirtualThread.park(VirtualThread.java:601)
      ...
      java.base/java.util.concurrent.locks.LockSupport.park(LockSupport.java:369)
      ..
      java.base/sun.nio.ch.NioSocketImpl.park(NioSocketImpl.java:201)
      java.base/sun.nio.ch.NioSocketImpl.implRead(NioSocketImpl.java:309)
      ....
      java.base/java.net.Socket$SocketInputStream.implRead(Socket.java:1116)
      java.base/java.net.Socket$SocketInputStream.read(Socket.java:1103)
      java.base/java.io.InputStream.read(InputStream.java:220)
      IoEchoThreadPoolServer$Worker.run(IoEchoThreadPoolServer.java:55)
   ```

If we follow the same procedure by modifying the server to use `Executors.newFixedThreadPool(5)`, but in the server log we could see that the platform thread is still blocked by waiting for the input from the client.
   ```
      #29 "pool-1-thread-1"
      java.base/sun.nio.ch.SocketDispatcher.read0(Native Method)
      java.base/sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:47)
      java.base/sun.nio.ch.NioSocketImpl.tryRead(NioSocketImpl.java:256)
      ....
      java.base/java.net.Socket$SocketInputStream.implRead(Socket.java:1116)
      java.base/java.net.Socket$SocketInputStream.read(Socket.java:1103)
      java.base/java.io.InputStream.read(InputStream.java:220)
      IoEchoThreadPoolServer$Worker.run(IoEchoThreadPoolServer.java:56)
   ``` 

  <!-- If you do `Thread.sleep(Duration.ofSeconds(1))` then  the virtual thread is actually sleeping. Not sure about the platform thread. -->
   The identity of the carrier is unavailable to the virtual thread. The value returned by `Thread.currentThread()` is always the virtual thread itself.
   Virtual threads is set to daemon thread as mostly because implementors don't want it to interact with the shutdown sequence otherwise program would need to wait for 1000s of virtual threads to be shut down and programmers could wait up on it's completion in other way as shown in [here](https://github.com/deleSerna/javanewfeatures/blob/main/java23/virtualthread/src/IoEchoThreadPoolServer.java#L20).
   <!-- How to make  thread-local data accidentally leaking from one task to another? -->


   The vast majority of blocking operations in the JDK will unmount the virtual thread, freeing its carrier and the underlying OS thread to take on new work.
   However, some blocking operations in the JDK do not unmount the virtual thread, and thus block both its carrier and the underlying OS thread. 
   This is because of limitations either at the OS level or at the JDK level (e.g., Object.wait()). 
   The maximum number of platform threads available to the scheduler can be tuned with the system property jdk.virtualThreadScheduler.maxPoolSize to increase the avaialble threads in those cases.

   There are a couple of scenarios in which a virtual thread cannot be unmounted during blocking operations because it is pinned to its carrier:
   - When it executes a native method or a foreign function.
     - Native method can’t be supported because virtual threrad implementation works by unwinding the stack into the heap. Doing that requires metadata describing what’s on the stack and where, which has to be produced by the JIT compiler. Native frames were produced by a non-JVM compiler and lack that metadata, so can’t be unwound. 
   - When it executes code inside a synchronized block or method (till Java 23).
     - This is mostly due to the how `synchronized` is implemented as mentioned [here](https://openjdk.org/jeps/491#Future:~:text=The%20reason%20for%20pinning). It is changed in this [pull request](https://github.com/openjdk/jdk/pull/21565). 
   - Other cases are mentioned [here](https://openjdk.org/jeps/491#Future:~:text=while%20holding%20locks.-,Future%20Work,-There%20are%20a).
   

**References**
1. [Virtual threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-68216B85-7B43-423E-91BA-11489B1ACA61)
2. [Networking-io-with-virtual-threads](https://inside.java/2021/05/10/networking-io-with-virtual-threads/)
3. [Rock the jvm](https://rockthejvm.com/articles/the-ultimate-guide-to-java-virtual-threads#some-virtual-threads-internals)
