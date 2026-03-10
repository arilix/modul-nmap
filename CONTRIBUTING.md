# Contributing to NMAP Learning Module

Terima kasih sudah tertarik untuk berkontribusi! Panduan ini menjelaskan cara terbaik untuk ikut serta dalam pengembangan modul ini.

---

## Cara Berkontribusi

### 1. Laporkan Kesalahan / Bug

Temukan typo, perintah yang salah, atau penjelasan yang membingungkan?

- Buka **[Issues](https://github.com/arilix/modul-nmap/issues)**
- Klik **New Issue** → pilih template **Bug Report**
- Sertakan: nama file, nomor bab, dan deskripsi masalah

### 2. Usulkan Materi Baru

Ingin menambahkan teknik, tool, atau skenario baru?

- Buka **[Issues](https://github.com/arilix/modul-nmap/issues)**
- Pilih template **Feature Request**
- Jelaskan topik yang ingin ditambahkan dan mengapa relevan

### 3. Pull Request (PR)

Langkah-langkah untuk kontribusi langsung:

```bash
# 1. Fork repository ini di GitHub

# 2. Clone fork kamu
git clone https://github.com/USERNAME/modul-nmap.git
cd modul-nmap

# 3. Buat branch baru
git checkout -b fix/bab03-typo
# atau
git checkout -b feat/bab16-wireless-scanning

# 4. Buat perubahan kamu

# 5. Commit dengan pesan yang jelas
git add .
git commit -m "fix: perbaiki contoh command di bab03 intermediate"

# 6. Push ke fork kamu
git push origin fix/bab03-typo

# 7. Buka Pull Request di GitHub
```

---

## Standar Penulisan

### Format Markdown

- Satu file per bab: `bab01.md`, `bab02.md`, dst.
- Gunakan heading hierarki: `#` → `##` → `###`
- Bungkus semua command dalam fenced code block dengan bahasa yang benar:
  ````
  ```bash
  nmap -sS 192.168.1.1
  ```
  ````
- Sertakan navigasi di akhir file:
  ```
  *← [BAB XX: Judul](babXX.md) · Selanjutnya → [BAB YY: Judul](babYY.md)*
  ```

### Bahasa

- Penjelasan dan narasi: **Bahasa Indonesia**
- Nama command, flag, protokol, variabel: **tetap bahasa Inggris**
- Tidak perlu menerjemahkan output terminal

### Tingkat Kesulitan

Pastikan kontribusi sesuai level modul:

| Level | Asumsi Pembaca | Kedalaman |
|-------|---------------|-----------|
| `beginner/` | Tidak ada background IT | Fundamental, langkah demi langkah |
| `intermediate/` | Paham jaringan dasar | Teknik lanjut, kombinasi tools |
| `Advance/` | Pengalaman pentest | Internals, scripting, opsec |

---

## Struktur Repository

```
modul-nmap/
├── README.md           ← Main index
├── LICENSE             ← MIT License
├── CONTRIBUTING.md     ← File ini
├── beginner/
│   ├── README.md
│   └── bab01.md ... bab15.md
├── intermediate/
│   ├── README.md
│   └── bab01.md ... bab15.md
└── Advance/
    ├── README.md
    └── bab01.md ... bab15.md
```

---

## Code of Conduct

- Semua kontribusi harus untuk tujuan **edukasi yang sah**
- Tidak boleh menambahkan konten yang mendorong penggunaan ilegal
- Hormati kontributor lain dalam diskusi Issue / PR
- Teknik ofensif yang dibahas harus disertai **konteks legal dan etika**

---

## Pertanyaan?

Buka [Discussion](https://github.com/arilix/modul-nmap/discussions) atau hubungi melalui Issues.

Kontribusimu sangat berarti untuk komunitas keamanan siber Indonesia! 🙏
