# BAB 13 — Threat Hunting dengan Nmap (Blue Team Perspective)

> **NMAP Advanced Guide · 2026 Edition**

---

## Nmap sebagai Alat Defensif

Nmap bukan hanya tool ofensif. Blue team dan security operations menggunakan Nmap untuk:
- **Baselining**: Inventarisasi semua assets dan services yang seharusnya ada
- **Change Detection**: Alert ketika ada port baru, host baru, atau service berubah
- **Rogue Device Detection**: Temukan device tidak sah di jaringan
- **Compliance Auditing**: Verifikasi policy diterapkan dengan benar
- **Incident Response**: Rapid assessment selama insiden

---

## Baseline Network Inventory

```bash
# Buat baseline awal: semua host dan services aktif
sudo nmap -sS -sV -O -T4 \
     10.0.0.0/8 \
     --open \
     --reason \
     -oX /var/db/nmap_baseline_$(date +%Y%m%d).xml

# Update baseline bulanan
sudo nmap -sS -sV -T4 10.0.0.0/8 --open -oX /var/db/nmap_$(date +%Y%m%d).xml

# Bandingkan dengan baseline
ndiff /var/db/nmap_baseline_20240101.xml /var/db/nmap_$(date +%Y%m%d).xml
```

---

## ndiff: Network Difference Detection

`ndiff` adalah tool bawaan Nmap untuk compare dua scan.

```bash
# Bandingkan dua scan
ndiff scan_old.xml scan_new.xml

# Output format:
# +192.168.1.55: (dihadiri kali ini, tidak ada sebelumnya) = HOST BARU
# -192.168.1.20: (ada sebelumnya, tidak ada kali ini) = HOST HILANG
# 192.168.1.10:
#   +tcp/8080 open  http  Apache httpd 2.4.51  = PORT BARU
#   -tcp/21   open  ftp   vsftpd 3.0.3         = PORT HILANG

# Verbose output
ndiff --verbose scan_old.xml scan_new.xml

# Hanya tampilkan perubahan penting
ndiff scan_old.xml scan_new.xml | grep "^[+-]" | grep -v "^---"
```

---

## Automated Change Monitoring

```python
#!/usr/bin/env python3
"""
threat_hunt.py — Network Baseline Monitoring
"""

import subprocess
import xml.etree.ElementTree as ET
import json
import sqlite3
from datetime import datetime, timedelta
from pathlib import Path


class NetworkMonitor:
    
    CRITICAL_PORTS = {
        21: "FTP (cleartext)",
        23: "Telnet (cleartext)",
        445: "SMB (potential lateral movement)",
        4444: "Metasploit default listener",
        5900: "VNC (remote access)",
        6667: "IRC (C2 channel)",
        1337: "Common backdoor port",
        31337: "Elite backdoor port",
        50050: "Cobalt Strike team server",
    }
    
    NEVER_OPEN = [4444, 6667, 31337, 50050, 1337, 9001]  # Red flags
    
    def __init__(self, db_path="/var/db/network_monitor.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_db()
    
    def _init_db(self):
        self.conn.executescript("""
            CREATE TABLE IF NOT EXISTS hosts (
                ip TEXT PRIMARY KEY,
                hostname TEXT,
                first_seen DATETIME,
                last_seen DATETIME,
                os_guess TEXT
            );
            CREATE TABLE IF NOT EXISTS services (
                ip TEXT,
                port INTEGER,
                protocol TEXT,
                service TEXT,
                version TEXT,
                first_seen DATETIME,
                last_seen DATETIME,
                still_open INTEGER DEFAULT 1,
                PRIMARY KEY (ip, port, protocol)
            );
            CREATE TABLE IF NOT EXISTS findings (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                detected_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                severity TEXT,
                category TEXT,
                ip TEXT,
                port INTEGER,
                description TEXT,
                remediation TEXT,
                resolved INTEGER DEFAULT 0
            );
        """)
        self.conn.commit()
    
    def scan(self, targets: str) -> dict:
        """Run Nmap and return parsed data"""
        xml_out = f"/tmp/monitor_{datetime.now().strftime('%s')}.xml"
        subprocess.run(
            ["nmap", "-sS", "-sV", "-O", "--open", "-T3", targets, "-oX", xml_out],
            check=True, capture_output=True
        )
        return self._parse(xml_out)
    
    def _parse(self, xml_path: str) -> dict:
        tree = ET.parse(xml_path)
        hosts = {}
        for host in tree.findall('host'):
            addr = host.find('.//address[@addrtype="ipv4"]')
            if addr is None:
                continue
            ip = addr.get('addr')
            
            ports = []
            for p in host.findall('.//port'):
                state = p.find('state')
                if state is None or state.get('state') != 'open':
                    continue
                svc = p.find('service') or {}
                ports.append({
                    'port': int(p.get('portid')),
                    'protocol': p.get('protocol'),
                    'service': svc.get('name', '?') if hasattr(svc, 'get') else '?',
                    'version': f"{svc.get('product','')} {svc.get('version','')}".strip()
                             if hasattr(svc, 'get') else ''
                })
            
            hosts[ip] = {'ports': ports}
        return hosts
    
    def analyze(self, current_data: dict) -> list:
        """Analyze scan data for threats and changes"""
        findings = []
        now = datetime.now().isoformat()
        
        for ip, data in current_data.items():
            # Check host
            exists = self.conn.execute("SELECT 1 FROM hosts WHERE ip=?", (ip,)).fetchone()
            if not exists:
                findings.append({
                    'severity': 'MEDIUM',
                    'category': 'new_host',
                    'ip': ip,
                    'port': None,
                    'description': f"New host discovered: {ip}",
                    'remediation': "Verify this device is authorized on the network"
                })
                self.conn.execute(
                    "INSERT INTO hosts (ip, first_seen, last_seen) VALUES (?,?,?)",
                    (ip, now, now)
                )
            else:
                self.conn.execute("UPDATE hosts SET last_seen=? WHERE ip=?", (now, ip))
            
            for p in data['ports']:
                port_num = p['port']
                
                # Check for suspicious ports
                if port_num in self.NEVER_OPEN:
                    findings.append({
                        'severity': 'CRITICAL',
                        'category': 'suspicious_port',
                        'ip': ip,
                        'port': port_num,
                        'description': f"Highly suspicious port {port_num} open on {ip} — possible malware/C2",
                        'remediation': "Immediately investigate this host. Isolate from network."
                    })
                
                if port_num in self.CRITICAL_PORTS:
                    findings.append({
                        'severity': 'HIGH',
                        'category': 'risky_service',
                        'ip': ip,
                        'port': port_num,
                        'description': f"{self.CRITICAL_PORTS[port_num]} open on {ip}",
                        'remediation': "Verify if this service is required. Disable if not."
                    })
                
                # Check if port is new
                existing = self.conn.execute(
                    "SELECT version FROM services WHERE ip=? AND port=? AND protocol=?",
                    (ip, port_num, p['protocol'])
                ).fetchone()
                
                if not existing:
                    findings.append({
                        'severity': 'MEDIUM',
                        'category': 'new_service',
                        'ip': ip,
                        'port': port_num,
                        'description': f"New service: {ip}:{port_num} ({p['service']} {p['version']})",
                        'remediation': "Verify this service is authorized and properly secured"
                    })
                    self.conn.execute(
                        "INSERT OR REPLACE INTO services VALUES (?,?,?,?,?,?,?,1)",
                        (ip, port_num, p['protocol'], p['service'], p['version'], now, now)
                    )
                elif existing[0] != p['version'] and p['version']:
                    findings.append({
                        'severity': 'LOW',
                        'category': 'version_change',
                        'ip': ip,
                        'port': port_num,
                        'description': f"Version change on {ip}:{port_num}: '{existing[0]}' → '{p['version']}'",
                        'remediation': "Verify the update was authorized"
                    })
        
        self.conn.commit()
        return findings
    
    def report(self, findings: list):
        """Print findings summary"""
        by_sev = {'CRITICAL': [], 'HIGH': [], 'MEDIUM': [], 'LOW': []}
        for f in findings:
            by_sev.get(f['severity'], []).append(f)
        
        print(f"\n{'='*60}")
        print(f"THREAT HUNT REPORT — {datetime.now().strftime('%Y-%m-%d %H:%M')}")
        print(f"Total findings: {len(findings)}")
        print(f"{'='*60}")
        
        for sev in ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW']:
            items = by_sev[sev]
            if items:
                print(f"\n[{sev}] ({len(items)} findings)")
                for f in items:
                    port_str = f":{f['port']}" if f['port'] else ""
                    print(f"  • {f['ip']}{port_str} — {f['description']}")
                    print(f"    → {f['remediation']}")
```

---

## Rogue Device Detection

```bash
# MAC OUI lookup untuk identify unknown vendors
# Isi vendor database:
nmap --script broadcast-dhcp-discover 2>/dev/null | grep "MAC\|Vendor"

# Scan dan group by vendor/manufacturer
sudo nmap -sn 192.168.1.0/24 --oX - 2>/dev/null |
python3 -c "
import xml.etree.ElementTree as ET
import sys

tree = ET.fromstring(sys.stdin.read())
vendors = {}
for host in tree.findall('host'):
    mac_elem = host.find('.//address[@addrtype=\"mac\"]')
    if mac_elem is not None:
        vendor = mac_elem.get('vendor', 'Unknown')
        ip = host.find('.//address[@addrtype=\"ipv4\"]').get('addr')
        vendors.setdefault(vendor, []).append(ip)

for vendor, ips in sorted(vendors.items()):
    print(f'{vendor}: {len(ips)} devices')
    if vendor == 'Unknown' or vendor.startswith('('):
        for ip in ips:
            print(f'  !! INVESTIGATE: {ip}')
"
```

---

## Honeypot Detection

```bash
# Honeypot sering membuka SEMUA port (atau sangat banyak port)
# dan merespons dengan cepat

# Cek host dengan port terlalu banyak (>500 = suspicious)
sudo nmap -sS --top-ports 1000 192.168.1.0/24 --open -oG - |
awk '/Ports:/{
    n=0
    for(i=1; i<=NF; i++) if($i~/open/) n++
    if(n>50) print $2 " has " n " open ports - POSSIBLE HONEYPOT"
}'

# Cek response time yang terlalu konsisten (honeypot characteristic)
for port in 22 80 443; do
    time nmap -sS -p $port 192.168.1.50 --packet-trace 2>&1 | grep "RCVD"
done
```

---

## SIEM Integration (Elastic/Splunk)

```bash
# Kirim Nmap findings ke Elasticsearch
python3 << 'EOF'
import subprocess
import xml.etree.ElementTree as ET
import json
import requests
from datetime import datetime

def scan_and_index(target, es_url="http://localhost:9200"):
    # Run scan
    proc = subprocess.run(
        ["nmap", "-sS", "-sV", target, "--open", "-oX", "-"],
        capture_output=True, text=True
    )
    
    # Parse and index
    tree = ET.fromstring(proc.stdout)
    for host in tree.findall('host'):
        doc = {
            "@timestamp": datetime.utcnow().isoformat(),
            "scanner": "nmap",
            "target": target,
        }
        
        addr = host.find('.//address[@addrtype="ipv4"]')
        if addr is None:
            continue
        doc["ip"] = addr.get("addr")
        doc["ports"] = []
        
        for p in host.findall('.//port'):
            state = p.find('state')
            if state is None or state.get('state') != 'open':
                continue
            svc = p.find('service')
            doc["ports"].append({
                "number": int(p.get("portid")),
                "protocol": p.get("protocol"),
                "service": svc.get("name","?") if svc is not None else "?",
                "version": f"{svc.get('product','')} {svc.get('version','')}".strip()
                          if svc is not None else ""
            })
        
        # Index to Elasticsearch
        requests.post(
            f"{es_url}/nmap-scans/_doc",
            json=doc,
            headers={"Content-Type": "application/json"}
        )
        print(f"Indexed: {doc['ip']} ({len(doc['ports'])} ports)")

scan_and_index("192.168.1.0/24")
EOF
```

---

## Compliance Scan: Hardening Verification

```bash
# Cek compliance: tidak ada port terlarang yang terbuka
FORBIDDEN_PORTS="21,23,69,79,515,1521,3306,5432,27017,6379"

cat << 'EOF' > compliance_scan.sh
#!/bin/bash
NETWORK=$1
FAIL=0

echo "[*] Compliance scan of $NETWORK"
nmap -sS -p $FORBIDDEN_PORTS --open $NETWORK -oG - 2>/dev/null |
grep "^Host" | while read line; do
    ip=$(echo $line | awk '{print $2}')
    ports=$(echo $line | grep -oP '\d+/open')
    
    for p in $ports; do
        port=$(echo $p | cut -d/ -f1)
        echo "[FAIL] $ip has forbidden port $port open"
        FAIL=1
    done
done

if [ $FAIL -eq 0 ]; then
    echo "[PASS] No forbidden ports found"
fi
EOF
chmod +x compliance_scan.sh
```

---

*← [BAB 12: Performance Tuning](bab12.md) · Selanjutnya → [BAB 14: Custom Vulnerability Scanner](bab14.md)*
