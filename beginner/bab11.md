# BAB 11 — Timing & Performance

> **NMAP Guide for Beginners · 2026 Edition · Written by Rocky**

---

Opsi timing Nmap mengontrol seberapa cepat atau lambat scan berjalan. Scan yang lebih cepat lebih "berisik" di jaringan (mudah terdeteksi), sedangkan scan yang lebih lambat lebih senyap tapi membutuhkan waktu lebih lama.

---

## Timing Templates

| Template | Flag  | Nama      | Use Case                                    |
|----------|-------|-----------|---------------------------------------------|
| T0       | `-T0` | Paranoid  | Sangat lambat; menghindari deteksi IDS      |
| T1       | `-T1` | Sneaky    | Lambat; menghindari kebanyakan IDS          |
| T2       | `-T2` | Polite    | Lebih lambat; mengurangi penggunaan bandwidth |
| T3       | `-T3` | Normal    | Kecepatan default                           |
| T4       | `-T4` | Aggressive| Cepat; mengasumsikan jaringan yang andal    |
| T5       | `-T5` | Insane    | Sangat cepat; mungkin melewatkan beberapa hasil |

---

## Fine-Tuning Performance

```bash
# Set jumlah probe port paralel maksimum
nmap --min-parallelism 10 --max-parallelism 100 192.168.1.0/24

# Set host timeout (abaikan host yang lambat)
nmap --host-timeout 30s 192.168.1.0/24

# Set probe timeout
nmap --min-rtt-timeout 100ms --max-rtt-timeout 1000ms 192.168.1.1

# Batasi kecepatan scan (paket per detik)
nmap --min-rate 100 --max-rate 500 192.168.1.0/24
```

---

> ⚠️ **Catatan:** Untuk scanning jaringan produksi, T3 (default) atau T2 (polite) paling aman. Gunakan T4 hanya di jaringan test yang terisolasi.

---

*← [BAB 10: Output & Reporting](bab10.md) · Selanjutnya → [BAB 12: Firewall & IDS Evasion](bab12.md)*
