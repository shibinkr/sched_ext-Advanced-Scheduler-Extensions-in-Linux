# Scheduler Architecture & Key Interfaces

`sched_ext` as built in layers, each with defined interfaces that communicate upward and downward:

---

## Layered Architecture

**Core Kernel Scheduler**  
- The base scheduling logic (in `kernel/sched/core.c`) defines core abstractions and operations that all scheduler classes must implement.

**sched_ext Framework (Kernel Layer)**  
- Sits on top of the core scheduler and implements a new scheduler class (`ext`) which satisfies the core scheduler’s interface (`struct sched_class`). The framework handles bridging to BPF and managing queues (local, global, custom).

**BPF Scheduler (Policy Layer)**  
- The BPF program implements `struct sched_ext_ops` callbacks (enqueue, dispatch, running, stopping, quiescent, etc.), defining custom behavior for scheduling decisions.

**Userspace Component**  
- A userspace program (C, Rust, etc.) interacts with the BPF scheduler via BPF maps, configuration, metrics, and control, enabling runtime updates and monitoring.

---

## Key Interfaces & Callbacks

Changwoo labels the interactions between these layers as Interface 1 through Interface 4:

### Interface 1: Core → sched_ext

- Core scheduler calls `enqueue_task`, `pick_next_task`, `update_curr` via `struct sched_class`.
- `sched_ext` installs its own implementations (`enqueue_task_scx`, `pick_next_task_scx`) to satisfy those hooks.  
- **Example:** When a task becomes runnable, `enqueue_task_scx()` delegates to `scx_ops.enqueue()` in the BPF scheduler.

### Interface 2: sched_ext → BPF Scheduler

- The framework passes arguments and context to BPF callbacks (`enqueue`, `dispatch`, `tick`, etc.).
- Provides default implementations if BPF scheduler does not override specific callbacks.

### Interface 3: BPF Scheduler → sched_ext

- The BPF scheduler uses helper functions to manage dispatch queues, select CPUs, move tasks, and wake idle CPUs.
- **Examples of helpers:**
  - `scx_bpf_create_dsq()`, `scx_bpf_destroy_dsq()`  
  - `scx_bpf_dispatch()`, `scx_bpf_dispatch_vtime()`  
  - `scx_bpf_consume()`, `scx_bpf_consume_task()`  
  - `scx_bpf_select_cpu_dfl()`  
  - `scx_bpf_kick_cpu()`
- **Dispatch Queues (DSQs)** mediate task delivery to CPU-local queues.

### Interface 4: BPF ↔ Userspace

- Userspace programs can read/write BPF maps, configure the scheduler, or monitor metrics.
- Allows dynamic updates, policy changes, and control of scheduling behavior at runtime.



```mermaid
flowchart TB
  subgraph L1["Core Scheduler (kernel/sched/core.c)"]
    A["struct sched_class"]
  end

  subgraph L2["sched_ext Framework (Kernel Layer)"]
    B["Scheduler Class 'ext' (enqueue_task_scx, pick_next_task_scx, etc.)"]
    C["Dispatch Queues (Global DSQ, Local DSQs, Custom DSQs)"]
    D["Helper Functions (scx_bpf_dispatch, scx_bpf_consume, etc.)"]
  end

  subgraph L3["BPF Scheduler (Policy Layer)"]
    E["struct sched_ext_ops (enqueue, dispatch, tick, running, stopping, quiescent, etc.)"]
  end

  subgraph L4["Userspace"]
    F["Control and Monitoring (BPF maps, configs, metrics)"]
  end

  A -->|Interface 1| B
  B -->|Interface 2| E
  E -->|"Interface 3 (Helpers)"| D
  E -->|Uses DSQs| C
  E <-->|"Interface 4 (BPF Maps)"| F

  subgraph L5["Task Lifecycle Flow"]
    T1["Task Runnable → ops.enqueue()"]
    T2["Insert into DSQ (local / global / custom)"]
    T3["ops.dispatch() picks task from DSQ"]
    T4["CPU executes task (ops.running())"]
    T5["Time slice decremented (ops.tick())"]
    T6["ops.stopping() → re-enqueue or exit"]
    T7["ops.quiescent() if idle"]
  end

  E --> T1
  T1 --> T2
  T2 --> T3
  T3 --> T4
  T4 --> T5
  T5 --> T6
  T6 --> T7
  T6 --> T2

```

```mermaid
flowchart TB
  %% Layer 1: Core Kernel Scheduler
  subgraph L1["Core Scheduler (kernel/sched/core.c)"]
    A["struct sched_class"]
    A1["enqueue_task() / pick_next_task() / task_tick()"]
  end

  %% Layer 2: sched_ext Framework
  subgraph L2["sched_ext Framework (Kernel Layer)"]
    B["Scheduler Class 'ext'"]
    B1["enqueue_task_scx()"]
    B2["pick_next_task_scx()"]
    B3["task_tick_scx()"]
    B4["init_task_scx(), exit_task_scx()"]
    C["Dispatch Queues (DSQs)"]
    C1["Global DSQ (SCX_DSQ_GLOBAL)"]
    C2["Local DSQs (per CPU)"]
    C3["Custom DSQs (user-defined)"]
    D["Helper Functions"]
    D1["scx_bpf_dispatch()"]
    D2["scx_bpf_consume()"]
    D3["scx_bpf_move_to_local()"]
    D4["scx_bpf_create_dsq() / destroy_dsq()"]
  end

  %% Layer 3: BPF Scheduler Policy
  subgraph L3["BPF Scheduler (Policy Layer)"]
    E["struct sched_ext_ops"]
    E1["enqueue()"]
    E2["dispatch()"]
    E3["tick()"]
    E4["running() / stopping() / quiescent()"]
    E5["select_cpu()"]
  end

  %% Layer 4: Userspace Control
  subgraph L4["Userspace"]
    F["Control and Monitoring"]
    F1["BPF Maps (DSQ state, task metadata)"]
    F2["Configurations / Parameters"]
    F3["Metrics Collection (latency, CPU usage, throughput)"]
  end

  %% Layer 5: Task Lifecycle
  subgraph L5["Task Lifecycle Flow"]
    T1["Task Runnable → ops.enqueue()"]
    T2["Insert into DSQ (local / global / custom)"]
    T3["ops.dispatch() picks task from DSQ"]
    T4["CPU executes task (ops.running())"]
    T5["Time slice decremented (ops.tick())"]
    T6["ops.stopping() → re-enqueue or exit"]
    T7["ops.quiescent() if idle"]
  end

  %% Connections
  A -->|Interface 1| B
  B -->|SCX Ops Interface| E
  E -->|Uses DSQs| C
  E -->|Helper Calls| D
  E <-->|BPF Maps / Configs| F
  D --> C
  E --> T1
  T1 --> T2
  T2 --> T3
  T3 --> T4
  T4 --> T5
  T5 --> T6
  T6 --> T7
  T6 --> T2  

```
---
## Diagram of the interaction between the sched_ext Framework (Kernel Layer) and the BPF Scheduler (Policy Layer)

```mermaid
flowchart TB
  subgraph Framework["sched_ext Framework (Kernel Layer)"]
    F1["Core Scheduler Hooks (enqueue_task_scx, pick_next_task_scx)"]
    F2["Dispatch Queues (Global, Local, Custom DSQs)"]
    F3["Helper Functions (scx_bpf_dispatch, scx_bpf_consume, scx_bpf_select_cpu)"]
  end

  subgraph BPF["BPF Scheduler (Policy Layer)"]
    B1["struct sched_ext_ops callbacks"]
    B2["Custom task placement & CPU selection"]
    B3["Custom policies (latency, fairness, batching)"]
  end

  subgraph Userspace["Userspace Control"]
    U1["BPF Maps & Metrics"]
    U2["Policy configuration & monitoring"]
  end

  %% Interaction
  F1 -->|Interface 1| B1
  B1 -->|Uses Helpers| F3
  B2 -->|Enqueue tasks into DSQs| F2
  B3 -->|Reads/Writes Config| U1
  U2 -->|Updates Policy/Maps| B1


```

### How to read this:

- Framework sits on top of the core scheduler, exposing hooks and helpers.

- BPF Scheduler implements custom scheduling logic using struct sched_ext_ops.

- Tasks flow from core scheduler → framework → BPF scheduler for decisions.

- DSQs (Dispatch Queues) are managed by the framework but populated according to the BPF policy.

- Userspace can read/write BPF maps for metrics or update scheduler policies at runtime.