# DDR4 Controller Implementation Plan

This file is the **authoritative implementation checklist** for the commercial server-grade DDR4 controller
described in `docs/ARCHITECTURE.md`. Agents implementing modules must check off items here and produce
the required artifacts (RTL, testbench, documentation, and results JSON).

---

## Phase 1 — Core Data Path

These modules implement the fundamental data movement and address decode. They have no dependencies on
other controller modules and can be implemented first.

### RTL Modules

- [ ] `ddr4_ecc_engine`: RTL implementation (encode + decode paths, SECDED (72,64))
- [ ] `ddr4_addr_mapper`: RTL implementation (configurable address field extraction)
- [ ] `ddr4_wr_buf`: RTL implementation (write data FIFO, 512-bit wide, 16-entry deep)
- [ ] `ddr4_rd_buf`: RTL implementation (read data return buffer with burst re-alignment)

### Testbenches

- [ ] `ddr4_ecc_engine`: Testbench (encode/decode round-trip, single-bit inject+correct, double-bit detect)
- [ ] `ddr4_addr_mapper`: Testbench (address decode for all rank/bg/ba/row/col combinations)
- [ ] `ddr4_wr_buf`: Testbench (FIFO full/empty, write/read ordering)
- [ ] `ddr4_rd_buf`: Testbench (burst re-alignment, FIFO backpressure)

### Documentation

- [ ] `ddr4_ecc_engine`: Module documentation (`docs/ddr4_ecc_engine.md`)
- [ ] `ddr4_addr_mapper`: Module documentation (`docs/ddr4_addr_mapper.md`)
- [ ] `ddr4_wr_buf`: Module documentation (`docs/ddr4_wr_buf.md`)
- [ ] `ddr4_rd_buf`: Module documentation (`docs/ddr4_rd_buf.md`)

### Verification / Coverage

- [ ] `ddr4_ecc_engine`: Coverage analysis (all syndrome values, single/double error paths)
- [ ] `ddr4_addr_mapper`: Coverage analysis (all address field configurations)
- [ ] `ddr4_wr_buf`: Coverage analysis (FIFO full/empty corner cases)
- [ ] `ddr4_rd_buf`: Coverage analysis (burst lengths, backpressure)

---

## Phase 2 — Timing and Refresh Infrastructure

Timing engine and refresh controller are needed before the command scheduler can enforce JEDEC timing.

### RTL Modules

- [ ] `ddr4_timing_engine`: RTL implementation (per-bank countdown timers, APB-programmable parameters)
- [ ] `ddr4_refresh_ctrl`: RTL implementation (tREFI counter, REFab/REFpb generation, deferred refresh logic)

### Testbenches

- [ ] `ddr4_timing_engine`: Testbench (timer load/expiry for all parameters, APB read-back)
- [ ] `ddr4_refresh_ctrl`: Testbench (REFab interval, deferred count up to 8x, REFpb bank rotation, high-temp mode)

### Documentation

- [ ] `ddr4_timing_engine`: Module documentation (`docs/ddr4_timing_engine.md`)
- [ ] `ddr4_refresh_ctrl`: Module documentation (`docs/ddr4_refresh_ctrl.md`)

### Verification / Coverage

- [ ] `ddr4_timing_engine`: Coverage analysis (tCL, tRCD, tRP, tRAS, tRFC1, tRFC2 expiry sequences)
- [ ] `ddr4_refresh_ctrl`: Coverage analysis (normal/deferred/high-temp refresh modes)

---

## Phase 3 — Initialization and Mode Register Control

Depends on Phase 2 (timing engine). These modules bring the DRAM out of reset and program MRS registers.

### RTL Modules

- [ ] `ddr4_init_fsm`: RTL implementation (RESET->CKE_LOW->MRS sequence->ZQCL->READY per JESD79-4 section 3.3)
- [ ] `ddr4_mr_ctrl`: RTL implementation (MR0-MR6 shadow registers + MRS command sequencer)

### Testbenches

- [ ] `ddr4_init_fsm`: Testbench (full power-on sequence, tPWstable and tXPR wait, MRS order verification)
- [ ] `ddr4_mr_ctrl`: Testbench (MRS write to each MR, APB shadow register read-back, re-program during operation)

### Documentation

- [ ] `ddr4_init_fsm`: Module documentation (`docs/ddr4_init_fsm.md`)
- [ ] `ddr4_mr_ctrl`: Module documentation (`docs/ddr4_mr_ctrl.md`)

### Verification / Coverage

- [ ] `ddr4_init_fsm`: Coverage analysis (all FSM states, timer expiry transitions)
- [ ] `ddr4_mr_ctrl`: Coverage analysis (all 7 mode registers, re-program path)

---

## Phase 4 — Command Scheduler and PHY Interface

Depends on Phases 1-3. Core command dispatch path.

### RTL Modules

- [ ] `ddr4_cmd_scheduler`: RTL implementation (open-page policy, bank state machines, timing gate, reorder buffer)
- [ ] `ddr4_phy_intf`: RTL implementation (DFI v4.0 command/data bus translation, DDR4 CA encoding)

### Testbenches

- [ ] `ddr4_cmd_scheduler`: Testbench (page hit/miss/empty sequences, refresh insertion, tCCD/tRRD/tWTR enforcement)
- [ ] `ddr4_phy_intf`: Testbench (ACTIVATE/READ/WRITE/PRECHARGE/REFRESH command encoding, DQS timing)

### Documentation

- [ ] `ddr4_cmd_scheduler`: Module documentation (`docs/ddr4_cmd_scheduler.md`)
- [ ] `ddr4_phy_intf`: Module documentation (`docs/ddr4_phy_intf.md`)

### Verification / Coverage

- [ ] `ddr4_cmd_scheduler`: Coverage analysis (all bank states, refresh defer up to 8x, write/read interleave)
- [ ] `ddr4_phy_intf`: Coverage analysis (all DDR4 commands, ODT assertion timing)

---

## Phase 5 — AXI4 Frontend and Top-Level Integration

Depends on all previous phases. Integrates the full design under the AXI4 slave interface.

### RTL Modules

- [ ] `ddr4_axi4_frontend`: RTL implementation (AXI4 slave, burst splitting, ID tracking, write/read channel management)
- [ ] `ddr4_ctrl_top`: RTL implementation (top-level integration of all sub-modules + APB register file)

### Testbenches

- [ ] `ddr4_axi4_frontend`: Testbench (AXI4 write/read burst, out-of-order IDs, backpressure)
- [ ] `ddr4_ctrl_top`: Testbench (end-to-end write+read with ECC, refresh during sustained traffic, APB timing register reprogramming)

### Documentation

- [ ] `ddr4_axi4_frontend`: Module documentation (`docs/ddr4_axi4_frontend.md`)
- [ ] `ddr4_ctrl_top`: Module documentation (`docs/ddr4_ctrl_top.md`)

### Verification / Coverage

- [ ] `ddr4_axi4_frontend`: Coverage analysis (AWBURST/ARID/AWID mixes, WSTRB patterns)
- [ ] `ddr4_ctrl_top`: System-level coverage (ECC SLVERR path, init-to-traffic latency, concurrent refresh and access)

---

## Dependency Graph

```
Phase 1 (ECC, Addr, Buffers)
       |
       v
Phase 2 (Timing, Refresh)
       |
       v
Phase 3 (Init FSM, MR Ctrl)
       |
       v
Phase 4 (Cmd Scheduler, PHY Intf)
       |
       v
Phase 5 (AXI4 Frontend, Top)
```

---

## JSON Traceability

For every completed module, create or update `results/phase-<phase_name>/<module>_result.json` with:

```json
{
  "module": "<module_name>",
  "rtl_done": true,
  "tb_done": true,
  "doc_done": true,
  "simulation_passed": true,
  "coverage_completed": true,
  "coverage_percentage": 95,
  "plan_item_completed": true,
  "error_summary": "",
  "sim_log": "results/phase-<phase_name>/<module>_sim.log"
}
```

Phase directory names: `phase-1-core-datapath`, `phase-2-timing-refresh`,
`phase-3-init-mr`, `phase-4-scheduler-phy`, `phase-5-axi4-top`.
