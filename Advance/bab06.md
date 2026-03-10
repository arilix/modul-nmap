# BAB 06 — Advanced Active Directory Reconnaissance

> **NMAP Advanced Guide · 2026 Edition**

---

## Mengapa AD Recon Penting?

Active Directory adalah tulang punggung infrastruktur enterprise Windows. Rekon mendalam terhadap AD memungkinkan:
- Pemetaan trust relationships antar domain
- Identifikasi akun dengan Kerberos delegation
- Enumeration Group Policy Objects (GPO)
- Deteksi misconfiguration yang dapat dieksploitasi (ASREPRoast, Kerberoast, ACL abuse)

---

## Port Landscape AD: Rekon Awal

```bash
# Port utama Active Directory
# 53   - DNS (Zone Transfer, SRV records)
# 88   - Kerberos
# 135  - RPC Endpoint Mapper
# 139  - NetBIOS Session Service
# 389  - LDAP
# 445  - SMB (NetLogon, SYSVOL, ADMIN$)
# 464  - Kerberos Password Change (kpasswd)
# 636  - LDAPS (LDAP over SSL)
# 3268 - Global Catalog
# 3269 - Global Catalog SSL
# 5985 - WinRM HTTP
# 5986 - WinRM HTTPS
# 49152-65535 - Dynamic RPC

# Scan semua port AD
sudo nmap -sS -sV -p 53,88,135,139,389,445,464,636,3268,3269,5985,5986 \
     --open -T4 192.168.1.0/24 -oX ad_scan.xml
```

---

## DNS Enumeration untuk Domain Discovery

### Zone Transfer Attack

```bash
# Coba zone transfer - sering masih berhasil di jaringan internal
nmap --script dns-zone-transfer \
     --script-args "dns-zone-transfer.domain=corp.local" \
     -p 53 192.168.1.1

# Jika berhasil, kamu mendapat semua records DNS:
# A, AAAA, CNAME, MX, NS, PTR, SRV, TXT
```

### DNS SRV Record Discovery

```bash
# SRV records mengungkap services AD
nmap --script dns-srv-enum \
     --script-args "dns-srv-enum.domain=corp.local" \
     -p 53 192.168.1.1

# SRV records penting:
# _ldap._tcp.corp.local       -> Domain Controllers
# _kerberos._tcp.corp.local   -> KDCs
# _kpasswd._tcp.corp.local    -> Password change servers
# _gc._tcp.corp.local         -> Global Catalog servers

# DNS brute force subdomain
nmap --script dns-brute \
     --script-args "dns-brute.domain=corp.local,dns-brute.threads=8" \
     -p 53 192.168.1.1
```

---

## LDAP Deep Enumeration

### RootDSE — Info Domain Tanpa Auth

```bash
# RootDSE accessible anonymously di banyak konfigurasi
nmap --script ldap-rootdse -p 389 192.168.1.10

# Output mengungkap:
# | ldap-rootdse:
# |   domainFunctionality: 7  (Windows Server 2016+)
# |   forestFunctionality: 7
# |   domainControllerFunctionality: 7
# |   currentTime: 20240115120000.0Z
# |   defaultNamingContext: DC=corp,DC=local
# |   rootDomainNamingContext: DC=corp,DC=local
# |   configurationNamingContext: CN=Configuration,DC=corp,DC=local
```

### LDAP Search dengan Credentials

```bash
# Dengan credentials valid, enumerasi komprehensif
nmap --script ldap-search \
     --script-args 'ldap.username="CN=user,CN=Users,DC=corp,DC=local",ldap.password="Password123",ldap.qfilter=users,ldap.attrib=sAMAccountName' \
     -p 389 192.168.1.10

# Cari Domain Admins
nmap --script ldap-search \
     --script-args 'ldap.username="user@corp.local",ldap.password="Password123",ldap.qfilter="(memberOf=CN=Domain Admins,CN=Users,DC=corp,DC=local)"' \
     -p 389 192.168.1.10
```

### Enumeration dengan ldapsearch (fallback)

```bash
# Enum Anonymous LDAP
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local" "(objectClass=*)" \
           dn description | head -50

# Enum users dengan credentials
ldapsearch -x -H ldap://192.168.1.10 \
           -D "user@corp.local" -w "Password123" \
           -b "DC=corp,DC=local" \
           "(objectClass=user)" \
           sAMAccountName userPrincipalName memberOf pwdLastSet \
           2>/dev/null | grep -v "^#\|^$"
```

---

## Kerberos Enumeration

### User Enumeration via AS-REQ

```bash
# Kerberos memungkinkan user enumeration tanpa credentials
# Karena error berbeda untuk user tidak ada vs password salah
nmap --script krb5-enum-users \
     --script-args krb5-enum-users.realm=corp.local,userdb=/usr/share/wordlists/seclists/Usernames/Names/names.txt \
     -p 88 192.168.1.10

# Output:
# | krb5-enum-users:
# | Discovered Kerberos principals:
# |     administrator@corp.local
# |     svc_backup@corp.local
```

### ASREPRoast Detection Setup

```bash
# Temukan akun dengan "Do not require Kerberos preauthentication"
# (Flag DONT_REQUIRE_PREAUTH / UF_DONT_REQUIRE_PREAUTH)

# Gunakan ldapsearch untuk cari flag ini
ldapsearch -x -H ldap://192.168.1.10 \
           -D "user@corp.local" -w "Password123" \
           -b "DC=corp,DC=local" \
           "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" \
           sAMAccountName

# Kemudian request TGT tanpa preauth (untuk crack offline)
python3 GetNPUsers.py corp.local/ -usersfile asreproast_targets.txt \
        -no-pass -dc-ip 192.168.1.10 -format hashcat
```

---

## SMB / NetBIOS Enumeration

```bash
# NetBIOS enumeration - mengungkap domain, workgroup, computer name
nmap --script nbstat -p 137 -sU 192.168.1.0/24

# SMB security level detection
nmap --script smb-security-mode -p 445 192.168.1.10

# Output:
# | smb-security-mode:
# |   account_used: guest
# |   authentication_level: user          (user-level auth)
# |   challenge_response: supported       (NTLM challenge)
# |_  message_signing: disabled           <-- RELAY ATTACK POSSIBLE

# SMB enum shares
nmap --script smb-enum-shares,smb-enum-users -p 445 192.168.1.10
nmap --script smb-enum-domains -p 445 192.168.1.10

# SMB vuln check
nmap --script "smb-vuln*" -p 445 192.168.1.10
```

### SMB Signing Check (Relay Attack Prerequisite)

```bash
# Hanya host dengan signing disabled yang vulnerable ke NTLM relay
nmap --script smb-security-mode -p 445 192.168.1.0/24 2>/dev/null |
awk '/Nmap scan report/{host=$NF} /message_signing: disabled/{print host " - RELAY POSSIBLE"}'
```

---

## RPC Endpoint Mapping

```bash
# Enum RPC endpoints - mengungkap services yang berjalan
nmap --script msrpc-enum -p 135 192.168.1.10

# Masing-masing endpoint menunjukkan UUID yang bisa diidentifikasi
# UUID 6BFFD098-A112-3610-9833-46C3F87E345A = Task Scheduler
# UUID 338CD001-2244-31F1-AAAA-900038001003 = WinReg (remote registry)
# UUID 1ff70682-0a51-30e8-076d-740be8cee98b = Task Scheduler (legacy)
```

---

## WinRM Detection & Analysis

```bash
# WinRM (Windows Remote Management)
nmap --script http-auth-finder -p 5985 192.168.1.10
nmap --script winrm-detect -p 5985,5986 192.168.1.10 2>/dev/null ||
nmap -sV -p 5985,5986 192.168.1.10

# WinRM aktif = bisa remote command execution jika ada credentials
# evil-winrm, ansible, pywinrm semua gunakan port ini
```

---

## BloodHound-Compatible Data Collection via Nmap

```bash
# Combine nmap data dengan BloodHound-style analysis
# Script untuk export AD users ke format JSON

cat << 'EOF' > ad_recon_nmap.sh
#!/bin/bash
TARGET_DC=$1
DOMAIN=$2
USER=$3
PASS=$4

echo "[*] Starting AD Recon against $TARGET_DC ($DOMAIN)"

# 1. DNS SRV Recon
echo "[*] DNS SRV Records:"
nmap --script dns-srv-enum \
     --script-args "dns-srv-enum.domain=$DOMAIN" \
     -p 53 $TARGET_DC -oN dns_srv.txt 2>/dev/null
cat dns_srv.txt | grep "|"

# 2. LDAP RootDSE
echo "[*] LDAP RootDSE:"
nmap --script ldap-rootdse -p 389 $TARGET_DC -oN ldap_rootdse.txt 2>/dev/null
cat ldap_rootdse.txt | grep "| "

# 3. SMB Info
echo "[*] SMB Security Mode:"
nmap --script smb-security-mode -p 445 $TARGET_DC -oN smb_info.txt 2>/dev/null
cat smb_info.txt | grep "message_signing"

# 4. Kerberos user enum
echo "[*] Kerberos User Enum:"
nmap --script krb5-enum-users \
     --script-args "krb5-enum-users.realm=$DOMAIN,userdb=/usr/share/seclists/Usernames/top-usernames-shortlist.txt" \
     -p 88 $TARGET_DC -oN krb5_users.txt 2>/dev/null
cat krb5_users.txt | grep "Discovered\|principal"

echo "[*] Recon complete. Results saved to *.txt files."
EOF
chmod +x ad_recon_nmap.sh

# Gunakan:
# ./ad_recon_nmap.sh 192.168.1.10 corp.local user Password123
```

---

## Trust Mapping antar Domain/Forest

```bash
# Cari domain trust via LDAP
ldapsearch -x -H ldap://192.168.1.10 \
           -D "user@corp.local" -w "Password123" \
           -b "CN=System,DC=corp,DC=local" \
           "(objectClass=trustedDomain)" \
           name trustDirection trustType flatName \
           2>/dev/null

# trustDirection:
# 1 = Inbound  (trusted domain auth ke kita)
# 2 = Outbound (kita auth ke trusted domain)
# 3 = Bidirectional

# Nmap against trusted domain
nmap -sS -p 389,445,88 forest-child-dc.othercorp.local
```

---

*← [BAB 05: Encrypted Services](bab05.md) · Selanjutnya → [BAB 07: Cloud & Container Scanning](bab07.md)*
