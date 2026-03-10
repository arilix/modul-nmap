# BAB 10 — Building a Scan Automation Pipeline

> **NMAP Advanced Guide · 2026 Edition**

---

## Arsitektur Pipeline Scanning Modern

```
┌─────────────┐    ┌──────────────┐    ┌────────────────┐    ┌──────────────┐
│ Trigger     │───▶│ Scan Engine  │───▶│ Parser/Enricher│───▶│ Output       │
│             │    │              │    │                │    │              │
│ - Cron job  │    │ - Nmap       │    │ - XML parse    │    │ - DefectDojo │
│ - CI/CD     │    │ - Masscan    │    │ - CVE lookup   │    │ - JIRA       │
│ - Webhook   │    │ - Custom NSE │    │ - ndiff        │    │ - Slack      │
│ - Manual    │    │              │    │ - CVSS scoring │    │ - ElasticSECM│
└─────────────┘    └──────────────┘    └────────────────┘    └──────────────┘
```

---

## Scan Manager: Full Python Implementation

```python
#!/usr/bin/env python3
"""
scan_manager.py — Advanced Nmap Automation Pipeline
"""

import subprocess
import xml.etree.ElementTree as ET
import json
import hashlib
import sqlite3
import smtplib
import requests
from datetime import datetime, timedelta
from pathlib import Path
from email.mime.text import MIMEText
from typing import Optional


class ScanManager:
    def __init__(self, db_path: str = "/var/db/scans.db"):
        self.db_path = db_path
        self.conn = sqlite3.connect(db_path)
        self._init_db()
    
    def _init_db(self):
        self.conn.executescript("""
            CREATE TABLE IF NOT EXISTS scans (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                target TEXT NOT NULL,
                scan_date DATETIME DEFAULT CURRENT_TIMESTAMP,
                scan_type TEXT,
                ports_found TEXT,  -- JSON array
                services TEXT,     -- JSON object
                xml_hash TEXT,
                xml_path TEXT
            );
            
            CREATE TABLE IF NOT EXISTS port_history (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                scan_id INTEGER,
                ip TEXT,
                port INTEGER,
                protocol TEXT,
                state TEXT,
                service TEXT,
                version TEXT,
                first_seen DATETIME DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(scan_id) REFERENCES scans(id)
            );
            
            CREATE TABLE IF NOT EXISTS changes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                detected_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                change_type TEXT,  -- new_port, closed_port, new_host, version_change
                target TEXT,
                port INTEGER,
                old_value TEXT,
                new_value TEXT,
                alerted INTEGER DEFAULT 0
            );
        """)
        self.conn.commit()
    
    def run_scan(self, target: str, ports: str = "top1000",
                 scan_type: str = "quick") -> Path:
        """Execute nmap scan and return path to XML output"""
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        safe_target = target.replace("/", "_").replace(".", "_")
        xml_path = Path(f"/tmp/nmap_{safe_target}_{timestamp}.xml")
        
        # Build nmap command
        cmd = ["nmap", "-oX", str(xml_path), "--reason"]
        
        if scan_type == "quick":
            cmd += ["-sS", "-T4", f"--top-ports 1000"]
        elif scan_type == "full":
            cmd += ["-sS", "-sV", "-sC", "-T3", "-p-"]
        elif scan_type == "stealth":
            cmd += ["-sS", "-T1", "--scan-delay", "5s", "--max-rate", "10"]
            cmd += ["-p", "22,80,443,8080,8443,3389,5432,3306,1433"]
        
        if ports != "top1000":
            cmd += ["-p", ports]
        
        cmd.append(target)
        
        print(f"[*] Running: {' '.join(cmd)}")
        proc = subprocess.run(cmd, capture_output=True, text=True)
        
        if proc.returncode != 0:
            raise RuntimeError(f"Nmap failed: {proc.stderr}")
        
        return xml_path
    
    def parse_xml(self, xml_path: Path) -> dict:
        """Parse nmap XML into structured dict"""
        tree = ET.parse(xml_path)
        root = tree.getroot()
        
        hosts = {}
        for host in root.findall('host'):
            addr = host.find('.//address[@addrtype="ipv4"]')
            if addr is None:
                continue
            ip = addr.get('addr')
            
            # Hostname
            hostname = host.findtext('.//hostname/@name') or ""
            
            # Ports
            ports = []
            for port in host.findall('.//port'):
                state = port.find('state')
                if state is None or state.get('state') != 'open':
                    continue
                
                service = port.find('service')
                version_str = ""
                if service is not None:
                    parts = [service.get('product',''), service.get('version','')]
                    version_str = " ".join(p for p in parts if p)
                
                ports.append({
                    'port': int(port.get('portid')),
                    'protocol': port.get('protocol'),
                    'service': service.get('name','?') if service else '?',
                    'version': version_str,
                    'reason': state.get('reason','')
                })
            
            hosts[ip] = {
                'hostname': hostname,
                'ports': ports
            }
        
        return hosts
    
    def save_and_diff(self, target: str, current: dict) -> list:
        """Save scan results and return list of changes"""
        changes = []
        
        # Load previous scan for this target
        cursor = self.conn.execute(
            "SELECT id, ports_found FROM scans WHERE target=? ORDER BY scan_date DESC LIMIT 1",
            (target,)
        )
        prev_row = cursor.fetchone()
        
        # Detect changes
        if prev_row:
            prev_data = json.loads(prev_row[1])
            
            for ip, data in current.items():
                prev_ip = prev_data.get(ip, {})
                curr_ports = {p['port'] for p in data['ports']}
                prev_ports = {p['port'] for p in prev_ip.get('ports', [])}
                
                for p in curr_ports - prev_ports:
                    changes.append({
                        'type': 'new_port',
                        'ip': ip,
                        'port': p,
                        'detail': next(x for x in data['ports'] if x['port'] == p)
                    })
                
                for p in prev_ports - curr_ports:
                    changes.append({
                        'type': 'closed_port',
                        'ip': ip,
                        'port': p,
                        'detail': None
                    })
            
            for ip in set(current.keys()) - set(prev_data.keys()):
                changes.append({'type': 'new_host', 'ip': ip, 'port': None})
        
        # Save current scan
        self.conn.execute(
            "INSERT INTO scans (target, scan_type, ports_found) VALUES (?,?,?)",
            (target, 'automated', json.dumps(current))
        )
        self.conn.commit()
        
        return changes
    
    def alert_slack(self, webhook_url: str, changes: list, target: str):
        if not changes:
            return
        
        text = f"*Nmap Scan Alert* — `{target}` — {datetime.now().strftime('%Y-%m-%d %H:%M')}\n"
        for c in changes[:10]:  # Max 10 changes
            if c['type'] == 'new_port':
                detail = c.get('detail', {})
                text += f"> :warning: NEW PORT: `{c['ip']}:{c['port']}` — {detail.get('service','')} {detail.get('version','')}\n"
            elif c['type'] == 'closed_port':
                text += f"> :x: CLOSED PORT: `{c['ip']}:{c['port']}`\n"
            elif c['type'] == 'new_host':
                text += f"> :new: NEW HOST: `{c['ip']}`\n"
        
        if len(changes) > 10:
            text += f"> _...and {len(changes)-10} more changes_\n"
        
        requests.post(webhook_url, json={"text": text}, timeout=10)


# Main usage
if __name__ == "__main__":
    import sys
    
    target = sys.argv[1] if len(sys.argv) > 1 else "192.168.1.0/24"
    manager = ScanManager()
    xml_path = manager.run_scan(target, scan_type="quick")
    current = manager.parse_xml(xml_path)
    changes = manager.save_and_diff(target, current)
    
    print(f"[+] Hosts found: {len(current)}")
    print(f"[+] Changes detected: {len(changes)}")
    for c in changes:
        print(f"    {c['type']}: {c['ip']}:{c.get('port','N/A')}")
    
    if changes and "SLACK_WEBHOOK" in __import__('os').environ:
        manager.alert_slack(__import__('os').environ["SLACK_WEBHOOK"], changes, target)
```

---

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/network-scan.yml
name: Scheduled Network Scan

on:
  schedule:
    - cron: '0 2 * * *'  # Setiap hari jam 02:00 UTC
  workflow_dispatch:      # Manual trigger

jobs:
  nmap-scan:
    runs-on: self-hosted  # Runner di internal network
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Install Nmap
        run: sudo apt-get install -y nmap python3-requests
      
      - name: Run Scan
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SCAN_TARGET: ${{ vars.SCAN_TARGET }}
        run: |
          python3 scan_manager.py "$SCAN_TARGET"
      
      - name: Upload Results
        uses: actions/upload-artifact@v4
        with:
          name: scan-results-${{ github.run_id }}
          path: /tmp/nmap_*.xml
          retention-days: 30
```

---

## DefectDojo Integration

```python
import requests

def upload_to_defectdojo(xml_path: str, product_id: int, engagement_id: int,
                          url: str, api_key: str):
    """Upload Nmap scan to DefectDojo"""
    
    with open(xml_path, 'rb') as f:
        resp = requests.post(
            f"{url}/api/v2/import-scan/",
            headers={"Authorization": f"Token {api_key}"},
            data={
                "scan_type": "Nmap Scan",
                "engagement": engagement_id,
                "product_id": product_id,
                "scan_date": datetime.now().strftime("%Y-%m-%d"),
                "verified": "false",
                "active": "true",
                "minimum_severity": "Info"
            },
            files={"file": ("scan.xml", f, "text/xml")},
            timeout=30
        )
    
    if resp.status_code == 201:
        print(f"[+] Uploaded to DefectDojo: {resp.json().get('test_id')}")
    else:
        print(f"[-] Upload failed: {resp.status_code} - {resp.text}")
```

---

## Scheduled Scanning dengan Cron

```bash
# /etc/cron.d/nmap-monitoring
# Setiap hari jam 03:00
0 3 * * * root /opt/scan_manager.py 192.168.0.0/16 >> /var/log/scan.log 2>&1

# Setiap Senin jam 01:00 - full port scan
0 1 * * 1 root /opt/scan_manager.py 192.168.0.0/16 --type full >> /var/log/scan_full.log 2>&1

# Setiap jam - cek host availability saja
0 * * * * root nmap -sn 192.168.1.0/24 -oG /var/tmp/hosts_$(date +\%H\%M).txt 2>/dev/null
```

---

## HTML Report Generation

```python
def generate_html_report(scan_data: dict, output_path: str, title: str = "Nmap Report"):
    """Generate HTML report dari parsed scan data"""
    
    html = f"""<!DOCTYPE html>
<html><head>
<title>{title}</title>
<style>
body {{ font-family: Arial; margin: 20px; background: #1a1a2e; color: #eee; }}
h1 {{ color: #e94560; }} h2 {{ color: #0f3460; background: #16213e; padding: 8px; }}
table {{ border-collapse: collapse; width: 100%; margin: 10px 0; }}
th {{ background: #0f3460; color: white; padding: 8px; text-align: left; }}
td {{ padding: 6px 8px; border-bottom: 1px solid #333; }}
.open {{ color: #4caf50; }} .filtered {{ color: #ff9800; }}
.critical {{ background: rgba(233,69,96,0.2); }}
</style></head><body>
<h1>{title}</h1>
<p>Generated: {datetime.now().isoformat()}</p>
<p>Hosts: {len(scan_data)} | Total open ports: {sum(len(h['ports']) for h in scan_data.values())}</p>
"""
    
    for ip, data in sorted(scan_data.items()):
        if not data['ports']:
            continue
        hostname = data.get('hostname', '')
        html += f"<h2>{ip} {f'({hostname})' if hostname else ''}</h2>\n"
        html += "<table><tr><th>Port</th><th>Service</th><th>Version</th><th>Reason</th></tr>\n"
        
        for p in sorted(data['ports'], key=lambda x: x['port']):
            row_class = "critical" if p['port'] in [21,23,3389,1433,3306] else ""
            html += f"<tr class='{row_class}'>"
            html += f"<td class='open'>{p['port']}/{p['protocol']}</td>"
            html += f"<td>{p['service']}</td><td>{p['version']}</td>"
            html += f"<td>{p.get('reason','')}</td></tr>\n"
        
        html += "</table>\n"
    
    html += "</body></html>"
    
    with open(output_path, 'w') as f:
        f.write(html)
    
    print(f"[+] Report saved: {output_path}")
```

---

*← [BAB 09: Evading EDR/NGFW](bab09.md) · Selanjutnya → [BAB 11: Red Team Operations](bab11.md)*
