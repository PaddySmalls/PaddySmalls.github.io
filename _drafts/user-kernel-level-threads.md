---
layout: post
title: How Operating Systems implement Threads 
tags: [Operating Systems, Linux]
---

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/a/a5/Multithreaded_process.svg/450px-Multithreaded_process.svg.png" class="title-image"
style="width:40%; height:40%"/><br/>


Nowadays, building high-performance applications which can be run concurrently 
and in parallel (on multicore CPUs) has become the norm. Such applications heavily rely on 
multi-threading. But have you ever asked yourself, how operating systems realize and manage
threads? That's what I recently did, and this post presents the results of what I've learned
about threads.

<!--more-->


<br/>

## What's a thread?

You probably already know what a process is: A program in execution. For each process, there is an entry in the 
operating 
system's process table. A single entry within this table tracks a certain process' CPU register values, memory 
maps, privileges, open file handles etc. Usually, each process has a single _thread_ of control.  
In contrast to processes as we know them, a thread is more like a _mini-process_. in theory, each process can have an 
arbitrary number of threads. Threads can be considered sub-units of a process, which can independently work on 
different tasks, whilst sharing most of the process' resources. Therefore, there's no need for a single thread to 
have e.g. its own address space. However, since threads execute independently from each other, they need their own 
stack and CPU context (program counter, stack pointer and register values). These resources cannot be shared among the 
threads of a single process.
The following figure provides a concise overview of the resources hold by processes and threads:
       

 <div class="image-div"> 
        <img src="{{site.url}}/assets/modern_os/figure-2-12-modern-operating-systems.png" 
        class="image-with-caption"/><br/>
        <small class="img-caption"><span>Figure 1: </span>Left colum: Resources per process. Right column:
         Resources per thread ([1], p.104)
        </small>
 </div>       
       
    
<br/>    
       
## Why threads?       


#### Single thread vs. multiple threads

Almost every program performs multiple tasks at the same time. Consider a simple word processor, which has several 
things to do at the same time: 

1. Waiting for input from the keyboard 
2. Doing a spell check on the existing text  
3. Constantly backing up the document to the disk. 
    
Now imagine the word processor is currently busy writing a backup to disk. During this time, it's not able to accept 
and print any input from the keyboard, since the whole process is blocked. How to get around that limitation?
The most obvious solution in this case is executing each of the tasks in a separate thread. Given that we're on a 
single-core CPU, the individual tasks are still processed sequentially. However, since the CPU rapidly switches 
between the threads of a process, the user is played the presence of simultaneity.
   
   
#### Can't we just use processes for that?
In theory, we could actually do this. In this case, we would need several processes which should be able to work on a
shared section of memory (the document) or some kind of IPC (Inter Process Communication). However, having threads as
an alternative, doing it this way makes not so much sense. Since threads already share a common address space (and 
other resources), preferring them over processes is the way to go here.         
Beyond that, figure 1 shows that threads are much more lightweight compared to processes, meaning that creating, 
destroying and scheduling threads is much cheaper as with processes (there're some exceptions from this rule, more on
that later). 
    


<br/>

## User Space Threads

The first place in an operating system where threads can be implemented is the user space. As a consequence, this 
means that the kernel is not aware of threads at all. All the kernel sees and manages is the existing processes. 
This requires each process to keep track of its threads and manage them entirely by itself. For that purpose, each 
process has its own _thread table_, which contains one entry per thread. Thread management in user space is usually 
done by means of dedicated threading packages for runtime systems (e.g. Java Runtime Environment). 
 
#### Advantages
Putting threads in user space allows for making use of threads even on operating systems which don't have any native 
support for threads. Moreover, having a thread waiting for another one in the same process only requires this thread 
to call a runtime system procedure. This procedure checks if the thread has to be switched into blocked state. In 
case that's true, the blocked thread's register values are saved in the thread table and another runnable thread is 
searched within the table. Thus, scheduling user space threads doesn't need any intervention of the kernel, what 
means that no trap is needed.
Another advantage is that any thread-aware runtime system might implement its own thread scheduling algorithm. In 
this way, thread management can be adjusted to special use cases and therefore be optimized. 
    

#### Drawbacks
A major drawback of implementing threads in user space is that a blocking system call issued by a single thread will 
stop the entire process. Why? Remember that in case of user space threads, the kernel still has not aware of the 
existence of threads. So if a thread does a syscall e.g. because it wants to perform I/O, all the kernel recognizes is
a syscall from a process. This process is therefore blocked and a context switch to another process takes place. The 
exact same thing happens if a single thread causes a page fault.
Second, since threads are not managed by the kernel, there're no clock interrupts for scheduling threads in a fair 
manner. Other threads have to rely on the currently running thread to call the scheduler after a certain amount of time.
If it doesn't, it could in theory run forever or rather as long as the owning process exists.
Another point is that the most popular domain for multithreading is exactly in the context of I/O-bound processes, e.g. Web 
servers, where system calls happen all the time, causing the entire server process to be blocked.  


#### Possible solutions
A possible approach to avoid that a single thread might block the entire process is do a _select_ call before a 
_write_ call is actually executed. The select call tells a caller if a subsequent read call would block. If that's 
the case, the thread could be suspended, repeating the select call at a later point in time and perform a read as 
soon as it's safe. As an alternative, using non-blocking system calls would spare the need to check if a system call 
is safe. Though, since that would require major changes to existing operating systems, that's not an attractive 
approach.
The problem of threads running forever could be solved by having the runtime system request a clock signal every time
a predefined amount of time has elapsed. But according to Bos and Tanenbaum [1], this is far from being a clean 
solution.
As for multithreading in the context of I/O-bound processes, a possible solution is to do thread management in the 
kernel instead of in user space. When doing I/O, the kernel is involved anyway. We'll take a look at kernel-level 
threads in the next section.
    
<br/>    
    
## Kernel Threads

<br/>

## Sources

* [1] Bos, Herbert and Tanenbaum, Andrew S. (2015). _Modern Operating Systems (Fourth Edition)_. New Jersey: Pearson 
Education, pp.97-114.

* Cover picture: https://upload.wikimedia.org/wikipedia/commons/thumb/a/a5/Multithreaded_process.svg/450px-Multithreaded_process.svg.png