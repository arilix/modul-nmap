# BAB 04 — Protocol-Level Packet Crafting dengan Nmap

> **NMAP Advanced Guide · 2026 Edition**

---

## Mengapa Packet Crafting?

Di level advanced, kamu tidak hanya membaca respons — kamu **merancang paket secara presisi** untuk:
- Menguji implementasi protokol non-standar
- Bypass filtering berdasarkan field header tertentu
- Fingerprint perangkat dari reaksinya terhadap paket malformed
- Riset kerentanan protocol-level

---

## NSE `packet` Library: Raw Packet Crafting

```lua
local packet = require "packet"
local nmap = require "nmap"

-- Craft raw IP+TCP packet
local tcp_pkt = packet.Frame:new()

-- Set IP header
tcp_pkt:ip_set_src(nmap.get_interface_info(nmap.get_interface()).address)
tcp_pkt:ip_set_dst(host.ip)
tcp_pkt:ip_set_ttl(128)
tcp_pkt:ip_set_df(true)

-- Set TCP header
tcp_pkt:tcp_set_sport(math.random(1024, 65535))
tcp_pkt:tcp_set_dport(port.number)
tcp_pkt:tcp_set_flags(packet.TH_SYN)
tcp_pkt:tcp_set_window(65535)
tcp_pkt:tcp_set_seq(math.random(0, 0xFFFFFFFF))

-- Set TCP options: MSS + SACK_PERM + NOP + WS
tcp_pkt:tcp_set_mss(1460)
tcp_pkt:tcp_parse_options("\x02\x04\x05\xb4" ..  -- MSS 1460
                           "\x04\x02" ..          -- SACK_PERM
                           "\x01" ..              -- NOP
                           "\x03\x03\x07")        -- Window Scale 7

-- Finalize (calculate checksums)
tcp_pkt:ip_set_len(tcp_pkt:raw():len())
tcp_pkt:ip_count_checksum()
tcp_pkt:tcp_count_checksum()

-- Kirim via dnet
local dnet = nmap.new_dnet()
dnet:ip_open()
dnet:ip_send(tcp_pkt:raw())
dnet:ip_close()
```

---

## Memahami TCP Options dalam Konteks Fingerprinting

TCP options adalah field penting untuk OS fingerprinting. Berbeda OS mengirim kombinasi options yang berbeda:

```
Linux:     MSS + SACK_PERM + Timestamps + NOP + WS
Windows:   MSS + NOP + WS + SACK_PERM + Timestamps
macOS:     MSS + NOP + WS + SACK_PERM + Timestamps + EOL
OpenBSD:   MSS + SACK_PERM + Timestamps + WS
Cisco IOS: MSS only (atau MSS + WS)
```

```bash
# Lihat packet-level detail saat scan
sudo nmap --packet-trace -O 192.168.1.1 2>&1 | grep -A 3 "SENT\|RCVD"

# Capture dan analisis dengan scapy
sudo python3 -c "
from scapy.all import *
pkt = IP(dst='192.168.1.1')/TCP(dport=80, flags='S', options=[('MSS',1460),('SAckOK',''),('Timestamp',(0,0)),('NOP',None),('WScale',7)])
resp = sr1(pkt, timeout=2, verbose=0)
if resp:
    print(resp[TCP].options)
"
```

---

## ICMP Crafting untuk Network Recon

```bash
# Kirim ICMP dengan Type/Code berbeda untuk bypass firewall
# Beberapa firewall memblokir ICMP Echo (type 8) tapi lewatkan type lain

# ICMP Timestamp (type 13)
sudo nmap -PE --packet-trace 192.168.1.1 2>&1 | head -20

# ICMP Address Mask (type 17) - meminta subnet mask dari host
sudo nmap -PP 192.168.1.1

# ICMP Info Request (type 15) - jarang digunakan tapi kadang tidak difilter
sudo nmap --script icmp-info 192.168.1.0/24
```

### Python/Scapy untuk ICMP Custom

```python
from scapy.all import *

# ICMP dengan TTL exceeded (type 11) - simulasi traceroute
def custom_traceroute(target, max_hops=30):
    for ttl in range(1, max_hops + 1):
        pkt = IP(dst=target, ttl=ttl) / ICMP()
        resp = sr1(pkt, timeout=2, verbose=0)
        
        if resp is None:
            print(f"{ttl}: * (filtered)")
        elif resp.type == 11:  # Time Exceeded
            print(f"{ttl}: {resp.src} (intermediate hop)")
        elif resp.type == 0:   # Echo Reply
            print(f"{ttl}: {resp.src} (destination!)")
            break
```

---

## TCP State Machine Exploitation

### Half-Open & Incomplete Handshakes

```lua
-- NSE script: kirim SYN+FIN bersamaan (invalid tapi informatif)
local packet = require "packet"

local pkt = packet.Frame:new()
-- Set SYN + FIN flags bersamaan
pkt:tcp_set_flags(packet.TH_SYN | packet.TH_FIN)
-- Beberapa OS merespons, yang lain tidak -- berguna untuk fingerprinting
```

```bash
# Cek respons terhadap invalid flag combinations via Nmap
# Null scan (no flags): closed ports kirim RST
sudo nmap -sN -p 80,22 192.168.1.1 --packet-trace

# Xmas scan (FIN+PSH+URG): closed ports kirim RST
sudo nmap -sX -p 80,22 192.168.1.1 --packet-trace

# Maimon scan (FIN+ACK): Beberapa BSD drop paket ini
sudo nmap -sM -p 80,22 192.168.1.1 --packet-trace
```

---

## UDP Crafting untuk Service Probing

### Custom UDP Payloads

```lua
local nmap = require "nmap"

-- Kirim UDP payload custom
local socket = nmap.new_socket("udp")
socket:connect(host.ip, port.number, "udp")

-- DNS Query yang dicraft manual
local dns_query = "\xAA\xBB" ..  -- Transaction ID
                  "\x01\x00" ..  -- Flags: Standard query
                  "\x00\x01" ..  -- Questions: 1
                  "\x00\x00\x00\x00\x00\x00" ..  -- 0 answers, ns, additional
                  "\x07example\x03com\x00" ..  -- example.com
                  "\x00\x01\x00\x01"  -- Type A, Class IN

socket:send(dns_query)
local status, data = socket:receive_bytes(512)
```

### SNMP Crafting

```python
from scapy.all import *

# SNMP v1 GET request (community string brute force)
communities = ["public", "private", "community", "manager", "admin"]

for community in communities:
    pkt = IP(dst="192.168.1.1")/UDP(dport=161)/SNMP(
        version=1,
        community=community.encode(),
        PDU=SNMPget(varbindlist=[SNMPvarbind(oid=ASN1_OID("1.3.6.1.2.1.1.1.0"))])
    )
    resp = sr1(pkt, timeout=2, verbose=0)
    if resp and resp.haslayer(SNMP):
        print(f"Valid community: {community}")
        print(resp.sprintf("%SNMP.PDU.varbindlist%"))
```

---

## Fragmentation Attack & Bypass

### Tiny Fragment Attack

```bash
# Fragment paket sedemikian kecil sehingga TCP header terbagi
# Beberapa firewall lama gagal reassemble dengan benar

# Fragment 8 byte (minimum)
sudo nmap -f -sS 192.168.1.1

# Overlap fragments (lanjut - memerlukan scapy)
```

### Fragment Overlap dengan Scapy

```python
from scapy.all import *
import random

# Kirim dua fragment yang saling overlap
# Fragment 1: SYN packet yang valid ke port "aman" (80)
# Fragment 2: Overlap yang mengubah destination port ke target sebenarnya

dst = "192.168.1.1"
sport = random.randint(1024, 65535)
seq = random.randint(0, 0xFFFFFFFF)

# Build inner TCP packet
inner_tcp = TCP(sport=sport, dport=80, flags="S", seq=seq)
inner_pkt = IP(dst=dst, id=1234) / inner_tcp

# Fragment dengan overlap
frags = fragment(inner_pkt, fragsize=8)

# Modifikasi fragment kedua untuk mengubah destination port
# (Teknik ini untuk bypass stateless packet filter)
send(frags, verbose=0)
```

---

## Analisis Raw Packet dengan `--packet-trace`

```bash
# Setiap paket yang dikirim dan diterima ditampilkan
sudo nmap --packet-trace -sS -p 22,80,443 192.168.1.1 2>&1

# Output format:
# SENT (0.0621s) TCP 192.168.1.100:49832 > 192.168.1.1:22 S ttl=49 id=57910 ...
# RCVD (0.0635s) TCP 192.168.1.1:22 > 192.168.1.100:49832 SA ttl=64 id=0 ...
#   SA = SYN+ACK (port open)
#   RA = RST+ACK (port closed)
#   (nothing) = timeout (port filtered)

# Filter hanya SENT atau RCVD
sudo nmap --packet-trace -sS 192.168.1.1 2>&1 | grep "^SENT"
sudo nmap --packet-trace -sS 192.168.1.1 2>&1 | grep "^RCVD"
```

---

## Scapy + Nmap: Workflow Hybrid

```python
from scapy.all import *
import subprocess
import xml.etree.ElementTree as ET

def hybrid_scan(target):
    """
    Fase 1: Scapy untuk low-level probe
    Fase 2: Nmap untuk version detection pada port ditemukan
    """
    open_ports = []
    
    # Fase 1: Custom SYN scan dengan Scapy
    ans, unans = sr(
        IP(dst=target)/TCP(dport=[21,22,23,25,80,443,8080,8443],
                           flags="S",
                           seq=1000),
        timeout=3,
        verbose=0
    )
    
    for sent, received in ans:
        if received.haslayer(TCP) and received[TCP].flags == 0x12:  # SYN-ACK
            port = received[TCP].sport
            open_ports.append(port)
            # Kirim RST untuk tutup koneksi
            send(IP(dst=target)/TCP(dport=port, sport=sent[TCP].sport,
                                    flags="R", seq=1001), verbose=0)
    
    print(f"Scapy found open ports: {open_ports}")
    
    if not open_ports:
        return
    
    # Fase 2: Nmap version detection pada port yang ditemukan
    ports_str = ",".join(map(str, open_ports))
    result = subprocess.run(
        ["nmap", "-sV", "-sC", f"-p{ports_str}", target, "-oX", "/tmp/nmap_out.xml"],
        capture_output=True
    )
    
    # Parse hasil
    tree = ET.parse("/tmp/nmap_out.xml")
    for port_elem in tree.findall('.//port[@protocol="tcp"]'):
        if port_elem.find('state').get('state') == 'open':
            portid = port_elem.get('portid')
            svc = port_elem.find('service')
            if svc is not None:
                print(f"Port {portid}: {svc.get('product','')} {svc.get('version','')}")

hybrid_scan("192.168.1.1")
```

---

*← [BAB 03: Advanced NSE Lua](bab03.md) · Selanjutnya → [BAB 05: Attacking Encrypted Services](bab05.md)*
