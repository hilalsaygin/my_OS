# CSE312 Operating Systems — Homework #1
**Gebze Technical University | Department of Computer Engineering**  
**Student:** R. Hilal Saygin | **ID:** 200104004111

---

## Project Overview

This project implements a custom micro-kernel operating system that runs in a VirtualBox virtual machine. It is split into two parts:

- **Part A (kernelA.iso):** Builds a multi-programming kernel with Round Robin scheduling, system calls, and process management.
- **Part B (kernelB.iso):** Extends Part A with Priority-Based (Preemptive) Scheduling and additional task strategies.

---

## Requirements

- **Host OS:** Ubuntu (recommended) or any Linux environment
- **VirtualBox:** Oracle VM VirtualBox (with a VM configured as described below)
- **Build Tools:** GCC cross-compiler, NASM assembler, GRUB, Make, xorriso

---

## Directory Structure

```
hw1/
├── kernelPartA/
│   ├── include/       # Header files
│   ├── src/           # Kernel source files
│   ├── linker.ld      # Linker script
│   ├── makefile       # Build system
│   └── ReadMe.txt
└── kernelPartB/
    ├── include/
    ├── src/
    ├── linker.ld
    ├── makefile
    └── ReadMe.txt
```

---

## Part A — kernelA.iso

### Build & Run

1. Navigate to the `kernelPartA/` directory.
2. Run the following command to compile and generate the ISO:
   ```bash
   make run
   ```
   This produces `kernelA.iso` and the `.bin` kernel binary.

3. In VirtualBox, create a virtual machine named **`kernel_A_OS`**.
4. Attach `kernelA.iso` as an optical disk under the VM's storage settings.
5. Start the VM from VirtualBox Manager, **or** re-run `make` from the project directory to auto-launch it.

> On Ubuntu, `make` will compile and automatically start the VM.

### Features

#### System Calls
System calls are invoked via software interrupt `int $0x80` using inline assembly. The call type is identified by an enum value placed in the `eax` register.

| System Call  | Description                              |
|--------------|------------------------------------------|
| `getPid`     | Returns the PID of the calling process   |
| `waitpid`    | Waits for a specified child process      |
| `exit_call`  | Terminates the calling process           |
| `sysprintf`  | Outputs a string to the console          |
| `fork`       | Creates a copy of the calling process    |
| `exec`       | Replaces process image with a new program|
| `addTask`    | Adds a new task to the process table     |

#### Process Management
- Processes are represented by a `Task` class holding: PID, PPID, state, CPU registers, stack (4 KiB), and wait PID.
- Process states: `READY`, `WAITING`, `FINISHED`
- Process table: array of up to 256 `Task` objects managed by `TaskManager`
- Task creation via `addTask` and `fork`; task termination via `exit_call`

#### Scheduling — Round Robin
- Each timer interrupt triggers a context switch via `robinScheduler(CPUState* cpustate)`
- Current task's CPU state is saved; the next READY task is selected using modular arithmetic (circular queue)
- Tasks in `WAITING` state are checked: if their waited-on process has `FINISHED`, they are promoted to `READY`

#### Interrupt Handling
- `InterruptManager` initialised at interrupt `0x20` (timer)
- `SyscallHandler` handles software interrupt `0x80`
- On interrupt: CPU state is saved, dispatch occurs, context switch happens if needed

#### Test Programs (Part A)
- **Collatz Conjecture:** Computes the Collatz sequence for all integers less than 100. Example output for input 60: `30 15 46 23 70 35 106 53 160 80 40 20 10 5 16 8 4 2 1`
- **Long Running Program:** Computes a double nested loop accumulation over n=1000. Expected result: `392146832`

Each program is loaded 3 times via `addTask`, with process table printed at every context switch.

---

## Part B — kernelB.iso

### Build & Run

1. Navigate to the `kernelPartB/` directory.
2. Run:
   ```bash
   make run
   ```
   This produces `kernelB.iso`.

3. In VirtualBox, create a virtual machine named **`kernel_B_OS`**.
4. Attach `kernelB.iso` as an optical disk.
5. Start the VM from VirtualBox Manager, or run `make`.

### New Features Over Part A

#### Priority-Based Preemptive Scheduling
- Tasks are assigned a `TaskPriority` attribute: `HIGH`, `MED`, or `LOW`
- `priorityScheduler(CPUState* cpustate)` scans all `READY` tasks and selects the one with the highest priority (lowest numeric value)
- The scheduler is invoked on every timer interrupt in place of (or alongside) Round Robin

#### Task Strategies

**initStrategy1 — Random Mixed Workload**
- A random number (0–3) is generated per iteration
- 10 tasks are added across 10 iterations, each randomly chosen from: `TaskLinearSearch`, `TaskBinarySearch`, `collatzTask`, `longRunningTask`
- Tests the scheduler's ability to handle diverse task types concurrently
- `waitpid` is called on the last added task before the strategy exits

**initStrategy2 — Repeated Same-Type Tasks**
- A single task type is randomly selected at the start
- Two outer iterations each add 3 instances of that same task type (6 total)
- `waitpid` is called on each of the 3 tasks per loop iteration
- Tests consistency and reliability when handling repetitive workloads

#### Test Programs (Part B)
- **Collatz Conjecture** — same as Part A
- **Long Running Program** — same as Part A
- **BinarySearch:** Input: `{10, 20, 80, 30, 60, 50, 110, 100, 130, 170}`, x = 110; Expected output: `6`
- **LinearSearch:** Input: `{10, 20, 80, 30, 60, 50, 110, 100, 130, 170}`, x = 175; Expected output: `-1`

---

## Key Design Decisions

- **Inline Assembly for Syscalls:** All system calls use `asm("int $0x80" ...)` with register constraints, closely mimicking real POSIX syscall conventions on x86.
- **Static PID Counter:** `Task::pIdCounter` is a static class variable, guaranteeing unique PIDs across all task instantiations.
- **4 KiB Per-Task Stack:** Each `Task` object allocates a fixed 4096-byte stack array. The CPU state is stored at the top of this stack.
- **Circular Round Robin:** Uses `(currentTask + 1) % totTaskCount` to cycle through tasks fairly.
- **Priority Enum Ordering:** `HIGH = 0, MED = 1, LOW = 2` — lower numeric value means higher priority, matching standard preemptive priority conventions.
- **WaitPid in Scheduler:** The Round Robin scheduler resolves waiting dependencies inline: if a `WAITING` task's waited-on process has `FINISHED`, it is promoted to `READY` in the same scheduling cycle.

---

## Notes

- Ensure VirtualBox is installed and the VM is created **before** running `make`.
- The VM name must match exactly: `kernel_A_OS` for Part A, `kernel_B_OS` for Part B.
- The process table is printed to the VM console on every context switch for observability.
- For grading purposes, syscall implementation and scheduling strategy correctness are the primary evaluation criteria.
