
# RISC-V Processor with Co-Processor Coupled AES-128 Accelerator

A high-performance, structurally verified 32-bit RISC-V processor implementation in Verilog HDL featuring a tightly coupled, hardware-accelerated **AES-128 Cryptographic Coprocessor**. 

The architecture features a specialized Custom Instruction Decoder within the CPU execution datapath. When a dedicated cryptographic opcode is decoded, the pipeline dynamically stalls instruction fetching, transfers spatial register matrices to the crypto-core, and computes a full 10-round NIST-compliant AES block in exactly 10 clock cycles with zero bus-latency overhead.

---

## 🛠️ Processor Architecture Overview

The core is structured as a modified single-cycle Harvard-architecture RISC-V processor augmented with a specialized hardware accelerator pipeline. Rather than routing data over a shared system bus (AHB/AXI) which creates memory bottlenecks, the AES engine sits directly inside the processor core as a co-processor module.

```text
       ┌───────────────────────── RISC-V CORE ─────────────────────────┐
       │                                                               │
       │   ┌───────────┐       ┌──────────────┐        ┌───────────┐   │
       │   │  Program  │──────>│ Instruction  │───────>│  Control  │   │
       │   │  Counter  │       │    Memory    │        │   Unit    │   │
       │   └─────▲─────┘       └──────────────┘        └─────┬─────┘   │
       │         │                                           │         │
       │         │ Stall Signal                              │ Stalls  │
       │         └───────────────────────────────────────────┼─────────┼──┐
       │                                                     │         │  │
       │                       ┌──────────────┐              │         │  │
       │                       │ Register File│              │         │  │
       │                       │  (X0 - X31)  │              │         │  │
       │                       └──────┬───────┘              │         │  │
       │                Plaintext Data│ (128-bit Matrix)     │         │  │
       │                (X10 - X13)   ▼                      ▼         │  │
       └──────────────────────────────┼──────────────────────┼─────────┘  │
                                      │                      │            │
       ┌──────────────────────────────┼──────────────────────┼────────────┘
       │                              ▼                      ▼
       │                    ┌──────────────────────────────────┐
       │                    │    AES-128 CO-PROCESSOR ENGINE   │
       │                    ├──────────────────────────────────┤
       │                    │  • Parallel S-Box Byte Sub       │
       │                    │  • Non-Linear ShiftRows Matrix   │
       │                    │  • GF(2^8) Galois MixColumns     │
       │                    │  • Dynamic On-The-Fly Key Gen    │
       │                    └──────────────────┬───────────────┘
       │                                       │ Ciphertext 
       │                                       ▼ (128-bit Writeback)
       └───────────────────────────────────────┘

```

---

## 📐 Datapath & Control Unit Mechanics

### 1. The Datapath

The datapath handles 32-bit scalar data routing but dynamically clusters registers for wide-word operations:

* **Program Counter (PC):** Sequential 32-bit memory word pointers matching the expression $PC_{next} = PC_{curr} + 4$.
* **Wide-Register Assembly:** The datapath intercepts standard 32-bit register channels and pairs registers `X10`, `X11`, `X12`, and `X13` side-by-side to construct a single, wide 128-bit plaintext block (`reg_plaintext`) fed directly into the encryption module inputs.
* **Writeback Safety:** Features a strict transition-edge pulse generator (`aes_done_pulse`). This ensures ciphertext data updates the general-purpose registers synchronously on a single clock edge when processing rounds conclude, preventing latch creation and pipeline hazards.

### 2. Control Unit & Pipeline Stalling

The control unit incorporates an auxiliary decoder that evaluates instruction bitfields continuously:

* **Instruction Sniffing:** It detects the custom hardware instruction sequence (`32'h04250533`).
* **Interlock Assertion:** Upon detection, the controller flags `is_aes = 1`, asserting a structural pipeline stall (`stall = aes_busy`).
* **FSM Handshake:** The program counter is frozen, keeping the current instruction locked inside the execution phase. Simultaneously, the controller raises the `start` signal to initiate the state machine of the AES coprocessor.

---

## 📋 Instruction Set Architecture (ISA) & Format

The core supports base standard ALU operations via standard RISC-V formatting alongside the custom hardware-acceleration instruction.

### Supported Instructions

* **Base System:** `NOP` (`32'h00000013`)
* **Custom ISA Extension:** `AES_ENC` (`32'h04250533`)

### Custom Crypto Instruction Format

The core decodes the customized `AES_ENC` instruction using a specialized R-type layout format:

```text
+-----------+-------+-------+--------+-------+----------+
|  funct7   |  rs2  |  rs1  | funct3 |  rd   |  opcode  |
+-----------+-------+-------+--------+-------+----------+
|  0000010  | 00101 | 00000 |  101   | 00110 |  0110011 | -> Hex: 32'h04250533
+-----------+-------+-------+--------+-------+----------+

```

---

## ⚡ Integrated AES-128 Coprocessor Architecture

The accelerator contains a fully compliant 128-bit structural implementation of the NIST AES-128 standard. It operates using an internal synchronous Finite State Machine (FSM) that drives four execution steps:

1. **Pre-Round Layer (`AddRoundKey`):** Bitwise XORs the raw 128-bit `reg_plaintext` matrix directly against the master 128-bit `base_key`.
2. **SubBytes Transformation:** A fully unrolled array of 16 structural instance lookup tables (`aes_sbox`) executes non-linear byte-substitution simultaneously within a single clock period.
3. **ShiftRows Layer:** A zero-latency hardwired routing matrix that transposes the data bytes across rows following standard matrix permutation rules.
4. **MixColumns Layer:** Employs Galois Field ($GF(2^8)$) linear multiplier functions ($g2(x)$) via conditional bit-shifting and polynomial reducing logic (`8'h1b`) to achieve inter-byte diffusion.
5. **Dynamic Key Expansion:** Incorporates a hardware key-generator that expands the round key on-the-fly using standard Round Constants (`rcon`). This saves massive amounts of silicon real-estate compared to a static pre-calculated lookup block memory.

---

## 💾 Memory & Interface Boundaries

* **Instruction Memory Interface:** Word-addressable structure (`Rom[addr[7:2]]`) that defaults to feeding execution `NOP` instructions (`32'h00000013`) to prevent core runaways, unless explicitly addressing loaded initial programs.
* **Register File Rules:** Implements a standard $32 \times 32$-bit dual-read asynchronous register matrix. Register `X0` remains securely anchored to zero through hardwired assignments (`assign read_data1 = (Rs1 == 0) ? 32'b0 : ...`), eliminating pipeline corruption during data writebacks.

---

## 📊 Verification & Simulation Methodology

The design was fully simulated and verified inside the **AMD Xilinx Vivado Design Suite** using a behavioral structural testbench (`tb_top`).

### Verification Strategy

1. **Reset Characterization:** The system is initially held inside a master reset loop (`rst = 1`) until `25ns` to verify that all registers drop to uniform, predictable initial states.
2. **NIST Vector Testing:** The registers `X10` through `X13` are injected with standard NIST hardware verification values:
* **Plaintext:** `00112233_44556677_8899aabb_ccddeeff`
* **Master Key:** `000102030405060708090a0b0c0d0e0f`


3. **Self-Checking Assertion:** The simulation runs a hardware verification routine comparing the runtime register results against the standard 128-bit NIST milestone target vector: `69c4e0d86a7b0430d8cdb78070b4c55a`.

### Simulation Log Sample

```text
--- Initial State Check (1ns) ---
R10 initial: 00112233 | R11 initial: 44556677

--- Reset Released at 25ns ---
[35ns] AES Unit started...
Status: AES Coprocessor detected BUSY. Encryption in progress...
[135ns] AES Unit finished.

--- AES Operation Complete ---
Final Result Stored in Registers:
R10: 69c4e0d8 | R11: 6a7b0430 | R12: d8cdb780 | R13: 70b4c55a

[RESULT] SUCCESS: Ciphertext matches NIST Test Vector!
--- Simulation Finished ---

```

---

## 📈 Performance Considerations

* **Deterministic Execution:** The encryption routine runs in exactly 10 clock cycles, guaranteeing absolute clock-cycle predictability for cryptographic workflows.
* **Bus Optimization:** Direct datapath registers-to-accelerator binding reduces standard setup times by avoiding typical high-latency system bus handshakes (like AXI or AHB).
* **Power-On Stability:** Register write operations rely on strict edge-pulses, guaranteeing that data remains pure and safe from glitches at time zero.

---

## 🚀 Future Improvements

* **Pipelined Crypto Core:** Upgrade the sequential 10-round loop into an unrolled 10-stage physical hardware pipeline to process one full data block every single clock cycle.
* **Decryption Pipeline:** Integrate a matching structural `aes_decrypt` block utilizing inverse S-Boxes and Inverse MixColumns logic to support symmetric decryption routines.
* **Full ISA Assembly Support:** Expand the basic control unit decoder to include full standard RV32I ALU arithmetic, branch handling, and standard data load/store memory instructions.

```

```
