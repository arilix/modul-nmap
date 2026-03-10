# BAB 14 — Membangun Custom Vulnerability Scanner

> **NMAP Advanced Guide · 2026 Edition**

---

## Arsitektur Custom Vuln Scanner

```
┌─────────────────────────────────────────────────────────────────┐
│                    Custom Vuln Scanner                          │
├──────────┬──────────────┬──────────────┬───────────────────────┤
│ Service  │  NSE Plugin  │  CVE Matcher │  CVSS Scorer          │
│ Detector │  Layer       │  Engine      │  + Remediation        │
│ (Nmap)   │  (Lua)       │  (Python)    │  (Report Generator)   │
└──────────┴──────────────┴──────────────┴───────────────────────┘
```

---

## Fase 1: Service Enumeration Layer

```python
#!/usr/bin/env python3
"""
vuln_scanner.py — Custom NSE-Based Vulnerability Scanner
"""

import subprocess
import xml.etree.ElementTree as ET
import json
import requests
import re
from dataclasses import dataclass, field, asdict
from typing import List, Optional
from pathlib import Path


@dataclass
class Service:
    ip: str
    port: int
    protocol: str
    name: str
    product: str
    version: str
    extra_info: str
    cpe: List[str] = field(default_factory=list)
    
    def version_tuple(self) -> Optional[tuple]:
        """Parse version string to comparable tuple"""
        match = re.search(r'(\d+)\.(\d+)(?:\.(\d+))?', self.version)
        if match:
            return tuple(int(x or 0) for x in match.groups())
        return None


@dataclass
class Vulnerability:
    cve_id: str
    cvss_score: float
    cvss_vector: str
    severity: str  # CRITICAL/HIGH/MEDIUM/LOW
    description: str
    service: Service
    remediation: str = ""
    references: List[str] = field(default_factory=list)


class ServiceScanner:
    def __init__(self):
        self.services: List[Service] = []
    
    def scan(self, target: str, ports: str = "-") -> List[Service]:
        """Run Nmap and collect services"""
        cmd = [
            "nmap", "-sS", "-sV", "--version-all",
            "-p", ports, "--open", target, "-oX", "-"
        ]
        
        proc = subprocess.run(cmd, capture_output=True, text=True)
        if proc.returncode != 0:
            raise RuntimeError(f"Scan failed: {proc.stderr}")
        
        return self._parse_xml(proc.stdout)
    
    def _parse_xml(self, xml_str: str) -> List[Service]:
        services = []
        root = ET.fromstring(xml_str)
        
        for host in root.findall('host'):
            addr = host.find('.//address[@addrtype="ipv4"]')
            if addr is None:
                continue
            ip = addr.get('addr')
            
            for p in host.findall('.//port'):
                state = p.find('state')
                if state is None or state.get('state') != 'open':
                    continue
                
                svc = p.find('service')
                if svc is None:
                    continue
                
                # Collect CPE identifiers
                cpes = [c.get('wk', '') for c in p.findall('.//cpe')]
                
                services.append(Service(
                    ip=ip,
                    port=int(p.get('portid')),
                    protocol=p.get('protocol'),
                    name=svc.get('name', ''),
                    product=svc.get('product', ''),
                    version=svc.get('version', ''),
                    extra_info=svc.get('extrainfo', ''),
                    cpe=cpes
                ))
        
        self.services = services
        return services
```

---

## Fase 2: CVE Matching Engine

```python
class CVEMatcher:
    NVD_BASE = "https://services.nvd.nist.gov/rest/json/cves/2.0"
    
    def __init__(self, cache_dir: str = "/var/cache/vuln_scanner"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
    
    def lookup_cpe(self, cpe: str) -> List[dict]:
        """Query NVD by CPE identifier"""
        cache_file = self.cache_dir / f"{cpe.replace('/', '_').replace(':', '_')}.json"
        
        if cache_file.exists() and (cache_file.stat().st_mtime > (__import__('time').time() - 86400)):
            with open(cache_file) as f:
                return json.load(f)
        
        try:
            resp = requests.get(
                self.NVD_BASE,
                params={"cpeName": cpe, "resultsPerPage": 20},
                timeout=15
            )
            resp.raise_for_status()
            cves = resp.json().get('vulnerabilities', [])
            
            with open(cache_file, 'w') as f:
                json.dump(cves, f)
            
            return cves
        except requests.RequestException:
            return []
    
    def lookup_keyword(self, product: str, version: str) -> List[dict]:
        """Query NVD by product keyword"""
        keyword = f"{product} {version}".strip()
        
        try:
            resp = requests.get(
                self.NVD_BASE,
                params={"keywordSearch": keyword, "resultsPerPage": 10},
                timeout=15
            )
            resp.raise_for_status()
            return resp.json().get('vulnerabilities', [])
        except requests.RequestException:
            return []
    
    def parse_cve(self, cve_data: dict, service: Service) -> Optional[Vulnerability]:
        """Parse CVE data into Vulnerability object"""
        cve = cve_data.get('cve', {})
        cve_id = cve.get('id', '')
        
        # Get CVSS score
        metrics = cve.get('metrics', {})
        cvss_score = 0.0
        cvss_vector = "N/A"
        
        for version in ['cvssMetricV31', 'cvssMetricV30', 'cvssMetricV2']:
            metrics_list = metrics.get(version, [])
            if metrics_list:
                cvss_data = metrics_list[0].get('cvssData', {})
                cvss_score = cvss_data.get('baseScore', 0.0)
                cvss_vector = cvss_data.get('vectorString', 'N/A')
                break
        
        if cvss_score == 0.0:
            return None
        
        # Severity mapping
        if cvss_score >= 9.0:
            severity = "CRITICAL"
        elif cvss_score >= 7.0:
            severity = "HIGH"
        elif cvss_score >= 4.0:
            severity = "MEDIUM"
        else:
            severity = "LOW"
        
        # Get description
        desc = ""
        for d in cve.get('descriptions', []):
            if d.get('lang') == 'en':
                desc = d.get('value', '')
                break
        
        # References
        refs = [r.get('url', '') for r in cve.get('references', [])[:3]]
        
        return Vulnerability(
            cve_id=cve_id,
            cvss_score=cvss_score,
            cvss_vector=cvss_vector,
            severity=severity,
            description=desc,
            service=service,
            references=refs
        )
    
    def match_service(self, service: Service) -> List[Vulnerability]:
        """Find all CVEs matching a service"""
        vulns = []
        cve_ids_seen = set()
        
        # Method 1: CPE lookup (more accurate)
        for cpe in service.cpe:
            for cve_data in self.lookup_cpe(cpe):
                cve_id = cve_data.get('cve', {}).get('id', '')
                if cve_id not in cve_ids_seen:
                    vuln = self.parse_cve(cve_data, service)
                    if vuln:
                        vulns.append(vuln)
                        cve_ids_seen.add(cve_id)
        
        # Method 2: Keyword lookup (fallback)
        if not vulns and service.product and service.version:
            for cve_data in self.lookup_keyword(service.product, service.version):
                cve_id = cve_data.get('cve', {}).get('id', '')
                if cve_id not in cve_ids_seen:
                    vuln = self.parse_cve(cve_data, service)
                    if vuln:
                        vulns.append(vuln)
                        cve_ids_seen.add(cve_id)
        
        return sorted(vulns, key=lambda v: v.cvss_score, reverse=True)
```

---

## Fase 3: NSE Custom Check Layer

```lua
-- cve_check.nse - NSE script yang check CVE berdasarkan version
local shortport = require "shortport"
local stdnse = require "stdnse"
local http = require "http"
local json = require "json"
local nmap = require "nmap"

description = "Checks service version against NVD CVE database"
author = "Advanced Nmap Module"
categories = {"vuln", "safe"}

-- Prorule: jalankan jika ada versi yang terdeteksi
portrule = function(host, port)
    return port.version and port.version.product ~= nil
end

action = function(host, port)
    local product = port.version.product or ""
    local version = port.version.version or ""
    
    if product == "" then
        return nil
    end
    
    -- Query NVD API
    local query = (product .. " " .. version):gsub(" ", "+")
    local resp = http.get(
        "services.nvd.nist.gov",
        443,
        "/rest/json/cves/2.0?keywordSearch=" .. query .. "&resultsPerPage=5",
        {timeout=10000, any_af=true}
    )
    
    if not resp or resp.status ~= 200 then
        return nil
    end
    
    local ok, data = json.parse(resp.body)
    if not ok or not data.vulnerabilities then
        return nil
    end
    
    -- Collect high/critical CVEs only
    local output = {}
    for _, item in ipairs(data.vulnerabilities) do
        local cve = item.cve
        local metrics = cve.metrics or {}
        local score = 0
        
        for _, mver in ipairs({"cvssMetricV31", "cvssMetricV30", "cvssMetricV2"}) do
            local m = metrics[mver]
            if m and m[1] then
                score = m[1].cvssData.baseScore or 0
                break
            end
        end
        
        if score >= 7.0 then
            local desc = ""
            for _, d in ipairs(cve.descriptions or {}) do
                if d.lang == "en" then desc = d.value:sub(1, 100) break end
            end
            table.insert(output, string.format(
                "%s (CVSS: %.1f) - %s...", cve.id, score, desc))
        end
    end
    
    if #output == 0 then return nil end
    
    return stdnse.format_output(true, output)
end
```

---

## Fase 4: HTML Report Generator

```python
def generate_vuln_report(vulns: List[Vulnerability], output_path: str):
    """Generate comprehensive HTML vulnerability report"""
    
    critical = [v for v in vulns if v.severity == 'CRITICAL']
    high = [v for v in vulns if v.severity == 'HIGH']
    medium = [v for v in vulns if v.severity == 'MEDIUM']
    low = [v for v in vulns if v.severity == 'LOW']
    
    sev_color = {
        'CRITICAL': '#d32f2f', 'HIGH': '#f57c00',
        'MEDIUM': '#f9a825', 'LOW': '#388e3c'
    }
    
    html = f"""<!DOCTYPE html>
<html><head><title>Vulnerability Scan Report</title>
<style>
body {{ font-family: 'Courier New', monospace; background: #0d1117; color: #c9d1d9; margin: 20px; }}
h1,h2 {{ color: #58a6ff; }} table {{ width: 100%; border-collapse: collapse; }}
th {{ background: #161b22; padding: 10px; color: #8b949e; text-align: left; }}
td {{ padding: 8px; border-bottom: 1px solid #21262d; }}
.CRITICAL {{ color: #d32f2f; font-weight: bold; }}
.HIGH {{ color: #f57c00; font-weight: bold; }}
.MEDIUM {{ color: #f9a825; }}
.LOW {{ color: #388e3c; }}
.score-bar {{ display: inline-block; height: 8px; background: currentColor; border-radius: 4px; }}
</style></head><body>
<h1>Vulnerability Assessment Report</h1>
<p>Generated: {__import__('datetime').datetime.now().isoformat()}</p>
<p>Total Findings: {len(vulns)} |
   <span class="CRITICAL">Critical: {len(critical)}</span> |
   <span class="HIGH">High: {len(high)}</span> |
   <span class="MEDIUM">Medium: {len(medium)}</span> |
   <span class="LOW">Low: {len(low)}</span>
</p>
<h2>Findings</h2>
<table>
<tr><th>CVE</th><th>CVSS</th><th>Severity</th><th>Target</th><th>Service</th><th>Description</th></tr>
"""
    
    for v in sorted(vulns, key=lambda x: x.cvss_score, reverse=True):
        target = f"{v.service.ip}:{v.service.port}"
        svc = f"{v.service.product} {v.service.version}".strip()
        desc = v.description[:120] + "..." if len(v.description) > 120 else v.description
        
        html += f"""<tr>
<td><a href="https://nvd.nist.gov/vuln/detail/{v.cve_id}" style="color:#58a6ff">{v.cve_id}</a></td>
<td>{v.cvss_score:.1f}</td>
<td class="{v.severity}">{v.severity}</td>
<td>{target}</td>
<td>{svc}</td>
<td>{desc}</td>
</tr>\n"""
    
    html += "</table></body></html>"
    
    with open(output_path, 'w') as f:
        f.write(html)
    
    print(f"[+] Report: {output_path} ({len(vulns)} findings)")


# Main entry point
if __name__ == "__main__":
    import sys
    
    target = sys.argv[1] if len(sys.argv) > 1 else "192.168.1.0/24"
    
    print(f"[*] Scanning {target}...")
    scanner = ServiceScanner()
    services = scanner.scan(target, ports="22,80,443,21,25,3306,5432,27017,6379,8080,8443")
    
    print(f"[+] Found {len(services)} services")
    
    matcher = CVEMatcher()
    all_vulns = []
    
    for svc in services:
        print(f"[*] Checking CVEs for {svc.ip}:{svc.port} {svc.product} {svc.version}...")
        vulns = matcher.match_service(svc)
        all_vulns.extend(vulns)
        if vulns:
            print(f"    Found {len(vulns)} CVEs (top: {vulns[0].cve_id} CVSS:{vulns[0].cvss_score})")
    
    generate_vuln_report(all_vulns, "/tmp/vuln_report.html")
    print(f"\n[+] Scan complete. {len(all_vulns)} total vulnerabilities found.")
```

---

*← [BAB 13: Threat Hunting](bab13.md) · Selanjutnya → [BAB 15: Advanced Methodology & Cheatsheet](bab15.md)*
