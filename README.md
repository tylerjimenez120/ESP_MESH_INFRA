# ESP32 Ad-Hoc Mesh Network — JPEG Streaming POC

> Exploring the limits of a self-contained Wi-Fi mesh built exclusively with ESP32 modules — no external router, no infrastructure.

---

## What is this?

This repository documents a Proof of Concept that answers one concrete question:

**Can three ESP32 nodes form an autonomous multi-hop Wi-Fi relay chain capable of transmitting live JPEG frames — with zero external infrastructure?**

The experiment uses a linear 3-node topology across physical spaces, a custom UDP fragmentation protocol, and JPEG frames from an ESP32-CAM as the maximum stress load.

---

## Architecture

```
[ NODE 1: Living Room ]        [ NODE 2: Hallway ]        [ NODE 3: Bedroom ]        [ PC ]
     ESP32-CAM                   ESP32-WROOM-32              ESP32-WROOM-32
     Transmitter          --->     The Bridge        --->      Receiver          ---> OpenCV
   (Subnet 192.168.5.x)         (APSTA mode)              (Subnet 192.168.4.x)
```

| Node | Role | Hardware | Key Detail |
|------|------|----------|------------|
| Node 1 | Transmitter | ESP32-CAM (OV2640) | 4MB PSRAM, JPEG capture via DMA |
| Node 2 | Bridge / Relay | ESP32-WROOM-32 | APSTA mode, single radio, no PSRAM |
| Node 3 | Receiver | ESP32-WROOM-32 | UDP server, Tag validation |
| PC | Visualizer | Linux + Python 3 | OpenCV JPEG reconstruction |

---

## Custom UDP Protocol

Every JPEG frame is sliced into **1000-byte chunks**. Each chunk gets a **13-byte header** prepended:

```
[ 0x77 Magic Tag | Frame ID (4B) | Chunk Index (4B) | Reserved (4B) | Payload (1000B) ]
```

Node 3 validates the `0x77` tag on every incoming packet. Invalid packets are silently discarded.

---

## What Was Discovered

### ✅ What worked
- Custom UDP fragmentation protocol — solid and reusable
- Dual-subnet topology (`192.168.5.x` / `192.168.4.x`) — clean, no broadcast collisions
- PMF fix for Reason 209 disconnections — `pmf_cfg = {capable: true, required: false}` on all nodes
- Single-hop JPEG delivery — stable at the protocol level

### ❌ What failed — and why
The bridge node (Node 2) in APSTA mode has a **single physical radio** that must time-share between receiving (STA role) and transmitting (AP role). Under continuous JPEG burst traffic, the radio's time-division multiplexing drops packets faster than the lwIP stack can queue them.

This was confirmed across **4 independent configurations**:

| Configuration | Result |
|---|---|
| ESP32-C6 single-core as bridge | Drops and latency |
| ESP32 dual-core as bridge | Same result |
| UART output on Node 3 | Drops and latency |
| Direct UDP server on Node 3 | Same result |

Same symptom across all four. The bridge radio was always the bottleneck — not the CPU, not the output method.

### The engineering conclusion
> The protocol works. The mesh works. **Continuous video relay on a single-radio chip is the limit.**

For telemetry and low-bandwidth control (small, infrequent packets), this architecture is 100% viable. For sustained high-throughput relay, dedicated RF infrastructure is required.

---

## Repository Structure

```
/
├── node1_sala/          # ESP32-CAM transmitter firmware (ESP-IDF)
│   ├── main.c
│   ├── camera_handler.c / .h
│   ├── network_handler.c / .h
│   └── network_protocol.h
│
├── node2_pasillo/       # Bridge relay firmware (ESP-IDF)
│   ├── main.c
│   └── pasillo_handler.c / .h
│
├── node3_cuarto/        # Receiver firmware (ESP-IDF)
│   ├── main.c
│   └── uart_handler.c / .h
│
├── pc_receiver/         # Python receiver script
│   └── receiver.py
│
└── 


---

## License

MIT — use it, break it, learn from it.
