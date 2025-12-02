# IEC‑104 4G Gateway End‑to‑End System

This repository documents the full setup for a three‑node IEC‑104 data forwarding system:

* **Sender (Laptop A)** → sends IEC‑104 frames.
* **Gateway (4G‑enabled Raspberry Pi)** → receives frames, forwards to LAN, Tailscale, or AWS; stores CSV offline; retries on reconnect.
* **Receiver (Laptop B / AWS / Laptop C)** → decodes IEC‑104 frames and stores them.

This README explains:

1. System Architecture
2. Flow of Data
3. Repository Structure
4. Hardware & Software Requirements
5. Network Setup
6. Raspberry Pi (Gateway) Setup
7. Sender Setup (Laptop A)
8. Receiver Setup (Laptop B / AWS)
9. AWS IoT Optional Setup
10. Systemd Services
11. CSV Offline Storage & Flushing
12. Troubleshooting Guide
13. Future Improvements

---

## 🚀 **1. System Architecture**

The system transfers IEC‑104 frames across multiple possible network paths.

```
Laptop A (Sender)  →  Raspberry Pi (4G Gateway)  →  Laptop B / AWS
```

The Raspberry Pi acts as an intelligent gateway:

* Accepts IEC‑104 packets from Laptop A.
* Prefers LAN forwarding.
* If LAN unreachable → tries Tailscale.
* If no network → stores frame in CSV.
* When network back → flushes unsent CSV entries.
* Also optionally publishes to AWS IoT.

---

## 🔄 **2. Data Flow Logic (Gateway Behavior)**

The Raspberry Pi script performs:

1. **Laptop A connects to RPi** on port `12345`.
2. For each incoming IEC‑104 frame:

   * Test LAN → If reachable, forward.
   * If LAN unreachable → Try Tailscale target.
   * If fallback unreachable → Save frame into CSV.
   * If AWS IoT enabled → Also publish.
3. A background thread periodically:

   * Reads CSV.
   * Tests network.
   * Resends unsent rows.
   * Deletes rows that are successfully sent.

This ensures **no IEC‑104 packet is ever lost**.

---

## 📂 **3. Repository Structure**

```
iec104-4g-gateway/
├── README.md
├── LICENSE
├── .gitignore
├── config_example.py
├── scripts/
│   ├── Sender.py
│   ├── Receiver.py
│   └── 4giec104withtelnet.py
├── systemd/
│   ├── 4giec.service
│   └── iec104-receiver.service
├── docs/
│   ├── architecture.md
│   └── troubleshooting.md
└── examples/
    ├── sample_csv_output.csv
    └── sample_test_commands.txt
```

---

## 🖥️ **4. Hardware & Software Requirements**

### **Sender (Laptop A)**

* Windows / Linux / Mac
* Python 3.7+
* Local network connection to RPi

### **Gateway (Raspberry Pi)**

* Raspberry Pi 3/4/5
* Python 3.7+
* 4G USB module with active SIM
* Tailscale installed (optional but recommended)
* AWS IoT certificates (optional)

### **Receiver (Laptop B / AWS / Laptop C)**

* Python 3.7+
* Static LAN IP or Tailscale IP

---

## 🌐 **5. Network Setup Requirements**

### **Important Ports**

| Purpose                               | Port         |
| ------------------------------------- | ------------ |
| Sender → Gateway                      | **12345**    |
| Gateway → Receiver (IEC‑104 standard) | **2404**     |
| Tailscale Tunnel                      | Auto-managed |
| MQTT (AWS IoT TLS)                    | **8883**     |

### Required Conditions

* Laptop A must reach the RPi on port 12345
* RPi must be able to reach Laptop B on port 2404
* Tailscale IPs must be same subnet if using fallback
* AWS IoT policy must allow MQTT publish

---

## 🧰 **6. Gateway Setup (Raspberry Pi)**

### Clone Repo

```
git clone https://github.com/youruser/iec104-4g-gateway
cd iec104-4g-gateway
```

### Install Dependencies

```
pip3 install AWSIoTPythonSDK
```

### Configure Values

Copy the example configuration:

```
cp config_example.py config.py
```

Edit:

* Primary target IP (Laptop B LAN)
* Fallback target IP (Tailscale)
* AWS cert paths
* CSV path for offline storage

### Start Gateway

```
python3 scripts/4giec104withtelnet.py
```

---

## 💻 **7. Sender Setup (Laptop A)**

```
cd scripts
python3 Sender.py
```

Sender will prompt:

* Enter **Server IP** → RPi IP
* Enter **Port** → 12345

Features:

* Manual or auto sending every 5 seconds
* Sends IEC‑104 formatted frames

---

## 📥 **8. Receiver Setup (Laptop B / AWS)**

Run:

```
cd scripts
python3 Receiver.py
```

Receiver:

* Listens on port 2404
* Parses IEC‑104 packets
* Saves decoded values into `iec104_data.csv`

---

## ☁️ **9. AWS IoT Optional Integration**

If enabled in RPi:

* RPi publishes received frames to AWS IoT MQTT topic

### AWS Requirements

* Root CA
* Device certificate
* Private key
* IoT policy allowing `iot:Publish` to topic

Update `config.py` with paths and host.

---

## ⚙️ **10. Systemd Services**

### Gateway on RPi

```
sudo cp systemd/4giec.service /etc/systemd/system/
sudo systemctl enable --now 4giec.service
```

### Receiver

```
sudo cp systemd/iec104-receiver.service /etc/systemd/system/
sudo systemctl enable --now iec104-receiver.service
```

---

## 🗂 **11. CSV Offline Storage Behavior**

If both LAN and Tailscale unreachable:

* Each IEC‑104 frame stored as a CSV row

A background flusher:

* Reads CSV in batches
* Tries sending to fallback target
* Deletes rows after successful forwarding

This makes the gateway **fully resilient**.

---

## 🛠 **12. Troubleshooting**

### Common Issues

| Issue                    | Cause                      | Fix                                |
| ------------------------ | -------------------------- | ---------------------------------- |
| Sender cannot connect    | Wrong RPi IP or port       | Check port 12345                   |
| Receiver gets no packets | LAN unreachable            | Ensure RPi → Laptop B connectivity |
| CSV growing too large    | Persistent network failure | Check connectivity                 |
| AWS publish errors       | Wrong cert paths           | Fix paths in config.py             |

---

## 🚧 **13. Future Improvements**

* Docker support
* Web dashboard for monitoring
* IEC‑104 integrity validation
* Automatic OTA updates for RPi

---

## 📞 Support

For improvements or customizations, reach out anytime.
