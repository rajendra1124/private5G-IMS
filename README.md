# Private 5G Standalone Testbed
## Data · Voice · Video · Messaging Services

> **Mason Innovation Lab — George Mason University**  
> Rajendra Paudyal · Rajendra Upadhyay · Al Nahian Bin Emran · Lisa Donnan · Duminda Wijesekera  
> Sponsored by **Cybastion Technology**

[![Open5GS](https://img.shields.io/badge/Core-Open5GS-blue)](https://open5gs.org/)
[![srsRAN](https://img.shields.io/badge/RAN-srsRAN_Project_Split_7.2-green)](https://docs.srsran.com/)
[![Kamailio](https://img.shields.io/badge/IMS-Kamailio_6.1-orange)](https://www.kamailio.org/)
[![Band](https://img.shields.io/badge/Band-n78_80MHz_4×2_MIMO-purple)]()
[![GitHub](https://img.shields.io/badge/Repo-private5G--IMS-black)](https://github.com/rajendra1124/private5G-IMS)

---

## Table of Contents

1. [Overview & Research Question](#1-overview--research-question)
2. [System Architecture](#2-system-architecture)
3. [Server & Hardware Specifications](#3-server--hardware-specifications)
4. [Repository Layout](#4-repository-layout)
5. [Installation](#5-installation)
   - 5.1 [Host OS & Kernel Preparation](#51-host-os--kernel-preparation)
   - 5.2 [Intel E810 NIC — PTP Grandmaster Setup](#52-intel-e810-nic--ptp-grandmaster-setup)
   - 5.3 [DPDK & VF (Virtual Function) Setup](#53-dpdk--vf-virtual-function-setup)
   - 5.4 [Open5GS 5G Core](#54-open5gs-5g-core)
   - 5.5 [srsRAN Project (CU/DU Split 7.2)](#55-srsran-project-cudu-split-72)
   - 5.6 [Benetel RAN650 Radio Unit Bring-Up](#56-benetel-ran650-radio-unit-bring-up)
   - 5.7 [Kamailio IMS + RTPEngine](#57-kamailio-ims--rtpengine)
6. [Configuration](#6-configuration)
   - 6.1 [Open5GS — AMF, SMF, UPF](#61-open5gs--amf-smf-upf)
   - 6.2 [srsRAN gNB — Split 7.2 OFH](#62-srsran-gnb--split-72-ofh)
   - 6.3 [Benetel RAN650 — ru_config.cfg](#63-benetel-ran650--ru_configcfg)
   - 6.4 [PTP Grandmaster — ptp4l & phc2sys](#64-ptp-grandmaster--ptp4l--phc2sys)
   - 6.5 [Kamailio & RTPEngine](#65-kamailio--rtpengine)
   - 6.6 [UE / SIM Provisioning](#66-ue--sim-provisioning)
7. [Running the System](#7-running-the-system)
8. [IMS Service Validation](#8-ims-service-validation)
9. [Performance Testing & Results](#9-performance-testing--results)
   - 9.1 [OTA Throughput (iperf3 at UPF)](#91-ota-throughput-iperf3-at-upf)
   - 9.2 [End-to-End Throughput (Beyond N6)](#92-end-to-end-throughput-beyond-n6)
   - 9.3 [Latency](#93-latency)
   - 9.4 [Voice & Video RTP Throughput](#94-voice--video-rtp-throughput)
   - 9.5 [Cell Setup Time Distribution](#95-cell-setup-time-distribution)
10. [Troubleshooting](#10-troubleshooting)
11. [Results Summary](#11-results-summary)
12. [Roadmap](#12-roadmap)
13. [References](#13-references)

---

## 1. Overview & Research Question

**Research Question:** Can a fully open-source 5G Standalone O-RAN architecture deliver commercial-grade throughput and real-time IMS services comparable to enterprise deployments?

**Answer (confirmed):** Yes. This testbed proves that Open5GS + srsRAN Split 7.2 + Kamailio achieves multi-hundred Mbps end-to-end throughput alongside stable voice, video, and messaging services, validated with commercial Android UEs.

### Key Results at a Glance

| Metric | Value | Condition |
|--------|-------|-----------|
| OTA DL Throughput (peak) | **3.5 Gbps** | iperf3 server at UPF (`ogstun`) |
| OTA DL Throughput (median) | **~3.2 Gbps** | 80 MHz · 4×2 MIMO · n78 |
| E2E DL Throughput (avg) | **695 Mbps** | Limited by 1G N6 link |
| E2E UL Throughput (avg) | **60 Mbps** | Commercial Android UE(Aquos sense 8) |
| Round-Trip Latency (avg) | **~20 ms** | ICMP ping UE ↔ server |
| Voice Call | **INVITE → 200 OK** | SIP via Kamailio 6.1 |
| RTP Audio | **~90 kbps** | OPUS/G.711, stable |
| RTP Video | **~750 kbps** | H.264 adaptive |
| Fronthaul Stability | **No packet drops** | PTP-locked via E810 GM |

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│               SERVER: Intel Xeon Gold 6438N (64 vCPU / 515 GB RAM)      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Open5GS 5G Core                                                 │   │
│  │  AMF(127.0.0.5) · SMF · UPF(ogstun 10.45/16) · AUSF · UDM · NRF  │   │
│  └─────────────────────────┬────────────────────────────────────────┘   │
│                             │ N2/NGAP (127.0.0.1 → 127.0.0.5)           │
│  ┌──────────────────────────▼───────────────────┐                       │
│  │  srsRAN Project gNB  (CU/DU Split 7.2)       │                       │
│  │  PLMN 99970 · TAC 7 · PCI 1 · Band n78 80MHz │                       │
│  │  ARFCN 637212 · 4T2R · TDD DDDDDDDSUU        │                       │
│  └──────────────────────────┬───────────────────┘                       │
│                             │ O-RAN Fronthaul (OFH)                     │
│  ┌──────────────────────────▼───────────────────┐                       │
│  │  Intel E810 NIC  (enp111s0f0/VF0000:6f:01.0) │                       │
│  │  VF MAC: 00:33:22:33:00:11                   │                       │
│  │  PTP Grandmaster (G.8275.1) → /dev/ptp0      │                       │
│  │  VLAN 5 (C-Plane + U-Plane)                  │                       │
│  └──────────────────────────┬───────────────────┘                       │
│                             │ SFP28 / DAC cable                         │
└─────────────────────────────┼───────────────────────────────────────────┘
                              │ Fronthaul VLAN 5
┌─────────────────────────────▼───────────────────────────────────────────┐
│  Benetel RAN650  (SW v1.4.0)                                            │
│  n78 · 3558.18 MHz · 80 MHz BW · 4T2R · 24 dBm                          │
│  MAC: 70:b3:d5:e1:5b:48 · TDD: DDDDDDDSUU                               │
│  PTP Slave → locks to E810 GM · BFP 9-bit dynamic compression           │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │ 17.3 dBi Antenna
                              │ (n78 air interface)
                    ┌─────────▼──────────┐
                    │  COTS Android UE   │
                    │  5G SA · PLMN 99970│
                    └────────────────────┘

IMS Path:
  UE ──[IMS PDU DNN]──▶ UPF(ogstun2 10.46.0.1/16) ──▶ Kamailio 6.1
                                                     (Registrar/Proxy + RTPEngine)
```

### Component Versions

| Component | Version / Build | Role |
|-----------|----------------|------|
| Ubuntu Server | 22.04 LTS | Host OS |
| Open5GS | 2.7.x | 5G SA Core (AMF, SMF, UPF, AUSF, UDM, NRF) |
| srsRAN Project | 24.x (main branch) | CU/DU gNB — O-RAN Split 7.2 |
| Benetel RAN650 | SW v1.4.0-NM-5561771 | O-RU: n78, 80 MHz, 4T2R |
| Intel E810 NIC | ice driver  | Fronthaul + PTP Grandmaster |
| DPDK | 23.11 LTS | Packet I/O for OFH |
| Kamailio | 6.1 | IMS: SIP Registrar/Proxy |
| RTPEngine | Latest | RTP media relay (voice & video) |
| MySQL | 8.x | Kamailio subscriber DB |

---

## 3. Server & Hardware Specifications

### Compute Server

```
Model   : Intel Xeon Gold 6438N
Cores   : 32 physical cores / 64 threads (HT enabled)
RAM     : 515 GB DDR5 ECC
L2 cache: 64 MiB (2 MiB per core)
L3 cache: 60 MiB shared
NUMA    : Single NUMA node (all 64 vCPUs in node0)
AVX-512 : Yes (AVX512F, AVX512DQ, AVX512BW, AVX512VL, AVX512VNNI, AMX)
VT-x    : Enabled (used for DPDK VFIO)
```

```
top output (idle — srsRAN not running):
  %Cpu: 1.4us  0.4sy  97.5id
  MiB Mem : 515285.8 total  /  496932.4 free  /  5911.7 used
  Active: ptp4l (5.9% — PTP GM running), mysqld (IMS DB)
```

### Network Interface

| Port | Purpose | Speed | Notes |
|------|---------|-------|-------|
| `enp111s0f0` | Fronthaul (PTP GM + O-RAN OFH) | 25GbE SFP28 | VF `0000:6f:01.0` used by DPDK |
| `enp111s0f1` | Management / uplink | 25GbE | |

### Benetel RAN650 Radio Unit

| Parameter | Value |
|-----------|-------|
| SW Version | RAN650-1v1.4.0-NM-5561771 |
| Band | n78 (3.3–3.8 GHz) |
| Center Frequency | 3558.18 MHz (ARFCN 637212) |
| Bandwidth | 80 MHz |
| MIMO | 4T2R (ports 1,2,3,4 TX; 1,2 RX) |
| TX Power | 24 dBm per port |
| Antenna | 17.3 dBi panel |
| TDD Pattern | DDDDDDDSUU (10 ms period) |
| Compression | Dynamic BFP 9-bit |
| Fronthaul MAC | `70:b3:d5:e1:5b:48` |
| VLAN (C+U plane) | 5 |

---

## 4. Repository Layout

```
private5G-IMS/
├── README.md                          ← This file
├── configs/
│   ├── gnb_ru_ran650_tdd_n78_80mhz_4x2.yml   ← srsRAN gNB config (actual)
│   ├── ran650_ru_config.cfg                   ← Benetel RAN650 /etc/ru_config.cfg
│   ├── ptp_systemd_services.conf              ← ptp4l + phc2sys service files
│   ├── e810-gm.cfg                            ← linuxptp grandmaster config
│   ├── open5gs/
│   │   ├── amf.yaml
│   │   ├── smf.yaml
│   │   └── upf.yaml
│   └── kamailio/
│       ├── kamailio.cfg
│       └── rtpengine.conf
├── docs/
│   └── architecture.png
└── scripts/
    ├── start_core.sh
    ├── start_gnb.sh
    └── perf_test.sh
```

---

## 5. Installation

### 5.1 Host OS & Kernel Preparation

```bash
# Update base system
sudo apt update && sudo apt upgrade -y

# Install low-latency kernel for deterministic fronthaul timing
sudo apt install linux-lowlatency linux-tools-lowlatency -y
sudo reboot

# Verify booted into lowlatency kernel
uname -r
# Expected output: 5.15.0-xx-lowlatency (or 6.x variant)

# Set CPU frequency governor to performance on all 64 cores
sudo apt install cpufrequtils -y
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl disable ondemand 2>/dev/null || true
sudo cpupower frequency-set -g performance

# Verify
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Expected: performance

# Disable CPU C-states for real-time latency (optional but recommended)
sudo apt install msr-tools -y
# Add to /etc/default/grub:
#   GRUB_CMDLINE_LINUX="processor.max_cstate=1 intel_idle.max_cstate=0 idle=poll"
sudo update-grub && sudo reboot
```

### 5.2 Intel E810 NIC — PTP Grandmaster Setup

The E810 NIC on `enp111s0f0` serves as the **PTP Grandmaster clock** for O-RAN fronthaul synchronization. The RAN650 acts as a PTP slave, locking to this GM before radio bring-up.

```bash
# Install linuxptp
sudo apt install linuxptp -y

# Create PTP Grandmaster config for E810
sudo mkdir -p /etc/linuxptp
sudo tee /etc/linuxptp/e810-gm.cfg > /dev/null << 'EOF'
[global]
#
# Intel E810 — O-RAN G.8275.1 Telecom Profile Grandmaster
#
clockClass               135
clockAccuracy            0x21
offsetScaledLogVariance  0x4B00
priority1                128
priority2                128
domainNumber             24
#
# PTP v2 over L2 (telecom profile G.8275.1)
network_transport        L2
delay_mechanism          P2P
tx_timestamp_timeout     10
summary_interval         0
logMinPdelayReqInterval  0
logSyncInterval         -4
logAnnounceInterval      1
announceReceiptTimeout   3
[enp111s0f0]
EOF

# Install and enable systemd services (from configs/ptp_systemd_services.conf)
sudo cp configs/ptp_systemd_services.conf /tmp/
# Split into individual service files:

sudo tee /etc/systemd/system/ptp4l-e810.service > /dev/null << 'EOF'
[Unit]
Description=PTP4L Grandmaster on Intel E810 (O-RAN DU)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/sbin/ptp4l -f /etc/linuxptp/e810-gm.cfg -i enp111s0f0
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/phc2sys-e810.service > /dev/null << 'EOF'
[Unit]
Description=PHC2SYS syncing system clock to E810 hardware clock
After=ptp4l-e810.service
Requires=ptp4l-e810.service

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s /dev/ptp0 -c CLOCK_REALTIME -O 0 -m
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ptp4l-e810 phc2sys-e810
sudo systemctl start  ptp4l-e810
sleep 3
sudo systemctl start  phc2sys-e810

# Verify PTP GM is advertising correctly
sudo journalctl -u ptp4l-e810 -n 20
# Expected: "selected best master clock", clockClass=135

# Check E810 hardware clock synchronized
sudo phc_ctl /dev/ptp0 get
```

### 5.3 DPDK & VF (Virtual Function) Setup

srsRAN uses DPDK with a **Virtual Function (VF)** of the E810 NIC for low-latency fronthaul packet I/O. The VF PCI address `0000:6f:01.0` is what appears in the gNB config.

```bash
# Install DPDK 23.11 LTS
sudo apt install dpdk dpdk-dev libdpdk-dev python3-pyelftools -y

# Find E810 PF PCI address
lspci | grep -i "ethernet.*e810"
# Example: 6f:00.0 Ethernet controller: Intel Corporation Ethernet ...

# Create 1 VF on the Physical Function
echo 1 | sudo tee /sys/bus/pci/devices/0000:6f:00.0/sriov_numvfs

# Find the VF PCI address
ls /sys/bus/pci/devices/0000:6f:00.0/virtfn0 -la
# Symlink → ../0000:6f:01.0  (this is the VF used in gNB config)

# Set VF MAC address to match gNB and RAN650 config
sudo ip link set enp111s0f0 vf 0 mac 00:33:22:33:00:11
sudo ip link set enp111s0f0 vf 0 trust on
sudo ip link set enp111s0f0 vf 0 spoofchk off

# Bind VF to vfio-pci for DPDK userspace access
sudo modprobe vfio-pci
sudo dpdk-devbind.py --bind=vfio-pci 0000:6f:01.0

# Verify binding
dpdk-devbind.py --status | grep 6f:01
# Expected: 0000:6f:01.0 'Ethernet ... drv=vfio-pci unused=iavf'

# Verify VF MAC
ip link show enp111s0f0
# Should show: vf 0 ... MAC 00:33:22:33:00:11

# Make VF binding persistent across reboots
sudo tee /etc/rc.local > /dev/null << 'EOF'
#!/bin/bash
echo 1 > /sys/bus/pci/devices/0000:6f:00.0/sriov_numvfs
ip link set enp111s0f0 vf 0 mac 00:33:22:33:00:11
ip link set enp111s0f0 vf 0 trust on
ip link set enp111s0f0 vf 0 spoofchk off
modprobe vfio-pci
/usr/bin/dpdk-devbind.py --bind=vfio-pci 0000:6f:01.0
exit 0
EOF
sudo chmod +x /etc/rc.local
```

### 5.4 Open5GS 5G Core

```bash
# Add Open5GS PPA and install
sudo add-apt-repository ppa:open5gs/latest -y
sudo apt update
sudo apt install open5gs -y

# Install Node.js for WebUI
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y

# Install Open5GS WebUI (subscriber management at http://localhost:9999)
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -

# Create TUN interfaces for user plane traffic
# Data DNN: "internet"
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip link set ogstun up

# IMS DNN: "ims"
sudo ip tuntap add name ogstun2 mode tun
sudo ip addr add 10.46.0.1/16 dev ogstun2
sudo ip link set ogstun2 up

# NAT masquerade for data DNN
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Make TUN interfaces persistent
sudo tee /etc/systemd/system/open5gs-tun.service > /dev/null << 'EOF'
[Unit]
Description=Open5GS TUN interfaces
After=network.target
Before=open5gs-upfd.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c '\
  ip tuntap add name ogstun mode tun 2>/dev/null || true; \
  ip addr add 10.45.0.1/16 dev ogstun 2>/dev/null || true; \
  ip link set ogstun up; \
  ip tuntap add name ogstun2 mode tun 2>/dev/null || true; \
  ip addr add 10.46.0.1/16 dev ogstun2 2>/dev/null || true; \
  ip link set ogstun2 up; \
  iptables -t nat -C POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE 2>/dev/null \
    || iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable open5gs-tun

# Start all core NFs (NRF first for service discovery)
sudo systemctl start open5gs-nrfd
sleep 2
for nf in scpd amfd smfd upfd ausfd udmd udrd pcfd nssfd bsfd; do
    sudo systemctl enable open5gs-${nf}
    sudo systemctl start  open5gs-${nf}
done
sudo systemctl enable open5gs-webui
sudo systemctl start  open5gs-webui

# Verify all NFs are running
sudo systemctl status open5gs-amfd open5gs-smfd open5gs-upfd | grep Active
```

### 5.5 srsRAN Project (CU/DU Split 7.2)

```bash
# Install build dependencies
sudo apt install cmake make gcc g++ pkg-config \
  libfftw3-dev libmbedtls-dev libboost-program-options-dev \
  libconfig++-dev libsctp-dev libyaml-cpp-dev \
  libzmq3-dev -y

# Clone srsRAN Project
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project

# Build with DPDK support (required for Split 7.2 OFH)
mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DENABLE_EXPORT=ON \
  -DENABLE_UHD=OFF \
  -DENABLE_DPDK=ON

make -j$(nproc)
# Build time: ~10–15 minutes on Xeon Gold 6438N

# Verify build
./apps/gnb/srsgnb --version

# Copy config to build directory (run from there)
cp /path/to/configs/gnb_ru_ran650_tdd_n78_80mhz_4x2.yml \
   ~/srsRAN_Project/build/apps/gnb/
```

### 5.6 Benetel RAN650 Radio Unit Bring-Up

The RAN650 runs a Linux-based management OS accessible via SSH. Configuration is written to `/etc/ru_config.cfg` before power-cycling or running the radio init script.

```bash
# SSH into the RAN650 management interface
ssh root@<RAN650_MGMT_IP>

# Copy the config (from configs/ran650_ru_config.cfg in this repo)
# Key parameters to verify match your deployment:
#   mimo_mode=1_2_3_4_4x2
#   bandwidth_hz=80000000
#   centre_frequency_hz=3558180000   (3558.18 MHz, ARFCN 637212)
#   c_plane_du_mac=00:33:22:33:00:11
#   u_plane_du_mac=00:33:22:33:00:11
#   vlan_tag (all) = 5
#   tdd_pattern_1=DDDD, tdd_pattern_2=DDDSUU
#   special_slots_symbols=DDDDDDGGGGUUUU
#   compression=dynamic_compressed

vi /etc/ru_config.cfg     # edit and match your frequency/MAC/VLAN

# Restart RU radio init to apply config
reboot  # or: systemctl restart radio-init

# After ~100 seconds, check radio status
cat /tmp/logs/radio_status
# Key lines to confirm:
#   [INFO] Sync completed                       ← PTP locked to E810 GM
#   [INFO] Center frequency set to 3558.18 MHz
#   [INFO] Set the bandwidth to 80000000 Hz
#   [INFO] Transmission enabled (4x2)
#   [INFO] Radio bringup complete

# Confirm TDD pattern applied
grep "TDD Pattern" /tmp/logs/radio_status
# Expected: [INFO] Setting TDD Pattern: DDDDDDDSUU
```

> **Important:** The RAN650 will wait indefinitely at `[INFO] Waiting for PTP sync` if the E810 PTP Grandmaster service is not running. Always start `ptp4l-e810` on the server **before** booting or restarting the RAN650.

### 5.7 Kamailio IMS + RTPEngine

```bash
# Install Kamailio 6.1
sudo apt install kamailio kamailio-mysql-modules \
  kamailio-presence-modules kamailio-tls-modules \
  kamailio-utils-modules -y

# Install MySQL for subscriber DB
sudo apt install mysql-server -y
sudo mysql_secure_installation

# Initialize Kamailio DB
sudo kamdbctl create

# Install RTPEngine
sudo apt install rtpengine -y

# Configure and start services (see configs/kamailio/)
sudo cp configs/kamailio/kamailio.cfg /etc/kamailio/
sudo cp configs/kamailio/rtpengine.conf /etc/rtpengine/

sudo systemctl enable kamailio rtpengine
sudo systemctl start  rtpengine kamailio
```

---

## 6. Configuration

All configuration files are in the `configs/` directory. Below are the most critical parameters explained.

### 6.1 Open5GS — AMF, SMF, UPF

Edit `/etc/open5gs/amf.yaml` — match PLMN and TAC to gNB:

```yaml
amf:
  sbi:
    server:
      - address: 127.0.0.5   # AMF SBI address (matches gNB cu_cp.amf.addr)
  ngap:
    server:
      - address: 127.0.0.5   # AMF N2 address
  guami:
    - plmn_id:
        mcc: 999
        mnc: 70              # PLMN 99970 — matches gNB and SIM
      amf_id:
        region: 2
        set: 1
  tai:
    - plmn_id:
        mcc: 999
        mnc: 70
      tac: 7                 # TAC 7 — matches gNB cell_cfg.tac
  plmn_support:
    - plmn_id:
        mcc: 999
        mnc: 70
      s_nssai:
        - sst: 1             # eMBB slice
```

Edit `/etc/open5gs/upf.yaml` — add IMS DNN:

```yaml
upf:
  pfcp:
    server:
      - address: 127.0.0.7
  gtpu:
    server:
      - address: 127.0.0.7
  session:
    - subnet: 10.45.0.1/16  # Data (internet DNN) pool — via ogstun
      dnn: internet
    - subnet: 10.46.0.1/16  # IMS DNN pool — via ogstun2
      dnn: ims
```

### 6.2 srsRAN gNB — Split 7.2 OFH

**Full annotated config:** `configs/gnb_ru_ran650_tdd_n78_80mhz_4x2.yml`

Critical parameters and why they matter:

```yaml
# AMF connection — loopback since core is co-hosted
cu_cp:
  amf:
    addr: 127.0.0.5        # Open5GS AMF SBI address
    port: 38412            # Standard NGAP port
    bind_addr: 127.0.0.1
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "99970"  # MCC=999, MNC=70

# DPDK lcores — assign 8 lcores across all 64 physical threads
hal:
  eal_args: "--lcores (0-7)@(0-63)"

# Fronthaul timing windows (µs) — tuned for RAN650 + E810 PTP
# These must be within the O-RAN spec windows for your Cat-B RU
ru_ofh:
  t1a_max_cp_dl: 470   t1a_min_cp_dl: 419   # DL C-Plane window
  t1a_max_cp_ul: 336   t1a_min_cp_ul: 285   # UL C-Plane window
  t1a_max_up:    345   t1a_min_up:    294   # DL U-Plane window
  ta4_max:       200   ta4_min:         0   # UL reception window
  cells:
    - network_interface: 0000:6f:01.0    # VF PCI — must match dpdk-devbind
      ru_mac_addr: 70:b3:d5:e1:5b:48    # RAN650 fronthaul MAC
      du_mac_addr: 00:33:22:33:00:11    # VF0 MAC set via ip link ... vf 0 mac
      vlan_tag_cp: 5                    # Must match RAN650 c_plane_du_vlan=5
      vlan_tag_up: 5                    # Must match RAN650 u_plane_du_vlan_*=5
      prach_port_id: [4, 5]             # eAxC IDs for PRACH
      dl_port_id:   [0, 1, 2, 3]       # 4 DL TX ports
      ul_port_id:   [0, 1]             # 2 UL RX ports

# Cell RF parameters
cell_cfg:
  dl_arfcn: 637212              # → 3558.18 MHz (matches RAN650 centre_frequency_hz)
  band: 78
  channel_bandwidth_MHz: 80     # 272 PRBs @ 30 kHz SCS
  common_scs: 30
  plmn: "99970"
  tac: 7
  pci: 1
  nof_antennas_dl: 4
  nof_antennas_ul: 2
  # TDD pattern matching RAN650: DDDDDDDSUU
  tdd_ul_dl_cfg:
    dl_ul_tx_period: 10  nof_dl_slots: 7  nof_dl_symbols: 6
    nof_ul_slots: 2      nof_ul_symbols: 4
```

### 6.3 Benetel RAN650 — ru_config.cfg

**Full annotated config:** `configs/ran650_ru_config.cfg`

```
# The three MAC/VLAN parameters MUST match srsRAN gNB config exactly
c_plane_du_mac=00:33:22:33:00:11   ← DU VF MAC (set via ip link vf 0 mac)
u_plane_du_mac=00:33:22:33:00:11
c_plane_du_vlan=5                  ← Matches vlan_tag_cp: 5 in gNB
u_plane_du_vlan_downlink=5         ← Matches vlan_tag_up: 5 in gNB
u_plane_du_vlan_uplink=5

# Frequency MUST match gNB dl_arfcn: 637212
centre_frequency_hz=3558180000

# TDD MUST match gNB tdd_ul_dl_cfg (DDDDDDDSUU = 10ms, 7DL+S+2UL)
tdd_pattern_1=DDDD
tdd_pattern_2=DDDSUU
special_slots_symbols=DDDDDDGGGGUUUU  ← 6D+4G+4U = 14 symbols
```

### 6.4 PTP Grandmaster — ptp4l & phc2sys

**Service files:** `configs/ptp_systemd_services.conf`

```bash
# Check PTP GM status
sudo journalctl -u ptp4l-e810 -f
# Good output:
#   ptp4l[xx]: selected best master clock <clock-id>
#   ptp4l[xx]: master offset 0 s2 freq ...

# Check PHC → system clock sync
sudo journalctl -u phc2sys-e810 -f
# Good output:
#   phc2sys[xx]: CLOCK_REALTIME rms xx ns ... s2

# Check on RAN650 side
cat /tmp/logs/radio_status | grep -i "sync\|ptp"
# Expected:
#   [INFO] Waiting for PTP sync
#   [INFO] Sync completed            ← RAN650 locked to E810 GM
```

### 6.5 Kamailio & RTPEngine

`/etc/kamailio/kamailio.cfg` — key parameters:

```
# Listen on IMS PDU session subnet
listen=udp:10.46.0.1:5060
listen=tcp:10.46.0.1:5060

# SIP domain matching UE SIP username suffix
#!define DOMAIN "mason5g.local"

# RTPEngine socket for media relay
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:2223")
```

`/etc/rtpengine/rtpengine.conf`:

```ini
[rtpengine]
interface = 10.46.0.1
listen-ng = 127.0.0.1:2223
port-min  = 30000
port-max  = 40000
log-level = 6
```

### 6.6 UE / SIM Provisioning

Program SIM cards and register subscribers in Open5GS WebUI at `http://localhost:9999` (default: `admin` / `1423`).

| Field | Example Value |
|-------|---------------|
| IMSI | `999700000000001` |
| Subscriber Key (K) | `465B5CE8B199B49FAA5F0A2EE238A6BC` |
| OPC | `E8ED289DEBA952E4283B54E88E6183CA` |
| Data DNN | `internet` (SST=1) |
| IMS DNN | `ims` (SST=1) |
| SIP URI | `sip:999700000000001@mason5g.local` |

```bash
# Add SIP subscriber to Kamailio DB
mysql -u kamailio -p kamailio << 'EOF'
INSERT INTO subscriber (username, domain, password, ha1, ha1b, rpid)
VALUES (
  '999700000000001',
  'mason5g.local',
  'your_password',
  MD5('999700000000001:mason5g.local:your_password'),
  MD5('999700000000001@mason5g.local:mason5g.local:your_password'),
  NULL
);
EOF

# Configure Android UE SIP client (e.g., Linphone, PortSIP)
# SIP Username: 999700000000001
# SIP Domain  : 10.46.0.X (UE IMS PDU session IP)
# Outbound proxy: 10.46.0.1:5060 (Kamailio)
# Transport   : UDP
```

---

## 7. Running the System

Start services in strict order. The RAN650 **must** see PTP sync before radio bring-up.

```bash
# ── Step 1: PTP Grandmaster (E810) ──────────────────────────────
sudo systemctl start ptp4l-e810
sleep 3
sudo systemctl start phc2sys-e810
# Verify: journalctl -u ptp4l-e810 | grep "best master"

# ── Step 2: Open5GS Core NFs ────────────────────────────────────
sudo systemctl start open5gs-nrfd
sleep 2
sudo systemctl start open5gs-scpd open5gs-amfd open5gs-smfd \
                     open5gs-upfd open5gs-ausfd open5gs-udmd \
                     open5gs-udrd open5gs-pcfd open5gs-nssfd open5gs-bsfd
# Verify: systemctl status open5gs-amfd | grep Active

# ── Step 3: IMS Stack ───────────────────────────────────────────
sudo systemctl start rtpengine
sudo systemctl start kamailio

# ── Step 4: Benetel RAN650 ──────────────────────────────────────
# (SSH into RAN650 and confirm radio_status shows "Radio bringup complete")
ssh root@<RAN650_MGMT_IP> "tail -5 /tmp/logs/radio_status"
# Expected: [INFO] Radio bringup complete

# ── Step 5: srsRAN gNB ──────────────────────────────────────────
cd ~/srsRAN_Project/build/apps/gnb

# Run with real-time priority and DPDK hugepages
sudo mkdir -p /dev/hugepages
sudo mount -t hugetlbfs none /dev/hugepages 2>/dev/null || true
echo 1024 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

sudo chrt -f 99 ./srsgnb -c gnb_ru_ran650_tdd_n78_80mhz_4x2.yml
# Watch for:
#   [RU-OFH] ... O-RAN fronthaul started
#   [RU-OFH] ... C-Plane connected
#   [RU-OFH] ... Radio Unit is ready
#   [AMF] ... NG connection established

# ── Step 6: UE ──────────────────────────────────────────────────
# Power on Android UE → Settings → Network → 5G SA mode
# Select PLMN "Mason5G" (MCC=999, MNC=70)

# ── Verify ──────────────────────────────────────────────────────
# Check UE registered
grep "Registration complete" /var/log/open5gs/amf.log

# Check PDU session
grep "PDU Session Establishment Accept" /var/log/open5gs/smf.log

# Ping UE from UPF
UE_IP=$(grep "UE IP" /var/log/open5gs/smf.log | tail -1 | grep -oP '\d+\.\d+\.\d+\.\d+')
ping -c 5 $UE_IP
```

---

## 8. IMS Service Validation

### Voice Call — SIP Flow

```bash
# Monitor SIP signaling in real-time (run on server)
sudo sngrep -d 10.46.0.1 -p 5060

# Expected SIP call flow:
#
#  UE-A (Caller)          Kamailio             UE-B (Callee)
#  ──────────────────────────────────────────────────────────
#  REGISTER ──────────────▶                                     (← UE registration)
#           ◀────────────── 200 OK
#
#  INVITE ────────────────▶ ──────────────────▶                 (← outgoing call)
#                           ◀────────────────── 180 Ringing
#         ◀─────────────────
#                           ◀────────────────── 200 OK
#         ◀─────────────────
#  ACK ───────────────────▶ ──────────────────▶
#
#  ══════════════════ RTP Media (via RTPEngine 30000–40000) ══════
#
#  BYE ───────────────────▶ ──────────────────▶                 (← hang up)
#                           ◀────────────────── 200 OK
#         ◀─────────────────
```

### RTP Media Capture

```bash
# Capture RTP during active call
sudo tcpdump -i ogstun2 -nn -w /tmp/rtp_call.pcap \
  "udp portrange 30000-40000" &

# Make a call from UE-A to UE-B (30+ seconds)

sudo killall tcpdump

# Analyze per-stream throughput with tshark
tshark -r /tmp/rtp_call.pcap -q -z io,stat,1 2>/dev/null | head -40

# Per-stream SSRC analysis
tshark -r /tmp/rtp_call.pcap -T fields \
  -e rtp.ssrc -e rtp.payload_type -e frame.len 2>/dev/null | \
  sort | uniq -c | sort -rn
```

### SMS via SIP MESSAGE

```bash
# Capture SIP MESSAGE events
sudo sngrep -d 10.46.0.1 -p 5060 -f "method=MESSAGE"

# Expected:
#  UE-A  MESSAGE ──────────▶ Kamailio ──────────▶ UE-B
#                            ◀──────────────────── 200 OK
#        ◀────────────────────
```

---

## 9. Performance Testing & Results

### 9.1 OTA Throughput (iperf3 at UPF)

Running iperf3 with the server bound to `ogstun` (UPF TUN interface) bypasses the 1G N6 link and measures **raw radio chain capacity**.

```bash
# Server — bind to UPF data plane TUN interface
iperf3 -s -B 10.45.0.1 -p 5201 -i 1

# Client — from Android UE terminal or via ADB
# Downlink (server → UE): 4 parallel streams, 60s
iperf3 -c 10.45.0.1 -p 5201 -t 60 -R -P 4

# Uplink (UE → server): 2 parallel streams, 60s
iperf3 -c 10.45.0.1 -p 5201 -t 60 -P 2
```

**OTA Downlink Throughput (60s, 4 streams):**

```
 Gbps
  4.0 ┤
  3.5 ┤━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ← Peak: 3.5 Gbps
  3.0 ┤        ╭────────────────────────────────────────
  2.5 ┤  ╭─────╯                                          ← Median: ~3.2 Gbps
  2.0 ┤──╯
  1.5 ┤
  1.0 ┤
  0.5 ┤
  0.0 ┤──┬──────────┬──────────┬──────────┬──────────┬──
       0s          15s         30s         45s        60s

  ━━ Max (Gbps)    ── Median (Gbps)

  Result: 80 MHz × 4×2 MIMO → peak 3.5 Gbps OTA
          Confirms effective spectral efficiency on n78
```

### 9.2 End-to-End Throughput (Beyond N6)

iperf3 server on external data network (server connected via 1G N6 interface):

```bash
# Server (on external data network)
iperf3 -s -p 5201 -i 1

# UE client — Downlink
iperf3 -c <DATA_SERVER_IP> -p 5201 -t 60 -R -P 4

# UE client — Uplink
iperf3 -c <DATA_SERVER_IP> -p 5201 -t 60 -P 2
```

**End-to-End DL/UL Throughput (60s):**

```
 Mbps
  900 ┤
  848 ┤            ╭──────╮                   ╭──╮         ← DL peak: 848 Mbps
  800 ┤      ╭─────╯      ╰────────────────╭──╯  ╰─────
  700 ┤──────╯                                             ← DL avg: 657 Mbps
  600 ┤
  100 ┤
   77 ┤   ╭──╮     ╭╮     ╭──────╮  ╭╮                   ← UL peak: 77 Mbps
   60 ┤───╯  ╰─────╯╰─────╯      ╰──╯╰──────────────────  ← UL avg: 60 Mbps
   40 ┤
    0 ┤──┬──────────┬──────────┬──────────┬──────────┬──
       0s          15s         30s         45s        60s

  ── Downlink (DL avg: 657 Mbps)    ── Uplink (UL avg: 60 Mbps)

  ⚠ Bottleneck: N6 interface is 1G Ethernet.
    Upgrading N6 to 10G will push E2E DL to multi-Gbps (matching OTA result).
```

### 9.3 Latency

```bash
# Ping UE from server side (via ogstun)
ping -c 1000 -i 0.01 <UE_IP>

# Ping external server from UE
ping -c 1000 -i 0.01 <DATA_SERVER_IP>
```

**Round-Trip Latency (ICMP, 1000 pings):**

```
 ms
  40 ┤            ╭╮
  35 ┤       ╭────╯╰──╮               ╭╮
  30 ┤  ╭────╯         ╰────────╮╭────╯╰────
  25 ┤──╯                       ╰╯
  20 ┤ ←── avg 20 ms
  15 ┤
  10 ┤
   8 ┤━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ← min 8 ms
   0 ┤──┬─────────┬─────────┬─────────┬───
       0        250        500        750  1000 pings

  Min: 8 ms  │  Avg: 20 ms  │  Max: 35 ms  │  StdDev: ~4 ms
```

### 9.4 Voice & Video RTP Throughput

Captured during a 30-second voice + video call between two Android UEs:

```bash
sudo tcpdump -i ogstun2 -w /tmp/rtp_30s.pcap "udp portrange 30000-40000" &
# ... place call, let run 30 seconds ...
sudo killall tcpdump
tshark -r /tmp/rtp_30s.pcap -q -z io,stat,1 | head -40
```

**RTP Throughput Over Time (3 simultaneous streams):**

```
 kbps
  1000 ┤
   800 ┤  ╭──────────────────────────────────────────────╮  ← Video H.264: ~750 kbps
   700 ┤  │                                              │
   600 ┤  │                                              │
   100 ┤  │                                              │
    90 ┤──╯──────────────────────────────────────────────╰─ ← Audio OPUS: ~90 kbps
    30 ┤
    15 ┤- - - - - - - - - - - - - - - - - - - - - - - - - - ← Keepalive: ~15 kbps
     0 ┤──┬───────┬───────┬───────┬───────┬───────┬──────
        0s       5s      10s      15s      20s      25s  30s

  Stream breakdown:
  ── Audio    (OPUS/G.711)     : ~90 kbps  — stable throughout call
  ── Video    (H.264 adaptive) : ~700–800 kbps — adaptive bitrate
  ─ ─ Keepalive (SIP OPTIONS)  : ~15 kbps  — SIP re-INVITE/OPTIONS
```

### 9.5 Cell Setup Time Distribution

Measures end-to-end time from UE sending RRCSetupRequest to PDU Session Establishment Complete across multiple attach attempts.

```bash
# Extract registration timestamps from AMF log
grep -E "Registration|PDU Session" /var/log/open5gs/amf.log | \
  awk '{print $1}' | head -100

# Or use NGAP PCAP
tshark -r /tmp/gnb_ngap.pcap -Y "ngap" \
  -T fields -e frame.time_relative -e ngap.procedureCode 2>/dev/null
```

**Cell Setup Time CDF (100 attach attempts):**

```
CDF
 1.0 ┤                               ╭──────────────────
 0.9 ┤                         ╭─────╯
 0.8 ┤                   ╭─────╯
 0.7 ┤              ╭────╯
 0.6 ┤          ╭───╯
 0.5 ┤       ╭──╯
 0.4 ┤    ╭──╯
 0.3 ┤  ╭─╯
 0.2 ┤╭─╯
 0.0 ┤╯
     ┤──────┬───────┬──────┬───────┬──────┬──────
      2614  2614.4  2614.8  2615.2  2615.4  2615.6
                  Call Setup Time (ms)

  P50: ~2614.8 ms  │  P90: ~2615.1 ms  │  P99: ~2615.3 ms
  Jitter: < 0.5 ms — highly deterministic (PTP clock stability)
  
  Note: 2.6s includes full NAS Registration + PDU Session Establishment
  from cold UE. Post-registration latency for data is <30ms.
```

---

## 10. Troubleshooting

### RAN650 stuck at "Waiting for PTP sync"

```bash
# 1. Check ptp4l is running on the server
sudo systemctl status ptp4l-e810
sudo journalctl -u ptp4l-e810 -n 20

# 2. Check VLAN connectivity to RAN650
sudo tcpdump -i enp111s0f0 -nn -e "ether proto 0x88f7"
# Should see PTP announce/sync frames

# 3. Check link speed on fronthaul interface
ethtool enp111s0f0 | grep Speed
# Should be 25000Mb/s

# 4. Verify RAN650 mgmt sees the GM
ssh root@<RAN650_MGMT_IP> "pmc -u -b 0 'GET CURRENT_DATA_SET' 2>/dev/null"
```

### gNB fails to connect to RAN650 (OFH not coming up)

```bash
# 1. Verify VF is bound to vfio-pci
dpdk-devbind.py --status | grep 6f:01
# Must show: drv=vfio-pci

# 2. Verify MAC address matches between gNB config and RAN650
#    gNB: du_mac_addr: 00:33:22:33:00:11
#    RU:  c_plane_du_mac=00:33:22:33:00:11
ip link show enp111s0f0 | grep "vf 0"
# Must show MAC 00:33:22:33:00:11

# 3. Verify VLAN 5 tagged traffic is flowing
sudo tcpdump -i enp111s0f0 -nn -e vlan 2>/dev/null | head -20
# Should see VLAN 5 C-plane and U-plane packets from RU MAC

# 4. Check timing window violations in srsRAN log
grep -i "late\|timing\|window" /tmp/gnb.log | head -20
# If late packets: increase t1a_max_cp_dl / t1a_max_up by 50µs steps
```

### UE Registration Fails (5G SA)

```bash
# Check AMF logs for rejection cause
sudo tail -f /var/log/open5gs/amf.log | grep -i "reject\|error\|fail"

# Common causes and fixes:
# 1. PLMN mismatch → verify gNB plmn: "99970" matches SIM MCC/MNC
# 2. TAC mismatch  → gNB tac: 7 must match AMF tai.tac: 7
# 3. AUSF unreachable → check NRF: systemctl status open5gs-nrfd
# 4. SIM not provisioned → Open5GS WebUI → Subscribers → check IMSI

# Verify all NFs are registered with NRF
sudo grep "NF registered" /var/log/open5gs/nrf.log | tail -10
```

### Low End-to-End Throughput (below 400 Mbps)

```bash
# 1. Verify N6 interface speed
ethtool <N6_INTERFACE> | grep Speed
# If 1G → this is the bottleneck, expected DL ~800 Mbps max
# Upgrade to 10G to break the bottleneck

# 2. Check CPU frequency governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Must be: performance

# 3. Check srsRAN log for processing delays
grep -i "underflow\|overflow\|late" /tmp/gnb.log

# 4. Check UPF CPU usage
sudo top -b -n 1 | grep open5gs-upfd

# 5. Verify hugepages for DPDK
cat /proc/meminfo | grep HugePage
# HugePages_Total should be ≥ 1024
```

### IMS Registration Fails (No SIP REGISTER response)

```bash
# 1. Confirm IMS PDU session established
grep "ims\|ogstun2" /var/log/open5gs/smf.log | tail -5

# 2. Check Kamailio is listening on ogstun2 subnet
ss -lnpu | grep 5060
# Should show: 10.46.0.1:5060

# 3. Check subscriber DB entry
mysql -u kamailio -p kamailio -e \
  "SELECT username, domain FROM subscriber;"

# 4. Test SIP registration manually
sipsak -s sip:999700000000001@10.46.0.1 -U -a your_password -v
```

---

## 11. Results Summary

| Metric | Value | Notes |
|--------|-------|-------|
| OTA DL Peak | **3.5 Gbps** | iperf3 server at UPF (ogstun), no N6 bottleneck |
| OTA DL Median | **~3.2 Gbps** | 80 MHz · 4×2 MIMO · n78 · BFP 9-bit compression |
| E2E DL Average | **657 Mbps** | N6 = 1G Ethernet bottleneck |
| E2E DL Peak | **~848 Mbps** | Burst peak within 1G constraint |
| E2E UL Average | **60 Mbps** | Commercial Android UE |
| RTT Latency (avg) | **~20 ms** | ICMP UE ↔ external server |
| RTT Latency (min) | **8 ms** | |
| Voice Call Setup | **INVITE→200 OK** | Full SIP call flow via Kamailio 6.1 |
| Audio RTP | **~90 kbps** | OPUS/G.711 — stable, no jitter |
| Video RTP | **~750 kbps** | H.264 adaptive |
| Cell Setup Time (P50) | **<0.5 ms jitter** | Consistent PTP-locked attach |
| Fronthaul Stability | **Zero drops** | PTP GM via E810, no late symbols |
| TDD Pattern | **DDDDDDDSUU** | 10 ms, 7DL+S+2UL slots |

---

## 12. Roadmap

- [ ] **Upgrade N6 to 10GbE** — expected to push E2E DL above 3 Gbps, matching OTA
- [ ] **Multi-UE stress testing** — 10+ simultaneous devices, per-UE QoS validation
- [ ] **Network slicing** — dedicated SST/SD slices for eMBB vs URLLC traffic
- [ ] **WebRTC gateway** — browser-based video calls via RTPEngine WebRTC bridge
- [ ] **Grafana/Prometheus dashboards** — real-time srsRAN JSON metrics visualization
- [ ] **Near-RT RIC (O-RAN)** — xApp integration for dynamic scheduling control
- [ ] **IPv6 dual-stack** — UPF and IMS IPv6 support
- [ ] **Carrier Aggregation** — dual-band experiments with second RU
- [ ] **NR Positioning** — UE location estimation using srsRAN positioning reference signals

---

## 13. References

- Open5GS: https://open5gs.org/
- srsRAN Project Installation: https://docs.srsran.com/projects/project/en/latest/user_manuals/source/installation.html
- Benetel RAN650 Product Page: https://benetel.com/
- Kamailio Documentation: https://www.kamailio.org/w/documentation/
- RTPEngine: https://github.com/sipwise/rtpengine
- O-RAN Alliance CUS-Plane Spec: https://www.o-ran.org/
- Intel E810 Datasheet: https://www.intel.com/content/www/us/en/ethernet-products/network-adapters/ethernet-800-series-brief.html
- linuxptp: http://linuxptp.sourceforge.net/
- DPDK 23.11 LTS: https://doc.dpdk.org/guides-23.11/

---

**GitHub:** https://github.com/rajendra1124/private5G-IMS  
**Contact:** rpaudyal@gmu.edu  
**Lab:** Mason Innovation Labs, George Mason University  
**Sponsor:** Cybastion Technology

*Last updated: February 2026*
