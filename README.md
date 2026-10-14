# Q2: Deadline Scheduler with Priority Donation Through Pipes

**Course:** CSL3030 — Operating Systems Lab
**Institute:** IIT Jodhpur
**Base OS:** xv6-riscv

## Objective
Implement workflow-aware Earliest Deadline First (EDF)
scheduling and priority donation through pipes.

## Project Structure
- `xv6-public/` — xv6-riscv source and modifications
- `tests/` — correctness and pipeline tests
- `experiments/` — experimental results and raw logs
- `docs/` — design notes and report drafts
- `qemu-setup/` — setup documentation

## Implementation Plan
1. Add setworkflow(wid, deadline_ticks).
2. Implement EDF scheduling.
3. Track pipe writers and waiting readers.
4. Propagate donated deadlines through dependency chains.
5. Detect cycles and withdraw stale donations.
6. Compare round-robin, EDF, and EDF with donation.
