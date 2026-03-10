# BAB 08 — ICS/SCADA & IoT Scanning

> **NMAP Advanced Guide · 2026 Edition**

---

## Peringatan & Etika

> **PENTING:** Scanning ICS/SCADA tanpa otorisasi eksplisit dapat menyebabkan gangguan operasional yang berbahaya — bahkan fatal. Peralatan industri tidak dirancang untuk menangani traffic scan dan bisa crash, hang, atau menghasilkan aksi fisik berbahaya. SELALU dapatkan izin tertulis dan koordinasi dengan operator sebelum scanning environment OT (Operational Technology).

---

## Protokol ICS/SCADA: Landscape

| Protokol | Port | Transport | Digunakan Di |
|----------|------|-----------|--------------|
| Modbus | 502/tcp | TCP | PLC, RTU, field devices |
| DNP3 | 20000/tcp, 20000/udp | TCP/UDP | Utilities (power, water) |
| EtherNet/IP | 44818/tcp, 2222/udp | TCP/UDP | Allen-Bradley PLC |
| PROFINET | 34962-34964/udp | UDP | Siemens, Rockwell |
| BACnet | 47808/udp | UDP | Building automation |
| Siemens S7 | 102/tcp | TCP | Siemens PLC (S7-300/400) |
| FINS | 9600/udp | UDP | Omron PLC |
| Mitsubishi MELSEC | 5006/tcp | TCP | Mitsubishi PLC |
| IEC 60870-5-104 | 2404/tcp | TCP | Power grid |
| OPC-UA | 4840/tcp | TCP | Vendor-neutral OT |
| MQTT | 1883/tcp | TCP | IoT/telemetry |
| CoAP | 5683/udp | UDP | IoT constrained devices |

---

## Nmap NSE Scripts untuk ICS

### Modbus

```bash
# Modbus discovery dan device identification
nmap --script modbus-discover -p 502 192.168.1.0/24

# Output:
# | modbus-discover:
# |   sid=0x1:
# |     Slave ID data: \xFF\x84\x00
# |     Device identification: Schneider Electric QUANTUM
# |   sid=0x64:
# |     Slave ID data: ...

# Modbus enum - baca holding registers
nmap --script modbus-enum -p 502 192.168.1.10
```

### Siemens S7

```bash
# S7comm protocol (port 102) - ISO-TSAP transport
nmap --script s7-info -p 102 192.168.1.10

# Output:
# | s7-info:
# |   Module: 6ES7 315-2EH14-0AB0
# |   Basic Firmware: V3.3.13
# |   Module Type: CPU 315-2 PN/DP
# |   Serial Number: S C-XXXXXXXX
# |   Plant Identification: (empty)
# |   Copyright: Original Siemens Equipment
```

### EtherNet/IP

```bash
# Allen-Bradley / Rockwell Automation PLC
nmap --script enip-info -p 44818 192.168.1.10

# Output mengungkap:
# Device type, product code, firmware revision
# Vendor ID, serial number
```

### BACnet (Building Automation)

```bash
# BACnet/IP untuk sistem HVAC, elevator, access control
nmap --script bacnet-info -sU -p 47808 192.168.1.10

# BACnet yang exposed bisa kontrol suhu ruangan, akses gedung
nmap --script bacnet-discover -sU -p 47808 192.168.1.0/24
```

### DNP3

```bash
# DNP3 untuk power/water utilities
nmap --script dnp3-info -p 20000 192.168.1.10

# Cari DNP3 over Ethernet dengan timing sangat hati-hati
nmap -T2 --max-parallelism 1 --script dnp3-info -p 20000 192.168.1.10
```

---

## ICS-Safe Scanning Practices

```bash
# SEMPRE gunakan timing yang sangat rendah untuk ICS
# T0 atau T1 saja, TIDAK PERNAH T4/T5

# Gunakan --max-rate untuk batasi bandwidth
sudo nmap -T1 --max-rate 10 --max-parallelism 1 \
     -p 102,502,44818 \
     --script s7-info,modbus-discover,enip-info \
     192.168.100.0/24

# Hindari scanning saat jam operasional puncak
# Koordinasi dengan tim OT
# Backup/snapshot kondisi sebelum scan

# Hindari port scan agresif (OS detection, full port)
# Gunakan -sV dengan intensitas rendah
sudo nmap -sV --version-intensity 1 -p 102,502,44818 192.168.100.10
```

---

## IoT Device Scanning

### Protokol IoT Umum

```bash
# MQTT (IoT messaging bus)
nmap --script mqtt-subscribe -p 1883 192.168.1.10
nmap --script mqtt-subscribe \
     --script-args mqtt.topic='#',mqtt.qos=0 \
     -p 1883 192.168.1.10

# Jika MQTT tanpa auth, bisa subscribe ke semua topic
# Sering berisi: sensor data, device commands, credentials

# CoAP (Constrained Application Protocol)
nmap -sU --script coap-resources -p 5683 192.168.1.10
```

### UPnP Exploitation

```bash
# UPnP sering aktif di router, NAS, smart TV, IP camera
nmap --script upnp-info -p 1900 -sU 192.168.1.0/24

# Dapatkan detail device dari SSDP reply
nmap --script broadcast-upnp-info

# UPnP SOAP injection (command ke router)
nmap --script upnp-info 192.168.1.1
```

### IP Camera (RTSP)

```bash
# RTSP (Real-Time Streaming Protocol) - IP cameras
nmap --script rtsp-url-brute -p 554 192.168.1.0/24

# Cari default credentials camera
nmap --script rtsp-url-brute \
     --script-args brute.firstonly=true \
     -p 554 192.168.1.10

# HTTP interface kamera
nmap -sV --script http-title,http-auth-finder -p 80,8080,443 \
     192.168.1.0/24 | grep -i "camera\|dvr\|nvr\|ipcam\|hikvision\|dahua\|axis"
```

### Industrial Router & Gateway

```bash
# Moxa serial-to-ethernet gateway
nmap --script moxa-enum -p 4800 192.168.1.10

# Lantronix device server
nmap --script lantronix-localinfo -p 30718 -sU 192.168.1.10

# Sierra Wireless gateway
nmap -sV -p 22,80,443 --script banner \
     192.168.1.0/24 | grep -i "sierra\|raven\|airlink"
```

---

## Shodan & Censys untuk ICS/IoT

```bash
# ICS assets di internet (gunakan dari host kamu sendiri / legal scope)

# Shodan CLI untuk Modbus devices
shodan search "port:502 modbus" --fields ip_str,port,org,country

# Python dengan Shodan API
python3 << 'EOF'
import shodan

api = shodan.Shodan("YOUR_API_KEY")
results = api.search('port:102 Siemens')
for r in results['matches'][:5]:
    print(f"{r['ip_str']} - {r.get('org','?')} - {r['_shodan']['module']}")
EOF

# Queries Shodan berguna:
# port:44818 product:ethernet/ip
# port:502 modbus
# port:102 s7comm
# port:47808 bacnet
# port:1883 mqtt
```

---

## Industrial Protocol NSE Collection

```bash
# Semua ICS scripts sekaligus (gunakan di lab)
nmap --script "ics-* or modbus-* or enip-* or s7-* or bacnet-* or dnp3-*" \
     -p 102,502,20000,44818,47808 \
     -T2 --max-rate 20 \
     192.168.100.0/24

# Scan Modbus network dengan logging komprehensif
sudo nmap -sS -sV \
     --script modbus-discover \
     --script-args modbus.unit_id=0-247 \
     -p 502 192.168.100.0/24 \
     -oA modbus_scan \
     --reason \
     -T1
```

---

## Threat Intelligence: ICS CVE Correlation

```bash
# Setelah identifikasi hardware, cari CVE terkait
# Contoh: S7-300 CPU + firmware version dari scan

# Gunakan searchsploit untuk quick lookup
searchsploit "Siemens S7"
searchsploit "Modbus"
searchsploit "EtherNet IP"

# Gunakan NIST NVD API
python3 << 'EOF'
import requests

# Cari CVE untuk produk tertentu
resp = requests.get(
    "https://services.nvd.nist.gov/rest/json/cves/2.0",
    params={
        "keywordSearch": "Siemens SIMATIC S7-300",
        "startIndex": 0,
        "resultsPerPage": 5
    }
)
for cve in resp.json().get('vulnerabilities', []):
    cve_id = cve['cve']['id']
    desc = cve['cve']['descriptions'][0]['value'][:100]
    print(f"{cve_id}: {desc}...")
EOF
```

---

*← [BAB 07: Cloud & Container](bab07.md) · Selanjutnya → [BAB 09: Evading EDR/NGFW](bab09.md)*
