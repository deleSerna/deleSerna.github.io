Version: 1.95.3 (Universal)
Commit: f1a4fb101478ce6ec82fe9627c43efbf9e98c813
Date: 2024-11-13T14:50:04.152Z
Electron: 32.2.1
ElectronBuildId: 10427718
Chromium: 128.0.6613.186
Node.js: 20.18.0
V8: 12.8.374.38-electron.0
OS: Darwin x64 23.0.0---
layout: post
title:  "Virtual threads overview"
date:   2024-10-22 16:51:10 +01
categories: java, virtual threads, 
published: false
---
 Virtual thread is originally introduced as a preview feature in Java 19 as part of [JEP 425][jep-425] and has added as a stable feature in Java 21.
 - It is not inherentally fast compared to normal java thread (aka platform thread).
 - Only useful, if tasks are mix of CPU and IO bound. If it is CPU bound then  it wont be better than platform threads.
 - It shhould not be pooled as it's only meant to use for some short lived synchronous tasks.

Limiting factor of current threads in java
  - The number of available threads are limited because the JDK implements threads as wrappers around operating system (OS)  
     threads. OS threads (and so platform threads)  are limited, so we cannot have too many of them, which makes the current platform threads ill-suited to the thread-per-request style. 
  - You have to  program in asynchronous style to increase the throughput and that is not easy for debugging because in stacktrace you won't always see the continuity between the calleee and the caller. 
   
   A virtual thread is an instance of java.lang.Thread that is not tied to a particular OS thread unlike the platform thread.

   When a code running in a virtual thread has invoked a blocking I/O operation in the `java.*` API, the runtime performs a non-blocking OS call and automatically suspends the virtual thread until it can be resumed later.
   Number of java libraries are changed it's internal implementation to support this non blocking operations by uisng NIO (eg: [JEP-353](https://openjdk.org/jeps/353), [JEP-373](https://openjdk.org/jeps/373) etc.) 
   or using Reentrant locks instead of `synchronized block`. 
   
   Here, I have adapted a small NIO based client server program using Virtual Thread to understand the chnages in the ServerSocket api for virtual thread.
   
   Steps to follow
   1. Start IoEchoThreadPoolServer
   2. Start IoEchoClient
   3. At thi spoint, one of the worker theread has connected to the client an waiting input from the client.
   4. Get `pid` of the IoEchoThreadPoolServer ( eg: ps -ef | grep IoEchoThreadPoolServer)
   5. Dump the server thread log.

If we go through the logs of the server then we could see in case of the virtual thread, the thread is going into a parked state (yielding the platform thread) when it is waiting for input from a client.
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

If we follow the same procedure by modifying the server to use `Executors.newFixedThreadPool(5)`, but in the log we could see that the platform thread is still blocked.
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

   Thread.sleep(Duration.ofSeconds(1));? is the virtual thread actually sleeping?
   The identity of the carrier is unavailable to the virtual thread. 
   The value returned by Thread.currentThread() is always the virtual thread itself

   In current scheduler, hw does the threadpool works? would not it aso, put the submiited job to sleep if it's waiting for IO?

   How to make  thread-local data accidentally leaking from one task to another?


   The vast majority of blocking operations in the JDK will unmount the virtual thread, freeing its carrier and the underlying OS thread to take on new work.
    However, some blocking operations in the JDK do not unmount the virtual thread, and thus block both its carrier and the underlying OS thread. 
    This is because of limitations either at the OS level (e.g., many filesystem operations) or at the JDK level (e.g., Object.wait()). 
    The implementation of these blocking operations will compensate for the capture of the OS thread by temporarily expanding the parallelism of the scheduler.
     Consequently, the number of platform threads in the scheduler's ForkJoinPool may temporarily exceed the number of available processors.
      The maximum number of platform threads available to the scheduler can be tuned with the system property jdk.virtualThreadScheduler.maxPoolSize

      There are two scenarios in which a virtual thread cannot be unmounted during blocking operations because it is pinned to its carrier:

Asynchronous debuuging vs virtual thread debugging?

When it executes code inside a synchronized block or method, or
When it executes a native method or a foreign function.
The scheduler does not compensate for pinning by expanding its parallelism. 

what is pinning blocking vs nonpinnig blocking?
Why is synchronized make it pinning but locking not?
   - Reentrant lock rewriiten to unpark in case of VT.

Why are virtual threads set to daemon thread?


[jep-425]: https://openjdk.org/jeps/425
