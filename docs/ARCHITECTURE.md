# DDR4 Controller Architecture

**Standard:** JEDEC JESD79-4 (DDR4 SDRAM)
**Revision:** 1.0

---

## 1. Project Overview

This project implements a **commercial server-grade DDR4 SDRAM memory controller** targeting the JEDEC JESD79-4 specification. The design supports:

- **Data widths:** 64-bit data bus + 8 ECC bits (72-bit DRAM interface using x8 devices, 1 rank x 8 DRAMs + 1 ECC device)
- **Speeds:** DDR4-1600 through DDR4-3200 (800 MHz to 1600 MHz data rate per pin; 400 MHz to 800 MHz controller clock)
- **ECC:** SECDED (Single-Error Correct, Double-Error Detect) over a 64-bit data word
- **Refresh:** Automatic refresh with tREFI/tRFC1/tRFC2 support; per-bank refresh (REFpb) optional
- **Timing:** All critical DDR4 timing parameters are register-programmable at runtime
- **Host interface:** AXI4 (64-bit address, 512-bit data burst) slave port for integration into server SoCs

The controller is structured as a **synthesizable RTL design** expressed in IEEE 1364-2001 / SystemVerilog-compatible Verilog (`.v` files under `rtl/`). Each sub-module is independently testable with a corresponding testbench in `tb/`.

---

## 2. Functional Blocks / Modules

| Module | File | Description |
|--------|------|-------------|
| `ddr4_ctrl_top` | `rtl/ddr4_ctrl_top.v` | Top-level wrapper; instantiates all sub-modules and connects them |
| `ddr4_axi4_frontend` | `rtl/ddr4_axi4_frontend.v` | AXI4 slave; accepts read/write transactions and converts them to internal command structs |
| `ddr4_addr_mapper` | `rtl/ddr4_addr_mapper.v` | Maps byte-addresses to {rank, bank-group, bank, row, column}; configurable interleave policy |
| `ddr4_cmd_scheduler` | `rtl/ddr4_cmd_scheduler.v` | Command scheduler/arbiter; reorders requests for bank-hit maximisation; enforces all timing constraints |
| `ddr4_timing_engine` | `rtl/ddr4_timing_engine.v` | Centralised programmable timing parameter store and per-bank countdown timers |
| `ddr4_refresh_ctrl` | `rtl/ddr4_refresh_ctrl.v` | Generates REF commands at tREFI intervals; tracks tRFC budget; handles deferred refresh (max 8x postponement) |
| `ddr4_ecc_engine` | `rtl/ddr4_ecc_engine.v` | SECDED encoder (write path) and decoder/corrector (read path); syndrome generation and bit-flip correction |
| `ddr4_mr_ctrl` | `rtl/ddr4_mr_ctrl.v` | Mode Register Set (MRS) sequencer; programs MR0-MR6 during init and on-the-fly DLL reset / ZQ calibration |
| `ddr4_init_fsm` | `rtl/ddr4_init_fsm.v` | Power-on reset and initialization FSM (RESET->CKE-LOW->MRS->ZQ-CAL->READY) per JESD79-4 section 3.3 |
| `ddr4_phy_intf` | `rtl/ddr4_phy_intf.v` | PHY / DFI v4.0 interface translation layer; drives CK, CKE, CS_n, ACT_n, RAS_n/CAS_n/WE_n (CA bus), address, DQ, DQS, DM/DBI, ODT |
| `ddr4_wr_buf` | `rtl/ddr4_wr_buf.v` | Write data buffer (FIFO); holds write data + byte enables until WRITE command is issued |
| `ddr4_rd_buf` | `rtl/ddr4_rd_buf.v` | Read data return buffer; re-aligns burst data from PHY; delivers to AXI4 read-data channel after ECC correction |

---

## 3. Interfaces

### 3.1 Top-Level Ports (`ddr4_ctrl_top`)

#### Clock and Reset

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `clk` | I | 1 | Controller clock (= CK frequency; e.g. 800 MHz for DDR4-3200) |
| `rst_n` | I | 1 | Active-low asynchronous reset |
| `pll_lock` | I | 1 | PLL/DLL lock indicator; init FSM waits until asserted |

#### AXI4 Slave (host to controller)

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `s_axi_awid` | I | 8 | Write address ID |
| `s_axi_awaddr` | I | 64 | Write address (byte-addressed) |
| `s_axi_awlen` | I | 8 | Burst length (AXI4 encoding; 0 = 1 beat) |
| `s_axi_awsize` | I | 3 | Beat size |
| `s_axi_awburst` | I | 2 | Burst type (INCR only supported) |
| `s_axi_awvalid` | I | 1 | Write-address channel valid |
| `s_axi_awready` | O | 1 | Write-address channel ready |
| `s_axi_wdata` | I | 512 | Write data (one AXI beat = one DDR4 BL8 burst) |
| `s_axi_wstrb` | I | 64 | Write byte strobes |
| `s_axi_wlast` | I | 1 | Last beat of write burst |
| `s_axi_wvalid` | I | 1 | Write-data channel valid |
| `s_axi_wready` | O | 1 | Write-data channel ready |
| `s_axi_bid` | O | 8 | Write response ID |
| `s_axi_bresp` | O | 2 | Write response (OKAY / SLVERR on uncorrectable ECC) |
| `s_axi_bvalid` | O | 1 | Write response valid |
| `s_axi_bready` | I | 1 | Write response ready |
| `s_axi_arid` | I | 8 | Read address ID |
| `s_axi_araddr` | I | 64 | Read address (byte-addressed) |
| `s_axi_arlen` | I | 8 | Read burst length |
| `s_axi_arsize` | I | 3 | Beat size |
| `s_axi_arburst` | I | 2 | Burst type (INCR only) |
| `s_axi_arvalid` | I | 1 | Read-address channel valid |
| `s_axi_arready` | O | 1 | Read-address channel ready |
| `s_axi_rid` | O | 8 | Read data ID |
| `s_axi_rdata` | O | 512 | Read data |
| `s_axi_rresp` | O | 2 | Read response (OKAY / SLVERR on uncorrectable ECC) |
| `s_axi_rlast` | O | 1 | Last beat indicator |
| `s_axi_rvalid` | O | 1 | Read-data valid |
| `s_axi_rready` | I | 1 | Read-data ready |

#### APB Register Interface (configuration)

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `apb_paddr` | I | 12 | APB address (register select) |
| `apb_pwrite` | I | 1 | APB write enable |
| `apb_pwdata` | I | 32 | APB write data |
| `apb_psel` | I | 1 | APB select |
| `apb_penable` | I | 1 | APB enable |
| `apb_prdata` | O | 32 | APB read data |
| `apb_pready` | O | 1 | APB ready |
| `apb_pslverr` | O | 1 | APB slave error |

#### DDR4 DRAM Interface (controller to PHY / DRAM)

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `ddr4_ck_t` | O | 1 | Differential clock - true |
| `ddr4_ck_c` | O | 1 | Differential clock - complement |
| `ddr4_cke` | O | RANKS | Clock enable (one per rank) |
| `ddr4_cs_n` | O | RANKS | Chip select (one per rank) |
| `ddr4_act_n` | O | 1 | Activate command strobe |
| `ddr4_ras_n` | O | 1 | RAS_n / A16 |
| `ddr4_cas_n` | O | 1 | CAS_n / A15 |
| `ddr4_we_n` | O | 1 | WE_n / A14 |
| `ddr4_bg` | O | 2 | Bank group address |
| `ddr4_ba` | O | 2 | Bank address |
| `ddr4_a` | O | 18 | Row/column address (multiplexed) |
| `ddr4_odt` | O | RANKS | On-Die Termination control |
| `ddr4_reset_n` | O | 1 | DRAM reset (active-low) |
| `ddr4_dq` | IO | 72 | Data bus (64 data + 8 ECC check bits) |
| `ddr4_dqs_t` | IO | 9 | Data strobe - true (one per byte lane including ECC) |
| `ddr4_dqs_c` | IO | 9 | Data strobe - complement |
| `ddr4_dm_n` | IO | 9 | Data mask / DBI_n (one per byte lane) |

#### Status / Interrupt

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `ecc_single_err` | O | 1 | Single-bit ECC error detected and corrected (sticky until cleared) |
| `ecc_double_err` | O | 1 | Double-bit ECC error detected; read data is unreliable |
| `ecc_err_addr` | O | 64 | Address of last ECC error |
| `init_done` | O | 1 | DRAM initialization complete |
| `irq` | O | 1 | Interrupt (ORed ECC double-error + scrub complete) |

---

### 3.2 Internal Module Interfaces

#### AXI4 Frontend to Command Scheduler (cmd_req_t)

```
// Internal command request (unpacked as separate wires in RTL)
// id       [7:0]   - AXI transaction ID (pass-through)
// addr     [63:0]  - byte address (pre-mapping)
// cmd      [0]     - 0=READ, 1=WRITE
// wdata    [511:0] - write data payload
// wstrb    [63:0]  - byte enable mask
```

#### Address Mapper to Command Scheduler (mapped_addr_t)

```
// rank  [1:0]  - rank select
// bg    [1:0]  - bank group
// ba    [1:0]  - bank address
// row   [17:0] - row address
// col   [9:0]  - column address
```

#### ECC Engine - Write Path

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `ecc_enc_din` | I | 64 | Raw data from write buffer |
| `ecc_enc_dout` | O | 72 | Data + 8 Hamming check bits to PHY |

#### ECC Engine - Read Path

| Signal | Dir | Width | Description |
|--------|-----|-------|-------------|
| `ecc_dec_din` | I | 72 | Data + check bits from PHY |
| `ecc_dec_dout` | O | 64 | Corrected data to read buffer |
| `ecc_syndrome` | O | 8 | Syndrome word (0 = no error) |
| `ecc_single_err` | O | 1 | Single-bit error (corrected) |
| `ecc_double_err` | O | 1 | Double-bit error (uncorrectable) |

---

## 4. Timing / Protocols / Constraints

### 4.1 Programmable Timing Parameters

All timing parameters are expressed in **controller clock cycles** and are programmable via the APB register interface. Default reset values correspond to DDR4-3200 (1600 MHz data rate, CL22) using a 800 MHz controller clock (t_ck = 1.25 ns).

| Parameter | APB Offset | Default cycles | Time (ns) | Description |
|-----------|-----------|---------------|-----------|-------------|
| `tCL` | `0x014` | 22 | 27.5 | CAS Latency - READ command to first valid data |
| `tCWL` | `0x018` | 18 | 22.5 | CAS Write Latency |
| `tRCD` | `0x01C` | 16 | 20.0 | ACTIVATE to READ/WRITE command delay |
| `tRP` | `0x020` | 16 | 20.0 | PRECHARGE to ACTIVATE minimum delay |
| `tRAS` | `0x024` | 39 | 48.8 | ACTIVATE to PRECHARGE minimum |
| `tRC` | `0x028` | 55 | 68.8 | ACTIVATE to ACTIVATE (same bank); tRC = tRAS + tRP |
| `tWR` | `0x02C` | 24 | 30.0 | Write recovery time (last write to PRECHARGE) |
| `tRTP` | `0x030` | 12 | 15.0 | Internal read to precharge command delay |
| `tFAW` | `0x034` | 26 | 32.5 | Four-Activate Window (same rank) |
| `tRRD_S` | `0x038` | 4 | 5.0 | ACTIVATE to ACTIVATE - different bank group |
| `tRRD_L` | `0x03C` | 6 | 7.5 | ACTIVATE to ACTIVATE - same bank group |
| `tCCD_S` | `0x040` | 4 | 5.0 | CAS to CAS - different bank group |
| `tCCD_L` | `0x044` | 8 | 10.0 | CAS to CAS - same bank group |
| `tWTR_S` | `0x048` | 4 | 5.0 | Write to Read - different bank group |
| `tWTR_L` | `0x04C` | 12 | 15.0 | Write to Read - same bank group |
| `tRTW` | `0x050` | 4 | 5.0 | Read to Write turnaround bubble |
| `tREFI` | `0x054` | 6240 | 7800 | Average periodic refresh interval (normal temp) |
| `tRFC1` | `0x058` | 296 | 370 | Refresh cycle time - REFab (8 Gb devices) |
| `tRFC2` | `0x05C` | 208 | 260 | Refresh cycle time - REFpb (8 Gb devices) |
| `tXPR` | `0x060` | 300 | 375 | Exit reset to first valid command after power-up |
| `tMOD` | `0x064` | 24 | 30.0 | MRS command to any non-MRS command |
| `tZQinit` | `0x068` | 1024 | 1280 | ZQ calibration - initial (ZQCL after power-up) |

> **Conversion formula:** `cycles = ceil(time_ns / t_ck_ns)` where `t_ck_ns = 1000 / f_mhz`.

### 4.2 DDR4 Command Encoding

DDR4 uses ACT_n and an 18-bit command/address bus to encode all commands (JESD79-4 Table 3):

| Command | ACT_n | RAS_n/A16 | CAS_n/A15 | WE_n/A14 | Notes |
|---------|-------|-----------|-----------|----------|-------|
| ACTIVATE | 0 | BG1 (addr) | BG0 (addr) | BA1 (addr) | ACT_n=0 distinguishes from all other commands; BG[1:0] on {A16,A15}, BA[1:0] on {A14,A12}; row addr on A[17,13,11:0] |
| WRITE | 1 | 1 | 0 | 0 | A[1:0]=BL, A[10]=AP, A[12]=BC, A[9:0]=col |
| READ | 1 | 1 | 0 | 1 | A[1:0]=BL, A[10]=AP, A[9:0]=col |
| PRECHARGE | 1 | 0 | 1 | 0 | A[10]=1 for all-bank precharge (PREAB) |
| REFRESH | 1 | 0 | 1 | 1 | A[10]=0: REFab; A[10]=1, A[3:0]=bank: REFpb |
| MRS | 1 | 0 | 0 | 0 | BG/BA select MR number; A[13:0] = mode data |
| NOP | 1 | 1 | 1 | 1 | |
| DESELECT | CS_n=1 | - | - | - | Equivalent to NOP when CKE high |

### 4.3 Burst Length

DDR4 supports **BL8** (Burst Length 8) as the baseline. One BL8 transfer delivers 8 beats x 72 bits = 576 bits total (64 B data + 8 B ECC check bytes) per rank access. OTF BL4 is not implemented in this design.

### 4.4 Refresh Policy

- **All-bank refresh (REFab):** the `ddr4_refresh_ctrl` module issues a REFab command every `tREFI` controller clock cycles (default 6240 cc at 800 MHz = 7.8 us).
- **Deferred refresh:** up to 8 consecutive REF commands may be deferred (postponed) by the scheduler while a latency-sensitive burst completes. Once the deferred count reaches 7, the next opportunity issues the accumulated REF commands before any new transactions.
- **Per-bank refresh (REFpb):** enabled via `REFRESH_CFG[0]`; rotates through 4 banks per refresh cycle, reducing the tRFC dead-time to tRFC2 per bank.
- **Extended temperature:** `REFRESH_CFG[8]` halves tREFI to 3120 cc (3.9 us) for operation above 85 C.
- During tRFC, no commands (other than NOP/DESELECT) may be issued to the refreshed rank.

### 4.5 ECC - (72,64) SECDED

The ECC engine implements the standard **(72, 64) SECDED** (Hamming-based) code:

- **72 bits total:** 64 data bits + 7 Hamming parity bits (P1, P2, P4, P8, P16, P32, P64) + 1 overall parity bit P0.
- **Encode (write):** the check bits are computed as XOR trees over specific subsets of the 64 data bits per the standard parity check matrix H.
- **Decode (read):** syndrome S[6:0] = XOR of received codeword through H. Overall parity bit P = XOR of all 72 bits.
  - `S == 0, P == 0` -> no error.
  - `S != 0, P == 1` -> single-bit error at bit position S; corrected by inverting that bit.
  - `S != 0, P == 0` -> double-bit error (uncorrectable); assert `ecc_double_err`.
  - `S == 0, P == 1` -> single-bit error in P0 itself (benign; corrected transparently).
- **Reporting:** `ecc_single_err` and `ecc_double_err` are sticky status flags cleared by writing `ECC_ERR_CLR`. The offending address is latched in `ECC_ERR_ADDR_HI/LO`. The `irq` output is raised on any uncorrectable error.

### 4.6 Address Mapping

Default bit-field interleave (optimised for sequential page-mode access):

```
Byte address [63:0]:
  [63:35] -> unused (supports up to 64 GB per 4-rank channel)
  [34:33] -> rank[1:0]
  [32:17] -> row[15:0]    (64K rows per bank; matches 8 Gb x8 DDR4 device row count)
  [16:15] -> bg[1:0]      (4 bank groups)
  [14:13] -> ba[1:0]      (4 banks per group = 16 banks total)
  [12: 3] -> col[9:0]     (1K columns x 8 bytes = 8 KB per row)
  [ 2: 0] -> byte offset (within 64-bit word)
```

> **Note:** The internal `mapped_addr_t` row field is `[17:0]` to match the full DDR4 18-bit address
> bus width (A[17:0]); bits [17:16] are zero for 8 Gb devices and non-zero only for higher-density
> parts with wider row addressing.

The start-bit position of each field is configurable via APB registers `0x070-0x07C` to support alternative interleave schemes (e.g. channel-first or cache-line interleave).

---

## 5. Block Diagram

```
Host SoC
   |
   |  AXI4 Slave (64-bit addr, 512-bit data, ID=8-bit)
   v
+----------------------------------------------------------------------+
|                        ddr4_ctrl_top                                  |
|                                                                       |
|  +-------------------+      +-----------------+                      |
|  | ddr4_axi4_frontend|      |  ddr4_init_fsm  |---> init_done        |
|  |   (AXI4 slave)    |      |  (power-on seq) |                      |
|  +--------+----------+      +--------+--------+                      |
|  cmd_req_t|                          | MRS / ZQ cmds                 |
|           v                          v                               |
|  +-------------------+      +--------------------+                   |
|  |  ddr4_addr_mapper  |      |   ddr4_mr_ctrl     |                   |
|  |  (row/col decode) |      |  (MRS sequencer)   |                   |
|  +--------+----------+      +--------+-----------+                   |
|  mapped   |                 MRS cmd  |                               |
|  addr_t   v                          v                               |
|  +----------------------------------------------------------+        |
|  |                  ddr4_cmd_scheduler                        |        |
|  |   open-page policy | timing gate | reorder buffer         |        |
|  +----+------------------------------------+-----------------+        |
|       | timing queries                     | issued cmd               |
|       v                                    v                         |
|  +--------------------+        +-----------------------+             |
|  | ddr4_timing_engine  |        |    ddr4_phy_intf       |             |
|  | (per-bank timers)  |        |  (DFI v4.0 / DRAM     |             |
|  +--------------------+        |   signal generation)  |             |
|                                +----------+------------+             |
|  +------------------+  +----------+       |                          |
|  | ddr4_refresh_ctrl |  |ddr4_wr_buf|      |  72-bit DDR4 bus         |
|  |  (REFab/REFpb)   |  |(wr FIFO) |      |  (CK, CKE, CS_n,         |
|  +--------+---------+  +----+-----+      |   ACT_n, RAS_n,          |
|  REF cmd  |          wdata  |            |   CAS_n, WE_n,           |
|           v                 v            |   BG, BA, A,             |
|  +--------------------------------------+|   DQ, DQS, DM,           |
|  |           ddr4_ecc_engine             |   ODT, RESET_n)          |
|  |  encode (wr) <---- 64b --------       |                          |
|  |  decode (rd) ----> 64b -------->      |                          |
|  +--------------------------------------+                            |
|                                                                       |
|  +------------------+                                                |
|  |   ddr4_rd_buf    |<-- corrected data <--(from PHY via ECC)        |
|  | (rd return buf)  |--> AXI4 RDATA                                  |
|  +------------------+                                                |
|                                                                       |
|  APB slave --> CTRL / TIMING / ECC / REFRESH / ADDR_MAP registers    |
+----------------------------------------------------------------------+
          |
          v
     DDR4 DIMM(s)  [x8 DRAM x 9 devices = 72-bit ECC DIMM]
```

---

## 6. Initialization FSM (`ddr4_init_fsm`)

Follows JESD79-4 section 3.3 power-on and initialization sequence:

```
S_RESET_ASSERT
  |  RESET_n=0, CKE=0
  |  Wait >= 200 us (tPWstable)
  v
S_CKE_LOW
  |  RESET_n=1, CKE=0
  |  Wait >= 500 us from CKE de-assert (or >= tXPR after power)
  v
S_MRS_MR3  -> issue MRS to MR3
  v
S_MRS_MR6  -> issue MRS to MR6
  v
S_MRS_MR5  -> issue MRS to MR5
  v
S_MRS_MR4  -> issue MRS to MR4
  v
S_MRS_MR2  -> issue MRS to MR2
  v
S_MRS_MR1  -> issue MRS to MR1
  v
S_MRS_MR0  -> issue MRS to MR0
  |  wait tMOD after last MRS
  v
S_ZQCL
  |  issue ZQCL; wait tZQinit (1024 ck)
  v
S_READY
  |  assert init_done; pass control to ddr4_cmd_scheduler
```

All wait timers use `ddr4_timing_engine` countdown registers. The FSM asserts `ddr4_reset_n` as its first action upon `rst_n` de-assertion.

---

## 7. Command Scheduler Policy (`ddr4_cmd_scheduler`)

Implements **open-page (page-hit-first)** arbitration:

1. **Bank state tracking:** each of the 16 banks (4 BG x 4 BA) has an independent state machine with states `IDLE`, `ACTIVATING`, `ACTIVE`, `PRECHARGING`, `REFRESHING`.
2. **Request classification:**
   - *Page hit* - open row matches request row -> issue READ or WRITE immediately (subject to tCCD/tWTR).
   - *Page empty* - bank is idle -> issue ACTIVATE, then READ/WRITE after tRCD.
   - *Page miss* - open row differs -> issue PRECHARGE, wait tRP, then ACTIVATE, wait tRCD, then READ/WRITE.
3. **Refresh priority:** when `ddr4_refresh_ctrl` deferred count >= 7, the scheduler inserts REF before the next transaction.
4. **Timing enforcement:** all inter-command constraints are checked against `ddr4_timing_engine` before a command is issued. Commands are stalled (held in reorder buffer) until all applicable timers expire.
5. **Write-to-read / read-to-write gaps:** tWTR_S/tWTR_L and tRTW bubbles are inserted automatically.

---

## 8. ECC Engine Details (`ddr4_ecc_engine`)

### 8.1 (72, 64) SECDED Parity Check Matrix

The standard Hamming code positions for a 72-bit codeword:

| Bit positions | Content |
|---|---|
| 1-64 | D[63:0] - data bits |
| 65-71 | P1, P2, P4, P8, P16, P32, P64 - Hamming check bits |
| 72 | P0 - overall parity bit |

Parity bit coverage follows powers-of-2 positions in the standard H matrix (Hamming distance 4).

### 8.2 Syndrome Decoding Table

| Syndrome S[6:0] | P_overall | Condition |
|---|---|---|
| 0b000_0000 | 0 | No error |
| 0b000_0000 | 1 | P0 bit itself is in error (data valid) |
| non-zero | 1 | Single-bit error at codeword position = S; correct by flipping |
| non-zero | 0 | Double-bit error; assert `ecc_double_err`; return SLVERR |

### 8.3 Scrub Engine (placeholder)

When `CTRL[2]` (scrub_enable) is asserted, the controller performs background read-modify-write passes over all memory to proactively correct accumulated single-bit errors. Full implementation is reserved for a future phase.

---

## 9. APB Register Map

| Offset | Name | R/W | Reset | Description |
|--------|------|-----|-------|-------------|
| `0x000` | `CTRL` | RW | `0x0` | [0] ctrl_enable, [1] ecc_enable, [2] scrub_enable, [3] refpb_enable |
| `0x004` | `STATUS` | RO | `0x0` | [0] init_done, [1] ecc_single_err_sticky, [2] ecc_double_err_sticky |
| `0x008` | `ECC_ERR_ADDR_LO` | RO | `0x0` | Lower 32 bits of last ECC error address |
| `0x00C` | `ECC_ERR_ADDR_HI` | RO | `0x0` | Upper 32 bits of last ECC error address |
| `0x010` | `ECC_ERR_CLR` | WO | - | Write any value to clear ECC sticky flags |
| `0x014` | `TIMING_tCL` | RW | 22 | CAS Latency in clock cycles |
| `0x018` | `TIMING_tCWL` | RW | 18 | CAS Write Latency |
| `0x01C` | `TIMING_tRCD` | RW | 16 | RAS-to-CAS delay |
| `0x020` | `TIMING_tRP` | RW | 16 | Row precharge |
| `0x024` | `TIMING_tRAS` | RW | 39 | Row active minimum |
| `0x028` | `TIMING_tRC` | RW | 55 | Row cycle (tRAS + tRP) |
| `0x02C` | `TIMING_tWR` | RW | 24 | Write recovery |
| `0x030` | `TIMING_tRTP` | RW | 12 | Read-to-precharge |
| `0x034` | `TIMING_tFAW` | RW | 26 | Four-activate window |
| `0x038` | `TIMING_tRRD_S` | RW | 4 | ACTIVATE-to-ACTIVATE diff BG |
| `0x03C` | `TIMING_tRRD_L` | RW | 6 | ACTIVATE-to-ACTIVATE same BG |
| `0x040` | `TIMING_tCCD_S` | RW | 4 | CAS-to-CAS diff BG |
| `0x044` | `TIMING_tCCD_L` | RW | 8 | CAS-to-CAS same BG |
| `0x048` | `TIMING_tWTR_S` | RW | 4 | Write-to-read diff BG |
| `0x04C` | `TIMING_tWTR_L` | RW | 12 | Write-to-read same BG |
| `0x050` | `TIMING_tRTW` | RW | 4 | Read-to-write turnaround |
| `0x054` | `TIMING_tREFI` | RW | 6240 | Refresh interval (cycles) |
| `0x058` | `TIMING_tRFC1` | RW | 296 | Refresh cycle (REFab) |
| `0x05C` | `TIMING_tRFC2` | RW | 208 | Refresh cycle (REFpb) |
| `0x060` | `TIMING_tXPR` | RW | 300 | Exit-reset to first cmd |
| `0x064` | `TIMING_tMOD` | RW | 24 | MRS settle time |
| `0x068` | `TIMING_tZQinit` | RW | 1024 | ZQ calibration initial |
| `0x070` | `ADDRMAP_RANK` | RW | 33 | Start bit of rank field in byte address |
| `0x074` | `ADDRMAP_ROW` | RW | 17 | Start bit of row field |
| `0x078` | `ADDRMAP_BG` | RW | 15 | Start bit of bank-group field |
| `0x07C` | `ADDRMAP_BA` | RW | 13 | Start bit of bank field |
| `0x080` | `MR0_SHADOW` | RW | DDR4 default | MR0 (burst length, CAS latency, DLL reset) |
| `0x084` | `MR1_SHADOW` | RW | DDR4 default | MR1 (DLL enable, output drive, RTT_NOM, AL) |
| `0x088` | `MR2_SHADOW` | RW | DDR4 default | MR2 (CWL, ASR, RTT_WR) |
| `0x08C` | `MR3_SHADOW` | RW | DDR4 default | MR3 (MPR, Geardown, fine granularity refresh) |
| `0x090` | `MR4_SHADOW` | RW | DDR4 default | MR4 (HS, RREFD, RDPRE, WR preamble, IDS) |
| `0x094` | `MR5_SHADOW` | RW | DDR4 default | MR5 (PL, ODTLoff, RTT_PARK, CA parity) |
| `0x098` | `MR6_SHADOW` | RW | DDR4 default | MR6 (VREF training range/value, tCCD_L) |
| `0x09C` | `REFRESH_CFG` | RW | `0x08` | [0] refpb_en, [7:4] max_defer, [8] hi_temp_en |
| `0x0A0` | `REFRESH_STATUS` | RO | `0x0` | [3:0] deferred_ref_count |

---

## 10. Notes and Assumptions

1. **PHY model:** `ddr4_phy_intf` is a **behavioral simulation model**. In silicon integration, replace with a vendor hardened DDR4 PHY (e.g. Synopsys DWC DDR4 PHY) connected via the DFI v4.0 protocol interface.
2. **Single rank default:** The parameter `RANKS=1` selects single-rank operation. Extend to `RANKS=2` or `RANKS=4` for registered DIMM (RDIMM) multi-rank support (requires CS_n, CKE, ODT per rank).
3. **DRAM geometry:** Default targets **8 Gb x 8 (x8)** DDR4 SDRAMs: 9 devices (8 data + 1 ECC) on a 72-bit ECC UDIMM or RDIMM.
4. **Temperature:** Normal operating range (0 to 85 C) uses tREFI = 7.8 us. Set `REFRESH_CFG[8]=1` for extended temp (85 to 95 C) to halve tREFI.
5. **Scrub engine:** `CTRL[2]` scrub_enable is a placeholder for a background read-modify-write scrubber (future phase).
6. **Burst order:** Only BL8 sequential burst order is supported.
7. **Standard reference:** JEDEC JESD79-4C, *DDR4 SDRAM Standard*. Available at: https://www.jedec.org/standards-documents/docs/jesd79-4a
8. **Module name authority:** All module names in section 2 are authoritative. Downstream Implementation issues **must** use these exact names for RTL files (`rtl/<module>.v`), testbenches (`tb/<module>_tb.v`), and module documentation (`docs/<module>.md`).
