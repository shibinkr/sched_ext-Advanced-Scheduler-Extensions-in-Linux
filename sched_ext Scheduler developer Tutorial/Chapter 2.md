# 2. Foundations of sched_ext

## 2.1 Core Concepts: DSQs, Callbacks, Helpers

At the heart of `sched_ext` are three building blocks:

- **Dispatch Queues (DSQs):**  
  Specialized queues that hold runnable tasks. DSQs can be global, per-CPU, or custom-defined.  
  The BPF scheduler decides how tasks are enqueued and dispatched from these DSQs.  

- **Callbacks (`struct sched_ext_ops`):**  
  The policy layer exposes hooks such as:  
  - `enqueue()` – called when a task becomes runnable.  
  - `dispatch()` – decides which task to run next from DSQs.  
  - `tick()` – invoked on every scheduler tick.  
  - `running()`, `stopping()`, `quiescent()` – lifecycle callbacks for tasks.  

- **Helpers (`scx_bpf_*` functions):**  
  Kernel-provided functions for BPF schedulers, e.g.  
  - `scx_bpf_dispatch()` → move task from DSQ to CPU.  
  - `scx_bpf_consume()` → take a task from a DSQ.  
  - `scx_bpf_kick_cpu()` → wake a CPU.  
  These helpers form the “glue” between the framework and the policy.  

---

## 2.2 Task Lifecycle under sched_ext

A task’s journey in `sched_ext` follows a predictable flow:

1. **Runnable → Enqueue:**  
   When a task becomes runnable, the core scheduler calls into `enqueue_task_scx()`, which triggers the BPF scheduler’s `enqueue()` callback.  

2. **Queued → Dispatch:**  
   The task is placed into a DSQ. Later, `dispatch()` selects a task from the DSQ for execution.  

3. **Execution:**  
   The chosen task runs on the CPU. While running, `tick()` may fire periodically to enforce timeslices or other policies.  

4. **Stopping or Yielding:**  
   When a task stops, blocks, or its timeslice expires, the `stopping()` callback is invoked. The BPF scheduler decides whether to re-enqueue it or not.  

5. **Idle State:**  
   If no task is runnable, the `quiescent()` callback is executed to signal idle conditions.  

---

## 2.3 Debugging Tools & Metrics Collection

Developing schedulers requires **visibility** into what tasks and DSQs are doing. `sched_ext` provides:

- **Tracing hooks:** Integration with ftrace, perf, and BPF tracepoints to log scheduler activity.  
- **Metrics via BPF maps:** Export counters, latency histograms, or runtime stats.  
- **Debug helpers:** Inspect DSQs, track enqueue/dispatch frequency, and measure per-task runtime.  
- **Graphing dependencies:** Using `dot` to visualize task dependencies when debugging build or scheduling interactions.  

These tools help identify bottlenecks, starvation, or unfair scheduling decisions.  

---

## 2.4 Userspace Interaction with BPF Maps

The scheduler’s policies are not hard-coded. Instead, they can be **tuned at runtime** using userspace interfaces:  

- **BPF Maps as Control Channels:**  
  - Update task weights, priorities, or quotas.  
  - Adjust DSQ configurations dynamically.  
  - Push scheduling hints from userspace programs.  

- **Monitoring from Userspace:**  
  - Query metrics (per-task runtime, queue length).  
  - Track fairness between tenants or workloads.  
  - Implement external controllers (e.g., load balancers).  

This design makes `sched_ext` **dynamic and adaptive**—you can deploy one scheduler binary but modify its behavior live through maps.  

## Task Lifecycle Diagram

```mermaid
flowchart TB
    T1["Task Becomes Runnable"] --> T2["ops.enqueue() → Insert into DSQ"]
    T2 --> T3["ops.dispatch() → Select Task from DSQ"]
    T3 --> T4["CPU Executes Task (ops.running())"]
    T4 --> T5["ops.tick() → Timeslice Decrement"]
    T5 --> T6["ops.stopping() → Re-enqueue or Exit"]
    T6 -->|Re-enqueue| T2
    T6 -->|Exit| T7["Task Finished"]
    T2 -.->|If DSQs Empty| T8["ops.quiescent() → CPU Idle"]
