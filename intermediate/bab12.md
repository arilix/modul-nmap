# BAB 12 — Automasi dengan Bash & Cron

> **NMAP Intermediate Guide · 2026 Edition**

---

## Mengapa Automasi?

- **Monitoring berkelanjutan** — deteksi perubahan jaringan secara otomatis
- **Konsistensi** — scan yang sama dijalankan setiap saat tanpa human error
- **Alerting** — notifikasi otomatis saat ada perubahan kritis
- **Audit trail** — riwayat scan tersimpan rapi dengan timestamp

---

## Script Bash: Full Recon Workflow

```bash
#!/bin/bash
# recon.sh - Full Nmap reconnaissance workflow
# Usage: ./recon.sh <target> <output_dir>

set -euo pipefail

TARGET="${1:-192.168.1.0/24}"
OUTDIR="${2:-./scan_$(date +%Y%m%d_%H%M%S)}"
LOGFILE="$OUTDIR/scan.log"

mkdir -p "$OUTDIR"

log() {
  echo "[$(date '+%H:%M:%S')] $*" | tee -a "$LOGFILE"
}

log "=== Mulai scanning: $TARGET ==="
log "Output dir: $OUTDIR"

# Fase 1: Host Discovery
log "[1/4] Host discovery..."
sudo nmap -sn -T4 --min-parallelism 50 "$TARGET" \
  -oN "$OUTDIR/01_hosts.txt" \
  -oX "$OUTDIR/01_hosts.xml" 2>>"$LOGFILE"

# Ambil daftar host yang up
grep "Nmap scan report" "$OUTDIR/01_hosts.txt" | awk '{print $NF}' > "$OUTDIR/live_hosts.txt"
TOTAL=$(wc -l < "$OUTDIR/live_hosts.txt")
log "Ditemukan $TOTAL host aktif"

if [ "$TOTAL" -eq 0 ]; then
  log "Tidak ada host yang ditemukan. Keluar."
  exit 1
fi

# Fase 2: Quick Port Scan
log "[2/4] Quick port scan (top 100)..."
sudo nmap -sS --top-ports 100 --open -T4 \
  -iL "$OUTDIR/live_hosts.txt" \
  -oA "$OUTDIR/02_quick_scan" 2>>"$LOGFILE"

# Fase 3: Version + Script Detection
log "[3/4] Version & service detection..."
sudo nmap -sS -sV -sC --open -T4 \
  -iL "$OUTDIR/live_hosts.txt" \
  -oA "$OUTDIR/03_version_scan" 2>>"$LOGFILE"

# Fase 4: Vulnerability Check
log "[4/4] Vulnerability assessment..."
sudo nmap --script=vuln --open -T4 \
  -iL "$OUTDIR/live_hosts.txt" \
  -oA "$OUTDIR/04_vuln_scan" 2>>"$LOGFILE"

log "=== Scan selesai! Hasil di: $OUTDIR ==="

# Ringkasan singkat
echo ""
echo "=== RINGKASAN ==="
echo "Total host aktif    : $TOTAL"
echo "Port terbuka umum   :"
grep "open" "$OUTDIR/03_version_scan.nmap" | grep -v "^#" | \
  awk '{print "  " $1, $3, $4, $5}' | sort | uniq -c | sort -rn | head -20
```

---

## Script Bash: Monitoring Perubahan Jaringan

```bash
#!/bin/bash
# monitor.sh - Deteksi perubahan port/host dibanding scan sebelumnya
# Usage: ./monitor.sh <target>

TARGET="${1:-192.168.1.0/24}"
SCAN_DIR="/var/log/nmap_monitor"
TODAY="$SCAN_DIR/scan_$(date +%Y%m%d).xml"
YESTERDAY=$(ls -t "$SCAN_DIR"/scan_*.xml 2>/dev/null | sed -n '2p')

mkdir -p "$SCAN_DIR"

echo "[*] Menjalankan scan baru..."
sudo nmap -sS -sV --open -T4 "$TARGET" -oX "$TODAY" -oN "${TODAY%.xml}.txt"

if [ -n "$YESTERDAY" ]; then
  echo "[*] Membandingkan dengan scan sebelumnya..."
  DIFF=$(ndiff "$YESTERDAY" "$TODAY")
  
  if echo "$DIFF" | grep -qE "^[+-]"; then
    echo "=== PERUBAHAN TERDETEKSI ==="
    echo "$DIFF" | grep -E "^[+-]"
    
    # Kirim notifikasi (opsional: ganti dengan curl/webhook)
    echo "Perubahan jaringan terdeteksi pada $(date)" >> /var/log/nmap_alerts.log
    echo "$DIFF" >> /var/log/nmap_alerts.log
  else
    echo "[+] Tidak ada perubahan."
  fi
else
  echo "[*] Ini scan pertama, tidak ada perbandingan."
fi
```

---

## Script Bash: Alerting via Webhook (Slack/Discord)

```bash
#!/bin/bash
# alert.sh - Kirim notifikasi ke Slack webhook jika ada perubahan

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
TARGET="192.168.1.0/24"
TODAY_XML="/tmp/nmap_today.xml"
PREV_XML="/tmp/nmap_prev.xml"

# Backup scan sebelumnya
[ -f "$TODAY_XML" ] && cp "$TODAY_XML" "$PREV_XML"

# Scan baru
sudo nmap -sS --open -T4 "$TARGET" -oX "$TODAY_XML" -q

# Bandingkan jika ada scan sebelumnya
if [ -f "$PREV_XML" ]; then
  CHANGES=$(ndiff "$PREV_XML" "$TODAY_XML" | grep -E "^[+-]" | head -20)
  
  if [ -n "$CHANGES" ]; then
    # Kirim ke Slack
    MESSAGE="🔔 *Nmap Alert* - Perubahan jaringan terdeteksi!\n\`\`\`${CHANGES}\`\`\`"
    curl -s -X POST "$WEBHOOK_URL" \
      -H 'Content-type: application/json' \
      --data "{\"text\":\"$MESSAGE\"}"
    echo "Alert terkirim!"
  fi
fi
```

---

## Menjadwalkan dengan Cron

```bash
# Edit crontab
crontab -e

# Contoh jadwal:

# Scan setiap hari jam 02:00 dini hari
0 2 * * * /home/user/scripts/monitor.sh 192.168.1.0/24 >> /var/log/nmap_cron.log 2>&1

# Scan setiap Senin jam 06:00
0 6 * * 1 /home/user/scripts/recon.sh 10.0.0.0/24 /var/log/weekly_scan >> /var/log/nmap_cron.log 2>&1

# Discovery setiap 4 jam
0 */4 * * * sudo nmap -sn 192.168.1.0/24 -oN /tmp/hosts_$(date +\%H).txt

# Cek port kritis setiap 15 menit
*/15 * * * * sudo nmap -sS -p 22,3389 --open 192.168.1.0/24 -oN /tmp/critical_ports.txt
```

---

## Rotasi & Manajemen Hasil Scan

```bash
#!/bin/bash
# cleanup.sh - Hapus scan yang lebih tua dari 30 hari

SCAN_DIR="/var/log/nmap_monitor"

# Hapus file lebih tua dari 30 hari
find "$SCAN_DIR" -name "scan_*.xml" -mtime +30 -delete
find "$SCAN_DIR" -name "scan_*.txt" -mtime +30 -delete

echo "Cleanup selesai: $(date)"

# Hitung dan tampilkan penggunaan disk
du -sh "$SCAN_DIR"
```

---

## One-Liner Berguna untuk Bash

```bash
# Ambil hanya IP dari scan output
nmap -sn 192.168.1.0/24 | grep "report" | awk '{print $5}'

# Ambil semua port terbuka dan service-nya dari scan
nmap -sS -sV 192.168.1.0/24 | grep "open" | awk '{print $1, $3, $4, $5}' | sort -u

# Scan dan langsung filter host dengan port 22 terbuka
nmap -p 22 --open 192.168.1.0/24 | grep "report" | awk '{print $5}' > ssh_hosts.txt

# Hitung statistik port yang paling banyak terbuka
nmap -sS 192.168.1.0/24 | grep "open" | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

---

*← [BAB 11: Advanced Evasion](bab11.md) · Selanjutnya → [BAB 13: Nmap + Tools Lain](bab13.md)*
