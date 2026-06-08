# UART-Based Sensor Data Logger in Verilog

![Language](https://img.shields.io/badge/Language-Verilog-blue)
![Simulator](https://img.shields.io/badge/Simulator-ModelSim%2010.5b-green)
![Baud Rate](https://img.shields.io/badge/Baud%20Rate-9600-orange)
![Clock](https://img.shields.io/badge/Clock-100%20MHz-red)
![Status](https://img.shields.io/badge/Simulation-Passing-brightgreen)

A fully functional UART-based sensor data logger implemented in Verilog HDL. The system simulates a temperature and humidity sensor, transmits 16-bit sensor readings serially over UART at 9600 baud, and receives/decodes them — all verified through ModelSim simulation with waveform output.

---

## Project Overview

This project implements a complete UART communication pipeline on an FPGA-targeted Verilog design. A simulated sensor generates temperature and humidity data every second. The top-level controller splits the 16-bit reading into two 8-bit bytes and transmits them sequentially using the UART protocol (8N1 format). A UART receiver decodes the serial stream and outputs the reconstructed bytes with a valid flag.

### Key Specifications

| Parameter | Value |
|---|---|
| System Clock | 100 MHz |
| Baud Rate | 9600 bps |
| UART Format | 8N1 (8 data bits, No parity, 1 stop bit) |
| Data Packet | 16-bit (8-bit temp + 8-bit humidity) |
| Simulation Tool | ModelSim Intel FPGA Starter Edition 10.5b |
| Target Platform | Xilinx/Intel FPGA (synthesizable RTL) |

---

## Repository Structure

```
uart-sensor-logger-verilog/
├── rtl/
│   ├── clk_div.v         # Clock divider — generates 9600 baud tick from 100 MHz
│   ├── uart_tx.v         # UART Transmitter — 8N1, baud-enable driven FSM
│   ├── uart_rx.v         # UART Receiver — mid-bit sampling, 8N1
│   ├── sensor_sim.v      # Sensor simulator — outputs temp + humidity every 1s
│   └── top_module.v      # Top-level — instantiates and connects all modules
├── tb/
│   ├── clk_div_tb.v      # Testbench for clock divider
│   └── testbench_uart.v  # Top-level system testbench with monitor + VCD dump
├── docs/
│   └── IEEE_Report.pdf   # Full technical report (see below)
└── README.md
```

---

## Module Descriptions

### 1. `clk_div.v` — Clock Divider
Divides the 100 MHz system clock down to a 9600 Hz baud tick using a parameterised counter. The output `clk_out` acts as a baud-enable pulse fed into the UART modules.

```
Parameter: DIV = 5208
Output frequency = 100 MHz / 5208 ≈ 9600 Hz
```

### 2. `uart_tx.v` — UART Transmitter
A 4-state FSM (IDLE → START → DATA → STOP) that serialises an 8-bit byte onto the TX line at 9600 baud. Advances state only on `baud_en` pulses — single clock domain design, FPGA-safe.

```
States:   IDLE → START → DATA (D0..D7, LSB first) → STOP → IDLE
Signals:  tx_start (trigger), tx_busy (status), tx_done (completion pulse)
```

### 3. `uart_rx.v` — UART Receiver
Mirror FSM of TX. Detects START bit falling edge, jumps to mid-bit (half baud period = 2604 cycles) for clean sampling, then samples each data bit on subsequent baud ticks.

```
Mid-bit sampling: waits HALF_PERIOD = 2604 cycles after START detection
Output: data_out[7:0], rx_valid pulse when byte complete
```

### 4. `sensor_sim.v` — Sensor Simulator
Generates incrementing temperature and humidity values, pulsing `data_ready` for one clock cycle every second (100,000,000 cycles at 100 MHz).

```
Initial values: temp = 0x18 (24°C), hum = 0x3C (60%)
Updates: both increment by 1 every second
```

### 5. `top_module.v` — Top Level
Connects all modules and implements a 3-state controller FSM (IDLE → SEND_TEMP → SEND_HUM) that fires the UART TX twice per sensor reading — temperature byte first, humidity byte second.

```
Data flow:
sensor_sim → [data_ready] → top FSM → uart_tx → tx wire
                                                      ↓ (loopback)
                                                   uart_rx → data_out
```

---

## How to Simulate

### ModelSim

```tcl
# 1. Compile all files
vlog rtl/clk_div.v rtl/uart_tx.v rtl/uart_rx.v rtl/sensor_sim.v rtl/top_module.v
vlog tb/testbench_uart.v

# 2. Start simulation
vsim top_tb

# 3. Add signals to wave window
add wave /top_tb/clk
add wave /top_tb/u_top/baud_en
add wave /top_tb/u_top/tx
add wave /top_tb/u_top/u_sensor/data_ready
add wave /top_tb/u_top/u_sensor/temp_data
add wave /top_tb/u_top/u_sensor/hum_data
add wave /top_tb/u_top/u_tx/tx_done
add wave /top_tb/u_top/u_rx/rx_valid
add wave /top_tb/u_top/u_rx/data_out

# 4. Run (use 10_000 for ONE_SECOND in sensor_sim for fast sim)
run 5000000ns
```

### Expected Output in Transcript
```
Time=2526245000 | rx_valid=0 | rx_data=0x??
Time=2526255000 | rx_valid=1 | rx_data=0x??   <-- byte received!
Time=2526265000 | rx_valid=0 | rx_data=0x??
```

Two `rx_valid` pulses per sensor reading — one for temperature, one for humidity.

---

## System Architecture

```
         100 MHz clk
              │
         ┌────▼────┐
         │ clk_div │──── baud_en (9600 Hz tick)
         └─────────┘           │
                               ├──────────────┐
         ┌───────────┐         │              │
         │sensor_sim │    ┌────▼────┐    ┌────▼────┐
         │temp + hum │───►│ uart_tx │    │ uart_rx │
         │data_ready │    └────┬────┘    └────┬────┘
         └───────────┘         │              │
               │          tx wire         data_out
         ┌─────▼─────┐         │         rx_valid
         │  top FSM  │◄────────┘
         │IDLE→TEMP  │    (loopback)
         │    →HUM   │
         └───────────┘
```

---

## Simulation Results

The system successfully transmits and receives both the temperature and humidity bytes over the UART loopback path. The `rx_valid` signal pulses correctly for each received byte, and `data_out` matches the expected sensor values.

Key waveform observations:
- `baud_en` pulses at regular 5208-cycle intervals confirming correct baud generation
- `tx` line shows clear UART framing: START (LOW) → 8 data bits → STOP (HIGH)
- `rx_valid` fires exactly twice per sensor reading cycle
- `data_out` correctly reflects the transmitted sensor byte
<img width="787" height="440" alt="image" src="https://github.com/user-attachments/assets/547dd16b-5921-4f39-a91b-ef3ae2ca7c5d" />

---

## Author

**Param Tomar**  
ECE Undergraduate, Batch 2028  
Jaypee Institute of Information Technology, Noida  
GitHub: [Param-ece](https://github.com/Param-ece)

---

## License

MIT License — free to use, modify, and distribute with attribution.
