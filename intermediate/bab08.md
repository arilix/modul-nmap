# BAB 08 — IPv6 Scanning

> **NMAP Intermediate Guide · 2026 Edition**

---

## Mengapa IPv6 Penting?

IPv6 semakin umum di jaringan enterprise dan cloud modern. Banyak administrator fokus pada keamanan IPv4 namun **melupakan IPv6** — ini menciptakan celah keamanan yang sering dieksploitasi. Nmap mendukung IPv6 sepenuhnya.

---

## Perbedaan IPv4 vs IPv6 dalam Scanning

| Aspek              | IPv4                        | IPv6                                     |
|--------------------|-----------------------------|------------------------------------------|
| Address space      | 4.3 miliar host             | 340 undecillion host                     |
| Host discovery     | ICMP Echo, ARP              | ICMPv6 Echo, NDP (Neighbor Discovery)   |
| Multicast         | Terbatas                    | Jauh lebih banyak digunakan              |
| Broadcast         | Ada                         | **Tidak ada** (diganti multicast)        |
| Scanning subnet   | /24 = 256 host             | /64 = 18 quintillion host (tidak praktis)|

> 💡 Karena subnet IPv6 sangat besar, **port scanning seluruh subnet IPv6 tidak praktis**. Kamu harus tahu alamat targetnya dulu.

---

## Sintaks Dasar Scanning IPv6

```bash
# Flag -6 untuk aktifkan IPv6
nmap -6 ::1                                # localhost IPv6
nmap -6 2001:db8::1                        # Scan single IPv6 address
nmap -6 2001:db8::/112                     # Subnet /112 (65536 host, masih banyak)
nmap -6 fe80::1%eth0                       # Link-local address (perlu interface)

# SYN scan IPv6
sudo nmap -6 -sS 2001:db8::1

# Aggressive scan
sudo nmap -6 -A 2001:db8::1

# Version + OS detection
sudo nmap -6 -sV -O 2001:db8::1
```

---

## Host Discovery untuk IPv6

### ICMPv6 (Menggantikan ICMP di IPv6)

```bash
# Ping scan IPv6
nmap -6 -sn 2001:db8::/112

# ICMPv6 Echo ping
nmap -6 -PE 2001:db8::1

# ICMPv6 Neighbor Solicitation (seperti ARP di IPv4)
sudo nmap -6 --script targets-ipv6-multicast-slaac --script-args newtargets -sL

# Multicast ping untuk menemukan semua host di jaringan lokal
sudo nmap -6 -sn ff02::1%eth0    # All-nodes multicast
```

### Menemukan IPv6 Addresses secara Pasif

```bash
# NDP (Neighbor Discovery Protocol) - seperti ARP di IPv4
sudo nmap --script targets-ipv6-multicast-slaac \
  --script-args 'newtargets,interface=eth0' -6 -sL

# Temukan alamat IPv6 via router advertisement
sudo nmap --script targets-ipv6-multicast-mld \
  --script-args 'newtargets,interface=eth0' -6 -sL
```

---

## DNS untuk IPv6

IPv6 menggunakan record **AAAA** (bukan A).

```bash
# Resolve hostname ke IPv6
nmap -6 -sn example.com         # Nmap otomatis resolve AAAA record

# Gunakan DNS server tertentu
nmap -6 --dns-servers 2001:4860:4860::8888 example.com

# DNS brute untuk AAAA records
nmap -6 --script dns-brute --script-args dns-brute.domain=example.com
```

---

## Port Scanning IPv6

```bash
# Scan port dasar
nmap -6 -p 22,80,443 2001:db8::1

# Scan semua port
nmap -6 -p- 2001:db8::1

# SYN scan, version, script
sudo nmap -6 -sS -sV -sC -T4 2001:db8::1

# Scan top 1000 port
nmap -6 2001:db8::1
```

---

## NSE Scripts untuk IPv6

```bash
# Info tentang IPv6 prefixes
nmap -6 --script ipv6-node-info 2001:db8::1

# Scan router advertisements
sudo nmap -6 --script targets-ipv6-multicast-echo \
  --script-args newtargets -sL -6

# Address autoconfiguration enumeration
sudo nmap -6 --script targets-ipv6-multicast-slaac \
  --script-args 'newtargets,interface=eth0' -6 -sn
```

---

## Scanning Link-Local Addresses

```bash
# Link-local dimulai dengan fe80::
# Harus menentukan interface dengan %
sudo nmap -6 -sn fe80::%eth0/64

# Scan link-local tertentu
sudo nmap -6 -p 22,80 fe80::a00:27ff:fec4:1234%eth0
```

---

## Dual-Stack Scanning (IPv4 + IPv6 Bersamaan)

```bash
# Nmap tidak scan kedua stack secara bersamaan secara native
# Solusi: scan terpisah dan gabungkan hasilnya

sudo nmap -4 -sS -sV -oX ipv4_scan.xml 192.168.1.0/24
sudo nmap -6 -sS -sV -oX ipv6_scan.xml 2001:db8::/112

# Gabungkan dengan tool seperti xsltproc atau ndiff
```

---

## Tantangan Umum IPv6

### 1. Firewall Blocking ICMPv6
```bash
# Treat semua host sebagai online (skip discovery)
sudo nmap -6 -Pn 2001:db8::1

# Gunakan TCP ping
sudo nmap -6 -PS80 2001:db8::1
```

### 2. Tidak Bisa Temukan Hosts
```bash
# Coba metode discovery berbeda
sudo nmap -6 -PE -PS80,443 -PA80 2001:db8::1

# Cek apakah ada di tabel neighbor
ip -6 neigh show
```

---

## Contoh Lengkap: IPv6 Network Audit

```bash
#!/bin/bash
IFACE="eth0"
SUBNET="2001:db8::/112"

echo "[1] Temukan hosts via multicast..."
sudo nmap -6 --script targets-ipv6-multicast-slaac \
  --script-args "newtargets,interface=$IFACE" -sL -6 \
  -oN 01_ipv6_hosts.txt 2>/dev/null

echo "[2] Ping scan subnet..."
sudo nmap -6 -sn $SUBNET -oN 02_ipv6_ping.txt

echo "[3] Port scan hosts yang ditemukan..."
sudo nmap -6 -sS -sV -sC -T4 \
  -iL 02_ipv6_ping.txt \
  -oA 03_ipv6_portscan

echo "Selesai!"
```

---

*← [BAB 07: Nmap + Metasploit](bab07.md) · Selanjutnya → [BAB 09: Scanning Jaringan Besar](bab09.md)*
