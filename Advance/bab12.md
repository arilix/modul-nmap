# BAB 12 — Performance Tuning & Internals

> **NMAP Advanced Guide · 2026 Edition**

---

## Bottleneck Analysis: Di Mana Nmap Lambat?

```bash
# Timing analysis dengan -d flag
sudo nmap -d2 -sS -p 1-1000 192.168.1.0/24 2>&1 | grep -i "timing\|rate\|timeout\|cwnd"

# Gunakan --stats-every untuk monitoring real-time
sudo nmap -sS -sV 192.168.1.0/24 --stats-every 5s

# Output setiap 5 detik:
# Stats: 0:00:05 elapsed; 0 hosts completed (5 up), 1 undergoing SYN Stealth Scan
# SYN Stealth Scan Timing: About 23.45% done; ETC: 15:32 (0:00:16 remaining)
```

---

## Timing Template Deep Dive

| Parameter | T0 | T1 | T2 | T3 | T4 | T5 |
|-----------|----|----|----|----|----|----|
| min-hostgroup | 1 | 1 | 1 | 1 | 1 | 1 |
| max-hostgroup | 10 | 20 | 30 | 30 | 50 | 256 |
| min-parallelism | - | - | - | - | - | - |
| max-parallelism | 1 | 1 | - | - | - | - |
| min-rtt-timeout | 100ms | 100ms | 100ms | 100ms | 100ms | 50ms |
| max-rtt-timeout | 5m | 15s | 10s | 8s | 1.25s | 300ms |
| initial-rtt | 5m | 15s | 4s | 1s | 500ms | 250ms |
| max-retries | 10 | 10 | 6 | 3 | 3 | 2 |
| host-timeout | 0 | 0 | 0 | 0 | 0 | 15m |
| scan-delay | 5m | 15s | 400ms | 0 | 0 | 0 |
| max-scan-delay | 5m | 20s | 1s | 1s | 10ms | 5ms |
| min-rate | 0 | 0 | 0 | 0 | 0 | 0 |
| max-rate | 0 | 0 | 0 | 0 | 0 | 0 |

```bash
# Contoh custom timing yang melebihi T5:
sudo nmap -sS \
    --min-rate 10000 \
    --max-retries 1 \
    --host-timeout 5m \
    --max-rtt-timeout 100ms \
    --initial-rtt-timeout 50ms \
    -p 1-1000 \
    192.168.1.0/24

# Ini lebih cepat dari T5 tapi juga lebih banyak false negative
```

---

## Optimasi untuk Large-Scale Scanning

### Strategi: Phased Scanning

```bash
# Fase 1: Ping sweep cepat (hanya host discovery)
# Tools: masscan lebih cepat untuk ini
masscan -p0 --ping 10.0.0.0/8 --rate 10000 -oG masscan_hosts.txt &

# Atau dengan Nmap:
sudo nmap -sn --min-rate 5000 10.0.0.0/8 -oG hosts.gnmap

# Fase 2: SYN scan pada port penting saja (dari live hosts)
grep "Up" hosts.gnmap | awk '{print $2}' > alive.txt
sudo nmap -sS --min-rate 2000 -p 22,80,443,8080,3389,445,3306,5432 \
     -iL alive.txt -oX open_ports.xml

# Fase 3: Version detection pada open ports saja (lebih lambat tapi akurat)
# Extract hanya host:port yang open
python3 -c "
import xml.etree.ElementTree as ET
for host in ET.parse('open_ports.xml').findall('host'):
    ip = host.find('.//address[@addrtype=\"ipv4\"]').get('addr')
    for p in host.findall('.//port'):
        if p.find('state').get('state') == 'open':
            print(f'{ip},{p.get(\"portid\")}')
" > open_services.csv

# Scan version detection per-host
while IFS=, read ip port; do
    sudo nmap -sV -p $port $ip --append-output -oX version_scans.xml
done < open_services.csv
```

---

## libpcap Buffer Tuning

```bash
# libpcap buffer size mempengaruhi seberapa cepat Nmap bisa kirim/terima paket

# Default buffer sering terlalu kecil untuk high-speed scan
# Set buffer lebih besar:
sudo sysctl -w net.core.rmem_max=134217728     # 128MB receive buffer
sudo sysctl -w net.core.wmem_max=134217728     # 128MB send buffer
sudo sysctl -w net.core.netdev_max_backlog=5000

# Cek apakah ada packet drops
cat /proc/net/softnet_stat | awk '{print "drops:", $2}'

# Compile Nmap dengan PF_RING support (untuk 10Gbps+ scanning):
apt-get install pfring-dkms
./configure --with-pfring=/usr/local/pfring
make
```

---

## Masscan + Nmap Pipeline (High Performance)

```bash
# Masscan untuk discovery (~40 juta paket/detik)
# Nmap untuk akurasi dan version detection

# Step 1: Masscan sweeps entire internet-scale ranges
sudo masscan 10.0.0.0/8 -p 22,80,443,8080 --rate 100000 \
     --output-format list --output-file masscan_results.txt

# Step 2: Parse masscan output, convert ke nmap format
# masscan output: open tcp 80 192.168.1.1
awk '/^open/ {print $4 " " $3}' masscan_results.txt | sort -u > nmap_targets.txt

# Step 3: Nmap deep scan pada hasil masscan
while read ip port; do
    nmap -sV -sC -p $port $ip --open -oX results_${ip//./_}.xml 2>/dev/null &
    # Rate limit: jangan spawn terlalu banyak process
    while (( $(jobs -r | wc -l) >= 50 )); do sleep 1; done
done < nmap_targets.txt
wait
```

---

## CPU & Memory Profiling Nmap

```bash
# Monitor resource usage saat scan
# Terminal 1: scan
sudo nmap -sS -sV -T4 192.168.1.0/24 &
NMAP_PID=$!

# Terminal 2: monitor
pidstat -p $NMAP_PID 1  # CPU usage setiap detik
pmap -x $NMAP_PID      # Memory map

# Atau gunakan time:
time sudo nmap -sS 192.168.1.0/24 -oX /dev/null

# Profiling dengan valgrind (untuk development):
valgrind --tool=callgrind sudo nmap -sS 192.168.1.1
callgrind_annotate callgrind.out.* | head -50
```

---

## Nmap Compilation Flags untuk Performa

```bash
# Clone dari source
git clone https://github.com/nmap/nmap.git
cd nmap

# Konfigurasi dengan optimasi
./configure \
    --with-libdnet=included \
    --with-libpcap=included \
    --with-liblinear=included \
    CFLAGS="-O3 -march=native -pipe" \
    CXXFLAGS="-O3 -march=native -pipe"

make -j$(nproc)  # Kompilasi paralel

# Verifikasi binary baru
./nmap --version

# Bandingkan waktu dengan binary sistem vs yang baru dikompilasi
time sudo ./nmap -sS -T4 192.168.1.0/24 -oX /dev/null
time sudo nmap -sS -T4 192.168.1.0/24 -oX /dev/null
```

---

## Scan Resumption untuk Large Scans

```bash
# Nmap bisa resume scan yang terganggu
# Mulai scan dengan output ke file
sudo nmap -sS -sV -p- 10.0.0.0/24 --resume-timeout 10s \
     -oX /tmp/bigsan.xml

# Jika scan terganggu (Ctrl+C):
# Resume scan dari checkpoint
sudo nmap --resume /tmp/bigscan.xml

# Verifikasi berapa % yang sudah selesai:
grep -c "<host " /tmp/bigscan.xml
grep "status=\"up\"" /tmp/bigscan.xml | wc -l
```

---

## Parallelism Math

```bash
# Hitung berapa host yang bisa discan paralel
# Formula: host-timeout / (avg-response-time + scan-delay)

# Contoh:
# host-timeout = tidak ada
# avg-response-time = 50ms
# ports per host = 1000
# Parallelism = tergantung RTT dan packet loss

# Nmap auto-tunes, tapi bisa di-force:
sudo nmap --min-parallelism 100 --max-parallelism 256 -sS 192.168.1.0/24

# Terlalu tinggi = packet loss, false filtered results
# Cek dengan:
sudo nmap -d3 -sS --min-parallelism 200 192.168.1.1 2>&1 |
grep "parallelism\|retransmit\|dropped"
```

---

## Benchmarking: Nmap vs Masscan vs RustScan

```bash
# Benchmark scan speed antar tools

TARGET="192.168.1.0/24"
PORTS="1-65535"

echo "--- Nmap benchmark ---"
time sudo nmap -sS -p $PORTS $TARGET --open -oX /dev/null -T4

echo "--- Masscan benchmark ---"
time sudo masscan -p $PORTS $TARGET --rate 100000

echo "--- RustScan benchmark ---"
time rustscan -a $TARGET -p 1-65535 --batch-size 65535 -- -oX /dev/null

# Hasil tipikal untuk /24, semua port:
# Nmap T4: ~5-15 menit
# Masscan 100k: ~1-3 menit  
# RustScan: ~15-60 detik (tapi lebih banyak false positive)
```

---

*← [BAB 11: Red Team Operations](bab11.md) · Selanjutnya → [BAB 13: Threat Hunting](bab13.md)*
