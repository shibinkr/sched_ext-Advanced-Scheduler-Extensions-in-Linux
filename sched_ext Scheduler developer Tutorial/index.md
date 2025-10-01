# 📑 Index — sched_ext Scheduler Tutorial

## 1. Introduction
- 1.1 What is `sched_ext`?  
- 1.2 Why Write Custom Schedulers?  
- 1.3 Overview of Workload-Specific Policies  

## 2. Foundations of sched_ext
- 2.1 Core Concepts: DSQs, Callbacks, Helpers  
- 2.2 Task Lifecycle under sched_ext  
- 2.3 Debugging Tools & Metrics Collection  
- 2.4 Userspace Interaction with BPF Maps  

## 3. Low-Latency Scheduler (Audio, Trading)
- 3.1 Goals & Challenges  
- 3.2 DSQ Design for Low-Latency  
- 3.3 Example: Audio Processing Tasks  
- 3.4 Example: High-Frequency Trading Threads  
- 3.5 Pitfalls & Best Practices  

## 4. High-Throughput Scheduler (AI/ML Training)
- 4.1 Goals & Challenges  
- 4.2 Designing for Batch Throughput  
- 4.3 Example: Parallel Training Workers  
- 4.4 Example: Data Preprocessing Pipelines  
- 4.5 Pitfalls & Best Practices  

## 5. Multi-Tenant Fair Scheduling (Cloud / Containers)
- 5.1 Goals & Challenges  
- 5.2 Per-Tenant DSQ and Token-Bucket Models  
- 5.3 Example: Two-Tenant Fair Share (70/30)  
- 5.4 Monitoring & Enforcing Quotas via Maps  
- 5.5 Pitfalls & Best Practices  

## 6. Real-Time / Edge Computing Scheduler
- 6.1 Goals & Challenges  
- 6.2 Deadline-Aware Scheduling in BPF  
- 6.3 Example: Sensor Fusion on Edge Device  
- 6.4 Example: Control Loop with Deadlines  
- 6.5 Pitfalls & Best Practices  

## 7. Cross-Cutting Techniques
- 7.1 Metrics Collection (Latency, Throughput, Fairness, Utilization)  
- 7.2 Debugging Common Issues (Fallbacks, Stalls, Starvation)  
- 7.3 Tuning & Hybrid Policies (Latency + Fairness)  

## 8. Testing & Deployment
- 8.1 Running sched_ext in QEMU  
- 8.2 Userspace Loader & Map Management  
- 8.3 Integration with Monitoring Systems (Prometheus, bpftrace)  
- 8.4 Safe Rollback and Fallback Handling  

## 9. Conclusion
- 9.1 Comparing the Four Schedulers  
- 9.2 Key Takeaways for Developers  
- 9.3 Future Directions (Hybrid & Adaptive schedulers)  
