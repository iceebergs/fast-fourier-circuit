# fast-fourier-circuit
Transistor-level VLSI implementation of a 4-point Fast Fourier Transform, laid out in Magic and simulated in IRSIM.

Given four complex time-domain samples `x[0..3]`, the chip computes the four complex
frequency-domain coefficients `F(0..3)` using a two-stage radix-2 butterfly, then holds them
for readout.

Top-level layout is in **`ring.mag`**.

## Specifications

| | |
|---|---|
| Function | 4-point DFT — 4 complex inputs → 4 complex coefficients |
| Data width | 6-bit real + 6-bit imaginary per sample |
| Input range | integers 0–15 (headroom reserved for butterfly growth) |
| Storage | 8 × 6-bit input RF, 8 × 6-bit scratch RF, 8 × 6-bit output RF |
| Arithmetic | 2 add/sub units, time-shared across both butterfly stages |
| Output | real and imaginary halves read out simultaneously, one coefficient per address |
| Process | SCMOS `SCN6M_DEEP`, 0.18 µm, 6 metal layers (λ = 0.09 µm) |
| Clocking | two-phase non-overlapping (`phi0` / `phi1`) |
| Pin count | 33 (21 input/power, 12 output) |
| Core size | ~3700 × 3700 λ ≈ 670 µm × 670 µm including pad ring |
| Status | designed for tape-out; further verification required before submitting |

---

## Algorithm

The chip evaluates the first four terms of the DFT with the twiddle factors reducing to sign swaps and real/imaginary exchanges, so no multiplier is needed — only addition, subtraction, and negation:

```
W₄⁰ =  1 = ( 1,  0)      W₄¹ = -j = ( 0, -1)
W₄² = -1 = (-1,  0)      W₄³ =  j = ( 0,  1)
```

**Stage 1** — four butterflies on the input pairs:

```
A_re = x0_re + x2_re      A_im = x0_im + x2_im
B_re = x0_re - x2_re      B_im = x0_im - x2_im
C_re = x1_re + x3_re      C_im = x1_im + x3_im
D_re = x1_re - x3_re      D_im = x1_im - x3_im
```

**Stage 2** — the twiddle factors fold in as the `D_im` / `D_re` cross-terms:

```
F_re(0) = A_re + C_re     F_im(0) = A_im + C_im
F_re(1) = B_re + D_im     F_im(1) = B_im - D_re
F_re(2) = A_re - C_re     F_im(2) = A_im - C_im
F_re(3) = B_re - D_im     F_im(3) = B_im + D_re
```

Both stages are the same add/subtract pattern, which is why one datapath serves both.

---

### Process
Values loaded into the input register file are selected in pairs by two 4:1 muxes, then a pair
of 2:1 muxes chooses between stage-1 and stage-2 operands — **the datapath is reused across
both stages to save area**. Each pass adds and subtracts simultaneously, latching one result
to the output register file and one into the scratch register file. Once `A_re` through `D_im`
are in the scratch RF, the same sequence runs again to produce `F_re(0)` through `F_im(3)`.

Two-phase non-overlapping clocking (`phi0` / `phi1`) is used throughout, and all state is held in
transmission-gate master–slave latches.

### Module inventory

| Block | Cells | Purpose |
|---|---|---|
| `ring.mag` | top level | Core + 4 × `pad` I/O pad groups, clock/reset distribution |
| `rf_wmux4copy.mag` | `new_regfile`, 48 × `latch_onefin`, 2 × `new_mux4` | Input register file with write muxing |
| `new_regfile.mag` | `decoder`, `reg_six0..3` | 8 × 6-bit register array + 3→8 write decoder |
| `scr_rf_wmux4.mag` | `scratch`, 2 × `new_mux4` | Scratchpad register file for intermediate butterfly results |
| `out_rf.mag` | 2 × `decoder2`, 8 × `reg_six` | Output register file, two simultaneous read ports (re + im) |
| `datapath_arrayed.mag` | `datapath_onefin` | Bit-sliced arrayed butterfly datapath |
| `datapath_onefin.mag` | 2 × `addsub_onefin`, 2 × `latch_onefinal` | One bit slice: two add/sub units + output latches |
| `addsub_onefin.mag` | `addsub_onev3`, `xorgate` | Full adder with XOR-controlled subtract (2's complement) |
| `fsm.mag`, `statereg_next.mag` | `statereg`, `s0_next`, `s1_next`, `s9_next`, `nextstate`, `ready`, `load`, `iter_*` | Control FSM: sequences load → butterfly iterations → done |
| `statereg.mag` | `masterslave` | Two-phase state register |
| `masterslave.mag` | `tg`, `tg_state`, `new_inv` | Transmission-gate master–slave flip-flop |
| `pad.mag` | — | Bidirectional I/O pad with output enable |

### Standard cell library

Primitives:
`inv` / `new_inv` / `fat_inv` / `tri_inv`, `nand2` / `nand3` / `nand4`,
`nor2` / `nor3` / `nor4`, `and2` / `and3`, `or2`, `xor2` / `xorgate`,
`tg` (transmission gate), `mux2` / `mux4` / `new_mux4`, `reg1` / `reg2` / `reg_one` / `reg_six`,
`latch_one*`, `decoder` / `decoder2`.

---

## Pinout

| # | Pin | Dir | Description |
|---:|---|:---:|---|
| 1 | `Reset` | in | Asynchronous reset |
| 2–3 | `phi0`, `phi1` | in | Two-phase non-overlapping clock |
| 4–5 | `GND` | — | Ground (×2) |
| 6–7 | `Vdd` | — | Supply (×2) |
| 8 | `START` | in | Begin computation once inputs are loaded |
| 9 | `ACK` | in | Acknowledge readout complete |
| 10–12 | `WA0`–`WA2` | in | Write address |
| 13–18 | `xin0`–`xin5` | in | 6-bit sample input |
| 19 | `REN` | in | Read enable |
| 20–21 | `RA0`, `RA1` | in | Read address |
| 22–27 | `outim_0`–`outim_5` | out | 6-bit imaginary result |
| 28–33 | `outre_0`–`outre_5` | out | 6-bit real result |

`DONE` is asserted by the controller when the coefficients are ready.

---

## Using the chip

### Writing inputs

Load each 6-bit sample one at a time by driving the write address with `WEN` high:

| WEN | WA0 | WA1 | WA2 | Loads |
|:---:|:---:|:---:|:---:|:---|
| 1 | 1 | 1 | 1 | x3,im |
| 1 | 1 | 1 | 0 | x1,im |
| 1 | 1 | 0 | 1 | x3,re |
| 1 | 1 | 0 | 0 | x1,re |
| 1 | 0 | 1 | 1 | x2,im |
| 1 | 0 | 1 | 0 | x0,im |
| 1 | 0 | 0 | 1 | x2,re |
| 1 | 0 | 0 | 0 | x0,re |

Then set `START` high and begin the clock. The FSM runs both butterfly stages and raises `DONE`.

### Reading results

With `DONE` high, drive `REN` high and select a coefficient; `outre[5:0]` and `outim[5:0]`
present the real and imaginary halves of that coefficient simultaneously. Assert `ACK` when
finished.

| REN | RA0 | RA1 | Reads |
|:---:|:---:|:---:|:---|
| 1 | 0 | 0 | F(0) |
| 1 | 0 | 1 | F(1) |
| 1 | 1 | 0 | F(2) |
| 1 | 1 | 1 | F(3) |

---

## Building and simulating

Requires **Magic** (with the `tapeout_SCN6M_DEEP.09.tech27` technology file in this directory)
and **IRSIM**.

In Magic, open the top level and extract:

```bash
magic ring.mag
```

```
extract all
ext2sim alias on
ext2sim
```

Then run the switch-level simulation:

```bash
irsim g180 ring.sim ring.al
```

And in IRSIM, load the top-level test vectors:

```
@ ring.irsim
```

`g180.prm` supplies the 0.18 µm (λ = 0.09 µm) device parameters.

### Test benches

| Script | Device under test |
|---|---|
| `ring.irsim` | Full chip — five input sets (regular, random, all-zeros, alternating) written, computed, and read back |
| `fsmnew.irsim` | Control FSM, including edge cases: mid-phase input changes, signals asserted while not `READY`, back-to-back iterations |
| `regfile.irsim` | Register file writes/reads |
| `setuprf.irsim` | Register file bring-up |
| `decoder.irsim` | 3→8 write decoder, all 16 address/enable combinations |
| `mux2.irsim`, `mux4.irsim` | Multiplexer select coverage |
| `shift_one.irsim` | Shift cell across all select/up-down combinations |
| `test_pad.irsim` | Bidirectional pad in both directions, plus high-Z when `oe` is low |

---

## Repository layout

- `*.mag` — Magic layout cells (the actual design)
- `*.ext` — extracted netlists from Magic
- `*.sim`, `*.al` — IRSIM netlists and alias files produced by `ext2sim`
- `*.irsim` — IRSIM test benches
- `tapeout_SCN6M_DEEP.09.tech27` — process technology file
- `g180.prm` — IRSIM 0.18 µm parameter file
- `VLSI Final Proj.pdf` — project report
- `README.txt` — original submission notes

Generated files (`.ext`, `.sim`, `.al`) are checked in so simulations can be run without
re-extracting, but they are reproducible from the `.mag` sources.

The report (`VLSI Final Proj.pdf`) contains the full derivation, the butterfly expressions,
and the pin assignment reproduced above.
