# WinWing MCDU Reverse Engineered Driver

**An open-source, reverse-engineered driver for the WinWing Airbus MCDU hardware.**

This project provides the necessary protocol documentation and proof-of-concept code to interface with the WinWing MCDU without using the official "SimApp Pro" software. It is designed to facilitate the creation of wireless bridges (using ESP32) or custom integrations for flight simulators.

## 📚 Acknowledgements & Sources

This project stands on the shoulders of giants. Special thanks to the initial reverse engineering efforts by the community:

* **XSchenFly (WinWing MCDU for X-Plane):** * Original repository: [github.com/schenlap/winwing_mcdu](https://github.com/schenlap/winwing_mcdu)
    * This repository provided the foundational initialization sequences (`0xF0...`) and the memory mapping logic used for the Toliss Airbus on Linux.

* **Packet Analysis:**
    * Timing strategies were derived from analyzing raw USB traffic dumps (`.pcap`) from the official Windows driver, isolating the specific timing constraints required by the hardware's microcontroller.

## ⚙️ Technical Findings: The "Paced Streaming" Strategy

One of the most critical discoveries of this project is the handling of USB transmission timing.

### 1. Official Behavior: The "Ping-Pong" Protocol
The official SimApp Pro software utilizes a safe, bidirectional handshake protocol (Request-Response).
1.  **Host** sends a 64-byte data chunk.
2.  **Device** processes it and sends an acknowledgement (ACK/Pong).
3.  **Host** waits for the ACK before sending the next chunk.

While safe, this method introduces latency due to USB polling intervals and software overhead.

### 2. Our Implementation: "Paced Streaming" (Fire-and-Forget)
Through stress testing, we discovered that the device's microcontroller does not *require* the handshake to process data. It simply needs time to clear its input buffer.

We implemented a **Unidirectional Paced Streaming** strategy:
* We ignore the return "Pong" signal.
* Instead, we introduce a precise **micro-delay (~2ms)** between every 64-byte packet sent by the host.
* This matches the hardware's internal "digestion" speed, preventing buffer overflows and screen corruption (tearing/flickering) without the overhead of waiting for a response.

## 📝 Protocol Specification

### Display Geometry
* **Resolution:** 14 Lines × 24 Columns.
* **Character Depth:** 3 Bytes (Color Low, Color High, Char).
* **Visual Data:** 1008 Bytes.

### Memory Layout & Wrap-Around
The hardware uses an internal ring buffer of exactly **1024 Bytes**.
* **Bytes 0-1007:** Visual Data.
* **Bytes 1008-1023:** Padding (Ignored/Control).

**Critical Finding:** When exactly 1024 bytes are written to the device, the internal write cursor automatically wraps back to position `(0,0)`. 
* **Do NOT** send "Reset Cursor" commands between frames. Doing so interrupts the hardware's write cycle and causes flickering.
* Only sync the cursor once at startup.

### USB Packet Structure
Data is sent via USB HID Interrupt Transfers (Endpoint 1).
* **Header:** `0xF2`
* **Payload:** 63 Bytes of pixel data.

## 📂 Repository Structure

* `src/python/`: **Golden Master** Python driver. Contains the stable implementation with 2ms pacing.
* `src/esp32/`: (Planned) C++ implementation for ESP32-S3 wireless bridge.
* `docs/`: Detailed protocol notes and initialization sequences.
* `captures/`: Wireshark dumps used for timing analysis.

## ⚠️ Disclaimer
This is an unofficial project and is not affiliated with WinWing. Use this driver at your own risk. Reverse engineering hardware may void your warranty.