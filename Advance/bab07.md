# BAB 07 — Cloud & Container Environment Scanning

> **NMAP Advanced Guide · 2026 Edition**

---

## Tantangan Scanning di Cloud & Container

Infrastruktur modern berbeda dari jaringan tradisional:
- IP address bersifat ephemeral dan berubah
- Auto-scaling membuat host list berubah setiap detik
- Security groups/NSG bertindak seperti stateful firewall di cloud level
- Container ports di-publish hanya saat dibutuhkan
- Metadata endpoints menjadi target utama SSRF

---

## Cloud Metadata Endpoint: Informasi Sensitif

### AWS IMDS (Instance Metadata Service)

```bash
# IMDSv1 — Tidak butuh token, langsung akses:
curl http://169.254.169.254/latest/meta-data/

# IMDSv2 — Butuh token (lebih aman, tapi masih bisa diakses dari instance):
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
        -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
     http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Dari perspektif Nmap - cari instance yang expose port 80
# lalu probe dengan NSE script:
nmap --script http-open-proxy \
     --script-args "http-open-proxy.url=http://169.254.169.254/latest/meta-data/" \
     -p 80 target-ec2.com
```

### Nmap Script: Cloud Metadata Probe

```lua
-- NSE script untuk detect SSRF ke metadata endpoints
local http = require "http"
local shortport = require "shortport"
local stdnse = require "stdnse"

description = "Detects SSRF via cloud metadata endpoint access"
author = "Advanced Nmap Module"
categories = {"discovery", "safe"}

portrule = shortport.http

action = function(host, port)
    -- Coba akses melalui open proxy atau SSRF
    local metadata_urls = {
        "http://169.254.169.254/latest/meta-data/",      -- AWS
        "http://169.254.169.254/computeMetadata/v1/",    -- GCP
        "http://169.254.169.254/metadata/instance",      -- Azure
        "http://100.100.100.200/latest/meta-data/"       -- Alibaba Cloud
    }
    
    for _, url in ipairs(metadata_urls) do
        local resp = http.get(host, port, url, {
            header = {
                ["Metadata"] = "true",  -- GCP header
                ["X-aws-ec2-metadata-token"] = ""
            }
        })
        
        if resp and resp.status == 200 then
            return string.format("METADATA ENDPOINT ACCESSIBLE: %s\n%s",
                                 url, resp.body:sub(1, 200))
        end
    end
end
```

---

## AWS EC2 Scanning

```bash
# Discover EC2 instances di range IP tertentu
# AWS public IPs: 3.0.0.0/8, 13.0.0.0/8, 18.0.0.0/8, 52.0.0.0/8, 54.0.0.0/8

# Scan dengan AWS-aware hostnames
nmap -sV --script ssl-cert \
     -p 443,8443,22,80 \
     --script-args "ssl-cert.subject-dns=true" \
     54.123.0.0/24

# Cari Security Group misconfiguration (port terbuka tidak semestinya)
nmap -sS -p 0-65535 ec2-instance.compute.amazonaws.com --open -T4

# AWS-specific services
# 2049  - EFS (Elastic File System / NFS)
# 9200  - Elasticsearch (sering exposed secara tidak sengaja)
# 5601  - Kibana (juga sering exposed)
# 6379  - Redis (tanpa auth)
# 27017 - MongoDB (tanpa auth)

nmap -sS -p 2049,9200,5601,6379,27017 \
     --open --script banner \
     $AWS_IP_RANGE
```

---

## GCP (Google Cloud Platform)

```bash
# GCP metadata endpoint: 169.254.169.254 atau metadata.google.internal
# Hanya accessible dari dalam instance

# Scanning GCP instances di range publik
# GCP: 34.0.0.0/8, 35.0.0.0/8, 104.196.0.0/14

nmap --script ssl-cert -p 443 35.200.0.0/24 2>/dev/null |
grep -A5 "Subject\|commonName" | grep "\.google\|appspot\|run\.app"

# Cari exposed Cloud Run / App Engine
nmap -sV -p 443 --script http-title,http-headers 35.200.0.0/24
```

---

## Azure Scanning

```bash
# Azure IP ranges: 20.0.0.0/8, 40.0.0.0/8, 52.168.0.0/14

# Azure-specific ports
# 1433 - Azure SQL (often exposed accidentally)
# 443  - Azure App Services, Azure Functions
# 5671/5672 - Azure Service Bus (AMQP)

# Cari Azure resources via SSL certificate
nmap --script ssl-cert -p 443 20.0.0.0/24 2>/dev/null |
awk '/commonName/{print}' | grep "\.azure\|\.windows\.net\|\.cloudapp\."

# Azure Blob Storage exposed
nmap --script http-methods -p 443 \
     --script-args http.useragent="Mozilla/5.0" \
     storage.azure.com
```

---

## Docker API Scanning

Docker Remote API yang exposed adalah kerentanan kritis — full container/host compromise.

```bash
# Port 2375 - Docker API (HTTP, tanpa TLS) — CRITICAL
# Port 2376 - Docker API (HTTPS, dengan TLS)

# Scan untuk Docker API
nmap -sS -sV -p 2375,2376 192.168.1.0/24

# Jika ditemukan port 2375 open:
nmap --script http-methods -p 2375 192.168.1.10
# Atau langsung:
curl http://192.168.1.10:2375/version
curl http://192.168.1.10:2375/containers/json

# NSE script untuk Docker API detection
nmap --script docker-version -p 2375 192.168.1.10 2>/dev/null ||
nmap --script "http-*" -p 2375 192.168.1.10 \
     --script-args http.max-redirects=0
```

### Docker API Enumeration Script (Python)

```python
import requests
import json

def docker_enum(host, port=2375):
    base = f"http://{host}:{port}"
    
    # Info daemon
    resp = requests.get(f"{base}/version", timeout=5)
    if resp.status_code != 200:
        return
    
    info = resp.json()
    print(f"[+] Docker Engine: {info.get('Version','?')}")
    print(f"[+] OS: {info.get('Os','?')}/{info.get('Arch','?')}")
    
    # List containers
    resp = requests.get(f"{base}/containers/json?all=true")
    containers = resp.json()
    print(f"[+] Containers ({len(containers)}):")
    for c in containers:
        print(f"    - {c['Names'][0]}: {c['Image']} ({c['State']})")
    
    # List images
    resp = requests.get(f"{base}/images/json")
    images = resp.json()
    print(f"[+] Images ({len(images)}):")
    for img in images[:5]:  # Top 5
        tags = img.get('RepoTags', ['<none>'])
        print(f"    - {tags[0]}")
    
    # List volumes (mungkin ada sensitive mounts)
    resp = requests.get(f"{base}/volumes")
    vols = resp.json().get('Volumes', [])
    print(f"[+] Volumes ({len(vols)}):")
    for v in vols:
        print(f"    - {v['Name']}: {v.get('Mountpoint','?')}")

docker_enum("192.168.1.10")
```

---

## Kubernetes API Scanning

```bash
# Kubernetes API Server
# Port 6443 - HTTPS (standard)
# Port 8080 - HTTP (insecure, deprecated tapi masih ditemukan)
# Port 10250 - Kubelet API
# Port 10255 - Kubelet read-only API (deprecated)
# Port 2379/2380 - etcd

nmap -sS -p 6443,8080,10250,10255,2379,2380 192.168.1.0/24 --open

# Cek apakah API anonymous access enabled
curl -sk https://192.168.1.10:6443/api/v1/namespaces
curl -sk https://192.168.1.10:6443/api/v1/nodes

# Kubelet API - bisa exec into containers jika not secured
curl -sk https://192.168.1.10:10250/pods
curl -sk https://192.168.1.10:10250/runningpods

# Kubelet read-only (tidak butuh auth di port 10255)
curl http://192.168.1.10:10255/pods 2>/dev/null | python3 -m json.tool | head -50
```

### etcd Enumeration

```bash
# etcd menyimpan seluruh cluster state termasuk secrets!
nmap -sV -p 2379 192.168.1.10

# Jika port 2379 open dan accessible:
curl http://192.168.1.10:2379/v2/keys/
curl http://192.168.1.10:2379/v3/kv/range  # etcd v3

# Ekstrak secrets dari etcd (jika unprotected)
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.1.10:2379 get "" \
             --from-key --keys-only | head -20
```

---

## Container Registry Scanning

```bash
# Docker Registry (self-hosted)
# Port 5000 - HTTP Docker Registry
# Port 443  - HTTPS Docker Registry

nmap -sV -p 5000 192.168.1.0/24

# Jika ditemukan:
curl http://192.168.1.10:5000/v2/_catalog      # List repositories
curl http://192.168.1.10:5000/v2/ubuntu/tags/list  # List tags
```

---

## Service Mesh: Istio & Envoy

```bash
# Envoy admin interface
# Port 15000 - Envoy Admin (jangan expose ke internet!)
# Port 15001 - Istio outbound traffic
# Port 15006 - Istio inbound traffic
# Port 9901  - Envoy admin (alternate)

nmap -sS -p 15000,15001,15006,9901 192.168.1.0/24 --open

# Envoy admin jika accessible:
curl http://192.168.1.10:15000/stats          # Prometheus-format stats
curl http://192.168.1.10:15000/config_dump    # Full config dump (sensitive!)
curl http://192.168.1.10:15000/clusters       # Upstream cluster info
```

---

*← [BAB 06: Active Directory](bab06.md) · Selanjutnya → [BAB 08: ICS/SCADA & IoT](bab08.md)*
