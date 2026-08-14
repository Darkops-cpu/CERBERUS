# Cerberus

An open-source iCE40 FPGA development board with a fully open toolchain.

No licence server, no 100 GB install. Yosys, nextpnr and IceStorm run on any laptop and
synthesise a design this size in seconds.

> **Status:** rev A, first boards in fabrication. Untested. Do not fabricate your own copy
> until this note is gone.

---

## Features

| | |
|---|---|
| **FPGA** | Lattice iCE40UP5K — 5,280 LUT4, 120 kbit BRAM, 1 Mbit SPRAM, 8 DSP blocks |
| **PSRAM** | 8 MB QSPI (LY68L6400SLIT) |
| **Flash** | 4 MB SPI configuration flash (W25Q32) |
| **Host** | FT2232HL — USB programming *and* a serial console over one cable |
| **Clock** | 12 MHz active oscillator into a global clock buffer |
| **Expansion** | 2× PMOD, 8 user GPIO on a 0.1" header |
| **LEDs** | 3 user RGB + power, USB and CDONE status |
| **Connector** | USB-C |
| **PCB** | 4-layer, hand-assembled |

The FT2232H's two channels run independently, so you can flash a bitstream and watch a UART
console at the same time without switching modes.

---

## Getting started

### 1. Install the toolchain

Download the [OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build/releases) and
add it to your `PATH`. That's Yosys, nextpnr, IceStorm, Icarus, Verilator, GTKWave and
`iceprog` in one archive. Nothing to compile, nothing to licence.

```sh
source <path-to-oss-cad-suite>/environment
```

### 2. Build

```sh
yosys -p 'synth_ice40 -top top -json top.json' top.v
nextpnr-ice40 --up5k --package sg48 --json top.json --pcf cerberus.pcf --asc top.asc
icepack top.asc top.bin
```

### 3. Flash

The config SPI is on the FT2232H's **channel B**, so the `-I B` flag is required:

```sh
iceprog -I B top.bin
```

The CDONE LED lights when configuration succeeds.

### 4. Console

Channel A enumerates as a second serial port. On Linux the two channels appear as
`/dev/ttyUSB0` (A) and `/dev/ttyUSB1` (B):

```sh
picocom -b 115200 /dev/ttyUSB0
```

---

## Pinout

Key pins. Everything else is in `cerberus.pcf`.

| Function | FPGA pin |
|---|---|
| 12 MHz clock in | 37 (G1) |
| UART TX (FPGA → host) | 18 |
| UART RX (host → FPGA) | 19 |
| PSRAM CE# | 6 |
| PSRAM SCK | 9 |
| PSRAM SIO0–3 | 10, 11, 12, 13 |
| RGB LEDs | 39, 40, 41 |
| CDONE | 7 |
| CRESET_B | 8 |

Pins 14–17 are the configuration SPI bus and are not available to user designs.

**Globals available on this package:** 35 (G0), 37 (G1), 20 (G3), 44 (G6). Put clocks on
these — a clock on an ordinary pin routes through general fabric with worse skew.

**RGB LEDs** are constant-current sinks driven by the `SB_RGBA_DRV` primitive. Current is
set in the IP, not by a resistor.

---

## Notes

**Reset.** The button pulls `CRESET_B` low and reconfigures the FPGA from flash on release.
Configuration takes under 140 ms.

**Internal oscillator.** The iCE40's internal 48 MHz `SB_HFOSC` is ±20% on industrial parts.
Fine for blinking an LED, useless for a UART — that's why the board has an external
oscillator. Use pin 37.

**PSRAM.** The controller must break long transfers into chunks and deassert CE# between
them — the part self-refreshes only while CE# is high, and holding it low past the tCEM
window loses data. Linear burst across page boundaries is not supported on this part.

**Powering.** Bus-powered from USB-C. There is no barrel jack and no external supply input.

---

## Licence

Hardware: [CERN-OHL-S v2](https://cern-ohl.web.cern.ch/)
Gateware and examples: MIT

---

## Acknowledgements

Built on the work of [YosysHQ](https://github.com/YosysHQ) and
[Project IceStorm](https://github.com/YosysHQ/icestorm). Pin conventions follow
[iCEBreaker](https://github.com/icebreaker-fpga/icebreaker), and the SG48 pinout is
documented in Lattice FPGA-UG-02001.