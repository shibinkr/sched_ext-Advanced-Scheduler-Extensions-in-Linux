# 1. Introduction

## 1.1 What is sched_ext?

`sched_ext` (Scheduler Extensions) is a Linux kernel framework that allows developers to write **custom schedulers in BPF** without modifying kernel C code directly.  
It introduces a new scheduler class (`ext`) that bridges the **core kernel scheduler** with **BPF-defined scheduling policies**, enabling flexibility to experiment with new strategies while still ensuring system stability and safe fallbacks.  

In short, `sched_ext` provides:  
- A standard interface between the kernel and BPF schedulers.  
- Dispatch queues (DSQs) for organizing runnable tasks.  
- Hooks (`enqueue`, `dispatch`, `tick`, etc.) where developers define custom scheduling logic.  

---

## 1.2 Why Write Custom Schedulers?

The Linux kernel already provides robust general-purpose schedulers like **CFS (Completely Fair Scheduler)**, **RT (Real-Time)**, and **Deadline Scheduler**. However, emerging workloads often demand **specialized scheduling strategies**.  

Some reasons to write custom schedulers:  
- **Low-latency control**: Audio processing or trading systems where microseconds matter.  
- **High-throughput optimization**: AI/ML training workloads where batch jobs dominate.  
- **Fair sharing in multi-tenant environments**: Ensuring cloud/container workloads get proportional CPU time.  
- **Domain-specific real-time needs**: Edge computing or robotics with deadline-driven requirements.  

With `sched_ext`, these policies can be **prototyped and tested in BPF** without risking kernel instability.  

---

## 1.3 Overview of Workload-Specific Policies

Different workloads require **different trade-offs** between latency, throughput, and fairness. `sched_ext` enables developers to create targeted schedulers for these scenarios:  

- **Low-Latency Workloads (Audio, Trading):** Prioritize ultra-fast task wake-up and execution.  
- **High-Throughput Batch Jobs (AI/ML):** Maximize CPU utilization and throughput for long-running jobs.  
- **Multi-Tenant Fair Scheduling (Cloud/Containers):** Ensure predictable and fair distribution of resources across tenants.  
- **Real-Time / Edge Computing:** Provide deadline-aware guarantees for tasks running close to sensors and actuators.  

By tailoring policies to these use cases, developers can achieve **better performance, fairness, or determinism** than general-purpose schedulers.  
