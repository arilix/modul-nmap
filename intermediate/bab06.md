# BAB 06 — Menulis Custom NSE Script

> **NMAP Intermediate Guide · 2026 Edition**

---

## Dasar Lua untuk NSE

NSE scripts ditulis dalam **Lua 5.3**. Kamu tidak perlu menjadi ahli Lua, tapi perlu memahami sintaks dasarnya.

### Sintaks Dasar Lua

```lua
-- Komentar satu baris
--[[ Komentar
     multi-baris ]]

-- Variabel
local nama = "Nmap"
local port = 80
local aktif = true

-- String
print("Hello, " .. nama)         -- Konkatenasi dengan ..

-- Kondisional
if port == 80 then
  print("HTTP port")
elseif port == 443 then
  print("HTTPS port")
else
  print("Port lain: " .. port)
end

-- Loop
for i = 1, 10 do
  print(i)
end

-- Tabel (seperti array + dict)
local hasil = {host="192.168.1.1", port=22, service="ssh"}
print(hasil.host)
print(hasil["service"])

-- Fungsi
local function sapa(nama)
  return "Halo, " .. nama
end
print(sapa("World"))
```

---

## Struktur NSE Script

Setiap NSE script memiliki 4 bagian utama:

```lua
-- 1. DESKRIPSI
description = [[
  Script ini mengecek apakah port 80 bisa diakses dan mengembalikan
  status code HTTP dan judul halaman.
]]

-- 2. AUTHOR & CATEGORIES
author = "Nama Kamu"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"discovery", "safe"}

-- 3. DEPENDENCIES (opsional)
-- Tambahkan library NSE yang dibutuhkan
local http = require "http"
local shortport = require "shortport"
local stdnse = require "stdnse"

-- 4. RULE (kapan script ini berjalan?)
-- portrule: jalankan jika kondisi port terpenuhi
portrule = shortport.http  -- jalankan untuk port HTTP

-- 5. ACTION (apa yang dilakukan?)
action = function(host, port)
  -- Logika script di sini
  return "Script berjalan!"
end
```

---

## Script Sederhana: HTTP Title Checker

```lua
-- File: /usr/share/nmap/scripts/my-http-check.nse

description = [[
  Cek status code dan title halaman web.
]]

author = "You"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"discovery", "safe"}

local http = require "http"
local shortport = require "shortport"
local stdnse = require "stdnse"

-- Hanya jalankan pada port HTTP
portrule = shortport.http

action = function(host, port)
  -- Ambil halaman root
  local response = http.get(host, port, "/")

  if not response then
    return "Tidak bisa terhubung"
  end

  local output = stdnse.output_table()

  -- Status code
  output["Status"] = tostring(response.status)

  -- Ambil title dari HTML
  if response.body then
    local title = response.body:match("<title>(.-)</title>")
    if title then
      output["Title"] = title
    end
  end

  -- Ambil header Server
  if response.header["server"] then
    output["Server"] = response.header["server"]
  end

  return output
end
```

**Cara menggunakannya:**
```bash
# Copy ke direktori scripts Nmap
sudo cp my-http-check.nse /usr/share/nmap/scripts/

# Update database
sudo nmap --script-updatedb

# Jalankan
nmap --script my-http-check -p 80,443,8080 192.168.1.1
```

---

## Library NSE yang Paling Berguna

### `shortport` — Filter Port

```lua
local shortport = require "shortport"

-- Jalankan untuk port HTTP terdeteksi
portrule = shortport.http

-- Jalankan untuk port SSL
portrule = shortport.ssl

-- Jalankan untuk port spesifik
portrule = shortport.port_or_service({22, 2222}, "ssh")
```

### `http` — HTTP Requests

```lua
local http = require "http"

-- GET request
local resp = http.get(host, port, "/path")

-- POST request
local resp = http.post(host, port, "/login", nil, nil, "user=admin&pass=test")

-- Custom headers
local opts = {header={["User-Agent"]="Mozilla/5.0"}}
local resp = http.get(host, port, "/", opts)

-- Cek status
if resp.status == 200 then ... end
if resp.body:find("admin") then ... end
```

### `stdnse` — Utilities

```lua
local stdnse = require "stdnse"

-- Formatted output
local output = stdnse.output_table()
output["Key"] = "Value"
return output

-- Debug print
stdnse.debug(1, "Debug message: %s", variable)

-- Delay
stdnse.sleep(2)  -- 2 detik
```

### `nmap` — Akses Host/Port Data

```lua
-- Cek apakah port terbuka
if nmap.get_port_state(host, port) == "open" then ... end

-- Dapatkan service name
local service = port.service

-- Dapatkan versi terdeteksi
local version = port.version.product
```

---

## Script Lanjutan: Port Knocking Detector

```lua
description = [[
  Cek apakah host mungkin menggunakan port knocking
  dengan membandingkan respons dari port filtered berbeda.
]]

author = "You"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"discovery"}

local nmap = require "nmap"
local stdnse = require "stdnse"

-- Jalankan satu kali per host (hostrule)
hostrule = function(host)
  return true
end

action = function(host)
  local output = stdnse.output_table()
  local filtered_ports = {}

  -- Iterasi semua port yang dipindai
  for _, portinfo in ipairs(host.ports or {}) do
    if portinfo.state == "filtered" then
      table.insert(filtered_ports, portinfo.number)
    end
  end

  output["Filtered ports count"] = #filtered_ports
  if #filtered_ports > 20 then
    output["Note"] = "Banyak port filtered - mungkin ada firewall ketat atau port knocking"
  end

  return output
end
```

---

## Praktik: Jalankan Script Custom

```bash
# Test script tanpa install (path langsung)
nmap --script ./my-script.nse -p 80 192.168.1.1

# Debug output penuh
nmap --script ./my-script.nse -p 80 -d3 192.168.1.1

# Test dengan --script-trace untuk lihat semua network calls
nmap --script ./my-script.nse -p 80 --script-trace 192.168.1.1
```

---

*← [BAB 05: Advanced NSE Categories](bab05.md) · Selanjutnya → [BAB 07: Nmap + Metasploit Integration](bab07.md)*
