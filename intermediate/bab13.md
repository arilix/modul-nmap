# BAB 13 — Nmap + Tools Lain

> **NMAP Intermediate Guide · 2026 Edition**

---

## Ekosistem Tools yang Bekerja dengan Nmap

Nmap jarang berdiri sendiri dalam workflow nyata. Berikut tools yang paling sering dikombinasikan:

```
Masscan ──→ Nmap ──→ Metasploit
                └──→ OpenVAS / Nessus
                └──→ EyeWitness (screenshot)
                └──→ Nuclei (vuln scanner)
                └──→ CrackMapExec (Windows)
```

---

## Nmap + Masscan (Discovery Cepat)

**Masscan** bisa scan jutaan IP per detik, tapi kurang akurat. Nmap lebih lambat tapi sangat akurat. Kombinasi idealnya:

```bash
# Langkah 1: Masscan untuk temukan port terbuka secara masif (cepat)
sudo masscan 10.0.0.0/8 -p 80,443,22,445,3389,8080,8443 \
  --rate 50000 \
  -oL masscan_results.txt

# Langkah 2: Ekstrak IP:Port dari masscan
awk '/^open/{print $4":"$3}' masscan_results.txt > targets_with_ports.txt
awk '/^open/{print $4}' masscan_results.txt | sort -u > alive_ips.txt

# Langkah 3: Nmap detail scan
sudo nmap -sS -sV -sC -T4 \
  -iL alive_ips.txt \
  -oA detailed_scan
```

---

## Nmap + EyeWitness (Screenshot Web)

EyeWitness mengambil screenshot dari semua web server yang ditemukan Nmap:

```bash
# Langkah 1: Scan web ports dengan Nmap
sudo nmap -sS -p 80,443,8080,8443 --open -T4 \
  192.168.1.0/24 \
  -oX web_targets.xml

# Langkah 2: Feed ke EyeWitness
eyewitness --xml web_targets.xml -d eyewitness_output --web

# Hasilnya: folder HTML dengan screenshot semua web server
```

---

## Nmap + Nikto (Web Vulnerability Scanner)

```bash
#!/bin/bash
# nmap_nikto.sh - Scan port web lalu jalankan Nikto

TARGET="192.168.1.0/24"

# Temukan web servers
echo "[*] Mencari web servers..."
nmap -sS -p 80,443,8080,8443 --open -T4 "$TARGET" \
  | grep "Nmap scan report" | awk '{print $NF}' > webservers.txt

# Jalankan Nikto untuk setiap web server
while read -r host; do
  echo "[*] Scanning $host dengan Nikto..."
  nikto -h "$host" -p 80 -output "nikto_${host}.txt" -Format txt
done < webservers.txt
```

---

## Nmap + CrackMapExec (Windows Networks)

```bash
# Langkah 1: Temukan semua Windows hosts
sudo nmap --script smb-os-discovery -p 445 --open 192.168.1.0/24 -oG smb_hosts.gnmap

# Ekstrak IP Windows hosts
grep "smb-os-discovery" smb_hosts.gnmap | awk '{print $2}' > windows_hosts.txt

# Langkah 2: CrackMapExec untuk enumeration lebih dalam
crackmapexec smb windows_hosts.txt       # Info dasar
crackmapexec smb windows_hosts.txt --shares    # Enum shares
crackmapexec smb windows_hosts.txt --users     # Enum users
crackmapexec smb windows_hosts.txt --pass-pol  # Password policy
```

---

## Nmap + Nuclei (Template-based Vuln Scanner)

```bash
# Langkah 1: Buat daftar URL dari scan Nmap
sudo nmap -sS -sV -p 80,443,8080,8443 --open 192.168.1.0/24 \
  -oX scan.xml

# Konversi XML ke daftar URL dengan Python
python3 - <<'EOF'
import xml.etree.ElementTree as ET
tree = ET.parse('scan.xml')
for host in tree.findall('host'):
    ip = host.find("address[@addrtype='ipv4']").get('addr')
    for port in host.findall('ports/port'):
        if port.find('state').get('state') == 'open':
            p = port.get('portid')
            proto = 'https' if p in ('443', '8443') else 'http'
            print(f"{proto}://{ip}:{p}")
EOF

# Langkah 2: Feed ke Nuclei
cat urls.txt | nuclei -t cves/ -t exposures/ -o nuclei_results.txt
```

---

## Nmap + OpenVAS / Greenbone

OpenVAS adalah vulnerability scanner yang bisa menerima input dari Nmap:

```bash
# Export Nmap ke format yang OpenVAS pahami
sudo nmap -sS -sV -O -T4 192.168.1.0/24 -oX nmap_for_openvas.xml

# Di OpenVAS GUI: Scans → Tasks → Import Nmap XML
# Atau via CLI dengan omp:
omp -u admin -w password --xml="<create_target>
  <name>My Network</name>
  <hosts>192.168.1.0/24</hosts>
</create_target>"
```

---

## Nmap + Searchsploit (Exploit Database)

```bash
#!/bin/bash
# Cari exploit berdasarkan versi software yang ditemukan Nmap

TARGET="192.168.1.1"

echo "[*] Scanning versi services..."
sudo nmap -sV -T4 "$TARGET" -oN /tmp/versions.txt

echo "[*] Mencari exploit yang relevan..."
grep "open" /tmp/versions.txt | while read -r line; do
  service=$(echo "$line" | awk '{print $4, $5, $6}')
  if [ -n "$service" ]; then
    echo "--- Searching: $service ---"
    searchsploit "$service" 2>/dev/null | head -5
  fi
done
```

---

## Nmap + Grep / Awk / Sed (Pipeline)

```bash
# Temukan semua web server dengan Apache
nmap -sV 192.168.1.0/24 | grep "Apache" | awk '{print $1, $5, $6}'

# Temukan semua host dengan SSH versi tua (< 7.x)
nmap -p 22 -sV 192.168.1.0/24 | grep "OpenSSH [1-6]"

# Daftar semua service unik yang ditemukan
nmap -sV 192.168.1.0/24 -oG - | grep "open" | \
  grep -oP '\d+/open/tcp/+/\K[^/]+' | sort | uniq -c | sort -rn

# Export format grepable, lalu filter
nmap -sS -p- --open 192.168.1.0/24 -oG - | \
  awk '/Up$/{print $2}' > all_hosts.txt
```

---

*← [BAB 12: Automation Bash & Cron](bab12.md) · Selanjutnya → [BAB 14: CTF & Pentest Scenarios](bab14.md)*
