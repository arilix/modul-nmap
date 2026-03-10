# BAB 03 — Advanced NSE Lua Programming

> **NMAP Advanced Guide · 2026 Edition**

---

## NSE Execution Model: Coroutines

NSE menggunakan **Lua coroutines** untuk konkuren tanpa threading. Setiap script instance berjalan sebagai coroutine, memungkinkan ribuan script berjalan "bersamaan" dalam satu thread OS.

```lua
-- Nmap mengelola coroutine di balik layar
-- Ketika script melakukan I/O (network call), coroutine di-yield
-- Nmap menjalankan coroutine lain sambil menunggu respons

-- Ini sebabnya NSE "concurrent" meskipun single-threaded
-- Script tidak perlu peduli threading - Nmap menanganinya
```

---

## Library NSE Tingkat Lanjut

### `nmap` — Core API

```lua
local nmap = require "nmap"

-- Registry: shared state antar script
nmap.registry["my_data"] = {found = true, value = "secret"}
local shared = nmap.registry["my_data"]

-- Akses scan info
local backend = nmap.get_interface()
local timing = nmap.timing_level()   -- 0-5 (T0-T5)
local verbosity = nmap.verbosity()   -- 0,1,2,...

-- Port management
local portstate = nmap.get_port_state(host, {number=80, protocol="tcp"})
nmap.set_port_state(host, port, "open")

-- Tambahkan target baru saat scan berlangsung!
nmap.new_targets = {"10.0.0.1", "10.0.0.2"}  -- newtargets arg harus aktif
```

### `socket` — Low-Level Network

```lua
local nmap = require "nmap"

-- Buat socket raw
local sd = nmap.new_socket()

-- Connect
local status, err = sd:connect(host.ip, port.number, "tcp")
if not status then return "Connect failed: " .. err end

-- Send & Receive dengan timeout
sd:set_timeout(5000)  -- 5 detik
local status, data = sd:receive_bytes(10)

-- SSL wrapper
local ssl_sd = nmap.new_socket("ssl")
ssl_sd:connect(host.ip, 443, "tcp")
```

### `comm` — High-Level Communication

```lua
local comm = require "comm"

-- Connect, kirim, terima dalam satu panggilan
local status, data = comm.exchange(host, port, "GET / HTTP/1.0\r\n\r\n",
  {timeout=5000, proto="tcp"})

-- Try open: coba koneksi yang sudah ada atau buat baru
local sd, data = comm.tryssl(host, port, "GET / HTTP/1.0\r\n\r\n")
```

### `httpspider` — Web Crawler

```lua
local httpspider = require "httpspider"
local http = require "http"

-- Crawl website secara rekursif
local crawler = httpspider.Crawler:new(host, port, nil, {
  scriptname = SCRIPT_NAME,
  maxdepth = 3,
  maxpagecount = 50,
})

crawler:set_timeout(10000)

while true do
  local response, url, parent_url, depth = crawler:crawl()
  if not response then break end
  
  -- Analisis setiap halaman
  if response.body:find("password") then
    print("Found sensitive keyword at: " .. url)
  end
end
```

### `brute` — Brute Force Framework

```lua
local brute = require "brute"
local creds = require "creds"

-- Implementasi Driver untuk service custom
Driver = {
  new = function(self, host, port, options)
    local o = setmetatable({}, self)
    self.__index = self
    o.host = host
    o.port = port
    return o
  end,

  connect = function(self)
    self.socket = brute.new_socket()
    return self.socket:connect(self.host, self.port)
  end,

  login = function(self, username, password)
    -- Kirim credentials ke service
    local status, err = self.socket:send(username .. ":" .. password .. "\n")
    local status, data = self.socket:receive_lines(1)
    
    if data:find("SUCCESS") then
      return true, brute.Account:new(username, password, creds.State.VALID)
    elseif data:find("LOCKED") then
      return false, brute.Error:new("Account locked", true)  -- true = abort
    end
    return false, brute.Error:new("Invalid credentials")
  end,

  disconnect = function(self)
    self.socket:close()
  end
}

action = function(host, port)
  local engine = brute.Engine:new(Driver, host, port)
  engine.options.script_name = SCRIPT_NAME
  
  local status, result = engine:start()
  return result
end
```

---

## Thread Management dalam NSE

```lua
local nmap = require "nmap"
local stdnse = require "stdnse"

-- Jalankan beberapa task secara concurrent menggunakan thread
action = function(host, port)
  local results = {}
  local threads = {}
  
  -- Buat 5 thread concurrent
  for i = 1, 5 do
    local thread = stdnse.new_thread(function(idx)
      -- Lakukan pekerjaan
      stdnse.sleep(idx * 0.1)
      results[idx] = "Result from thread " .. idx
    end, i)
    table.insert(threads, thread)
  end
  
  -- Tunggu semua thread selesai
  for _, thread in ipairs(threads) do
    local status, err = stdnse.await(thread)
    if not status then
      stdnse.debug(1, "Thread error: %s", err)
    end
  end
  
  return stdnse.format_output(true, results)
end
```

---

## Menulis Script Multi-Phase yang Kompleks

### Script: Recursive Web Crawler + Vuln Scanner

```lua
description = [[
  Advanced web crawler yang menemukan form, endpoint, dan
  mencari potential vulnerabilities (exposed credentials, misconfigs).
]]

author = "Advanced User"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"discovery", "safe"}

local http = require "http"
local httpspider = require "httpspider"
local shortport = require "shortport"
local stdnse = require "stdnse"
local table = require "table"
local url = require "url"

portrule = shortport.http

-- Pola sensitif yang dicari
local SENSITIVE_PATTERNS = {
  {pattern = "password%s*=%s*['\"][^'\"]+['\"]", desc = "Hardcoded password"},
  {pattern = "api[_%-]?key%s*=%s*['\"][^'\"]+['\"]", desc = "API Key exposed"},
  {pattern = "BEGIN RSA PRIVATE KEY", desc = "Private key leaked"},
  {pattern = "DB_PASSWORD", desc = "Database password config"},
  {pattern = "AWS_SECRET", desc = "AWS credentials"},
}

action = function(host, port)
  local output = stdnse.output_table()
  local findings = {}
  local crawled_count = 0
  
  local crawler = httpspider.Crawler:new(host, port, "/", {
    scriptname = SCRIPT_NAME,
    maxdepth = nmap.registry[SCRIPT_NAME] and 
               nmap.registry[SCRIPT_NAME].depth or 2,
    maxpagecount = 100,
    withinhost = true,
  })
  
  crawler:set_timeout(8000)
  
  while true do
    local response, path = crawler:crawl()
    if not response then break end
    
    crawled_count = crawled_count + 1
    if not response.body then goto continue end
    
    -- Cek setiap pola sensitif
    for _, check in ipairs(SENSITIVE_PATTERNS) do
      if response.body:match(check.pattern) then
        table.insert(findings, string.format("[%s] %s", check.desc, path))
      end
    end
    
    -- Cek backup files
    if path:match("%.bak$") or path:match("%.old$") or path:match("%.orig$") then
      table.insert(findings, string.format("[Backup file] %s", path))
    end
    
    ::continue::
  end
  
  output["Pages crawled"] = crawled_count
  if #findings > 0 then
    output["Findings"] = findings
  else
    output["Note"] = "No sensitive patterns found"
  end
  
  return output
end
```

---

## Menggunakan `unpwdb`: Username & Password Database

```lua
local unpwdb = require "unpwdb"
local brute = require "brute"

action = function(host, port)
  local usernames, passwords

  -- Load username list (dari --script-args userdb= atau default)
  local status, usernames = unpwdb.usernames()
  if not status then return "Failed to load usernames: " .. usernames end
  
  local status, passwords = unpwdb.passwords()
  if not status then return "Failed to load passwords: " .. passwords end
  
  -- Iterasi
  for username in usernames do
    for password in passwords do
      -- Test credentials
      if try_login(host, port, username, password) then
        return string.format("Credentials found: %s:%s", username, password)
      end
    end
    passwords("reset")  -- Reset password list untuk user berikutnya
  end
  
  return "No valid credentials found"
end
```

---

## Registry: Sharing Data Antar Script

```lua
-- Script A: producer (berjalan lebih dulu via prerule)
prerule = function() return true end

action = function()
  -- Simpan data ke registry untuk dipakai script lain
  nmap.registry["shared_targets"] = {"10.0.0.1", "10.0.0.2"}
  nmap.registry["scan_config"] = {aggressive = true, depth = 3}
end

-- Script B: consumer (berjalan setelah Script A)
portrule = shortport.http

action = function(host, port)
  local targets = nmap.registry["shared_targets"]
  if targets then
    -- Gunakan data dari script lain
    for _, t in ipairs(targets) do
      -- process...
    end
  end
end
```

---

## Postrule: Script yang Berjalan Setelah Semua Scan

```lua
-- Script ini berjalan sekali setelah SEMUA host di-scan
postrule = function() return true end

action = function()
  local output = stdnse.output_table()
  
  -- Aggregasi semua data dari registry
  local all_vulns = nmap.registry["found_vulns"] or {}
  
  output["Total vulnerabilities found"] = #all_vulns
  
  if #all_vulns > 0 then
    -- Sort berdasarkan severity
    table.sort(all_vulns, function(a, b)
      return a.severity > b.severity
    end)
    output["Critical findings"] = all_vulns
  end
  
  return output
end
```

---

## Tips Optimasi NSE Script

```lua
-- 1. Cache HTTP responses
local cache = nmap.registry[SCRIPT_NAME] or {}
if not cache[host.ip] then
  cache[host.ip] = http.get(host, port, "/")
  nmap.registry[SCRIPT_NAME] = cache
end
local response = cache[host.ip]

-- 2. Gunakan pcre2 untuk regex kompleks (lebih cepat dari Lua patterns)
local pcre = require "pcre2"
local re = pcre.new("(?i)(passw(?:or)?d|secret|token)\\s*[:=]\\s*(\\S+)")

-- 3. Batasi output (jangan flood terminal)
if #results > 20 then
  results = {table.unpack(results, 1, 20)}
  table.insert(results, string.format("... dan %d lainnya", #results - 20))
end
```

---

*← [BAB 02: OS & Service Fingerprinting](bab02.md) · Selanjutnya → [BAB 04: Protocol-Level Packet Crafting](bab04.md)*
