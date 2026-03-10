# BAB 02 — Installation & Setup

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

## Instalasi di Linux

Kebanyakan distribusi Linux sudah menyertakan Nmap di repository paket mereka. Gunakan perintah yang sesuai dengan distribusimu:

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install nmap

# Red Hat / CentOS / Fedora
sudo dnf install nmap

# Arch Linux
sudo pacman -S nmap
```

---

## Instalasi di Windows

Di Windows, unduh installer resmi dari situs Nmap (nmap.org). Installer tersebut sudah mencakup:
- **Nmap** — tool utama
- **Zenmap** — antarmuka GUI
- **Ncat** — tool jaringan serbaguna
- **Npcap** — library packet capture yang dibutuhkan di Windows

```powershell
# Setelah instalasi, gunakan lewat Command Prompt atau PowerShell:
nmap --version
```

---

## Instalasi di macOS

```bash
# Menggunakan Homebrew (direkomendasikan)
brew install nmap

# Atau unduh installer .dmg dari nmap.org
```

---

## Verifikasi Instalasi

Setelah instalasi selesai, verifikasi bahwa Nmap berhasil terinstall:

```bash
nmap --version
```

**Contoh output yang diharapkan:**
```
Nmap version 7.95 ( https://nmap.org )
Platform: x86_64-pc-linux-gnu
```

---

*← [BAB 01: Introduction](bab01.md) · Selanjutnya → [BAB 03: Nmap Basics & Syntax](bab03.md)*
