# Breath_Break-HackEdge_2026-

An Edge-Powered Wearable System for Detecting Real-Time Fatigue in Employees

This repository holds the hardware and firmware baseline for the biosensing front end of the project. It is built on the Texas Instruments AFE4950 evaluation module (AFE4950EVM). The AFE4950 is an analog front end for optical (PPG, SpO2) and electrical (ECG) biosignal capture. An MSP430F5528 microcontroller on the board configures the AFE4950 and streams the captured samples to a PC over USB.

Note: this repository contains the TI EVM firmware, board design files, documentation, and PC tools. It does not contain a fatigue-detection algorithm. The fatigue model is expected to consume the PPG/ECG data captured with this setup.

## Table of Contents

1. [Repository Layout](#repository-layout)
2. [What Each Code File Does](#what-each-code-file-does)
3. [Setup](#setup)
4. [How to Run](#how-to-run)
5. [Expected Results](#expected-results)
6. [PC Command Protocol](#pc-command-protocol)
7. [Troubleshooting](#troubleshooting)
8. [Git Hygiene](#git-hygiene)

## Repository Layout

| Path | Description |
|------|-------------|
| `SourceCode/V 3.0.0/AFE4950 V3.0.0/` | Code Composer Studio project with the MSP430 firmware (version 3.0.0). |
| `BioSensing/` | Windows installers: `Bio-Sensing_Installer_V5.3.0.exe` (PC GUI) and `AFE4950EVM_Installer_V2.0.exe` (EVM software package). |
| `DesignFile/` | Altium Designer project, schematics, PCB layout, BOM, Gerber, ODB and netlist files for the DC116A board. |
| `AFE4950_Register_Generation_v2.1_PPG_ECG.xlsm` | Excel tool that generates AFE register values for PPG and ECG. |
| `AFE4950_Register_Generation_v2.1_SpO2_10ph.xlsm` | Excel tool that generates AFE register values for SpO2 (10 phase). |
| `AFE4950EVM_UsersGuide_slau832.pdf` | EVM user guide. |
| `AFE4950_FullDatasheet_sbasa36E.pdf` | AFE4950 datasheet. |
| `AFE4950_IEC_Report_v1.1.pdf`, `EU_DoC_sszqnz4a.pdf` | Compliance report and EU declaration of conformity. |
| `Links_to_purchase_EVM_and_ICs.docx` | Purchase links for the EVM and ICs. |

Inside the firmware project folder:

| Folder | Description |
|--------|-------------|
| `driverlib/` | TI MSP430 driver library (GPIO, clock, timer, SPI, I2C, power management). Vendor code. |
| `USB_API/` | TI MSP430 USB stack (CDC, HID, MSC, PHDC). Vendor code. |
| `USB_app/` | USB helper constructs and event handling. |
| `USB_config/` | USB descriptors (VID 0x2047, PID 0x0300) and the Windows CDC driver `MSP430_CDC.inf`. |
| `targetConfigs/` | Debug target configuration for MSP430F5528 and MSP430F5529. |
| `.launches/` | Debug launch configurations. |
| `Debug/`, `Release/` | Build output (map files, object files, `.out` binary). |

## What Each Code File Does

All files are in `SourceCode/V 3.0.0/AFE4950 V3.0.0/`.

| File | Purpose |
|------|---------|
| `main.c` | Entry point. Stops the watchdog, calls `initEvmHardware()`, then loops forever. Each pass it checks the USB state: when enumeration is active it calls `processPcCommands()`; in any other USB state it raises the error LED indication, and when suspended it enters low power mode 3. Also holds the NMI interrupt handler (oscillator fault recovery and USB bus error handling). |
| `EVM_Hardware.h` | Hardware definitions: 24 MHz system clock, pin and port assignments (SPI, I2C, AFE reset, ADC ready, LDO enable, LEDs), the I2C address `0x5B`, LED and communication enums, and the AFE register bit structures. Declares the hardware functions. |
| `EVM_Hardware.c` | Board and AFE driver. Initializes the power core (VCORE 3), USB, GPIO, the clock (XT2) and a 50 ms timer. Implements SPI (1 MHz) and I2C (400 kbps) register read and write, interface selection, AFE reset, LED control, and capture enable/disable. Contains the timer interrupt (LED timing) and the data-ready (FIFO ready) interrupt that triggers sample capture. Also reads and writes the 128-byte information memory at `0x1980`. |
| `pcComm.h` | Declares the PC command enum, the USB buffer structures (RX 512 bytes, TX 2048 bytes per CDC channel), capture and extra-register capture structures, and helper prototypes. |
| `pcComm.c` | PC command handling over USB CDC. Decodes ASCII commands, runs register read/write, reset, interface select, firmware and device queries, bootloader entry, and information memory access. Implements `capture()`, which reads the FIFO pointer (register `0x6D`) and FIFO data (register `0xFF`) from the AFE and packs it into the capture channel buffer. Also contains the hex/ASCII conversion helpers. |
| `hal.h`, `hal.c` | USB hardware abstraction (port and clock initialization). Port initialization is partly commented out in `hal.c`; the active pin setup is done in `EVM_Hardware.c`. |
| `lnk_msp430f5528.cmd`, `lnk_msp430f5529.cmd` | Linker command files (memory map) for the two supported MCUs. |
| `USB_API/`, `USB_app/`, `USB_config/`, `driverlib/` | TI-provided libraries and USB configuration used by the files above. |

### How the firmware works

1. On power-up the MCU configures the clock, USB, GPIO, timer and the SPI interface, then resets the AFE4950.
2. The board enumerates over USB as a composite device with two virtual COM ports (CDC0 and CDC1).
3. CDC0 is the command channel (register access and control). CDC1 is the data channel (captured samples).
4. When the PC starts a capture, the firmware enables the AFE data-ready interrupt. On each interrupt it reads the AFE FIFO and sends the samples to the PC over CDC1.

## Setup

### Hardware required

- AFE4950EVM board (DC116A)
- USB cable (to connect the EVM to the PC)
- Optional: MSP-FET or other MSP430 debug probe, only if you want to flash the firmware yourself
- Sensors or probes for the signals you want to capture (PPG/SpO2 optical sensor, or ECG electrodes)

### Software required

- Windows PC
- Code Composer Studio (CCS) with MSP430 support, to build and flash firmware
- TI MSP430 compiler (the project is configured for TI CGT; the `__MSP430F5528__` symbol is defined for the build)
- Microsoft Excel with macros enabled, for the register generation workbooks
- Altium Designer (only to open the PCB and schematic files)

### Step 1: Clone the repository

```powershell
git clone https://github.com/hxt200010/Breath_Break-HackEdge_2026-.git
cd Breath_Break-HackEdge_2026-
```

### Step 2: Install the PC software

1. Run `BioSensing\AFE4950EVM_Installer_V2.0.exe` and follow the prompts.
2. Run `BioSensing\Bio-Sensing_Installer_V5.3.0.exe` and follow the prompts.

### Step 3: Connect the board

1. Connect the EVM to the PC with the USB cable.
2. Open Windows Device Manager and confirm two COM ports appear under "Ports (COM & LPT)". If the device is not recognized, install the driver from `SourceCode\V 3.0.0\AFE4950 V3.0.0\USB_config\MSP430_CDC.inf`.

### Step 4 (optional): Build and flash the firmware

The EVM normally ships with firmware already loaded. Only do this to modify or restore the firmware.

1. Open Code Composer Studio.
2. Choose File > Import > Code Composer Studio > CCS Projects, and select the folder `SourceCode\V 3.0.0\AFE4950 V3.0.0`.
3. Select the project and click Finish.
4. Build with Project > Build Project. Output goes to the `Debug` folder.
5. Connect a debug probe, then use Run > Debug (or one of the launch configurations in `.launches/`) to flash the board.

Alternatively, the firmware can be updated over USB through the bootloader (the `ENTER_BSL` command, see below).

## How to Run

### Using the PC GUI (recommended)

1. Connect the EVM over USB.
2. Start the Bio-Sensing GUI (installed in Step 2).
3. Select the COM ports for the command and capture channels if the GUI asks.
4. Load register settings. Generate them with one of the Excel workbooks:
   - `AFE4950_Register_Generation_v2.1_PPG_ECG.xlsm` for PPG and ECG
   - `AFE4950_Register_Generation_v2.1_SpO2_10ph.xlsm` for SpO2
5. Write the registers to the device and start a capture.
6. Watch the live waveform and save the data as needed.

### Using a serial terminal (for testing)

Open the command COM port (CDC0) at any baud rate (USB CDC ignores baud rate) and send ASCII commands such as:

```
C:CDC?
DEVICE?
FIRMWARE_VER?
```

See the [PC Command Protocol](#pc-command-protocol) section for the full list.

## Expected Results

| Action | Expected result |
|--------|-----------------|
| Plug in the board | The board-up LED activates, and two COM ports appear on the PC. |
| Send `C:CDC?` on the command port | Reply `REG_EDIT`. On the capture port the reply is `CAPTURE`. |
| Send `DEVICE?` | Reply `AFE4950`. |
| Send `FIRMWARE_VER?` | Reply `V3.0.0`. |
| Register write or read | Reply `SUCCESS`; a read also returns the 24-bit register value as 6 hex characters. The write and read LEDs blink for about 250 ms. |
| Unknown command | Reply `FAILED. UNKNOWN COMMAND`. |
| Start capture | The read and write LEDs stop blinking, and the capture port streams samples. The GUI shows live PPG, SpO2 or ECG waveforms. |
| Stop capture | Streaming stops and the LEDs return to normal operation. |
| USB disconnected, suspended, or enumeration in progress | The error LED indication is raised. This is normal until the host completes enumeration. |

## PC Command Protocol

Commands are ASCII text. The command channel (CDC0) handles control commands. The capture channel (CDC1) handles capture commands. The firmware matches commands by prefix.

| Command | Channel | Description | Reply |
|---------|---------|-------------|-------|
| `C:CDC?` | Both | Identify the channel. | `REG_EDIT` (CDC0) or `CAPTURE` (CDC1) |
| `DEVICE?` | CDC0 | Query the device name. | `AFE4950` |
| `FIRMWARE_VER?` | CDC0 | Query the firmware version. | `V3.0.0` |
| `C:RST_AFE` | CDC0 | Hardware reset of the AFE4950. | `SUCCESS` |
| `C:RST_UC` | CDC0 | Software reset of the microcontroller. | `SUCCESS`, then reboot |
| `C:COMM_SEL` | CDC0 | Select the interface. The character at position 11 is `I` for I2C or `S` for SPI. | `SUCCESS` or a failure message |
| `C:REG_W` | CDC0 | Write an AFE register: 2 hex digits of address, then 6 hex digits of data, with the AFE select digit at position 15. | `SUCCESS` |
| `C:REG_R` | CDC0 | Read an AFE register: 2 hex digits of address. | `SUCCESS` plus 6 hex digits |
| `C:W_INFO` | CDC0 | Write the 128-byte information memory. | `SUCCESS` |
| `C:R_INFO` | CDC0 | Read information memory. | Hex data |
| `ENTER_BSL` | CDC0 | Jump to the MSP430 bootloader for firmware update. | `SUCCESS` |
| `C:CAP_START` | CDC1 | Start capture (finite or continuous, with an optional FIFO count). | Streamed data |
| `C:CAP_STOP` | CDC1 | Stop capture. | None |
| `C:XTRA_REG` | CDC1 | Enable or disable reading extra registers with each capture frame. | Extra data frames |

Capture data frames are binary. Each sample is 4 bytes: three data bytes (least significant first) followed by an identifier byte. Identifier 2 marks the frame header, 3 the FIFO pointer, 5 the FIFO samples, and 7 the extra registers.

## Troubleshooting

- Board not detected: try another USB cable or port, and install `MSP430_CDC.inf`. The error LED stays active until USB enumeration finishes.
- Register reads return zero or garbage: confirm the interface selection (SPI is the default at power-up) and that the AFE is powered and out of reset.
- No data during capture: confirm registers were written, the timing engine is enabled, and the FIFO is configured. The firmware only streams data when the AFE data-ready pin fires.
- Build errors in CCS: confirm the MSP430 compiler is installed and the target device is MSP430F5528.
- Windows blocks an installer: right-click the `.exe` and choose Run as administrator.

## Git Hygiene

The firmware folder contains CCS workspace and build artifacts (`.metadata/`, `.jxbrowser-data/`, `Debug/`, `Release/`). These are not needed to build the project and bloat the repository. Consider adding a `.gitignore` such as:

```
.metadata/
.jxbrowser-data/
.jxbrowser.userdata/
Debug/
Release/
*.obj
*.pdom
```

## References

- AFE4950 datasheet: `AFE4950_FullDatasheet_sbasa36E.pdf`
- EVM user guide: `AFE4950EVM_UsersGuide_slau832.pdf`
- Firmware code is Copyright Texas Instruments Incorporated (BSD license, see headers in source files).
