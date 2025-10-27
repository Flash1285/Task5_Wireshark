# 🦈 Network Traffic Analysis with Wireshark and tcpdump

This repository documents a step-by-step laboratory exercise focused on capturing, filtering, and analyzing network traffic using tools like **Wireshark**, **tshark**, and **tcpdump**.

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

***

## 📥 Wireshark Download Links

| Platform | Installer Link |
| :--- | :--- |
| **Windows x64 Installer** | [Wireshark-4.6.0-x64.exe](https://2.na.dl.wireshark.org/win64/Wireshark-4.6.0-x64.exe) |
| **Windows Arm64 Installer** | [Wireshark-4.6.0-arm64.exe](https://2.na.dl.wireshark.org/win64/Wireshark-4.6.0-arm64.exe) |
| **Windows x64 Portable** | [WiresharkPortable64_4.6.0.paf.exe](https://2.na.dl.wireshark.org/win64/WiresharkPortable64_4.6.0.paf.exe) |
| **macOS Universal** | [Wireshark 4.6.0.dmg](https://2.na.dl.wireshark.org/osx/Wireshark%204.6.0.dmg) |
| **Ubuntu (PPA)** | [Launchpad PPA](https://launchpad.net/~wireshark-dev/+archive/ubuntu/stable) |
| **Source Code** | [wireshark-4.6.0.tar.xz](https://2.na.dl.wireshark.org/src/wireshark-4.6.0.tar.xz) |

***

## ⚙️ Phase-Wise Network Capture & Analysis

### Phase 1 — Install & Prepare Wireshark (Linux One-Time Setup)

Install the necessary packet capture tools:
```
sudo apt update 
sudo apt install -y wireshark tshark tcpdump
```

Grant non-root packet-capture capability (recommended for safety):
#enable dumpcap capabilities so non-root users in group 'wireshark' can capture
```
sudo dpkg-reconfigure wireshark-common 
```
select 'Yes'
```
sudo usermod -aG wireshark $USER
```
Either log out/in or run:
```
newgrp wireshark
```

*Alternative method using `setcap`:*
```
sudo setcap cap_net_raw,cap_net_admin+eip /usr/bin/dumpcap
```

### Phase 2 — Find Your Active Interface

Determine the network interface connected to the internet (e.g., `eth0`, `wlan0`, `ens33`).

**Method A (Ask the kernel):**
```
ip route get 8.8.8.8 | awk '{print $5; exit}'
```

**Method B (List interfaces):**
```
ip link show
```
or
```
ifconfig -a
```

### Phase 3 — Start Capturing (GUI and CLI Options)

#### GUI (Wireshark)

1.  Start Wireshark:
    ```
    wireshark &
    ```
2.  In the Wireshark window, select the interface found in Phase 2 and click the shark-fin icon (Start).
3.  Generate traffic (Phase 4).
4.  After ~60s, click the red square (Stop).
5.  Save the file: `File` → `Save As` → `capture.pcap`.

#### CLI (Recommended for Reproducible Lab Work)

**Automatic interface detection and 60s capture:**
```
IF=$(ip route get 8.8.8.8 | awk '{print $5; exit}')
timeout 60 tcpdump -i "$IF" -w ~/capture.pcap
```

Using tshark
```
timeout 60 tshark -i "$IF" -w ~/capture.pcap
```
*Note: `timeout 60` stops the capture after 60 seconds. File is saved as `~/capture.pcap`.*

**Fixed number of packets:**
```
tcpdump -i "$IF" -c 500 -w ~/capture.pcap
```

### Phase 4 — Generate Traffic (While Capture Running)

From another terminal or browser, execute the following during the 60-second capture window:

Browse a few pages in a browser
```
ping -c 4 google.com 
curl -I https://www.google.com
```

*This traffic will produce **ICMP**, **DNS**, **TCP**, **HTTP/HTTPS**, and potentially **TLS** traffic.*

### Phase 5 — Filter Captured Packets

Open `capture.pcap` in Wireshark or use `tshark`/`tcpdump` to apply filters.

| Common Wireshark Display Filters | Description |
| :--- | :--- |
| `http` | HTTP requests/responses |
| `dns` | DNS queries/responses |
| `tcp` | All TCP segments |
| `icmp` | Ping echo/request/reply |
| `tls` or `ssl` | TLS handshakes/encrypted sessions |

**CLI Examples:**
show DNS packets
```
tshark -r ~/capture.pcap -Y dns
```
show HTTP packets
```
tshark -r ~/capture.pcap -Y http
```
show ICMP
```
tshark -r ~/capture.pcap -Y icmp
```

**To extract the top protocols seen (Protocol Hierarchy Summary):**
```
tshark -r ~/capture.pcap -q -z io,phs,0,ip
```
*(If the `-z` command errors on your tshark version, use Wireshark GUI → Statistics → Protocol Hierarchy.)*

### Phase 6 — Identify at Least 3 Protocols (Proof)

Identify at least three distinct protocols in your capture and use `tshark` to provide evidence.

| Protocol | Purpose | Command to Prove It |
| :--- | :--- | :--- |
| **DNS** | Name resolution (used before HTTP) | `tshark -r ~/capture.pcap -Y dns -T fields -e frame.number -e dns.qry.name -e ip.src -e ip.dst | head` |
| **TCP** | TCP handshake (SYN / SYN-ACK / ACK) | `tshark -r ~/capture.pcap -Y "tcp.flags.syn==1 && tcp.flags.ack==0" -T fields -e frame.number -e ip.src -e ip.dst -e tcp.flags` |
| **HTTP (GETs)** | Unencrypted web requests | `tshark -r ~/capture.pcap -Y http.request.method -T fields -e http.request.method -e http.host -e http.request.uri` |
| **TLS** | Encrypted handshake (ClientHello) | `tshark -r ~/capture.pcap -Y "tls.handshake.type==1" -T fields -e ip.src -e ip.dst -e tls.handshake.extensions_server_name` |
| **ICMP** | Ping traffic | `tshark -r ~/capture.pcap -Y icmp -T fields -e frame.number -e icmp.type -e ip.src -e ip.dst` |

### Phase 7 — Export / Save .pcap

If you used the CLI, the capture is already saved as `~/capture.pcap`.

**To make a copy with only specific protocols (e.g., HTTP/DNS):**
```
tshark -r ~/capture.pcap -Y "http or dns" -w ~/http_dns_only.pcap
```

