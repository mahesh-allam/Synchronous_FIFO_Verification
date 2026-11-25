# Synchronous FIFO: RTL Design and Directed Verification

A parameterized synchronous FIFO written in SystemVerilog, with a directed testbench simulated in Xilinx Vivado.

> **Status:** work in progress. The RTL is complete. Verification so far is a **directed testbench whose output is checked manually** from the log and waveform. A self-checking, class-based testbench with SVA and functional coverage is planned (see [Roadmap](#roadmap)).

---

## Design

### Parameters

| Parameter | Default | Description |
| :--- | :---: | :--- |
| `FIFO_DEPTH` | `8` | Number of entries (must be a power of 2) |
| `DATA_WIDTH` | `32` | Width of each entry in bits |

### Ports (module `fifo_sync`)

| Signal | Direction | Width | Description |
| :--- | :--- | :--- | :--- |
| `clk` | Input | 1 | Clock |
| `rst_n` | Input | 1 | **Asynchronous**, active-low reset |
| `cs` | Input | 1 | Chip select: reads and writes happen only when `cs = 1` |
| `wr_en` | Input | 1 | Write enable |
| `rd_en` | Input | 1 | Read enable |
| `data_in` | Input | `DATA_WIDTH` | Write data |
| `data_out` | Output | `DATA_WIDTH` | Read data (registered) |
| `full` | Output | 1 | High when the FIFO holds `FIFO_DEPTH` entries |
| `empty` | Output | 1 | High when the FIFO holds no entries |

### Behaviour

- **Write:** accepted on a rising clock edge when `cs && wr_en && !full`. A write while full is ignored.
- **Read:** accepted on a rising clock edge when `cs && rd_en && !empty`. A read while empty is ignored.
- **Read latency:** `data_out` is registered, so the value appears **one clock cycle after** the read is accepted. It holds its value until the next accepted read.
- **Simultaneous read and write:** both are accepted in the same cycle if each is individually allowed.
- **Reset:** `rst_n = 0` clears both pointers immediately (asynchronous), which makes the FIFO empty. Memory contents and `data_out` are not cleared.

### Full/empty detection

The read and write pointers are each `$clog2(FIFO_DEPTH) + 1` bits wide. The lower bits address the memory, and the extra MSB records wrap-around:

- `empty`: both pointers are exactly equal.
- `full`: the lower bits are equal and the MSBs differ, which means the write pointer has wrapped once more than the read pointer.

This avoids a separate counter and tells full and empty apart without wasting an entry.

---

## Verification

### Testbench (`tb/tb_fifo_sync.sv`)

A directed testbench with two tasks, `write_data()` and `read_data()`. Each one drives a single operation over two clock cycles and prints `data_in`/`data_out`, `full` and `empty` to the log.

### Scenarios

| # | Scenario | Stimulus | Observed result |
| :---: | :--- | :--- | :--- |
| 1 | Basic write/read | Write 111, 999; read twice | Read back 111 then 999; `empty` asserts after the last read |
| 2 | Alternating write/read | 8× (write 2^i, then read) | Each read returns the value just written; FIFO returns to empty every iteration |
| 3 | Fill and overflow attempt | Write 9 values (1 … 256) into the 8-deep FIFO, then read 8 | `full` asserts after the 8th write; the 9th write (256) is ignored; reads return 1 … 128 in order, then `empty` asserts |

Results were checked by reading the simulation log and waveform. The testbench does not yet compare against expected values automatically.

### Known limitations of the current testbench

- Not self-checking: there is no reference model and no automatic PASS/FAIL.
- Reads and writes are never issued in the same cycle, so simultaneous read/write is not exercised.
- No read is issued while the FIFO is empty, so underflow is not exercised.
- Stimulus is driven with blocking assignments at the clock edge, so the `full`/`empty` values printed in `write_data()` show the state before that write.

---

## Repository structure

```text
Synchronous_FIFO_Verification/
├── rtl/
│   └── sync_fifo.sv               # FIFO RTL (module fifo_sync)
├── tb/
│   └── tb_fifo_sync.sv            # Directed testbench (module tb_fifo_sync)
├── sim/
│   └── waveforms/
│       └── tb_fifo_sync_behav.wcfg  # Vivado waveform configuration
├── .gitignore
└── README.md
```

---

## How to run (Xilinx Vivado)

### GUI

1. Create a new **RTL Project** in Vivado.
2. Add `rtl/sync_fifo.sv` as a design source.
3. Add `tb/tb_fifo_sync.sv` as a simulation source and set `tb_fifo_sync` as the top module.
4. **Flow Navigator → Run Simulation → Run Behavioral Simulation**.
5. Optional: open `sim/waveforms/tb_fifo_sync_behav.wcfg` to load the saved waveform layout.

### Command line (xsim)

```bash
xvlog -sv rtl/sync_fifo.sv tb/tb_fifo_sync.sv
xelab tb_fifo_sync -debug typical -s fifo_sim
xsim fifo_sim -runall
```

---

## Roadmap

Planned work. None of this is in the repository yet.

- [ ] Interface with clocking blocks to remove driver/monitor race conditions
- [ ] Class-based layered testbench: transaction, generator, driver, monitor, scoreboard, environment
- [ ] Constrained-random stimulus, including simultaneous read/write and reads while empty
- [ ] Queue-based reference model in the scoreboard (accounting for the one-cycle read latency)
- [ ] SVA: full/empty mutual exclusion, overflow and underflow protection, reset behaviour
- [ ] Functional coverage: read/write combinations crossed with full/empty, fill levels, flag transitions
- [ ] Bug-injection experiments and a "bugs found" table
- [ ] One-command regression script

---

## Author

[@mahesh-allam](https://github.com/mahesh-allam)