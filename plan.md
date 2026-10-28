# Q2 Implementation Plan: EDF Scheduling with Pipe Donation

## 1. Repository Audit

### Confirmed repository and xv6 variant

- Repository root: `/home/aadi/q2-deadline-scheduler`.
- The xv6-riscv source tree is in `xv6-public/`; therefore source paths below are relative to the repository root and include that prefix.
- The initial working tree was clean, and no root `plan.md` existed.
- This is a RISC-V multiprocessor xv6 variant with `NCPU` 8, `NPROC` 64, `PIPESIZE` 512, syscall numbers through `SYS_sync` 22, and a custom `printk()`.
- Unlike the handout's older-code examples, this tree has `kfork()` and `kexit()` in `kernel/proc.c`, no `fork()`/`exit()` implementation names there, and split sleep registration: `sleep_prepare(chan)` followed by `sleep()`.
- The repository root README describes `tests/`, `experiments/`, `docs/`, and `qemu-setup/`, but these directories are absent. `xv6-public/test-xv6.py` is the existing host-side QEMU test harness; it writes `test-xv6.out` when run.
- No existing workflow, deadline, scheduling-policy, or pipe-writer tracking fields/mechanisms were found.

### Existing relevant responsibilities and behavior

| File | Confirmed responsibility and relevant symbols |
|---|---|
| `xv6-public/kernel/proc.c` | Process allocation/lifecycle (`allocproc`, `freeproc`, `userinit`, `kfork`, `kexit`, `kwait`, `kkill`), process table and lock initialization (`procinit`), round-robin-ish `scheduler`, `yield`, and split sleep/wakeup (`sleep_prepare`, `sleep`, `wakeup`). Current scheduler scans table order and immediately runs each RUNNABLE process it encounters. |
| `xv6-public/kernel/proc.h` | `struct proc` has `lock`, state, `chan`, `killed`, pid, parent, address-space/context, files, cwd, and name. No workflow or dependency fields exist. `p->lock` protects state/chan/killed/pid; several other fields are private to the process by existing convention. |
| `xv6-public/kernel/param.h` | Defines process and CPU counts and general limits; no scheduling constants or no-deadline sentinel. |
| `xv6-public/kernel/defs.h` | Kernel cross-file prototypes; it already forward-declares `struct pipe` and `struct proc`. |
| `xv6-public/kernel/pipe.c` | Private `struct pipe` contains spinlock, ring buffer, monotonically increasing `nread`/`nwrite`, and open flags. `pipealloc`, `pipeclose`, `pipewrite`, `piperead` implement pipe lifecycle and blocking. No process writer is recorded. Readers call `sleep_prepare(&pi->nread)`, release the pipe lock, then `sleep()`; writers wake readers after writing. A full writer similarly sleeps on `&pi->nwrite`. |
| `xv6-public/kernel/file.c`, `kernel/file.h` | `struct file` references a pipe and has a shared refcount. `filedup()` increments the reference; `fileclose()` calls `pipeclose()` only when the shared file refcount reaches zero. Fork duplicates descriptor references. Thus pipe endpoint lifetime does not identify the currently executing writer process. |
| `xv6-public/kernel/sysproc.c` | Process and time syscall handlers, including `sys_fork`, `sys_exit`, `sys_kill`, `sys_pause`, and `sys_uptime`; uses `argint()`/`myproc()` and kernel process APIs. |
| `xv6-public/kernel/syscall.c`, `kernel/syscall.h` | Syscall declarations, dispatch table, and integer-number definitions. Highest current number is 22 (`sync`). `argint`, `argaddr`, `argstr` are provided in `syscall.c`. |
| `xv6-public/user/user.h`, `user/usys.pl` | User prototypes and generated syscall stubs. `usys.pl` emits the `ecall` wrappers from `entry("name")` declarations. |
| `xv6-public/Makefile` | Kernel object list, user program build rules, `UPROGS` filesystem-image inclusion, and QEMU configuration. `CPUS` defaults to 3 and is used by `QEMUOPTS`. |
| `xv6-public/user/usertests.c` | Large existing syscall/regression suite, with `pipe1`, `killstatus`, `killzero`, and extensive fork/pipe behavior tests. It can host focused correctness regressions, but is not a convenient configurable benchmark generator. |
| `xv6-public/test-xv6.py` | Host test runner that boots QEMU and can run named `usertests`; running it creates an output file, so it is only a future validation tool, not used in this planning task. |
| `xv6-public/kernel/printk.c` | Kernel diagnostic output facility, relevant to the requested donation event diagnostics. |

### Handout versus repository differences

The PDF's syscall, process, and pipe files are present in the expected xv6 subdirectory, but its code fragments are schematic. In particular, the example scheduler assumes one selected process at a time per scan, whereas the actual scheduler's loop is the per-CPU dispatch loop; EDF must select one runnable process per dispatch under correct `p->lock` handling. The example donation walk directly dereferences raw `struct proc *` links without synchronization and its `<=` early-stop condition is not sufficient to recompute stale donations. The actual split sleep protocol and descriptor reference sharing also require more lifecycle handling than the PDF sketch shows.

## 2. Requirements Breakdown

| Requirement | Planned work |
|---|---|
| Workflow deadlines and `setworkflow(wid, deadline_ticks)` | Add process workflow ID and own absolute deadline, define `NO_DEADLINE`, implement syscall validation and registration path. Treat argument as a relative number of ticks and convert to an absolute deadline using the current tick count under `tickslock`; `wid == 0` clears workflow and own deadline. A nonzero `wid` update establishes/updates this process's deadline; all processes in a workflow must share the same deadline, so define a serialized update policy that applies the supplied absolute deadline to current members of that `wid`. This policy should be explicit in implementation/report because the syscall signature alone does not specify whether calls update one process or the whole group. |
| EDF scheduling | Replace round-robin dispatch in `scheduler()` with minimum effective-deadline RUNNABLE selection, retain chosen process lock through switch, and rotate equal-deadline selection fairly. |
| Pipe writer/dependency tracking | Capture writer process identities on pipe writes and associate a reader blocked on an empty pipe with a candidate writer. Track dependency changes around sleep/wakeup and pipe closure. |
| Transitive donation and cycle safety | Propagate effective deadline along waiting dependencies for at least four stages; bound traversal to `NPROC` and detect repeated PIDs/slots. |
| Withdrawal and multiple donors | Clear/recompute donation when wait edges disappear or participants exit/are killed. Recommend recomputing from all live waiters rather than subtracting one stored donated value, because multiple waiters can target the same receiver. |
| Workload and measurements | Add a configurable user workload and host-side QEMU sweep description; test pipeline lengths, CPU work, deadlines, policies, CPU counts, 20 repetitions, misses, and donation events. |

## 3. Exact File Modification Manifest

All paths are relative to the repository root. The proposed code scope is only for a later implementation; no source changes are made by this plan.

| Repository-relative path | Existing or new | Functions/structures involved | Planned changes | Reason |
|---|---|---|---|---|
| `xv6-public/kernel/proc.c` | Existing | `procinit`, `allocproc`, `freeproc`, `userinit`, `kfork`, `kexit`, `kkill`, `scheduler`, `sleep_prepare`, `sleep`, `wakeup`; new helper(s) such as `effective_deadline` and donation recomputation | Initialize/reset workflow and donation fields; inherit workflow ID and own deadline on fork while resetting child donation/dependency; clear dependency before exit/reap; implement `ksetworkflow`; use EDF selection under per-process locks with tie rotation; add safe bounded dependency propagation and recomputation helpers. Keep lock scopes compatible with `sched()`'s requirement that only current `p->lock` is held. | Workflow lifecycle, EDF, dependency propagation, cycle bounds, cleanup. |
| `xv6-public/kernel/proc.h` | Existing | `struct proc`, `enum procstate` | Add `wid`, `deadline`, `donated`, and `waiting_on` fields; document lock ownership and define `NO_DEADLINE` in an appropriate shared header (recommended `param.h`). Consider dependency identity as PID plus validated process-table lookup instead of an unguarded raw pointer, or ensure raw pointer links are cleared before process reuse. | Required per-process scheduling and dependency state. |
| `xv6-public/kernel/param.h` | Existing | `NPROC`, `NCPU`; new `NO_DEADLINE`, optional scheduling policy selection constants | Define the infinity sentinel consistently with `uint` and add compile-time policy selector/configuration only if needed for RR/EDF/EDF+donation measurements. If scheduler policy remains runtime fixed to EDF, separate RR/EDF comparison may instead use named build configurations. | Shared sentinel, traversal bound, and reproducible policy comparison. |
| `xv6-public/kernel/pipe.c` | Existing | private `struct pipe`; `pipealloc`, `pipeclose`, `pipewrite`, `piperead` | Add bounded writer-candidate identity tracking and lock-protected access. Register active writer before/while supplying data; set reader dependency only while empty and about to sleep; remove it after wake/return; recompute donation when bytes arrive, writer side closes, or reader exits. Keep lock ordering explicit and do not sleep while holding another process lock. | Dependency source and edge lifecycle for donation. |
| `xv6-public/kernel/defs.h` | Existing | Process/scheduler prototypes | Declare `ksetworkflow`, deadline/dependency helpers, and any narrow pipe cleanup interface used across translation units. Avoid exposing private helpers unnecessarily. | Cross-file syscall and pipe-to-process integration. |
| `xv6-public/kernel/sysproc.c` | Existing | New `sys_setworkflow` | Parse `wid` and deadline argument with actual syscall argument helpers; validate values and call the process-layer routine. For absolute conversion, sample `ticks` under `tickslock`; handle overflow and reject negative relative deadlines. | Kernel syscall implementation. |
| `xv6-public/kernel/syscall.h` | Existing | `SYS_*` definitions | Allocate `SYS_setworkflow` after current `SYS_sync` number 22. | New syscall number. |
| `xv6-public/kernel/syscall.c` | Existing | syscall prototypes and `syscalls[]` | Declare `sys_setworkflow` and add the dispatch table entry. | Kernel dispatch registration. |
| `xv6-public/user/user.h` | Existing | system-call declarations | Declare `int setworkflow(int wid, int deadline_ticks);` using repository user API conventions. | User-visible syscall prototype. |
| `xv6-public/user/usys.pl` | Existing | `entry(...)` list | Add `entry("setworkflow")` to generate the user `ecall` stub. | User syscall wrapper. |
| `xv6-public/Makefile` | Existing | `UPROGS`, `QEMUOPTS`, user build pattern | Add the new workload executable to `UPROGS`; document/enable policy build configuration if chosen. Use existing `CPUS` variable for one-CPU and three-CPU runs. | Build and package benchmark into xv6 image; enable experiment configurations. |
| `xv6-public/user/deadlinebench.c` | NEW | `main`, stage worker logic | Add user workload with arguments for stage count, CPU work, output volume, workflow ID/deadline, and mode; launch pipeline stages, assign workflow, produce/consume more than `PIPESIZE` data, and report start/completion/miss information. Ensure each stage closes unused descriptors so pipe EOF behavior is correct. | Configurable user-space workload required by Q2. |
| `xv6-public/user/usertests.c` | Existing | test functions and test registry | Add narrowly scoped deterministic checks for syscall acceptance/rejection, deadline inheritance/reassignment semantics, EDF order/ties, and donation withdrawal or route integration to the dedicated workload. Keep long comparative experiments out of this large regression binary. | Existing regression suite is the right place for short kernel behavior checks. |
| `experiments/q2_sweep.py` | NEW | Host-side experiment runner | Add a host runner that boots each policy/CPU configuration for 20 runs, passes workload arguments, captures raw QEMU output, and extracts miss/donation records. It must not silently regenerate or overwrite evidence; choose unique run log names and record configuration metadata. | Reproducible matrix and raw logs required for evaluation; repository currently has no `experiments/` directory. |

### Writer tracking recommendation and limitation

`struct pipe` is the correct object for this metadata because fork/dup share the pipe endpoint through `struct file`, and tracking only a descriptor would miss shared endpoints. A single raw `struct proc *writer` is unsafe: multiple processes may write through inherited descriptors, and a process slot can be reused after exit. Recommend a bounded list of writer PIDs (monotonic IDs, not raw pointers), validated against live process-table entries whenever a donation is made; register a writer when it actually calls `pipewrite`, not merely when `pipealloc` runs. Bound the list by `NPROC`, de-duplicate PIDs, and make overflow explicit (skip donation and count/log the limitation rather than corrupt memory). Since this only records observed writers, donation targets the most recently active eligible writer; for multiple legitimate writers it is a best-effort expected-supplier model, not a proof that the chosen process will write next. A stronger future design would associate active writer processes with endpoint references and remove entries on descriptor close/exit, but the current shared `struct file` refcount does not identify which process dropped a reference. This limitation must be tested and reported.

## 4. Data Structures and Invariants

Proposed `struct proc` scheduling fields:

- `int wid`: `0` means no workflow.
- `uint deadline`: this process/workflow's own absolute deadline in the same tick domain as `uptime`; `NO_DEADLINE` means infinity.
- `uint donated`: best effective deadline currently lent by blocked dependents; `NO_DEADLINE` when no donation applies.
- `struct proc *waiting_on` is the handout's conceptual model, but implementation should prefer a `waiting_on_pid` plus validity/generation strategy or strictly clear raw pointers before any target slot is freed/reused. A PID is safer than a naked pointer but must be resolved under process-table locking and validated against process state.

Invariants and semantics:

1. `effective(p) = min(p->deadline, p->donated)`; never overwrite `deadline` with donated urgency.
2. `NO_DEADLINE` compares later than every valid finite deadline. Tick wraparound must be considered; either reject deadlines that overflow the supported horizon or use wrap-safe signed-difference comparisons consistently.
3. New processes start with `wid = 0`, own and donated deadlines `NO_DEADLINE`, and no wait dependency. `userinit` follows the same initialization.
4. `fork()` inherits `wid` and own deadline. Donation is derived from current waits and is not inherited; child starts with `donated = NO_DEADLINE`, no `waiting_on` edge.
5. `setworkflow(0, ...)` clears workflow and own deadline. A nonzero workflow assignment changes membership and deadline atomically under the chosen workflow-state serialization. Recommended semantics: `deadline_ticks` is relative to now, converted to an absolute tick deadline; all live members of the supplied workflow ID receive the same deadline. Moving a process to another ID removes its previous membership and makes it adopt the new workflow deadline; `wid=0` removes it from any workflow. The supplied PDF does not define reassignment/group-update semantics, so this is an explicit design decision requiring report documentation.
6. A process becoming runnable, sleeping on a non-pipe channel, waking from an empty-pipe wait, or exiting must have a correct dependency edge. The pipe path owns setting/clearing the pipe dependency; other sleeps must not retain stale pipe edges.
7. Recompute `donated` from live donor edges (minimum of their effective deadlines), then propagate downstream. Never attempt to undo a donation by assigning the donor's old deadline: a second, more urgent donor may remain.
8. Lock invariant: process scheduling fields are read/written under `p->lock` or a dedicated global donation/workflow lock with a documented order. Pipe wait metadata is guarded by `pi->lock`. Establish one lock order and avoid acquiring process locks in reverse from wakeup/exit paths. Do not hold multiple process locks across `swtch()`.
9. Pointer/PID invariant: no dependency may be followed after its target is ZOMBIE/UNUSED or its identity no longer matches; clear/recompute edges before slot reuse in `freeproc`.

## 5. System-Call Registration Plan

End-to-end registration for `setworkflow(wid, deadline_ticks)`:

1. `xv6-public/user/user.h`: add the public C prototype.
2. `xv6-public/user/usys.pl`: add `entry("setworkflow")`; the existing generator emits a stub loading `SYS_setworkflow` into `a7` and executing `ecall`.
3. `xv6-public/kernel/syscall.h`: allocate the next unused syscall ID, 23.
4. `xv6-public/kernel/syscall.c`: add `extern uint64 sys_setworkflow(void)` and `[SYS_setworkflow] = sys_setworkflow` to the dispatch table.
5. `xv6-public/kernel/sysproc.c`: implement `sys_setworkflow`, parsing both values with `argint()` and rejecting invalid workflow/deadline values before calling `ksetworkflow` in `proc.c`.
6. `xv6-public/kernel/defs.h`: declare the process-layer entry point.

There is no generated syscall assembly checked into the tracked source list; `usys.S` is produced by the Makefile rule during a future build and must not be manually added to this plan's source manifest. `setworkflow` should return `0` on success and `-1` for invalid arguments or deadline arithmetic overflow.

## 6. Scheduler Algorithm Plan

1. Replace `scheduler()`'s current in-order “dispatch each RUNNABLE process found” loop with one candidate selection per CPU scheduling iteration.
2. Scan `proc[]`, acquiring each candidate's `p->lock`. Read state and effective deadline while that lock is held. Keep the chosen process lock held while releasing a prior candidate lock; do not release all locks and later reacquire a winner that could have changed state.
3. For equal effective deadlines, use a per-CPU rotating start index or monotonically advancing round-robin cursor among ties. Avoid the PDF sketch's `<=` behavior as a sole fairness mechanism: a fixed scan order with equal deadlines needs an explicit rotating cursor, and multiple CPUs need per-CPU or globally synchronized cursor handling.
4. With winner lock held, set `RUNNING`, set `c->proc`, call `swtch`, clear `c->proc` on return, restore the existing interrupt bookkeeping behavior, and release the winner lock. If no process is runnable, use the existing `wfi` path.
5. Keep `intr_on()`/`intr_off()` behavior and the xv6 lock transfer convention: scheduler acquires process lock; the process releases it after switching into its context and reacquires it before returning to scheduler. `sched()` requires only `p->lock`, interrupts disabled, and `noff == 1`.
6. For NCPU>1, each CPU may select a different runnable process because selection and `RUNNING` transition occur under that process's lock. A process cannot be chosen by two CPUs at once. EDF is best effort across simultaneous dispatches; there is no global reservation or preemption of a lower urgency process already running on another CPU.
7. Re-read effective deadlines during each fresh selection. Changes to a running process's urgency affect future dispatches; this design does not preempt a currently running task solely because a more urgent task becomes runnable.

## 7. Priority Donation and Withdrawal Plan

### Initial donation and transitivity

- In `piperead()`, when `nread == nwrite && writeopen`, mark the reader as depending on the pipe's current eligible writer before the sleep handoff. Use the actual `sleep_prepare(&pi->nread)` sequence and ensure the edge is visible before the writer can wake it.
- In `pipewrite()`, record the calling PID as a writer candidate before making bytes visible/waking readers. On an empty-pipe waiter, donate the reader's effective deadline to the writer if it is earlier than the writer's current effective deadline.
- Follow the writer's own `waiting_on` dependency to the preceding producer, recomputing each receiver's donated value. Support at least four edges; use a visited set or bounded `NPROC` traversal and stop on repetition, dead target, no edge, or no improvement. Never block indefinitely or recurse without a bound.
- Donation event diagnostics should be behind a kernel print mode/configuration and include exactly donor PID, receiver PID, and new donated deadline. Avoid printing while holding locks that could make the console path deadlock; snapshot event fields and print after releasing locks where possible.

### Multiple donors and recomputation

Recommend recomputation by scanning all live process slots for dependency edges to a receiver and taking the minimum donor effective deadline, then propagating changes through affected upstream waiters. `NPROC` is only 64, making a bounded scan tractable and avoiding a separate dynamically allocated donor list. A donor list would make edge removal faster but adds lifetime, capacity, and lock-order hazards. Scans must snapshot state safely and must not hold one process lock while acquiring arbitrary other process locks in a cycle. A global donation lock can serialize graph updates; if selected, define order as donation lock before pipe lock/process locks everywhere, and never enter donation update while holding a lock that another graph operation takes first. Alternatively use snapshot/retry with single-process locks. The implementation should choose one complete scheme before coding.

### Withdrawal, kill, exit, and pipe closure

- On successful wake from an empty pipe, on the reader's killed return, and before it exits, clear that reader's dependency and recompute donated urgency along the formerly affected chain.
- When a writer is killed or exits, prune its PID from candidate tracking and recompute any reader's dependency to another eligible writer or leave the edge unresolved. A `kill()` syscall only sets `killed` and may wake a sleeper; actual cleanup must occur in the pipe read/write exit path and `kexit()`.
- When the last write endpoint closes, `pipeclose()` wakes empty readers; those readers should observe EOF and clear dependency. When last read endpoint closes, blocked writers wake and should cease being considered possible suppliers.
- If data arrives but does not satisfy a reader's full request, the current pipe code returns available bytes after waking; its dependency is removed for that read return and installed again only if a later read blocks empty.
- If a dependency changes because a writer candidate becomes stale or another writer is selected, recompute the graph from current edges rather than attempting local subtraction.
- Multiple readers may independently donate to the same writer. Removing one edge must preserve the minimum deadline from all remaining live waiters.

### Tricky correctness cases

- **Multiple writers:** a single `writer` pointer is incorrect for inherited/duplicated descriptors. Bounded PID candidates are safer than raw pointers but still approximate which writer will supply the next byte. Document this limitation and test competing writers.
- **Writer exits while a pipe remains open:** inherited descriptors mean `writeopen` can remain true after one writer exits. Candidate cleanup must not infer endpoint closure from one process's exit.
- **Stale PID and slot reuse:** PID lookup must compare the complete identity and live state. Never dereference a freed `proc *` retained in pipe metadata.
- **Cycles:** two processes can wait on pipes whose writers depend back on each other. Detect repeated nodes and stop; do not spin with interrupts disabled.
- **Lock inversion:** `piperead`/`pipewrite` hold `pi->lock` around `sleep_prepare`, which briefly acquires `p->lock`; `wakeup` scans and acquires process locks. Donation scans while holding `pi->lock` can add additional order edges. Keep critical sections short and establish/document a global lock order; do not naively call a recursive donation helper under `pi->lock` if it acquires unrelated process locks.
- **Concurrent CPUs:** donor removal and new donation can race. Serialize graph mutation or use versioned snapshots and retry; per-field unsynchronized writes are not acceptable.
- **Workflow changes during waits:** update own deadline and recompute effective donation for every dependent chain; previously propagated donations cannot simply remain cached.

## 8. Testing Plan

Use `xv6-public/user/usertests.c` for short deterministic regressions and new `xv6-public/user/deadlinebench.c` for configurable end-to-end scenarios. Use the existing `xv6-public/test-xv6.py` harness for future regression runs, recognizing it creates `test-xv6.out`.

| Test | Validates |
|---|---|
| `setworkflow` valid, zero workflow, invalid/overflow arguments | Syscall registration, dispatch, sentinel and validation behavior. |
| Fork a workflow member | Child inherits workflow ID and own deadline but begins without copied transient donation/dependency. |
| Reassign and clear workflow | Old workflow membership/deadline removed; new workflow semantics applied consistently to members. |
| Controlled CPU-bound jobs with distinct deadlines | EDF dispatch ordering and that finite deadlines precede `NO_DEADLINE`. |
| Many equal-deadline runnable jobs over repeated yields | Tie fairness; each runnable process receives CPU and high process-table indices do not starve. |
| One reader blocks on empty pipe; one writer later writes | Reader dependency is established and donation event uses donor PID, receiver PID, and effective deadline. |
| Write more than 512 bytes, then drain | Pipe capacity behavior really forces blocking; donation can occur and dependency clears as data arrives. |
| Four-stage and five-stage pipelines plus unrelated no-deadline CPU hog | Transitive propagation to all upstream stages; hidden five-stage scenario. |
| Two readers/donors targeting one writer, remove the more urgent donor first | Recompute preserves the remaining donor's deadline. |
| Reader exits or is killed while blocked | Dependency and donation are withdrawn without stale state. |
| Writer exits while pipe descriptor remains inherited/open | No stale pointer or false claim that the pipe is closed; candidate fallback/limitation is observable. |
| Two-process cyclic pipe dependency | Traversal terminates safely, no kernel hang, cycle policy is deterministic. |
| Multiple potential writers on same pipe | Candidate selection does not corrupt state; verify and document approximate supplier behavior. |
| Existing full `usertests`, `grind`, pipe and kill regressions | No regressions to existing process, pipe, sleep/wakeup, and file descriptor behavior. |
| One CPU and three CPUs | Lock correctness and required evaluation CPU configurations. |

Existing named `usertests` include pipe/fork and kill coverage; add specific new tests to the test table rather than replacing unrelated tests. The dedicated benchmark program is new because the existing shell/user tests are not parameterized for stage count, workload, deadline, and repeated measurements.

## 9. Experiment and Measurement Plan

Run the following matrix for 20 independent runs per cell:

| Dimension | Values |
|---|---|
| Scheduler policy | Round-robin baseline, EDF without donation, EDF with donation |
| QEMU CPU count | 1 and 3 (`CPUS=1` / `CPUS=3`) |
| Pipeline stages | 4-stage required demonstration; 5-stage hidden/adversarial scenario; selected lengths configurable in the workload |
| CPU work per stage | At least light, medium, and heavy calibrated loop counts, held constant within each comparison cell |
| Deadline | Configurable absolute target from workflow-relative tick budget; include an urgent pipeline and unrelated no-deadline hog |
| Runs | 20 per policy × CPU count × workload configuration |

Record per run: run ID, policy, CPU count, stage count, per-stage work, byte volume, assigned deadline, start/end tick or elapsed tick count, completion status, deadline miss (boolean and lateness), and each donation event `(donor_pid, receiver_pid, donated_deadline)`. Preserve raw QEMU logs and summarize miss count/rate and donation-event count in the host runner output.

To force actual blocking, have upstream stages generate more than `PIPESIZE` (512) bytes before the consumer can drain the pipe, and use CPU work to delay downstream reads. A writer's normal write fills the ring and sleeps on `nwrite`; the urgent reader must reach an empty pipe and sleep for donation to be exercised as well. Merely producing fewer than 512 bytes or allowing the reader to consume immediately is not evidence of donation. Synchronization gates/barriers in the workload should make the empty-read and full-write conditions reproducible and should avoid waiting forever if a stage fails.

Round-robin comparison can be provided through a compile-time policy selector in `param.h` or a documented separate source/build configuration; EDF without donation should use the same EDF selection but disable edge propagation. Keep policy changes reproducible and record the selected policy in every log. The user workload itself should not require separate hand-edited source for each policy.

## 10. Implementation Order

1. Record baseline process/pipe invariants and choose workflow update and time arithmetic semantics.
2. Add `struct proc` state, sentinel, initialization/reset, fork inheritance, and workflow syscall plumbing.
3. Implement EDF-only selection and tie rotation; validate against existing process tests and fairness cases.
4. Add selectable RR/EDF-no-donation/EDF-donation policy configurations so measurements compare consistent kernels.
5. Add pipe writer candidate tracking and waiting-reader dependency establishment/cleanup, with lifecycle rules.
6. Add bounded transitive donation and cycle detection; first validate four-stage and five-stage chains.
7. Implement donation recomputation/withdrawal for multiple donors, wakeups, workflow changes, kill/exit, and pipe closure.
8. Add deterministic adversarial regressions and configurable user workload; register it in `UPROGS`.
9. Add and run host experiment sweep for 1/3 CPUs × 3 policies × 20 runs per configuration, preserving raw logs and analyzing misses.
10. Run existing regression suite and document limits, observed races, and measurements.

## 11. Risks and Open Questions

### Confirmed facts

- Pipe writer identity is not currently recorded.
- A pipe's write endpoint may be shared by fork/dup because `filedup()` shares a refcounted `struct file`; `pipeclose()` sees only final endpoint-reference release.
- `sleep_prepare()` acquires the current process lock, while the pipe code calls it with `pi->lock` held. Any added donation locks must account for this existing order.
- The actual scheduler runs on multiple CPUs and the process lock is handed across `swtch()`.
- The build's `CPUS` default is 3; `PIPESIZE` is private to `kernel/pipe.c`.

### Design decisions that must be fixed before implementation

- Does `setworkflow` update every current member of `wid` or just the caller? Recommended group update, because the requirement says members share a deadline; serialize workflow updates and make reassignment semantics precise.
- Is `deadline_ticks` relative or absolute? Recommended relative budget converted to absolute uptime ticks; the signature name alone is ambiguous.
- Which writer should receive donation if several processes have written or can write to an open pipe? Recommended bounded live PID candidate set with best-effort active supplier selection; this remains a documented model limitation.
- Which synchronization strategy will protect the dependency graph? A single graph lock is simple but must have a strict order with `pi->lock` and process locks; snapshots/retry avoid nested locks but require careful validation.
- How will RR and EDF modes be selected reproducibly without a runtime policy syscall? Recommended compile-time policy selector and separate recorded builds.

### Concrete implementation risks

- Deadlock from adding a process-lock acquisition under pipe lock or from reverse-order graph scans.
- Stale donor or writer identity after exit/reap/slot reuse.
- Donation values remaining too urgent after wait edges, workflow deadline, or writer candidate changes.
- A fixed writer candidate list filling under many writers; overflow policy must be explicit.
- Time arithmetic wraparound and `NO_DEADLINE` ordering.
- Strict EDF can starve no-deadline work under a continuous stream of finite deadlines; decide whether assignment expects strict EDF (likely) and document starvation behavior.
- Equal deadlines across CPUs need fairness in candidate starting position; a shared cursor itself requires synchronization.
- A task already running on another CPU will not be preempted by this dispatch-only scheduler.
- External host sweep may need QEMU/toolchain environment not guaranteed by source tree; keep raw logs and separate environment failures from kernel results.

## 12. Final Allowed-Changes Checklist

Proposed future implementation scope from the manifest:

- Existing: `xv6-public/kernel/proc.c`
- Existing: `xv6-public/kernel/proc.h`
- Existing: `xv6-public/kernel/param.h`
- Existing: `xv6-public/kernel/pipe.c`
- Existing: `xv6-public/kernel/defs.h`
- Existing: `xv6-public/kernel/sysproc.c`
- Existing: `xv6-public/kernel/syscall.c`
- Existing: `xv6-public/kernel/syscall.h`
- Existing: `xv6-public/user/user.h`
- Existing: `xv6-public/user/usys.pl`
- Existing: `xv6-public/Makefile`
- Existing: `xv6-public/user/usertests.c`
- New: `xv6-public/user/deadlinebench.c`
- New: `experiments/q2_sweep.py`

This is the proposed implementation scope only. It is not permission to modify these files during the current planning task; the only file authorized for this task is repository-root `plan.md`.
