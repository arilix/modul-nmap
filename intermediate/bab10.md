# BAB 10 — Parsing Output Nmap dengan Python & XML

> **NMAP Intermediate Guide · 2026 Edition**

---

## Mengapa Parse Output Nmap?

- Memproses ratusan/ribuan host secara otomatis
- Filter hasil sesuai kriteria tertentu
- Generate laporan custom (HTML, CSV, JSON)
- Integrasi dengan tools lain (Jira, Slack, database)
- Membandingkan scan dari waktu ke waktu

---

## Format Output XML Nmap

Nmap XML adalah format paling terstruktur dan mudah di-parse.

```bash
# Simpan scan ke XML
sudo nmap -sS -sV -O -T4 192.168.1.0/24 -oX scan.xml

# Simpan semua format (XML, normal, grepable)
sudo nmap -sS -sV -T4 192.168.1.0/24 -oA hasil_scan
```

### Struktur XML Nmap

```xml
<?xml version="1.0" encoding="UTF-8"?>
<nmaprun scanner="nmap" args="nmap -sV ..." start="1741000000" ...>

  <host starttime="..." endtime="...">
    <status state="up" reason="echo-reply"/>
    <address addr="192.168.1.1" addrtype="ipv4"/>
    <hostnames>
      <hostname name="router.local" type="PTR"/>
    </hostnames>
    <ports>
      <port protocol="tcp" portid="22">
        <state state="open" reason="syn-ack"/>
        <service name="ssh" product="OpenSSH" version="8.9" .../>
      </port>
      <port protocol="tcp" portid="80">
        <state state="open" reason="syn-ack"/>
        <service name="http" product="Apache httpd" version="2.4.52"/>
      </port>
    </ports>
    <os>
      <osmatch name="Linux 5.15" accuracy="95"/>
    </os>
  </host>

</nmaprun>
```

---

## Parsing dengan Python (xml.etree)

### Parser Sederhana

```python
import xml.etree.ElementTree as ET

def parse_nmap_xml(filename):
    tree = ET.parse(filename)
    root = tree.getroot()
    
    results = []
    
    for host in root.findall('host'):
        # Cek apakah host up
        status = host.find('status')
        if status is None or status.get('state') != 'up':
            continue
        
        # Ambil IP address
        addr_elem = host.find("address[@addrtype='ipv4']")
        ip = addr_elem.get('addr') if addr_elem is not None else 'unknown'
        
        # Ambil hostname
        hostname = ''
        hostnames = host.find('hostnames')
        if hostnames is not None:
            hn = hostnames.find('hostname')
            if hn is not None:
                hostname = hn.get('name', '')
        
        # Ambil port terbuka
        open_ports = []
        ports_elem = host.find('ports')
        if ports_elem is not None:
            for port in ports_elem.findall('port'):
                state = port.find('state')
                if state is not None and state.get('state') == 'open':
                    portid = port.get('portid')
                    proto = port.get('protocol')
                    
                    service = port.find('service')
                    svc_name = service.get('name', '') if service is not None else ''
                    svc_ver  = service.get('product', '') + ' ' + service.get('version', '') if service is not None else ''
                    
                    open_ports.append({
                        'port': int(portid),
                        'protocol': proto,
                        'service': svc_name,
                        'version': svc_ver.strip()
                    })
        
        results.append({
            'ip': ip,
            'hostname': hostname,
            'open_ports': open_ports
        })
    
    return results

# Gunakan
hasil = parse_nmap_xml('scan.xml')
for host in hasil:
    print(f"\n[+] {host['ip']} ({host['hostname']})")
    for p in host['open_ports']:
        print(f"    {p['port']}/{p['protocol']}  {p['service']}  {p['version']}")
```

---

## Parsing dengan Library `python-libnmap`

Library ini lebih lengkap dan mudah digunakan:

```bash
pip install python-libnmap
```

```python
from libnmap.parser import NmapParser

# Parse dari file
report = NmapParser.parse_fromfile('scan.xml')

print(f"Scan selesai: {report.endtime}")
print(f"Total hosts: {len(report.hosts)}")

for host in report.hosts:
    if host.is_up():
        print(f"\n[+] {host.address} ({host.hostnames})")
        print(f"    OS: {host.os_fingerprint}")
        
        for svc in host.services:
            if svc.state == 'open':
                print(f"    {svc.port}/{svc.protocol}  {svc.service}  {svc.banner}")
```

---

## Generate Laporan CSV

```python
import csv
import xml.etree.ElementTree as ET

def nmap_to_csv(xml_file, csv_file):
    hasil = parse_nmap_xml(xml_file)  # Fungsi dari contoh sebelumnya
    
    with open(csv_file, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['IP', 'Hostname', 'Port', 'Protocol', 'Service', 'Version'])
        
        for host in hasil:
            for port in host['open_ports']:
                writer.writerow([
                    host['ip'],
                    host['hostname'],
                    port['port'],
                    port['protocol'],
                    port['service'],
                    port['version']
                ])
    
    print(f"CSV tersimpan di: {csv_file}")

nmap_to_csv('scan.xml', 'hasil_scan.csv')
```

---

## Generate Laporan HTML

```python
def nmap_to_html(xml_file, html_file):
    hasil = parse_nmap_xml(xml_file)
    
    html = """<!DOCTYPE html>
<html><head>
<style>
  body { font-family: monospace; background: #1a1a1a; color: #00ff00; }
  table { border-collapse: collapse; width: 100%; }
  th, td { border: 1px solid #333; padding: 8px; }
  th { background: #333; }
  .host { color: #ffff00; font-size: 1.1em; margin-top: 20px; }
</style>
</head><body>
<h1>Nmap Scan Report</h1>
"""
    
    for host in hasil:
        html += f'<p class="host">► {host["ip"]} {host["hostname"]}</p>'
        html += '<table><tr><th>Port</th><th>Protocol</th><th>Service</th><th>Version</th></tr>'
        for p in host['open_ports']:
            html += f'<tr><td>{p["port"]}</td><td>{p["protocol"]}</td>'
            html += f'<td>{p["service"]}</td><td>{p["version"]}</td></tr>'
        html += '</table>'
    
    html += '</body></html>'
    
    with open(html_file, 'w') as f:
        f.write(html)
    
    print(f"HTML report: {html_file}")

nmap_to_html('scan.xml', 'report.html')
```

---

## Membandingkan Dua Scan (Ndiff)

Nmap menyertakan tool `ndiff` untuk membandingkan dua scan:

```bash
# Bandingkan scan minggu lalu vs sekarang
ndiff scan_lama.xml scan_baru.xml

# Output ringkas
ndiff --verbose scan_lama.xml scan_baru.xml

# Hanya tampilkan perubahan
ndiff scan_lama.xml scan_baru.xml | grep -E "^[+-]"
```

**Output contoh:**
```
-192.168.1.5: Host is up.
+192.168.1.5: Host is up.
 80/tcp  open  http
+443/tcp open  https      ← port baru terbuka!
-22/tcp  open  ssh        ← port ini sekarang tertutup!
```

---

## Workflow Otomatis: Scan + Parse + Alert

```python
import subprocess
import xml.etree.ElementTree as ET

def run_and_parse(target):
    # Jalankan nmap
    cmd = ['sudo', 'nmap', '-sS', '-sV', '-T4', '--open', '-oX', '/tmp/scan.xml', target]
    subprocess.run(cmd, capture_output=True)
    
    # Parse hasil
    hasil = parse_nmap_xml('/tmp/scan.xml')
    
    # Cari port berbahaya
    dangerous = {22, 23, 3389, 5900, 4444, 1337}
    
    for host in hasil:
        for port in host['open_ports']:
            if port['port'] in dangerous:
                print(f"⚠️  ALERT: {host['ip']}:{port['port']} ({port['service']}) OPEN!")

run_and_parse('192.168.1.0/24')
```

---

*← [BAB 09: Scanning Jaringan Besar](bab09.md) · Selanjutnya → [BAB 11: Advanced Evasion Techniques](bab11.md)*
