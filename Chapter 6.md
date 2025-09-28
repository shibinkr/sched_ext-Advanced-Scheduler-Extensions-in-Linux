# 6. Use Cases and Examples

`sched_ext` enables Linux to go beyond the traditional "one-size-fits-all" CPU scheduler. By allowing custom scheduling policies in BPF, it unlocks new opportunities for specialized workloads that demand either lower latency, higher throughput, or domain-specific task management.

This chapter highlights practical scenarios where `sched_ext` shines.

---

## 6.1 Low-Latency Workloads

### Context
Latency-sensitive systems (e.g., trading systems, live audio/video processing, or gaming engines) require **deterministic response times**.  
Traditional fair scheduling (CFS) optimizes for throughput and fairness, which may introduce jitter or delays.

### sched_ext Advantage
- Developers can implement **real-time–like policies** tuned for their workload without replacing the entire kernel scheduler.  
- Tasks can be dispatched **immediately to idle CPUs** using `scx_bpf_dsq_insert()`.  
- Custom DSQs can enforce **strict ordering**, ensuring critical tasks run without being delayed by background jobs.  

### Example
A **real-time media server** might implement:  
- Separate DSQs for high-priority audio streams and background encoding tasks.  
- A rule: always schedule audio DSQ tasks first, even preempting encoding DSQs.  

```c
void BPF_STRUCT_OPS(media_enqueue, struct task_struct *p, u64 enq_flags) {
    if (is_audio_task(p))
        scx_bpf_dsq_insert(p, AUDIO_DSQ, SCX_SLICE_DFL, enq_flags);
    else
        scx_bpf_dsq_insert(p, ENCODING_DSQ, SCX_SLICE_DFL, enq_flags);
}
```
## 6.2 Multi-Tenant / Cloud Scheduling

### Context
In cloud environments, a single machine may host workloads from **multiple tenants**.  
The scheduler must balance **fairness** with **performance isolation**.

### sched_ext Advantage
- Tenants can be mapped to **separate DSQs** or queues inside a BPF map.  
- The scheduler can enforce **CPU quotas per tenant**, preventing noisy neighbors from dominating.  
- Policies like **weighted fair queuing (WFQ)** or **priority-based allocation** can be implemented in BPF.  

### Example
- Tenant A has **70% CPU share**, Tenant B has **30%**.  
- The BPF scheduler keeps counters of CPU slices consumed and dispatches accordingly.

```c
if (tenant_usage[A] < 70)
    scx_bpf_dsq_insert(task_A, SCX_DSQ_GLOBAL, slice, 0);
else if (tenant_usage[B] < 30)
    scx_bpf_dsq_insert(task_B, SCX_DSQ_GLOBAL, slice, 0);
```
This ensures **predictable resource sharing** without kernel modifications.
## 6.3 AI/ML Task Scheduling

### Context
AI/ML workloads are often **heterogeneous**:
- Short control tasks (e.g., parameter updates)
- Long compute tasks (e.g., training batches)
- I/O-heavy tasks (e.g., data preprocessing)

Traditional schedulers may not distinguish between them effectively.

### sched_ext Advantage
- Tasks can be **categorized dynamically** using BPF maps (e.g., task annotations).  
- **DSQs** can separate short-latency control tasks from long training jobs, ensuring responsiveness.  
- **Custom batching**: CPUs can be dedicated to AI kernels while leaving some cores free for lightweight tasks.  

### Example
A deep learning framework might:
- Always prioritize **gradient update threads**  
- Run **compute-intensive GPU feeder threads** at lower priority

```c
if (is_gradient_task(p))
    scx_bpf_dsq_insert(p, PRIORITY_DSQ, SCX_SLICE_DFL, 0);
else
    scx_bpf_dsq_insert(p, BACKGROUND_DSQ, SCX_SLICE_DFL, 0);
```
This leads to **faster convergence** and **lower training jitter**.

## 6.4 Research Prototypes

### Context
Scheduling is a long-standing area of systems research.  
Traditional kernel schedulers are difficult to extend without **recompiling** and risking system stability.

### sched_ext Advantage
- Researchers can **prototype new scheduling algorithms** in BPF without kernel patches.  
- **Misbehaving schedulers** are automatically reverted by fallback mechanisms.  
- **Debugging tools** (`sched_ext_dump`, SysRq-D) provide detailed insights.  

### Example Prototypes
- **Lottery Scheduling**: Randomly select tasks proportional to “tickets” assigned  
- **Energy-Aware Scheduling**: Group tasks on fewer cores to save power  
- **Deadline Scheduling**: Prioritize tasks with the earliest deadlines  

#### Prototype code snippet for lottery scheduling:
```c
u32 winner = bpf_get_prandom_u32() % total_tickets;
task = pick_task_from_lottery(winner);
scx_bpf_dsq_insert(task, SCX_DSQ_GLOBAL, SCX_SLICE_DFL, 0);
```
This makes sched_ext an ideal platform for ***academic and industrial experiments***.

## 📊 Comparison Table: Workload vs Scheduler Approach

| Workload Type                  | Traditional Scheduler Limitation                  | sched_ext Solution                                    |
|--------------------------------|-------------------------------------------------|------------------------------------------------------|
| Low-Latency (e.g., trading, audio) | Jitter due to fairness and batching           | Direct DSQ insertion, preempt background tasks      |
| Multi-Tenant Cloud             | Noisy neighbor can starve other tenants        | Tenant-specific DSQs, weighted quotas in BPF       |
| AI/ML Training                 | Long compute jobs delay short control tasks    | Task categorization and priority DSQs               |
| Research Prototypes            | Kernel recompilation needed, unsafe            | BPF prototyping, safe fallback, rich debugging     |

## ✅ Summary

**sched_ext** is not just a replacement for CFS—it’s a framework for innovation:

- **Low-latency**: Deterministic response for critical workloads.
- **Multi-tenant**: Fair and isolated CPU sharing in cloud systems.
- **AI/ML**: Specialized handling of heterogeneous AI tasks.
- **Research**: Safe prototyping of novel algorithms without kernel recompilation.

By combining **BPF flexibility** with **kernel safety nets**, sched_ext provides both developers and researchers with a powerful tool for next-generation scheduling.
