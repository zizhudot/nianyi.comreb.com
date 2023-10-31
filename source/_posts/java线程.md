---
title: Let's Talk about Java Threads and CPU Scheduling!
date: 2023-10-31 15:28:28
categories:
  - Technical section
tags: 
  - Cpu scheduling
  - Thread
description: Understanding the concepts of Java threads and thread scheduling is critical to developing efficient concurrent applications.
cover: https://www.fita.in/wp-content/themes/zeft/images/threads-in-java.jpg
---

## Introduction

In modern operating systems, when a program is run, a process is created. For example, when we start a Java program, the system creates a Java process. Within a process, multiple threads can be created, each having its own set of attributes such as a counter, stack, and local variables. Introducing the concept of threads allows for the separation of resource allocation and execution scheduling within a process. Threads can also access shared memory variables, such as memory addresses and file I/O. Threads are lightweight units of execution in a computer system and serve as the smallest unit for system scheduling. In Java, a program starts executing from the main() method, and it may appear that no other threads are involved. However, in reality, a Java program is inherently a multithreaded program because the execution of the main() method is carried out by a thread called the "main" thread. Let's take a closer look at the threads involved in a typical Java program.

```java
public class Thread {
    public static void main(String[] args) {
        // 获取Java线程管理
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 仅获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        // 遍历线程信息并打印线程ID和名称
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println("[" + threadInfo.getThreadId() + "]" + threadInfo.getThreadName());
        }
    }
}
```
![](https://cdn.jsdelivr.net/gh/PirlosM/image@main/20231031201746.png)

The above code snippet demonstrates that the execution of a Java program involves not only the main() method but also multiple other threads running simultaneously.


## Thread Implementation

Mainstream operating systems provide various ways to implement threads. In Java, each instance of the java.lang.Thread class, which has been started but not yet terminated, represents a thread. The Thread class in the JDK has some notable differences compared to other Java APIs as its key methods are declared as Native. In Java API, the use of native methods usually implies that the method is not implemented using platform-independent means, indicating that it operates at a low-level beyond the scope of the Java language.


There are primarily three ways to implement threads: using kernel threads (1:1 implementation), using user threads (N:1 implementation), and using a hybrid implementation of user threads and lightweight processes (N:M implementation).


### Kernel Threads (1:1 Implementation)

Kernel threads (KLTs) are directly supported by the operating system kernel. The kernel manipulates the scheduler to schedule threads and maps their tasks to different processors. Each KLT corresponds to a lightweight process (LWP) as depicted in the image below. Each kernel thread can be seen as a "clone" of the kernel, enabling the operating system to handle multiple tasks simultaneously and support multithreading. Generally, programs do not directly use kernel threads but rather a higher-level interface called lightweight processes (LWPs). LWPs are what we commonly refer to as threads since each LWP is supported by a kernel thread. LWPs and kernel threads have a one-to-one relationship, known as a 1:1 thread model. With the support of kernel threads, each LWP becomes an independent scheduling unit, even if one LWP is blocked in a system call, it does not affect the overall functioning of the process. However, LWP also has some limitations. Since it is implemented based on kernel threads, various thread operations such as creation, destruction, and synchronization require system calls. System calls incur a relatively high cost due to the need for switching between user mode and kernel mode. Additionally, each LWP requires the support of a kernel thread, which consumes certain kernel resources, such as the stack space of the kernel thread. Therefore, the number of LWPs supported by a system is limited.

![](https://cdn.jsdelivr.net/gh/PirlosM/image@main/20231031191743.png)
<center>Schematic of 1:1 between lightweight processes and kernel threads</center>



### User Threads (N:1 Implementation)

User threads are threads that are completely implemented in user space and are invisible to the operating system kernel. Each user thread corresponds to a single kernel thread. The creation, synchronization, destruction, and scheduling of user threads are all performed in user space without relying on the kernel. If implemented properly, these threads can avoid switching to kernel mode, resulting in fast and low-cost operations. They can also support a larger number of threads, which is why they are often used in high-performance databases and similar scenarios. The relationship between processes and user threads follows a many-to-one thread model. The advantage of using user threads is that they do not require support from the operating system kernel. However, the disadvantage is that they also lack support from the kernel, meaning that all thread operations need to be handled by the user program itself. This includes thread creation, switching, and scheduling. If one thread is blocked, it may cause the entire process to be blocked. Java previously used user threads, but eventually abandoned them. However, newer programming languages such as Golang and Erlang, which focus on high concurrency, widely support user threads.

![](https://cdn.jsdelivr.net/gh/PirlosM/image@main/20231031192424.png)
<center>Schematic of the N:1 relationship between processes and user threads</center>


### Hybrid Implementation (N:M Implementation)

The hybrid implementation combines user threads and lightweight processes. User threads are still implemented entirely in user space, so their creation, switching, and destruction are still inexpensive and can support a large number of user threads. The operating system provides support for lightweight processes, which act as a bridge between user threads and kernel threads. This allows for the utilization of the kernel's thread scheduling and processor mapping capabilities. System calls made by user threads are handled through lightweight processes, greatly reducing the risk of the entire process being completely blocked. In this hybrid model, the ratio between user threads and lightweight processes can vary, forming an N:M relationship. Many UNIX-based operating systems provide N:M thread model implementations, making it easier to apply the N:M thread model to applications running on these systems.

![](https://cdn.jsdelivr.net/gh/PirlosM/image@main/20231031193015.png)
<center>Schematic of the N:M relationship between user threads and lightweight processes</center>


## Java Thread Implementation

The implementation of Java threads is heavily influenced by the thread models supported by the underlying operating systems. The JVM specification does not dictate the specific thread model that must be used. The choice of thread model primarily affects the concurrency scale and operational costs of threads. However, for Java developers, these differences are transparent. Java, as a higher-level application, does not require developers to be concerned with the specific details of thread models. Before JDK 1.2, Java threads used a user-level thread implementation called "Green Threads". However, starting from JDK 1.3, the thread model was replaced with an implementation based on the native thread model of the operating system, using a 1:1 thread model. The most commonly used JVM for Java SE is the HotSpot VM developed by Oracle/Sun. In the newer versions of HotSpot VM that are supported on all platforms except Solaris, the 1:1 thread model is used. This means that a Java thread is directly implemented using a native operating system thread, without any additional indirect structures in between. Furthermore, HotSpot VM does not interfere with thread scheduling and leaves it entirely to the underlying operating system.


## Thread Scheduling

Thread scheduling refers to the process of assigning processor usage rights to threads in a system. There are two main scheduling approaches: cooperative thread scheduling and preemptive thread scheduling. In a cooperative scheduling system, each thread controls its own execution time. After completing its work, a thread needs to actively notify the system to switch to another thread. Cooperative scheduling has the advantage of simplicity as there are no thread synchronization issues since the thread switch is known by the thread itself. However, cooperative scheduling also has significant drawbacks. The execution time of a thread is not controlled, and if a thread encounters a problem and fails to notify the system to switch threads, the entire process may be indefinitely blocked. In a preemptive scheduling system, the system allocates execution time to each thread. Thread switches are not controlled by the thread itself (although in Java, Thread.yield() can be used to give up execution time, but the thread itself cannot control acquiring execution time). In this scheduling implementation, the execution time of threads is controlled by the system, and a thread blocking the process indefinitely is avoided. Java uses preemptive scheduling as its thread scheduling mechanism. If a process encounters a problem, we can terminate it using the "Task Manager" without causing the entire system to crash.


When a system has a single processor, the operating system can handle multitasking and switch between multiple threads. The processor assigns CPU time slices to each thread for execution. A CPU time slice is the duration of time allocated by the CPU for a thread to execute its tasks and is usually a few tens of milliseconds. During this short time, threads are switched back and forth so rapidly that we do not perceive the switch, making it appear as if they are running simultaneously. The time slice determines how long a thread can continuously occupy the processor for execution. When a thread's time slice is exhausted or it is forced to pause due to its own reasons, another thread (either the same thread or a thread from another process) is chosen by the operating system to occupy the processor. This process of one thread pausing and another thread being selected for execution is known as a context switch. A context switch involves saving the entire execution context of a thread to resume execution from where it left off. It includes variables, computed results, program counters, and more. It's like taking a snapshot of the thread's running environment so that when it regains CPU time, it can quickly restore the previous execution context by retrieving the saved data. This process is called a "context switch". In a system with multiple CPUs, the operating system assigns CPUs to different threads in a round-robin manner, resulting in more frequent context switches, especially when switching across different CPUs, which are more expensive than context switches within a single CPU. In the context of multithreaded programming, we are mainly concerned with the performance impact of context switches between threads. Now, let's explore the reasons behind context switches in multithreading.


## Thread States

System threads primarily have five states: "New", "Runnable", "Running", "Blocked", and "Dead". In the Java context, they are mapped to the following six states: "NEW", "RUNNABLE", "BLOCKED", "WAITING", "TIMED_WAITING", and "TERMINATED". The transition of a thread's state from "RUNNING" to "BLOCKED" or from "BLOCKED" to "RUNNABLE" triggers a context switch between threads. Let's take a look at the different scenarios that can lead to context switches.

- Time Slice Exhaustion: When a thread's time slice is exhausted, the operating system forcefully switches to another thread to ensure fair CPU time allocation to other threads.

- Preemption by Higher Priority Thread: If a thread with a higher priority needs to execute, the operating system interrupts the execution of the current thread and switches to the higher priority thread.

- Blocked Operation: When a thread performs a blocked operation, such as waiting for I/O completion or waiting for a lock to be released, the operating system places that thread in a blocked state and switches to another executable thread to fully utilize CPU resources.

- Thread Synchronization: When multiple threads need to access shared resources, thread synchronization operations such as mutex locks or semaphores are used. In this case, when one thread acquires the synchronization resource, other threads may need to wait, leading to a context switch.

- Interrupt Handling: When a hardware or software interrupt occurs, the operating system interrupts the execution of the current thread and switches to handling the interrupt event, which can cause a thread switch.


In these scenarios, the operating system determines which thread to switch to based on the scheduling algorithm and priority rules, and performs a context switch by saving and restoring the thread's execution context.


## Performance Impact of Context Switches

Context switches have a performance impact on a system due to the overhead incurred during the switch. Context switches involve saving the execution context of a thread and restoring it when it resumes execution. This process requires memory operations and can be time-consuming, especially when switching across different CPUs, since the processor caches may need to be invalidated and reloaded. The frequency and duration of context switches can affect the overall performance of a system. However, the impact of context switches on system performance depends on various factors, such as the frequency of thread switches, the number of threads, the efficiency of the scheduling algorithm, and the workload characteristics. In general, reducing the number of unnecessary context switches and optimizing the scheduling algorithm can help improve system performance.


## Conclusion

Understanding Java threads and the concept of thread scheduling is crucial for developing efficient and concurrent applications. Java provides a unified interface for thread operations across different hardware and operating system platforms. The implementation of Java threads is influenced by the underlying thread models supported by the operating system. The choice of thread model affects the scalability and operational costs of threads. Thread scheduling determines how CPU time is allocated to different threads, and context switches play a significant role in achieving fair and efficient resource utilization. By optimizing thread usage and minimizing unnecessary context switches, developers can improve the performance of their Java applications. So, whether you're a beginner or an experienced Java developer, it's important to have a solid understanding of Java threads and the impact of thread scheduling on application performance.


Remember, the key to successful multithreaded programming lies in balancing the workload, minimizing contention, and ensuring efficient resource utilization. With the right knowledge and skills, you can leverage the power of Java threads to build robust and high-performance applications.

