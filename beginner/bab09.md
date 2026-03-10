# BAB 09 — Nmap Scripting Engine (NSE)

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

**Nmap Scripting Engine (NSE)** adalah salah satu fitur paling powerful di Nmap. NSE memungkinkan pengguna menulis skrip (dalam bahasa **Lua**) yang mengotomatiskan tugas mulai dari network discovery hingga deteksi vulnerability tingkat lanjut.

---

## Apa itu NSE?

- NSE scripts disimpan di `/usr/share/nmap/scripts/` (Linux) atau direktori instalasi Nmap (Windows)
- Terdapat lebih dari **600 built-in scripts** yang mencakup berbagai tugas
- Scripts dapat berinteraksi dengan layanan target dan mengembalikan hasil yang detail

---

## Kategori Script

| Kategori    | Tujuan                                                        |
|-------------|---------------------------------------------------------------|
| `auth`      | Menguji credential autentikasi (mis. password default)        |
| `broadcast` | Temukan host yang tidak ada di target list via broadcast      |
| `brute`     | Serangan brute-force credential                               |
| `default`   | Dijalankan dengan `-sC`; script aman yang memberikan info berguna |
| `discovery` | Query registri publik, SNMP, DNS, dll.                        |
| `dos`       | Tes denial-of-service (gunakan dengan hati-hati!)             |
| `exploit`   | Eksploitasi kerentanan yang diketahui                         |
| `external`  | Berinteraksi dengan resource eksternal (Whois, DNS)           |
| `fuzzer`    | Kirim payload fuzzing untuk mendeteksi bug                    |
| `intrusive` | Script berisiko yang mungkin menyebabkan masalah              |
| `malware`   | Periksa backdoor dan infeksi malware                          |
| `safe`      | Script berisiko rendah, tidak mungkin crash service           |
| `version`   | Membantu mengidentifikasi versi service                       |
| `vuln`      | Periksa kerentanan CVE yang diketahui                         |

---

## Menjalankan Scripts

```bash
# Jalankan default scripts
nmap -sC 192.168.1.1

# Jalankan script spesifik
nmap --script=http-title 192.168.1.1

# Jalankan beberapa script
nmap --script=http-title,http-headers 192.168.1.1

# Jalankan semua script dalam satu kategori
nmap --script=vuln 192.168.1.1

# Jalankan script dengan argumen
nmap --script=http-brute --script-args userdb=users.txt 192.168.1.1

# Update database script
sudo nmap --script-updatedb
```

---

## Script-Script Populer

| Script                  | Deskripsi                                          |
|-------------------------|----------------------------------------------------|
| `http-title`            | Ambil judul halaman web                            |
| `ssh-brute`             | Brute-force login SSH                              |
| `smb-vuln-ms17-010`     | Deteksi kerentanan EternalBlue (WannaCry)          |
| `ssl-heartbleed`        | Tes kerentanan Heartbleed SSL                      |
| `ftp-anon`              | Cek apakah FTP mengizinkan login anonymous         |
| `dns-brute`             | Brute-force subdomain DNS                          |
| `mysql-empty-password`  | Cek MySQL dengan password root kosong              |
| `http-robots.txt`       | Ambil dan tampilkan file robots.txt                |
| `banner`                | Ambil banner layanan sederhana                     |
| `whois-domain`          | Lakukan WHOIS lookup pada domain target            |

---

*← [BAB 08: OS Detection](bab08.md) · Selanjutnya → [BAB 10: Output & Reporting](bab10.md)*
