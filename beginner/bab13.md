# BAB 13 — Zenmap – The Nmap GUI

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

**Zenmap** adalah antarmuka grafis (GUI) resmi untuk Nmap. Zenmap bersifat cross-platform, gratis, dan sudah termasuk dalam installer Windows dan macOS. Zenmap membuat Nmap lebih mudah diakses oleh pemula, sekaligus tetap memberikan akses penuh ke fitur-fitur lanjutan.

---

## Fitur Utama Zenmap

- **Profile-based scanning** — scan type sudah dikonfigurasi sebelumnya
- **Interactive topology map** — peta jaringan yang menunjukkan hubungan antar host
- **Side-by-side scan comparison** — membandingkan dua scan untuk mendeteksi perubahan
- **Searchable results** — filter hasil yang powerful
- **Save and load scan results** — simpan dan buka kembali hasil scan nanti
- **Host list** — kolom yang bisa diurutkan dan panel info detail

---

## Profil Scan yang Sudah Ada (Predefined Scan Profiles)

| Nama Profil          | Perintah yang Digunakan                                     | Tujuan                      |
|----------------------|-------------------------------------------------------------|-----------------------------|
| Intense scan         | `nmap -T4 -A -v`                                           | Info terlengkap, agresif    |
| Intense scan + UDP   | `nmap -sS -sU -T4 -A -v`                                   | Scan TCP + UDP              |
| Ping scan            | `nmap -sn`                                                  | Temukan host yang aktif saja |
| Quick scan           | `nmap -T4 -F`                                               | Scan cepat 100 port teratas |
| Quick traceroute     | `nmap -sn --traceroute`                                     | Peta jalur jaringan         |
| Regular scan         | `nmap`                                                      | Scan default                |
| Slow comprehensive   | `nmap -sS -sU -T4 -A -v -PE --script discovery,safe`       | Audit lengkap               |

---

> 💡 **Tips:** Zenmap sangat berguna untuk pemula yang ingin belajar Nmap karena menampilkan perintah yang digunakan secara real-time saat kamu memilih profil.

---

*← [BAB 12: Firewall & IDS Evasion](bab12.md) · Selanjutnya → [BAB 14: Real-World Use Cases](bab14.md)*
