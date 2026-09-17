# AES-128 Hardware Implementation in Verilog

A synthesizable RTL implementation of the **Advanced Encryption Standard (AES-128)** written in Verilog HDL. The design supports both **encryption and decryption** of 128-bit data blocks using a 128-bit secret key.

The project is organized into independent AES transformation modules and a top-level sequential controller. It is intended for RTL simulation, FPGA implementation, and further study of cryptographic hardware architecture.

## Features

- AES-128 encryption
- AES-128 decryption
- 128-bit plaintext/ciphertext
- 128-bit encryption key
- 10 AES rounds
- Combinational AES transformation blocks
- Dedicated AES-128 key expansion
- S-Box and inverse S-Box implementations
- Clocked top-level control using an FSM-like round counter
- `start`, `done`, and `encrypt` control signals
- Testbench for encryption followed by decryption
- Suitable as a starting point for FPGA/ASIC RTL exploration

## AES-128 Overview

AES-128 operates on:

- **Block size:** 128 bits
- **Key size:** 128 bits
- **Number of rounds:** 10
- **State:** 16 bytes arranged as a 4 × 4 byte matrix

### Encryption

After the initial AddRoundKey operation, AES performs:

```text
Round 1–9:
    SubBytes
       ↓
    ShiftRows
       ↓
    MixColumns
       ↓
    AddRoundKey

Round 10:
    SubBytes
       ↓
    ShiftRows
       ↓
    AddRoundKey
```

The final round intentionally omits MixColumns.

### Decryption

Decryption applies the inverse transformations in the corresponding reverse-key order:

```text
Initial AddRoundKey
       ↓
InvShiftRows
       ↓
InvSubBytes
       ↓
AddRoundKey
       ↓
InvMixColumns
       ↓
...
Final InvShiftRows
       ↓
InvSubBytes
       ↓
AddRoundKey
```

## Project Structure

```text
.
├── AES-128 top.v       # Top-level AES-128 controller
├── gpt_sbox.v          # AES S-Box
├── sub_bytes.v         # SubBytes transformation
├── shift_rows.v        # ShiftRows transformation
├── mix_columns.v       # MixColumns transformation
├── key_expansions.v    # AES-128 key schedule
├── InvSubs.v           # Inverse SubBytes
├── InvShiftrows.v      # Inverse ShiftRows
├── InvMixColumns.v     # Inverse MixColumns (required by top module)
└── testbench.v         # Encryption/decryption testbench
```

> **Note:** The supplied top-level design instantiates a module named `InvMixColumns`. Make sure the corresponding `InvMixColumns.v` source file is present in the simulation/implementation project.

## Module Description

### `aes_128`

Top-level AES controller.

**Inputs**

| Signal | Width | Description |
|---|---:|---|
| `clk` | 1 | Clock |
| `rst` | 1 | Reset |
| `start` | 1 | Starts an AES operation |
| `encrypt` | 1 | `1` = encryption, `0` = decryption |
| `plaintext` | 128 | Input block |

| Signal | Width | Description |
|---|---:|---|
| `key` | 128 | AES-128 key |
| `ciphertext` | 128 | Output block |
| `done` | 1 | Indicates operation completion |

The implementation uses a round counter and processes one AES round per clock cycle after the initial AddRoundKey operation.

### `key_expansion`

Generates the complete AES-128 key schedule.

The 128-bit key is expanded into:

```text
11 round keys × 128 bits = 1408 bits
```

The implementation uses:

- RotWord
- SubWord
- Rcon
- XOR-based AES key schedule generation

### `sbox`

Implements the standard AES substitution box as a combinational lookup table.

### `sub_bytes`

Applies the S-Box independently to all 16 bytes of the AES state.

```text
128-bit state
     ↓
16 × 8-bit S-Box lookups
     ↓
128-bit transformed state
```

### `shift_rows`

Performs the AES ShiftRows transformation by cyclically shifting the state rows.

### `mix_columns`

Implements the AES MixColumns operation using GF(2^8) multiplication.

The implementation provides:

- Multiplication by 2
- Multiplication by 3
- `xtime()` operation

The standard AES MixColumns matrix is implemented as:

```text
[02 03 01 01]
[01 02 03 01]
[01 01 02 03]
[03 01 01 02]
```

### `InvShiftRows`

Performs the inverse ShiftRows transformation during decryption.

### `InvSubstitutionMatrix`

Applies the AES inverse S-Box to all 16 state bytes.

### `InvMixColumns`

Performs the inverse MixColumns transformation. This module is referenced by the top-level design and should be included in the project.

## Top-Level Operation

The top-level interface uses a simple handshake:

```text
        start
          │
          ▼
    ┌─────────────┐
    │ Initial Key │
    │   Addition  │
    └──────┬──────┘
           │
           ▼
     AES Round 1
           │
           ▼
          ...
           │
           ▼
     AES Round 10
           │
           ▼
       ciphertext
           │
           ▼
          done
```

A new operation is accepted only when the core is not busy.

### Encryption

Set:

```verilog
encrypt = 1'b1;
```

Then provide the 128-bit input block and 128-bit key and pulse `start`.

### Decryption

Set:

```verilog
encrypt = 1'b0;
```

and provide the ciphertext as the input block with the same AES key.

## Simulation

The supplied testbench uses a 100 MHz clock:

```verilog
forever #5 clk = ~clk;
```

It first performs encryption and then feeds the resulting ciphertext into the decryption path.

### Standard AES-128 Test Vector

The testbench uses the well-known AES-128 test vector:

```text
Plaintext:
00112233445566778899aabbccddeeff

Key:
000102030405060708090a0b0c0d0e0f
```

Expected AES-128 ciphertext:

```text
69c4e0d86a7b0430d8cdb78070b4c55a
```

The subsequent decryption should recover:

```text
00112233445566778899aabbccddeeff
```

## Running with Icarus Verilog

If Icarus Verilog is installed, compile all RTL files together.

Example:

```bash
iverilog -o aes_sim \
  "AES-128 top.v" \
  gpt_sbox.v \
  sub_bytes.v \
  shift_rows.v \
  mix_columns.v \
  key_expansions.v \
  InvSubs.v \
  InvShiftrows.v \
  InvMixColumns.v \
  testbench.v
```

Run the simulation:

```bash
vvp aes_sim
```

Expected output should contain the AES-128 ciphertext:

```text
Encrypted Ciphertext: 69c4e0d86a7b0430d8cdb78070b4c55a
```

and after decryption:

```text
Decrypted Plaintext: 00112233445566778899aabbccddeeff
```

## Running with Verilator

The RTL can also be checked with Verilator for linting and simulation.

For example:

```bash
verilator --lint-only \
  "AES-128 top.v" \
  gpt_sbox.v \
  sub_bytes.v \
  shift_rows.v \
  mix_columns.v \
  key_expansions.v \
  InvSubs.v \
  InvShiftrows.v \
  InvMixColumns.v
```

Address any tool-specific warnings before proceeding to synthesis.

## Timing and Architecture

The implementation is primarily **iterative**, rather than fully unrolled.

The same transformation datapath is reused across AES rounds, while the round counter determines which round is currently being processed.

Conceptually:

```text
             ┌───────────────────────┐
             │     Key Expansion     │
             └──────────┬────────────┘
                        │
                        ▼
Input ──► AddRoundKey ─► AES Round Datapath ─► Output
                              │
                              │
                       Round Counter
                              │
                              └──► 10 rounds
```

This approach reduces hardware duplication compared with a completely unrolled ten-round implementation, at the cost of multiple clock cycles per block.

## RTL Design Concepts Demonstrated

This project demonstrates several important RTL/VLSI concepts:

- Hierarchical Verilog design
- Module instantiation
- Combinational logic
- Sequential logic
- Round-based control
- Lookup-table implementation
- GF(2^8) arithmetic
- Key scheduling
- Datapath/control separation
- Parameterized byte extraction and concatenation
- Testbench-driven functional verification
- Encryption/decryption datapath reuse

## Verification

The testbench verifies the complete data path by performing:

```text
Plaintext
   +
AES Key
   │
   ▼
Encryption
   │
   ▼
Ciphertext
   │
   ▼
Decryption
   │
   ▼
Recovered Plaintext
```

The strongest basic functional check is:

```text
Decrypted Plaintext == Original Plaintext
```

For standards-based verification, the implementation should also be tested against multiple official AES-128 known-answer test vectors.

## FPGA / ASIC Next Steps

This RTL can be extended toward a complete digital design flow.

### FPGA

Possible next steps:

1. Simulate using Vivado/ModelSim/Questa/Verilator.
2. Synthesize the AES core.
3. Inspect LUT, FF, BRAM and DSP utilization.
4. Run timing analysis.
5. Add FPGA I/O interfaces.
6. Implement on a target FPGA board.
7. Measure throughput and latency.

### ASIC / RTL-to-GDSII

The design can also be used as an RTL starting point for:

```text
Verilog RTL
   ↓
Simulation / Verification
   ↓
Synthesis
   ↓
Gate-Level Netlist
   ↓
Floorplanning
   ↓
Placement
   ↓
Clock Tree Synthesis
   ↓
Routing
   ↓
DRC / LVS
   ↓
GDSII
```

Useful metrics to evaluate include:

- Area
- Maximum clock frequency
- Latency
- Throughput
- Power
- Power-delay product
- Area-delay product

## Possible Improvements

The current implementation is a functional RTL-oriented AES-128 core. For a more production-oriented hardware implementation, possible improvements include:

- Add an explicit `busy` output.
- Improve reset/start handshake behavior.
- Add parameterization for AES-128/192/256.
- Add SystemVerilog assertions.
- Add self-checking testbench assertions.
- Add more NIST/FIPS known-answer tests.
- Add waveform-based verification.
- Register critical datapath boundaries for higher clock frequency.
- Explore composite-field or optimized S-Box architectures.
- Explore resource-shared versus unrolled architectures.
- Add side-channel countermeasures where required.
- Perform synthesis and post-synthesis timing analysis.
- Evaluate FPGA and ASIC area/power/throughput trade-offs.

## Repository Usage

This project is intended for educational and research use in:

- RTL Design
- Digital VLSI
- FPGA Design
- Cryptographic Hardware
- Hardware Security
- Computer Architecture
- ASIC Design
- RTL-to-GDSII flows

## Author

**G Rahul Reddy**

B.Tech — Electronics & Communication Engineering  
PDPM IIITDM Jabalpur

## Disclaimer

This repository is an educational RTL implementation of AES-128. Functional correctness should be independently verified using recognized AES test vectors before the design is used in any security-critical application. Hardware cryptographic implementations may also require protection against timing, power-analysis, electromagnetic, fault-injection, and other side-channel attacks.
