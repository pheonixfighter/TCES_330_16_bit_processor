<div align="center">

# TCES 330 16-Bit Processor

### Authors: Landon Wardle and Robert Cromer
### Spring 2026

A 16-bit programmable processor written in SystemVerilog, simulated in ModelSim and run on an Altera DE2-115 FPGA board.

<!-- Badge row - these render as little pills on GitHub -->
![Language](https://img.shields.io/badge/language-SystemVerilog-8A2BE2?style=for-the-badge)

![Tools](https://img.shields.io/badge/tools-Quartus%20%7C%20ModelSim-0071C5?style=for-the-badge)

![Board](https://img.shields.io/badge/board-DE2--115%20(Cyclone%20IV%20E)-2b6cb0?style=for-the-badge)

</div>

---

## Table of Contents

1. [Overview](#overview)
2. [Repository Layout](#repository-layout)
3. [Instruction Set](#instruction-set)
4. [Architecture](#architecture)
5. [Simulation in ModelSim](#simulation-in-modelsim)
6. [Running on the DE2-115](#running-on-the-de2-115)
7. [Modules and Files](#modules-and-files)
8. [Example Program](#example-program)

---

# Overview

The processor has a **16-bit datapath**, a **16 x 16 register file**, **128 words of instruction ROM**, and **256 words of data RAM**.
A finite state machine in the control unit drives the fetch, decode, and execute cycle for six instructions: `NOOP`, `STORE`, `LOAD`, `ADD`, `SUB`, and `HALT`.

We built it in phases, testing each one before starting the next:

| Phase | Focus |
|:------|:------|
| **I** | ALU design and testbench, hand-assembled test program (`ProjectPhaseIPart1.txt`), ALU tested on the DE2-115 |
| **II** | 16 x 16 register file, then the register file and ALU combined (`regALU`) |
| **III** | Control FSM state diagram (`ControlFSM.drawio`) |
| **IV** | Datapath: RAM, 2-to-1 mux, ALU, and register file |
| **V** | Control unit: program counter, instruction ROM, instruction register, and FSM |
| **Final** | Full processor in `Processor/`, wired to the DE2-115 board through `Project.sv` |

# Repository Layout

```
TCES_330_16_bit_processor/
├── PhaseI/        ALU, 7-segment decoder, and the hand-assembled test program
├── PhaseII/       Register file and register file + ALU
├── PhaseIII/      Control FSM diagram (draw.io)
├── PhaseIV/       Datapath and data memory (RAM)
├── PhaseV/        Controller, FSM, PC, IR, and instruction memory (ROM)
├── Processor/     The complete processor: the files you will actually build and simulate
└── TestMIFs/      Extra memory initialization files for testing
```

> [!NOTE]
> The `Processor/` folder holds the final version of every module. The `Phase*` folders keep the earlier versions, so a module in a phase folder may be older than its copy in `Processor/`.

# Instruction Set

Each instruction is 16 bits. The top 4 bits are the opcode and the remaining 12 bits are operands.

| Opcode | Mnemonic | Encoding | Operation |
|:------:|:---------|:---------|:----------|
| `0` | `NOOP` | `0000 xxxx xxxx xxxx` | Do nothing |
| `1` | `STORE` | `0001 rrrr dddd dddd` | `D[d] = RF[r]` |
| `2` | `LOAD` | `0010 dddd dddd rrrr` | `RF[r] = D[d]` |
| `3` | `ADD` | `0011 aaaa bbbb cccc` | `RF[c] = RF[a] + RF[b]` |
| `4` | `SUB` | `0100 aaaa bbbb cccc` | `RF[c] = RF[a] - RF[b]` |
| `5` | `HALT` | `0101 xxxx xxxx xxxx` | Stop the processor |

`r`, `a`, `b`, and `c` are 4-bit register addresses. `d` is an 8-bit RAM address.

# Architecture

The processor is split into two halves, joined in `Processor.sv`:

- **Controller** (`Controller.sv`): the program counter, instruction ROM, and instruction register (wrapped together in `ROM_PC_IR.sv`), plus the control FSM.
- **Datapath** (`Datapath.sv`): the data RAM, a 2-to-1 mux that selects the register file's write data (ALU or RAM), the register file, and the ALU.

### Control FSM

| # | State | What it does | Next State |
|:-:|:------|:-------------|:-----------|
| 0 | `Init` | Clears the PC | `Fetch` |
| 1 | `Fetch` | Loads the IR and increments the PC | `Decode` |
| 2 | `Decode` | Reads the opcode | Depends on opcode |
| 3 | `NOOP` | Nothing | `Fetch` |
| 4 | `LOAD_A` | Sets the RAM address and selects RAM as the register file input | `LOAD_B` |
| 5 | `STORE` | Writes register `r` to RAM address `d` | `Fetch` |
| 6 | `ADD` | Writes `RF[a] + RF[b]` to `RF[c]` | `Fetch` |
| 7 | `HALT` | Stays here until reset | `HALT` |
| 8 | `SUB` | Writes `RF[a] - RF[b]` to `RF[c]` | `Fetch` |
| 9 | `LOAD_B` | Writes the RAM output into register `r` | `Fetch` |

`LOAD` takes two states because the RAM is synchronous, so its output is not ready until the cycle after the address is set. Holding `ResetN` low sends the FSM back to `Init`, and an unknown opcode does the same.

### ALU Operations

The FSM only uses `ADD` (1) and `SUB` (2), but the ALU supports eight operations through its 3-bit select line:

| `Sel` | Operation | | `Sel` | Operation |
|:-----:|:----------|-|:-----:|:----------|
| `0` | Zero | | `4` | `A ^ B` |
| `1` | `A + B` | | `5` | `A \| B` |
| `2` | `A - B` | | `6` | `A & B` |
| `3` | `A` (pass-through) | | `7` | `A + 1` |

# Simulation in ModelSim

All simulation is done from the `Processor/` folder. The project was built with **Quartus Prime / ModelSim-Intel FPGA Edition 20.1**.

1. Run `Launch_ModelSim.bat` to open ModelSim in the `Processor/` folder.
   It expects ModelSim at `C:\intelFPGA\20.1\modelsim_ase\`. Edit the path if yours is installed elsewhere.
2. In the ModelSim transcript, run the script for the testbench you want:

```tcl
do runrtl.do
```

| Script | Testbench | What it tests |
|:-------|:----------|:--------------|
| `runrtl.do` | `testProcessor` | **The full processor** running `ROM_instructions.mif` until `HALT` |
| `controllerrtl.do` | `Controller_tb` | The PC and IR follow the FSM, and reset works mid-program |
| `Datapathrtl.do` | `Datapath_tb` | Load, store, add, and subtract through the datapath |
| `runrtlFSM.do` | `FSM_tb` | Control signals for every instruction, and that `HALT` holds |
| `runROM_PC_IR.do` | `ROM_PC_IR_tb` | The PC, ROM, and IR working together (uses `ROM_test.mif`) |
| `InstMemoryrtl.do` | `InstMemory_tb` | Instruction ROM reads |
| `DataMemoryrtl.do` | `DataMemory_tb` | Data RAM reads and writes |
| `runALU.do` | `ALU_tb` | Every select line against every input (at 3 bits wide) |
| `runRegFile.do` | `regfile16x16_tb` | Register file writes and reads on both ports |
| `runPC.do` | `PC_tb` | Counting, holding, clearing, and wrapping |
| `runIR.do` | `IR_tb` | Loading, holding, and clearing |
| `runMux.do` | `Mux_tb` | Fixed and random inputs |
| `runButtonSynchronizer.do` | `ButtonSynchronizer_tb` | One-cycle pulse output |

> [!IMPORTANT]
> The ROM and RAM are Quartus IP cores (`InstMemory.v`, `DataMemory.v`) that load `.mif` files when the simulation starts.
> The full processor needs **`ROM_instructions.mif`** in the ROM and **`RAM_processor.mif`** in the RAM, which is the default.
> `ROM_PC_IR_tb` expects `ROM_test.mif` instead, so change `init_file` in `InstMemory.v` before running it.

> [!NOTE]
> Testbenches that Quartus can't synthesize are wrapped in `` `ifdef MODEL_TECH ``, so they only compile in ModelSim and the same source files work in both tools.

# Running on the DE2-115

`Project.sv` is the top-level module for the board. Create a Quartus project targeting the **Cyclone IV E (EP4CE115F29C7)**, add the `.sv` and `.v` files from `Processor/` (not the testbench-only files), set `Project` as the top-level entity, import the DE2-115 pin assignments, then compile and program the board.

### Controls

| Input | Function |
|:------|:---------|
| `KEY[2]` | **Clock.** Each press steps the processor forward one clock cycle |
| `KEY[1]` | **Reset** (active-low). Hold it down and press `KEY[2]` to return to `Init` |
| `SW[17:15]` | Chooses what `HEX7` to `HEX4` show (see below) |
| `LEDR` / `LEDG` | Light up to match the switches and keys |

`KEY[2]` goes through `ButtonSynchronizer` and `KeyFilter`, so one press gives exactly one clean clock pulse.

### Display

`HEX3` to `HEX0` always show the **current instruction register**. `SW[17:15]` selects what `HEX7` to `HEX4` show:

| `SW[17:15]` | `HEX7` to `HEX4` |
|:-----------:|:-----------------|
| `0` | `HEX7` and `HEX6` = PC, `HEX5` = 0, `HEX4` = current state |
| `1` | ALU A input |
| `2` | ALU B input |
| `3` | ALU output |
| `4` | `HEX7` = next state, the rest are 0 |
| `5` to `7` | Unused (all zeros) |

# Modules and Files

### Top Level:
 - **Project.sv**: Board-level wrapper. Connects the processor to the keys, switches, LEDs, and 7-segment displays.
 - **Processor.sv**: The processor itself, made of the controller and the datapath.
 - **testProcessor.sv**: Full-processor testbench. Runs until the IR holds `HALT` (`5000`).

### Control Unit:
 - **Controller.sv**: Joins `ROM_PC_IR` with the `FSM`.
 - **FSM.sv**: Control state machine. Sets every control signal from the current state and instruction.
 - **ROM_PC_IR.sv**: Program counter, instruction ROM, and instruction register wired together.
 - **PC.sv**: 7-bit program counter with clear and increment.
 - **IR.sv**: 16-bit instruction register with load and clear.
 - **InstMemory.v**: 128 x 16 instruction ROM (Quartus IP), initialized from `ROM_instructions.mif`.

### Datapath:
 - **Datapath.sv**: Joins the RAM, mux, register file, and ALU.
 - **DataMemory.v**: 256 x 16 data RAM (Quartus IP), initialized from `RAM_processor.mif`.
 - **regfile16x16.sv**: 16 registers of 16 bits each. Two read ports, one write port, and writes happen on the falling clock edge.
 - **ALU.sv**: 8-operation ALU with a parameterized width.
 - **Mux.sv**: 16-bit 2-to-1 mux that picks the register file's write data.

### Board I/O:
 - **ButtonSynchronizer.sv**: Moore machine that turns a button press into a single-cycle pulse.
 - **KeyFilter.sv**: Allows at most 10 pulses per second on the 50 MHz clock to debounce the clock key.
 - **MuxnW_8to1.sv**: n-bit 8-to-1 mux that picks what `HEX7` to `HEX4` show.
 - **Decoder.sv**: Hex digit to 7-segment display decoder.

### Memory Files:
 - **ROM_instructions.mif**: The program the processor runs (see [Example Program](#example-program)).
 - **RAM_processor.mif**: The starting RAM values the program uses.
 - **ROM_test.mif / RAM_test.mif**: Test patterns for the memory and `ROM_PC_IR` testbenches.

# Example Program

The program in `ROM_instructions.mif` is the one from `PhaseI/Part1/ProjectPhaseIPart1.txt`. It loads four values from RAM, does two subtractions and an addition, then stores the result.

<details>
<summary><b>Program listing</b></summary>

<br>

| Addr | Hex | Instruction | Effect |
|:----:|:----|:------------|:-------|
| 0 | `21B1` | `LOAD 1B, R1` | `R1 = D[1B]` |
| 1 | `22A2` | `LOAD 2A, R2` | `R2 = D[2A]` |
| 2 | `23C3` | `LOAD 3C, R3` | `R3 = D[3C]` |
| 3 | `27E4` | `LOAD 7E, R4` | `R4 = D[7E]` |
| 4 | `4121` | `SUB R1, R2, R1` | `R1 = R1 - R2` |
| 5 | `4342` | `SUB R3, R4, R2` | `R2 = R3 - R4` |
| 6 | `312A` | `ADD R1, R2, RA` | `RA = R1 + R2` |
| 7 | `1A6A` | `STORE RA, 6A` | `D[6A] = RA` |
| 8 | `5000` | `HALT` | Stop |

</details>

<details>
<summary><b>Expected results</b></summary>

<br>

With the starting values in `RAM_processor.mif`:

| Register | Value | How |
|:---------|:------|:----|
| `R1` | `21BA` | Loaded from `D[1B]` |
| `R2` | `A04E` | Loaded from `D[2A]` |
| `R3` | `71AC` | Loaded from `D[3C]` |
| `R4` | `B17F` | Loaded from `D[7E]` |
| `R1` | `816C` | `21BA - A04E` |
| `R2` | `C02D` | `71AC - B17F` |
| `RA` | `4199` | `816C + C02D` (the carry out is dropped) |

The program ends with **`4199`** stored at `D[6A]`, and the IR showing `5000` on `HEX3` to `HEX0`.
In simulation, `testProcessor` prints each state change with the PC, IR, and ALU values, and stops at the `HALT`.

</details>

---

<div align="center">

<sub>`ButtonSynchronizer.sv` was co-written with Jenny Sheng. Built for TCES 330 at the University of Washington Tacoma.</sub>

</div>
