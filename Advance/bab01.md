# BAB 01 — Nmap Architecture & Internals Deep Dive

> **NMAP Advanced Guide · 2026 Edition**
>
> Prasyarat: Sudah selesai modul Beginner & Intermediate. Familiar dengan C/Linux networking, Lua, Python, dan konsep jaringan layer 2-4.

---

## Arsitektur Internal Nmap

Nmap bukan sekadar tool — ini adalah **framework multi-subsistem**. Memahami arsitekturnya memungkinkan kamu melakukan tuning yang tidak terdokumentasi dan menulis kontribusi.

```
┌─────────────────────────────────────────────────────────┐
│                      NMAP PROCESS                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────┐ │
│  │  Target  │  │  Option  │  │   Output Engine        │ │
│  │  Parser  │→ │  Parser  │→ │  (Normal/XML/Grepable) │ │
│  └──────────┘  └──────────┘  └────────────────────────┘ │
│        ↓              ↓                                  │
│  ┌─────────────────────────┐                             │
│  │    Host State Machine   │ ← Track per-host scan state │
│  └─────────────────────────┘                             │
│        ↓                                                 │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────────┐ │
│  │    Ultra     │  │  Port      │  │   Service/OS     │ │
│  │  Scan (core) │  │  Scanner   │  │   Detection      │ │
│  └──────────────┘  └────────────┘  └──────────────────┘ │
│        ↓                                                 │
│  ┌──────────────────────────────────────────────────┐    │
│  │         NSE Engine (Lua 5.3 + coroutines)        │    │
│  └──────────────────────────────────────────────────┘    │
│        ↓                                                 │
│  ┌─────────────┐  ┌──────────────────────────────────┐   │
│  │   libpcap   │  │   Raw Socket / libdnet            │   │
│  │  (capture)  │  │   (send packets)                  │   │
│  └─────────────┘  └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Ultra Scan: Mesin Inti Nmap

**Ultra Scan** adalah algoritma scanning paling penting di Nmap — mengelola semua probe TCP/UDP/SCTP secara konkuren. Memahaminya membantu kamu melakukan tuning yang tepat.

### Konsep: Congestion Window

Seperti TCP, Ultra Scan menggunakan **congestion window** (`cwnd`) untuk mengatur berapa probe yang boleh "in flight" sekaligus:

```
cwnd (initial) = 10 probe
↓
Jika probe berhasil → cwnd naik (slow start / congestion avoidance)
Jika probe timeout → cwnd turun drastis (congestion detected)
```

```bash
# Debug level 9: lihat cwnd berubah real-time
sudo nmap -d9 -sS 192.168.1.0/24 2>&1 | grep -i "cwnd\|probes\|sent\|rcvd"
```

### Konsep: Timing Groups

Nmap mengelompokkan host ke dalam **timing groups** berdasarkan RTT mereka. Host dengan RTT serupa digroup bersama untuk efisiensi.

```bash
# Lihat RTT per host saat scan berlangsung
sudo nmap -d3 -sS 192.168.1.0/24 2>&1 | grep "Timing"
```

---

## Source Code: Lokasi Kunci (Nmap C++)

Jika kamu ingin membaca atau memodifikasi Nmap:

```
nmap/
├── nmap.cc           ← Entry point, argument parsing
├── scan_engine.cc    ← UltraScan implementation (terpenting!)
├── scan_engine.h     ← UltraScan data structures
├── portreasons.cc    ← Logika port state determination
├── osscan.cc         ← OS fingerprinting engine
├── osscan2.cc        ← OS fingerprinting v2 (lebih baru)
├── service_scan.cc   ← Version/service detection
├── nse_main.cc       ← NSE engine (Lua integration)
├── tcpip.cc          ← TCP/IP utility functions
├── Target.cc         ← Per-host state management
└── output.cc         ← Output formatting
```

---

## Packet Flow Analysis dengan Wireshark

Pahami setiap paket yang Nmap kirim dengan menganalisisnya:

```bash
# Terminal 1: Capture traffic Nmap
sudo tcpdump -i eth0 -w /tmp/nmap_capture.pcap host 192.168.1.1

# Terminal 2: Jalankan scan spesifik
sudo nmap -sS -p 22,80,443 192.168.1.1

# Buka di Wireshark
wireshark /tmp/nmap_capture.pcap

# Filter di Wireshark:
# tcp.flags.syn == 1 && tcp.flags.ack == 0   → SYN packets
# tcp.flags.rst == 1                          → RST packets
# icmp                                         → ICMP responses
```

### Menganalisis Setiap Scan Type

```bash
# Bandingkan paket untuk SYN scan vs Connect scan
sudo tcpdump -i lo -n -X &

# SYN scan: hanya SYN dikirim, RST dikirim balik setelah SYN-ACK
sudo nmap -sS -p 80 127.0.0.1

# Connect scan: full three-way handshake + RST/FIN
sudo nmap -sT -p 80 127.0.0.1
```

---

## Libpcap Internals: Bagaimana Nmap Capture Responses

Nmap menggunakan **libpcap** untuk menangkap paket masuk secara low-level:

```c
/* Pseudocode dari scan_engine.cc */
pcap_t *pd = pcap_open_live(iface, 100, 1, 200, errbuf);
pcap_setfilter(pd, &fcode);  // BPF filter

while (scan_not_done) {
    // Send probes
    send_syn_probe(target_ip, target_port);
    
    // Capture responses via pcap
    struct pcap_pkthdr hdr;
    const u_char *pkt = pcap_next(pd, &hdr);
    process_response(pkt);
}
```

### BPF Filters yang Nmap Gunakan

```bash
# Lihat BPF filter yang aktif saat scan
sudo nmap -d5 -sS 192.168.1.1 2>&1 | grep "BPF\|pcap\|filter"

# Contoh BPF filter Nmap untuk SYN scan:
# "tcp and dst host OUR_IP and (tcp[tcpflags] & (tcp-syn|tcp-rst|tcp-ack) != 0)"
```

---

## Nmap Timing System: Detail Algoritma

### Parameter Timing Internal

```
ssthresh       = slow start threshold
cwnd           = congestion window  
srtt           = smoothed round trip time
rttvar         = RTT variance
timeout        = probe timeout = srtt + 4*rttvar
```

### Hubungan dengan Template -T

| Parameter         | T0      | T1      | T2      | T3       | T4        | T5     |
|-------------------|---------|---------|---------|----------|-----------|--------|
| min_rtt_timeout   | 100ms   | 100ms   | 100ms   | 100ms    | 100ms     | 50ms   |
| max_rtt_timeout   | 5 min   | 15s     | 10s     | 10s      | 1250ms    | 300ms  |
| initial_rtt_timeout | 5 min | 15s   | 1s      | 1s       | 500ms     | 250ms  |
| max_retries       | 10      | 10      | 10      | 10       | 6         | 2      |
| scan_delay        | 5 min   | 15s     | 400ms   | 0        | 0         | 0      |
| max_scan_delay    | 5 min   | 15s     | 10s     | 1s       | 10ms      | 5ms    |
| host_timeout      | 0       | 0       | 0       | 0        | 0         | 0      |

```bash
# Override parameter individual (lebih presisi dari -T)
sudo nmap --min-rtt-timeout 10ms \
          --max-rtt-timeout 300ms \
          --initial-rtt-timeout 50ms \
          --max-retries 2 \
          --min-parallelism 50 \
          --max-parallelism 300 \
          192.168.1.0/24
```

---

## Debug Mendalam: Memahami Keputusan Nmap

```bash
# Debug level 1-9 (9 = paling verbose)
sudo nmap -d1 -sS 192.168.1.1   # Summary decisions
sudo nmap -d3 -sS 192.168.1.1   # RTT, timing adjustments  
sudo nmap -d5 -sS 192.168.1.1   # Per-probe details
sudo nmap -d9 -sS 192.168.1.1   # Everything (sangat verbose)

# Kombinasikan dengan grep untuk filter
sudo nmap -d5 -sS 192.168.1.1 2>&1 | grep -E "Timeout|RTT|retry|cwnd"

# Packet trace level
sudo nmap --packet-trace -sS -p 80 192.168.1.1
```

---

## Membangun Nmap dari Source

```bash
# Clone source
git clone https://github.com/nmap/nmap.git
cd nmap

# Konfigurasi dengan semua fitur
./configure --with-libpcap --with-liblua --with-openssl

# Build
make -j$(nproc)

# Install (atau jalankan langsung)
sudo make install
# atau
./nmap --version
```

### Patch Kecil: Ubah Default User-Agent HTTP

```c
// Di nse_nsock.cc atau nse_main.cc
// Cari: "Mozilla/5.0 (compatible; Nmap Scripting Engine"
// Ganti dengan user-agent yang kamu inginkan
```

---

*Selanjutnya → [BAB 02: Advanced OS & Service Fingerprinting](bab02.md)*
