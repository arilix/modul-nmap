# BAB 09 — Scanning Jaringan Besar & Optimasi

> **NMAP Intermediate Guide · 2026 Edition**

---

## Masalah Saat Scanning Jaringan Besar

Saat memindai ribuan host, kamu akan menghadapi tantangan:
- Scan terlalu lambat
- Terlalu banyak false positives / false negatives
- Resource CPU/RAM tinggi
- Kemungkinan terdeteksi IDS/IPS
- Host lambat menyebabkan timeout yang menumpuk

---

## Strategi Phased Scanning

Jangan langsung scan semua port di semua host. Gunakan pendekatan bertahap:

```bash
# FASE 1: Host Discovery cepat
sudo nmap -sn -T4 --min-parallelism 50 10.0.0.0/8 -oN fase1_hosts.txt

# Ambil hanya IP yang up
grep "Nmap scan report" fase1_hosts.txt | awk '{print $5}' > live_hosts.txt

# FASE 2: Quick port scan (top 100 port) pada host yang hidup
nmap -sS --top-ports 100 --open -T4 -iL live_hosts.txt -oA fase2_quick

# FASE 3: Version detection hanya pada host yang menarik
nmap -sV -sC -T4 -iL interesting_hosts.txt -oA fase3_detailed

# FASE 4: Full port scan pada target prioritas
nmap -p- -sS -T4 -iL priority_hosts.txt -oA fase4_fullscan
```

---

## Tuning Performance untuk Jaringan Besar

### Paralelisme

```bash
# Paksa lebih banyak probe berjalan bersamaan
nmap --min-parallelism 100 --max-parallelism 1000 10.0.0.0/24

# Kombinasi dengan timing agresif
nmap -T4 --min-parallelism 50 10.0.0.0/24
```

### Rate Limiting

```bash
# Minimal 500 paket/detik (Nmap akan usahakan lebih cepat)
nmap --min-rate 500 10.0.0.0/24

# Maksimal 1000 paket/detik (jangan bebani jaringan)
nmap --max-rate 1000 10.0.0.0/24

# Untuk jaringan besar: 2000-5000 pps (perlu test dulu)
nmap --min-rate 2000 --max-rate 5000 10.0.0.0/8
```

### Timeout Management

```bash
# Abandon host setelah 30 detik
nmap --host-timeout 30s 10.0.0.0/24

# RTT timeout lebih agresif
nmap --min-rtt-timeout 50ms --max-rtt-timeout 200ms --initial-rtt-timeout 100ms 10.0.0.0/24

# Retry lebih sedikit (lebih cepat, sedikit lebih banyak false negative)
nmap --max-retries 1 10.0.0.0/24
```

### Kombinasi Optimal untuk Scan Cepat

```bash
# Fast scan besar (ribuan host)
sudo nmap -sS -T4 \
  --min-rate 1000 \
  --max-retries 2 \
  --host-timeout 60s \
  --open \
  -p 80,443,22,445,3389 \
  -iL hosts.txt \
  -oA fast_scan

# Ultra-fast discovery only
sudo nmap -sn -T5 \
  --min-parallelism 200 \
  --max-retries 0 \
  10.0.0.0/16
```

---

## Distributed / Parallel Scanning

### Membagi Subnet dengan Bash

```bash
#!/bin/bash
# Split /16 menjadi 256 /24 dan scan paralel

for i in $(seq 0 255); do
  subnet="10.0.$i.0/24"
  outfile="scan_10_0_$i"
  sudo nmap -sS -T4 --min-rate 1000 -oA "results/$outfile" "$subnet" &
  
  # Limit: jalankan max 10 proses bersamaan
  if (( (i+1) % 10 == 0 )); then
    wait
    echo "Batch $((i/10+1)) selesai..."
  fi
done
wait
echo "Semua subnet selesai!"
```

### Menggunakan `masscan` + Nmap

Masscan jauh lebih cepat untuk discovery, Nmap lebih akurat untuk version detection:

```bash
# Phase 1: masscan untuk temukan host/port cepat
sudo masscan 10.0.0.0/8 -p 80,443,22,445 --rate 100000 -oL masscan_results.txt

# Konversi hasil masscan ke daftar IP
awk '/open/{print $4}' masscan_results.txt | sort -u > alive_hosts.txt

# Phase 2: Nmap detail scan pada hasil masscan
sudo nmap -sS -sV -sC -T4 -iL alive_hosts.txt -p 80,443,22,445 -oA nmap_detail
```

---

## Menangani False Positives & Negatives

### UDP Scan yang Lebih Akurat

```bash
# UDP default sering false positive; tambahkan version detection
sudo nmap -sU -sV --version-intensity 0 -T4 -p 53,161,123,69 10.0.0.0/24

# Retry lebih banyak untuk UDP (lebih akurat)
sudo nmap -sU --max-retries 3 -p 161 10.0.0.0/24
```

### Deteksi Host yang Sengaja Menyembunyikan Diri

```bash
# Host mungkin memblokir ICMP, coba TCP
nmap -PS80,443,22 -PE 192.168.1.1

# Coba berbagai metode discovery
nmap -PE -PP -PS80 -PU53 192.168.1.1

# Skip discovery sepenuhnya
nmap -Pn 192.168.1.1
```

---

## Menyimpan dan Melanjutkan Scan yang Terinterupsi

```bash
# Simpan scan dalam format yang bisa diresume
sudo nmap -oA scan_output --resume scan_output.nmap 192.168.1.0/24

# Jika terputus, lanjutkan dengan:
sudo nmap --resume scan_output.nmap
```

---

## Monitoring Progress Scan

```bash
# Tampilkan progress saat scan berjalan
# Tekan 'p' saat scan berlangsung untuk toggle statistik

# Atau tampilkan stats setiap 30 detik
nmap --stats-every 30s 10.0.0.0/24

# Verbose output
nmap -v 10.0.0.0/24

# Sangat verbose (tampilkan setiap host saat ditemukan)
nmap -vv 10.0.0.0/24
```

---

## Estimasi Waktu Scan

| Target       | Metode              | Estimasi Waktu     |
|--------------|---------------------|--------------------|
| /24 (256)    | SYN, top-1000       | ~2-5 menit         |
| /24 (256)    | SYN, all ports      | ~15-30 menit       |
| /16 (65K)    | SYN, top-100        | ~30-60 menit       |
| /8 (16M)     | Ping only           | ~6-8 jam           |
| /8 (16M)     | SYN, top-100        | ~24-48 jam         |

> 💡 Dengan `--min-rate 10000` dan hardware yang kuat, waktu bisa dipotong 5-10x. Tapi perhatikan dampak pada jaringan produksi.

---

*← [BAB 08: IPv6 Scanning](bab08.md) · Selanjutnya → [BAB 10: Parsing Output dengan Python & XML](bab10.md)*
