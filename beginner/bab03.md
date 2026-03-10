# BAB 03 — Nmap Basics & Syntax

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

## Struktur Perintah Dasar

Sintaks umum Nmap sangat sederhana. Setiap perintah Nmap mengikuti struktur berikut:

```
nmap [Scan Type] [Options] {target specification}
```

**Contoh sederhana:**
```bash
nmap 192.168.1.1          # Scan satu IP
nmap scanme.nmap.org      # Scan hostname
nmap 192.168.1.0/24       # Scan satu subnet
```

---

## Jenis-Jenis Target (Target Specification)

| Target Type   | Contoh                | Deskripsi                         |
|---------------|-----------------------|-----------------------------------|
| Single IP     | `192.168.1.1`         | Scan satu host spesifik           |
| IP Range      | `192.168.1.1-254`     | Scan range IP                     |
| CIDR Notation | `192.168.1.0/24`      | Scan seluruh subnet (256 host)    |
| Hostname      | `example.com`         | Scan berdasarkan nama domain      |
| From File     | `-iL targets.txt`     | Baca target dari file             |
| Exclude       | `--exclude 192.168.1.5` | Lewati host tertentu            |

---

## Memahami Output Nmap

Saat Nmap selesai melakukan scan, ia menampilkan laporan terstruktur yang berisi status host, state port, nama layanan, dan informasi versi (jika diminta).

**Contoh output:**
```
Nmap scan report for 192.168.1.1
Host is up (0.0023s latency).

PORT      STATE   SERVICE   VERSION
22/tcp    open    ssh       OpenSSH 8.9
80/tcp    open    http      Apache 2.4
443/tcp   open    https     Apache 2.4
3306/tcp  closed  mysql
```

> 💡 **Catatan:** Nilai *latency* menunjukkan waktu yang dibutuhkan untuk menerima respons dari host. Nilai lebih rendah berarti host lebih cepat/dekat.

---

*← [BAB 02: Installation & Setup](bab02.md) · Selanjutnya → [BAB 04: Host Discovery](bab04.md)*
