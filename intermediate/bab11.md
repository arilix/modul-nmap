# BAB 11 — Advanced Evasion Techniques

> **NMAP Intermediate Guide · 2026 Edition**

---

> ⚠️ **Perhatian:** Semua teknik di bab ini hanya boleh digunakan dalam konteks **authorized penetration testing** atau pada jaringan/sistem yang kamu miliki. Penggunaan tanpa izin adalah ilegal.

---

## Bagaimana IDS/IPS Mendeteksi Scan?

Sebelum bisa evade, pahami dulu apa yang dideteksi:

| Pola Deteksi                  | Contoh Alat yang Mendeteksi      |
|-------------------------------|-----------------------------------|
| SYN flood dari satu IP        | Snort, Suricata, firewall stateful |
| Port scan signature           | Suricata, Zeek (Bro)              |
| Scan terlalu cepat            | Rate-based IPS                    |
| Fragmented packets            | Deep Packet Inspection (DPI)      |
| Unusual TCP flag combinations | Snort rules                       |
| Port scanning sequence        | PortSentry, fail2ban              |

---

## Teknik 1: Timing-Based Evasion

```bash
# T0: Sangat lambat - satu probe per 5 menit (tidak terdeteksi rate-based IDS)
sudo nmap -T0 192.168.1.1

# T1: Satu probe per 15 detik
sudo nmap -T1 192.168.1.1

# Custom timing: satu probe per detik, max 1 host bersamaan
sudo nmap --scan-delay 1s --max-parallelism 1 192.168.1.1

# Random delay antara 1-5 detik
sudo nmap --scan-delay 1s --max-scan-delay 5s 192.168.1.1
```

---

## Teknik 2: Packet Fragmentation

```bash
# Fragment paket menjadi 8-byte chunk
sudo nmap -f 192.168.1.1

# Fragment lebih kecil lagi (16-byte)
sudo nmap -ff 192.168.1.1

# Tentukan ukuran fragment manual (harus kelipatan 8)
sudo nmap --mtu 24 192.168.1.1

# Gabungkan dengan scan type
sudo nmap -f -sS -T2 192.168.1.1
```

> 💡 Banyak modern IDS dapat melakukan **IP defragmentation** sebelum inspeksi, jadi teknik ini kurang efektif melawan IDS modern namun masih berguna melawan packet filters sederhana.

---

## Teknik 3: Decoy Scanning

Buat traffic scanmu bercampur dengan traffic dari IP palsu, sehingga sulit diidentifikasi sumber aslinya.

```bash
# Decoy spesifik (ME = posisi IP aslimu di antara decoys)
sudo nmap -D 10.0.0.1,10.0.0.2,ME,10.0.0.4 192.168.1.1

# Generate 10 decoy acak
sudo nmap -D RND:10 192.168.1.1

# Decoy dengan timing lambat
sudo nmap -D RND:5 -T2 192.168.1.1

# Gabungkan dengan fragmentation
sudo nmap -D RND:10 -f -T2 -sS 192.168.1.1
```

**Cara kerjanya:** Target akan melihat SYN packets datang dari 11 IP berbeda (10 decoy + IP aslimu), sehingga sulit menentukan siapa yang benar-benar scanning.

> ⚠️ Pilih decoy yang benar-benar ada di jaringan, atau yang ada di internet — jika decoy tidak ada, SYN-ACK dari target akan dikirim ke IP tidak valid dan tidak akan menyebabkan masalah, tapi lebih mudah diidentifikasi oleh IDS yang cerdas.

---

## Teknik 4: Source Port Manipulation

Beberapa firewall mengizinkan traffic dari port tertentu (DNS=53, HTTP=80) secara implisit.

```bash
# Gunakan source port 53 (DNS - sering di-whitelist)
sudo nmap --source-port 53 192.168.1.1
sudo nmap -g 53 192.168.1.1

# Gunakan source port 80 (HTTP)
sudo nmap -g 80 192.168.1.1

# Source port 443 (HTTPS)
sudo nmap -g 443 192.168.1.1

# Kombinasi dengan scan type dan decoy
sudo nmap -g 53 -D RND:5 -T2 -sS 192.168.1.1
```

---

## Teknik 5: Data Length Manipulation

Beberapa IDS memiliki tanda tangan berdasarkan ukuran paket. Tambahkan data acak:

```bash
# Tambahkan 25 byte data acak ke paket
sudo nmap --data-length 25 192.168.1.1

# Ukuran besar untuk menyerupai traffic normal
sudo nmap --data-length 200 192.168.1.1

# Gabungkan dengan teknik lain
sudo nmap --data-length 15 -f -T2 192.168.1.1
```

---

## Teknik 6: MAC Address Spoofing

```bash
# MAC address acak
sudo nmap --spoof-mac 0 192.168.1.1

# MAC dari vendor tertentu (menyamar sebagai device Cisco)
sudo nmap --spoof-mac Cisco 192.168.1.1

# MAC spesifik
sudo nmap --spoof-mac 00:11:22:33:44:55 192.168.1.1

# MAC dari Apple (menyamar sebagai MacBook)
sudo nmap --spoof-mac Apple 192.168.1.1
```

> 💡 MAC spoofing hanya efektif di **jaringan lokal (layer 2)**. Tidak berpengaruh saat scanning melalui router.

---

## Teknik 7: Randomisasi Target & Port

```bash
# Acak urutan host (hindari pattern scan yang linier)
nmap --randomize-hosts 192.168.1.0/24

# Acak urutan port (default Nmap sudah randomize port)
# Untuk disable randomisasi (lebih cepat tapi lebih terdeteksi):
nmap -r 192.168.1.1   # -r = scan port secara berurutan (JANGAN pakai untuk evasion)
```

---

## Teknik 8: Bad Checksum

```bash
# Kirim paket dengan checksum salah (beberapa IDS melewatinya)
sudo nmap --badsum 192.168.1.1

# Hanya host yang benar-benar menerima paket akan membalas
# Host normal akan drop paket checksum salah
# IDS yang tidak melakukan checksum validation akan melewatinya
```

---

## Kombinasi Teknik Evasion (Contoh Realistis)

```bash
# Stealth scan yang menyeluruh
sudo nmap \
  -sS \                      # SYN scan
  -T1 \                      # Timing sneaky
  -f \                       # Fragment packets
  -D RND:5 \                 # 5 random decoys
  -g 53 \                    # Source port DNS
  --data-length 20 \         # Extra data
  --randomize-hosts \        # Acak urutan host
  -p 22,80,443,445,3389 \
  192.168.1.0/24
```

---

## Menguji Apakah Scan Terdeteksi

```bash
# Di sistem target/monitor, jalankan tcpdump untuk melihat paket
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' -n

# Cek log IDS (jika ada akses)
sudo tail -f /var/log/snort/alert

# Gunakan Wireshark untuk lihat semua traffic yang mencurigakan
```

---

*← [BAB 10: Parsing Output Python/XML](bab10.md) · Selanjutnya → [BAB 12: Automation dengan Bash & Cron](bab12.md)*
