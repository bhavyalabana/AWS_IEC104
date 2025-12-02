# AWS_IEC104
Sender(Laptop A) send the data to 4giec(RPI) and then 4giec will send the data to Receiver.py(Laptop B or AWS and laptop C)

# IEC-104 4G Gateway & Receiver

This repository contains a small IEC-104 test system:
- `scripts/Sender.py` — IEC-104 client used by **Laptop A** to send test frames. :contentReference[oaicite:6]{index=6}
- `scripts/4giec104withtelnet.py` — Raspberry Pi gateway (4G + fallback to Tailscale + AWS + CSV persistence). :contentReference[oaicite:7]{index=7}
- `scripts/Receiver.py` — IEC-104 server/parser used by **Laptop B** (or AWS VM) to receive/process frames and save decoded values to CSV. :contentReference[oaicite:8]{index=8}

---

## Goals

- Forward IEC-104 frames from Laptop A → RPi (4G) → Laptop B/AWS.  
- If local LAN reachable: RPi forwards only to LAN target.  
- If LAN unreachable: RPi attempts Tailscale, publishes to AWS IoT (if configured), and saves frames to a local CSV as persistent backup. A background flusher will retry sending stored rows when a target becomes reachable.

---

## Prerequisites

- Python 3.8+ on all machines (RPi, laptops, servers)
- Network connectivity as described below
- (Optional) AWS IoT Core certificates (if you want to enable AWS publish)
- (Optional) Tailscale or another VPN for remote device addressing
- On RPi: 4G module + internet access and tailscale installed (if using tailscale fallback)

---

## Quickstart — local test (no AWS/Tailscale)

1. On the machine meant to receive frames (Laptop B), run:

```bash
cd scripts
python3 Receiver.py
# default listens on 0.0.0.0:2404 and writes iec104_data.csv
