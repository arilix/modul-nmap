# BAB 09 — Evading Modern EDR, NGFW & IDS

> **NMAP Advanced Guide · 2026 Edition**

---

## Landscape Pertahanan Modern

Berbeda dengan firewall tradisional yang hanya melihat port dan IP, pertahanan modern beroperasi di beberapa layer:

| Komponen | Cara Mendeteksi | Apa yang Dicegah |
|----------|-----------------|------------------|
| **NGFW** (Palo Alto, Fortinet) | Deep Packet Inspection, App-ID | L7 traffic analysis, TLS inspection |
| **IDS/IPS** (Snort, Suricata) | Signature matching, anomaly | Known scan patterns, rate anomaly |
| **EDR** (CrowdStrike, Defender) | Syscall monitoring, API hooking | Process behavior, network anomaly |
| **SIEM** (Splunk, Elastic) | Log aggregation, correlation | Multi-stage attack patterns |
| **NDR** (Darktrace, ExtraHop) | ML behavioral baseline | Deviation dari baseline |

---

## Timing-Based Evasion

### Melawan Threshold-Based Deteksi

IDS menggunakan threshold seperti "lebih dari 100 port probes dalam 1 detik = scan".

```bash
# T0: Paranoid — 5 menit antar probe
# Tidak ada paralelisme, evade hampir semua rate-based IDS
sudo nmap -T0 -sS 192.168.1.1

# T1: Sneaky — 15 detik antar probe
sudo nmap -T1 -sS -p 22,80,443 192.168.1.1

# Custom timing — kontrol presisi:
# --min-rtt-timeout / --max-rtt-timeout (response timeout)
# --initial-rtt-timeout (first probe timeout)
# --host-timeout (total time per host)
# --scan-delay (minimum delay antar probes)
# --max-scan-delay (maximum delay antar probes)
# --min-rate / --max-rate (packet rate limit)

sudo nmap --scan-delay 5s --max-rate 1 \
     -p 20-25,80,443,3389 192.168.1.1
```

### Jittter untuk Bypass ML-Based Anomaly Detection

```bash
# Sistem ML belajar pola timing yang konsisten
# Tambahkan variasi acak pada timing

# Bash wrapper dengan random sleep:
for port in 22 80 443 3389 8080; do
    sudo nmap -sS -p $port --open 192.168.1.1 &
    sleep $((RANDOM % 30 + 5))  # Tunggu 5-35 detik secara acak
done
wait
```

---

## Fragmentation & MTU Manipulation

```bash
# Fragment paket menjadi 8-byte chunks
# Banyak IDS tidak reassemble sebelum inspect
sudo nmap -f 192.168.1.1

# Double fragment (lebih kecil lagi = 16 byte)
sudo nmap -ff 192.168.1.1

# Set MTU custom (harus kelipatan 8)
sudo nmap --mtu 16 192.168.1.1
sudo nmap --mtu 24 -sS 192.168.1.1

# Kombinasi dengan decoy
sudo nmap -f -D RND:5 192.168.1.1
```

---

## Decoy Scanning

Decoy membuat scan terlihat berasal dari banyak sumber, mempersulit attribution.

```bash
# Gunakan IP acak sebagai decoy
sudo nmap -D RND:10 192.168.1.1

# Gunakan IP decoy spesifik (real atau fiktif)
# Kamu harus ada di antara decoy (gunakan ME)
sudo nmap -D 10.0.0.1,10.0.0.2,ME,10.0.0.3 192.168.1.1

# Decoy dengan source port tertentu
sudo nmap -D RND:5 -g 53 192.168.1.1

# PENTING: Decoy hanya efektif terhadap non-stateful analysis
# Tool seperti CrowdStrike bisa tetap korelasi meski dengan decoys
```

---

## Source Port Manipulation

```bash
# Beberapa firewall mengizinkan traffic dari source port "trusted"
# DNS (53), HTTP (80), HTTPS (443), FTP (20) sering di-allowlist

# Source port 53 (DNS) - sering diizinkan melewati firewall
sudo nmap -g 53 192.168.1.1
sudo nmap --source-port 53 -sS 192.168.1.1

# Source port 80
sudo nmap -g 80 -sS 192.168.1.1

# Source port 443 (paling efektif karena HTTPS traffic dianggap aman)
sudo nmap -g 443 192.168.1.1
```

---

## Packet Padding & Data Obfuscation

```bash
# Tambahkan random data ke paket untuk mengacaukan signatures
sudo nmap --data-length 50 192.168.1.1   # Tambah 50 byte random
sudo nmap --data-length 100 192.168.1.1  # Tambah 100 byte random

# Kirim string spesifik sebagai payload
sudo nmap --data-string "GET / HTTP/1.0\r\n\r\n" 192.168.1.1

# Hex data
sudo nmap --data 0xdeadbeef 192.168.1.1

# Kombinasi: fragment + padding + decoy + source port
sudo nmap -f --data-length 50 -D RND:5 -g 443 \
     --scan-delay 2s -T1 192.168.1.1
```

---

## IP Spoofing & MAC Spoofing

```bash
# Spoof source IP (tapi response tidak kembali ke kamu - hanya berguna untuk DoS/recon)
sudo nmap -S 10.10.10.10 -e eth0 192.168.1.1

# Spoof MAC address - berguna di network yang filter via MAC
sudo nmap --spoof-mac 0 192.168.1.0/24        # Random MAC
sudo nmap --spoof-mac AA:BB:CC:DD:EE:FF 192.168.1.1  # Specific MAC
sudo nmap --spoof-mac Apple 192.168.1.1         # Vendor prefix lookup

# Kombinasi dengan interface specification
sudo nmap -e eth0 --spoof-mac Cisco 192.168.1.1
```

---

## Bypassing TLS/SSL Inspection (NGFW)

NGFW modern seperti Palo Alto melakukan TLS inspection. Cara Nmap sering terdeteksi:
1. JA3 fingerprint Nmap dikenal
2. Cipher order abnormal
3. TLS extensions tidak normal

```bash
# Gunakan proxychains untuk route melalui proxy
# dengan TLS stack yang berbeda
proxychains nmap -sV -p 443 target.com

# Atau gunakan stunnel sebagai TLS wrapper
# yang menggunakan cipher suite normal browser

# Cek apakah NGFW melakukan intercept (cert berbeda)
nmap --script ssl-cert -p 443 target.com | grep "Issuer"
# Jika issuer adalah "Palo Alto" atau "Fortinet" = TLS inspection aktif
```

---

## Evading Snort/Suricata Signatures

Snort rules sering match pada:
- TCP options tertentu (OS fingerprint probes)
- Nmap probe payloads (service detection strings)
- Scan rate

```bash
# Disable OS detection (mengurangi suspicious probes)
nmap -sS -sV --no-os-probe 192.168.1.1

# Gunakan hanya versi detection intensitas rendah
nmap -sV --version-intensity 2 192.168.1.1

# Randomize scan order (hindari sequential port scan signature)
nmap --randomize-hosts -iL targets.txt
nmap -r -  # --no-random untuk sequential (sering LEBIH evasive karena tidak seperti scan)

# Bad checksum untuk bypass beberapa IDS (tidak semua)
nmap --badsum 192.168.1.1
# IDS yang tidak check checksum: paket diterima
# Real host: paket diabaikan
# Berguna: jika SEMUA port terlihat filtered dengan --badsum = ada IDS
```

---

## Passive Recon (Zero Scan Traffic)

```bash
# p0f - passive OS fingerprinting (hanya listen, tidak kirim)
sudo p0f -i eth0 -w /tmp/p0f.log

# Scan dengan hanya menganalisis traffic yang sudah ada
# Berguna di segmen yang kamu sudah memiliki akses

# Zeek (Bro) untuk passive asset discovery
zeek -i eth0 /opt/zeek/share/zeek/policy/misc/scan.zeek

# Wireshark/tcpdump passive recon
sudo tcpdump -i eth0 -w capture.pcap
tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e tcp.dstport |
sort | uniq -c | sort -rn | head -20
```

---

## Distributed Scanning untuk Evade Rate Limiting

```bash
# Bagi scan responsibility antar beberapa host
# Masing-masing host scan subset port

# Host A (pentest-vm-1.lab):
nmap -sS -p 1-10000 192.168.1.0/24 -oX scan_part1.xml

# Host B (pentest-vm-2.lab):
nmap -sS -p 10001-20000 192.168.1.0/24 -oX scan_part2.xml

# Host C (pentest-vm-3.lab):
nmap -sS -p 20001-65535 192.168.1.0/24 -oX scan_part3.xml

# Merge hasil dengan ndiff atau python
python3 << 'EOF'
import xml.etree.ElementTree as ET

def merge_nmap_xml(files, output):
    base = ET.parse(files[0])
    base_root = base.getroot()
    
    for f in files[1:]:
        tree = ET.parse(f)
        for host in tree.findall('host'):
            # Merge ports dari scan berbeda ke host yang sama
            ip = host.find('.//address[@addrtype="ipv4"]').get('addr')
            base_host = base_root.find(f'.//host[address[@addr="{ip}"]]')
            if base_host:
                for port in host.findall('.//port'):
                    base_ports = base_host.find('ports')
                    if base_ports is None:
                        base_host.append(ET.Element('ports'))
                        base_ports = base_host.find('ports')
                    base_ports.append(port)
    
    base.write(output)

merge_nmap_xml(['scan_part1.xml', 'scan_part2.xml', 'scan_part3.xml'], 'merged.xml')
EOF
```

---

## Deteksi "Apakah Kamu Terdeteksi?"

```bash
# Check apakah IP kamu di-block setelah scan
# Sebelum scan: catat RTT baseline
ping -c 10 192.168.1.1 | tail -1

# Setelah scan: jika latency naik drastis atau 100% loss = terkena ACL
ping -c 5 192.168.1.1

# Gunakan traceroute untuk lihat apakah ada drop point baru
traceroute 192.168.1.1

# Jika ada firewall yang blacklist IP:
# Tunggu TTL blacklist (biasanya 1-24 jam)
# Atau rotasi IP source
```

---

## Kombinasi Teknik Evasion

```bash
# Full evasion scan (slow, stealthy):
sudo nmap \
    -sS \                           # SYN scan (half-open)
    -T1 \                           # Sneaky timing
    --scan-delay 10s \              # 10 detik antar probes
    --max-rate 5 \                  # Maks 5 packet/detik
    -f \                            # Fragment paket
    --data-length 50 \              # Random padding
    -D RND:5 \                      # 5 random decoys
    -g 53 \                         # Source port 53
    --randomize-hosts \             # Acak urutan hosts
    --spoof-mac 0 \                 # Random MAC
    --reason \                      # Show why ports are in state
    -p 22,80,443,8080,3389 \
    192.168.1.0/24

# Note: Tidak ada teknik yang 100% guarantee evasion
# Defense-in-depth dari defender juga berlaku
```

---

*← [BAB 08: ICS/SCADA & IoT](bab08.md) · Selanjutnya → [BAB 10: Automation Pipeline](bab10.md)*
