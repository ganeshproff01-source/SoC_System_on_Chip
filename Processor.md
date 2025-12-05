# 🧠 TinyRISC Processor Core Documentation

## Overview
The **TinyRISC Processor Core** is a modular RISC-based CPU designed as the central processing unit for the TinySoC platform.  
It supports both **pipelined** and **non-pipelined** execution, enabling scalable performance and easier FPGA verification.

The processor implements a **3-stage pipeline**:
1. **Instruction Fetch (IF)**
2. **Instruction Decode (ID)**
3. **Execute / Memory / Writeback (EX–MEM–WB)**

It follows a **Harvard architecture**, keeping separate instruction and data memory interfaces.  
This allows concurrent access to program and data memory, improving throughput.

---

## 🔹 Key Features
- 16-bit instruction set, 16 general-purpose registers (R0–R15)
- 3-stage pipeline with hazard detection and forwarding
- Arithmetic, logical, branch, and interrupt support
- Integrated **CSR File** for control/status management
- **APB Bus** interface for peripherals
- Interrupt + Watchdog integration
- FPGA verified

---

## ⚙️ Internal Architecture

| Block | Description |
|:------|:-------------|
| **Program Counter (PC)** | Tracks current instruction and updates on branch/interrupt. |
| **Instruction Memory** | Stores program instructions for IF stage. |
| **Register File** | 16 × 16-bit registers (R0–R15). |
| **ALU** | Executes arithmetic/logic ops. |
| **Control Unit** | Decodes instructions and drives control signals. |
| **Pipeline Regs** | Hold inter-stage data. |
| **Data Memory I/F** | Manages read/write via APB. |

---

## 🧩 Pipeline Stages

### 1️⃣ Instruction Fetch (IF)

Fetches instructions from memory using PC.

```verilog
if (branch_taken)
    pc_next = branch_target;
else if (interrupt_pending)
    pc_next = isr_address;
else
    pc_next = pc_current + 1;
```

**Notes**
- Program Counter resets to address `0x40` on system reset.  
- Flash programming mode is isolated via `prg_mode_top`.

| Problem | Cause | Fix |
|:---------|:------|:----|
| PC increment error during flash mode | No gating on `clk_top` | Added gating with `prg_mode_top` |
| Wrong fetch after branch | Missing pipeline flush | Introduced IF/ID flush signal |

---

### 2️⃣ Instruction Decode (ID)

Interprets the instruction, reads operands and generates control.

```verilog
case (opcode)
    4'b0000: control = ADD_OP;
    4'b0001: control = SUB_OP;
    4'b0010: control = AND_OP;
    4'b0011: control = OR_OP;
    4'b0100: control = MOV_OP;
    4'b0101: control = JMP_OP;
    default: control = NOP;
endcase
```

Hazard detection example:

```verilog
if (ID_EX_Reg.rd == IF_ID_Reg.rs)
    stall_pipeline = 1;
```

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Operand fetched before previous writeback | Missing stall detection | Added hazard detection logic |
| Incorrect register read | Control signal timing | Synchronized register file read with clock edge |

---

### 3️⃣ Execute (EX)

Performs arithmetic, logical, and comparison operations.

```verilog
always @(*) begin
    case (alu_ctrl)
        ADD_OP: alu_result = opA + opB;
        SUB_OP: alu_result = opA - opB;
        AND_OP: alu_result = opA & opB;
        OR_OP : alu_result = opA | opB;
        CMP_OP: alu_result = (opA == opB) ? 1 : 0;
        default: alu_result = 0;
    endcase
end
```

Forwarding logic:

```verilog
if (EX_MEM_Reg.rd == ID_EX_Reg.rs)
    forwardA = 2'b10;
```

Branch mispredictions cause IF/ID flush.

---

### 4️⃣ Memory Access (MEM)

Interacts with data memory and peripherals via the APB bus.

```verilog
PADDR   <= alu_result;
PWDATA  <= reg_data2;
PWRITE  <= mem_write;
PRDATA  => mem_data_out;
```

| Cycle | Transition | Description |
|:------|:------------|:-------------|
| 1 | IDLE → SETUP | Bus address phase |
| 2 | SETUP → ACCESS | Data transfer starts |
| 3 | ACCESS → IDLE | Transfer completes |

| Problem | Cause | Solution |
|:---------|:------|:----------|
| Partial read | Premature `PREADY` assertion | Delayed FSM transition |
| Data write glitch | Unsynced address latch | Latch `PADDR` on negedge |

---

### 5️⃣ Writeback (WB)

Updates register file values.

```verilog
if (mem_to_reg)
    regfile[rd] <= mem_data_out;
else
    regfile[rd] <= alu_result;
```

**Notes**
- Writeback occurs only if `write_enable` = 1.  
- Forwarding prevents stale data usage.

---

## 🧠 Processor Control FSM

| State | Function | Transition Condition |
|:------|:-----------|:----------------------|
| RESET | Initialize registers, PC, and flags | After reset release |
| FETCH | Retrieve next instruction | Always |
| DECODE | Generate control signals | Instruction valid |
| EXECUTE | Perform ALU operations | Decode done |
| MEMORY | Read/write from memory | Memory access required |
| WRITEBACK | Update register | If `write_enable` active |

---

## 🧩 Interrupt and Watchdog Integration

### Interrupt Flow
1. `int_req` asserted by controller.  
2. Processor pushes `PC` and `Flags` into stack.  
3. Loads ISR address into PC.  
4. Executes ISR and returns via `IRET`.

### Watchdog Reset
- Watchdog timeout triggers `wdt_rst`, resetting the processor.  
- The reset vector reloads `PC = 0x40`.

---

## 🧪 Simulation and Verification

| Test Case | Description | Result |
|:------------|:-------------|:---------|
| Basic ALU operations | ADD, SUB, AND, OR, CMP | ✅ Passed |
| Branch handling | Conditional and unconditional jumps | ✅ Passed |
| Load/store operations | Memory access via APB | ✅ Passed |
| Interrupt response | High/Low priority ISR check | ✅ Passed |
| Watchdog timeout | Forced reset after missed refresh | ✅ Passed |

Artifacts:  
`simulation_waveforms/fetch_decode_pipeline.png`  
`timing_diagrams/branch_flow.png`  
`docs/fpga_test.jpg`

---

## 🧰 Problems Faced & Debugging Insights

| Problem | Cause | Solution |
|:----------|:--------|:-----------|
| Branch misprediction | PC updated before branch valid | Added PC mux gating |
| Load-use hazard | No stall mechanism | Implemented stall control |
| Data misalignment | APB access phase mismatch | Introduced negedge synchronization |
| Interrupt nesting failure | Stack overflow | Added nested flag in CSR |

---

## 🚀 Future Improvements
- Implement instruction cache for performance boost  
- Add hardware multiply/divide unit  
- Extend pipeline to 5 stages (separate MEM/WB)  
- Support dual ISR vector mapping  
- Integrate debug mode and trace buffer  

---

✅ **End of Processor Module Documentation**
