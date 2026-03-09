## LDA Instruction – Example Waveform Values (Step‑By‑Step)

This file focuses **only on concrete values** you would see on the waveform for an **LDA** instruction, using one consistent example.

> These values are illustrative but consistent with the datapath and control described in `LDA_datapath.md`. Your exact hex values may differ depending on how you initialize instruction and data memory, but the **pattern** will match.

---

## 1. Example Setup

Assume the following initial conditions and memory contents:

- **Program Counter before LDA fetch**:  
  - `PC = 0x00000000`
- **Instruction memory**:  
  - `M[0x00000000] = 0xABCD0010`  
    - Upper 16 bits: opcode `0xABCD` (LDA)  
    - Lower 16 bits: address field `0x0010`
- **Data memory**:  
  - `M[0x00000010] = 0xDEADBEEF`
- **Registers**:  
  - `IR` initially undefined / previous value.  
  - `A` initially `0x00000000`.

We track the main signals you see in the LDA simulation:

- `clk`
- `PC`
- `addr` (to data memory)
- `EN`, `WEN`
- `MEM_OUT` / `data_out`
- `out_down/IR` (IR output)
- `out_down/A` (A output)
- `LD_IR`, `LD_A`

We describe values **just before** and **just after** the active clock edge of each cycle.

---

## 2. Cycle N – Fetch LDA Instruction

### 2.1 Control intent

- Use `PC` as the memory address to **fetch the instruction**.
- Load the fetched word into `IR`.
- Increment `PC` to point at the **next** instruction.

So control does:

- `addr_src = PC`
- `EN = 1`, `WEN = 0`
- `DATA_MUX = MEM_OUT`
- `LD_IR = 1`
- `LD_PC = 1`, `INC_PC = 1`
- `LD_A = 0`

### 2.2 Signal values

| Time point                   | PC              | addr            | EN,WEN | MEM_OUT       | out_down/IR   | out_down/A   |
|-----------------------------:|----------------:|----------------:|:------:|--------------:|--------------:|-------------:|
| **Just before edge (N)**     | `0x00000000`    | `0x00000000`    | `1,0`  | `0xXXXXXXXX`  | `0xXXXXXXXX`  | `0x00000000` |
| **Just after edge (N)**      | `0x00000004`    | `0x00000000`    | `1,0`  | `0xABCD0010`  | `0xABCD0010`  | `0x00000000` |

Explanation:

- Before the edge, `PC` already drives `addr = 0x00000000`, so memory is prepared to output `M[0]`.
- At the active clock edge:
  - Memory output becomes `MEM_OUT = 0xABCD0010`.
  - `IR` loads from the bus (`LD_IR=1`), so `out_down/IR` steps from an unknown/old value to `0xABCD0010`.
  - `PC` increments to `0x00000004`.  
  - Register `A` is untouched (`LD_A=0`), so `out_down/A` remains `0x00000000`.

On the waveform you see:

- **First big step** in `out_down/IR` to the instruction word `0xABCD0010`.
- `PC` line stepping from `0x00000000` to `0x00000004`.
- `out_down/A` still flat at zero.

---

## 3. Cycle N+1 – Execute LDA (Load from Memory into A)

### 3.1 Control intent

- Use the **low 16 bits of IR** as a memory address.
- Read the data word at that address.
- Load the result into register `A`.
- Keep `PC` and `IR` unchanged during this cycle.

So control does:

- `addr_src = zero_extend(IR[15..0])`
- `EN = 1`, `WEN = 0`
- `DATA_MUX = MEM_OUT`
- `LD_IR = 0`
- `LD_PC = 0`, `INC_PC = 0`
- `LD_A = 1`

### 3.2 Derived address

- `IR = 0xABCD0010`
- `IR[15..0] = 0x0010`
- Zero‑extend:
  - `addr = 0x00000010`

### 3.3 Signal values

| Time point                   | PC              | IR             | addr            | EN,WEN | MEM_OUT       | out_down/IR   | out_down/A   |
|-----------------------------:|----------------:|---------------:|----------------:|:------:|--------------:|--------------:|-------------:|
| **Just before edge (N+1)**   | `0x00000004`    | `0xABCD0010`   | `0x00000010`    | `1,0`  | `0xABCD0010`* | `0xABCD0010`  | `0x00000000` |
| **Just after edge (N+1)**    | `0x00000004`    | `0xABCD0010`   | `0x00000010`    | `1,0`  | `0xDEADBEEF`  | `0xABCD0010`  | `0xDEADBEEF` |

\*Immediately after `addr` changes to `0x00000010`, `MEM_OUT` starts to reflect `M[0x00000010]`. In many simulators you will see this transition happen slightly before or at the clock edge.

Explanation:

- `PC` is held (`LD_PC=0`), so it stays at `0x00000004`.
- `IR` is held (`LD_IR=0`), so `out_down/IR` stays `0xABCD0010`.
- `addr` has switched to `0x00000010` via the lower zero extender.
- With `EN=1`, `WEN=0`, the memory outputs `M[0x00000010] = 0xDEADBEEF`.
- At the clock edge:
  - `A` samples the bus (`LD_A=1`), so `out_down/A` steps from `0x00000000` to `0xDEADBEEF`.

On the waveform you see:

- `addr` jumping from `0x00000000` (instruction fetch) to `0x00000010` (data read).
- `MEM_OUT` transitioning from `0xABCD0010` to `0xDEADBEEF`.
- **Second big step** in `out_down/A` to `0xDEADBEEF`, while `out_down/IR` stays flat.

---

## 4. After Execute – Between Instructions

Until the control unit starts the next **fetch**:

- `PC` remains at `0x00000004`.
- `out_down/IR` remains at `0xABCD0010`.
- `out_down/A` remains at `0xDEADBEEF`.
- `EN`/`WEN` and mux selects may be idle / don’t‑care depending on your micro‑sequencer.

The waveform appears as **flat segments** for these signals, showing that no registers are being loaded and memory is not being actively accessed for LDA anymore.

---

## 5. Quick Reference Table (All Key Values)

| Cycle / phase      | PC            | IR             | IR[15..0] | addr          | MEM_OUT     | out_down/IR | out_down/A |
|--------------------|--------------:|---------------:|----------:|--------------:|------------:|------------:|-----------:|
| Before fetch (N)   | `0x00000000`  | `XXXXXXXX`     | `XXXX`    | `0x00000000`  | `XXXXXXXX`  | `XXXXXXXX`  | `0x00000000` |
| After fetch (N)    | `0x00000004`  | `0xABCD0010`   | `0x0010`  | `0x00000000`  | `0xABCD0010`| `0xABCD0010`| `0x00000000` |
| Before exec (N+1)  | `0x00000004`  | `0xABCD0010`   | `0x0010`  | `0x00000010`  | `0xABCD0010`| `0xABCD0010`| `0x00000000` |
| After exec (N+1)   | `0x00000004`  | `0xABCD0010`   | `0x0010`  | `0x00000010`  | `0xDEADBEEF`| `0xABCD0010`| `0xDEADBEEF` |

This is the **value‑only view** of LDA: you can line it up directly with your waveform viewer and check that the simulated values follow this pattern, even if the exact hex constants differ.

