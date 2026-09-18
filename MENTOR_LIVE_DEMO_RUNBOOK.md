# Mentor Live Demo Runbook
# Fokus: yang TERLIHAT di terminal (percakapan, multi-user, compact, dll.)
# Bukan: daleman D2–D13 / pytest / SQL / lease / outbox

**Untuk:** Anda (presenter) menjalankan perintah manual di depan mentor  
**Durasi:** ~10–15 menit  
**Alat:** simulator CRM WhatsApp lokal (`python -m src.crm_sim`)

Ini **bukan** Meta WhatsApp produksi. Ini simulasi lokal dengan mock WhatsApp.

---

## 1. Setup terminal (sekali di awal)

Semua dari root repo:

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
```

(Ganti `PY` ke venv Anda jika beda.)

### Terminal 1 — Server (biarkan hidup)

```bash
$PY -m src.crm_sim server
```

Tunggu sampai muncul port (default `8080`) dan “Waiting for messages…”.

### Terminal 2 — Customer (chat pelanggan)

```bash
$PY -m src.crm_sim customer 628111111111
```

### Terminal 3 — Operator (eskalasi / agent on-off / balas)

```bash
$PY -m src.crm_sim operator
```

Perintah operator yang dipakai:

```text
/contacts
/open <wa_id>
/history
/escalations
/agent on|off
/reply <pesan>
/resolve <escalation_id>
/quit
```

**Stop server/CLI:** `Ctrl+C`.

---

## 2. Script pembuka (30 detik)

> “Saya demo CRM WhatsApp lokal: pelanggan chat di Terminal 2, operator pantau di Terminal 3, server di Terminal 1. Mentor lihat hasil eksternal — jawaban agent, eskalasi, multi-kontak, agent off, dan compact — bukan isi database.”

---

## 3. DEMO UTAMA (10 menit) — case percakapan

### CASE 1 — Pertanyaan yang ada di knowledge (known)

**Anda ketik di Terminal 2 (customer):**

```text
Ada Lenovo Legion?
```

**Yang mentor harus lihat:**

- Terminal 2: muncul balasan agent (mock inbox)
- Terminal 1: log CRM (retrieval / outcome)
- Terminal 3: `/open 628111111111` lalu `/history` → ada pesan customer + jawaban agent

**Kalimat Anda:**

> “Kalau knowledge ada, agent menjawab. Ini jalur normal.”

**Lanjut follow-up (opsional, 20 detik):**

```text
Apakah ada laptop RTX 4090?
```

atau:

```text
Apakah ada laptop ASUS?
```

---

### CASE 2 — Knowledge gap → eskalasi (bukan ngarang)

**Anda ketik di Terminal 2:**

```text
Apakah bisa cicilan 24 bulan?
```

**Yang mentor harus lihat:**

- Balasan aman / tidak mengarang skema cicilan sebagai fakta katalog
- Terminal 3:

```text
/escalations
```

atau:

```text
/open 628111111111
/history
```

→ ada escalation **OPEN**, reason knowledge gap.

**Kalimat Anda:**

> “Kalau knowledge tidak ada, sistem tidak mengarang. Operator mendapat eskalasi.”

**Lanjut resolve (tunjukkan kontrol manusia):**

Di Terminal 3, salin `id` eskalasi, lalu:

```text
/resolve <id>
```

Cek lagi:

```text
/escalations
```

→ status jadi **resolved**.

**Kalimat Anda:**

> “Eskalasi hanya selesai kalau operator resolve secara eksplisit.”

**Gap kedua (opsional):**

```text
Bisa custom rakit laptop?
```

→ eskalasi baru lagi (boleh lebih dari satu per kontak).

---

### CASE 3 — Agent OFF (customer chat, agent diam)

Di Terminal 3:

```text
/open 628111111111
/agent off
```

Di Terminal 2, ketik:

```text
Berapa harganya?
```

**Yang mentor harus lihat:**

- Tidak ada balasan agent baru (atau outcome agent disabled di log)
- Pesan customer tetap tercatat
- Operator masih bisa balas manual:

```text
/reply Sebentar, saya cek dulu harganya.
```

Lalu `/history` → pesan terakhir dari **operator**.

**Nyalakan lagi:**

```text
/agent on
```

Customer:

```text
Ada laptop RTX 4070?
```

→ agent jawab lagi.

**Kalimat Anda:**

> “Agent bisa dimatikan per kontak. Chat tetap masuk; yang menjawab bisa manusia.”

---

### CASE 4 — Multi-user / multi-kontak (3 pelanggan)

Buka **2 customer CLI tambahan** (Terminal 4 & 5), atau ganti wa_id:

```bash
# Terminal 4
$PY -m src.crm_sim customer 628222222222

# Terminal 5
$PY -m src.crm_sim customer 628333333333
```

**Script percakapan (gilir cepat):**

| Siapa | Ketik |
| --- | --- |
| `628111111111` | `Apakah ada laptop RTX 4090?` |
| `628222222222` | `Apakah bisa cicilan 24 bulan?` |
| `628333333333` | `Apakah ada laptop ASUS?` |
| `628111111111` | `Ada Lenovo Legion?` |
| `628222222222` | `Bisa custom rakit laptop?` |
| `628333333333` | `Ada laptop gaming?` |

Di Terminal 3:

```text
/contacts
```

**Yang mentor harus lihat:**

- Tiga kontak berbeda
- Kontak A & C: agent menjawab (known)
- Kontak B: eskalasi OPEN (gap)
- History per kontak **tidak campur**

```text
/open 628111111111
/history

/open 628222222222
/history

/open 628333333333
/history
```

**Kalimat Anda:**

> “Ini multi-user: tiap nomor punya percakapan sendiri. Gap di satu kontak tidak merusak jawaban kontak lain.”

**Alternatif satu perintah (tanpa ketik manual semua):**

```bash
$PY -m src.crm_sim scenario multi-contact
```

atau simulasi concurrent:

```bash
$PY -m src.crm_sim scenario concurrent
```

(Catatan di output: concurrent = simulasi lokal, bukan bukti multi-worker produksi.)

---

### CASE 5 — Compaction (riwayat dipadatkan)

Compaction **tidak** mudah ditunggu real-time (butuh idle ~2 hari / volume besar).  
Untuk mentor, jalankan scenario yang sudah ada — **hasil eksternal** tetap muncul di terminal:

```bash
$PY -m src.crm_sim scenario compaction-idle
```

**Yang ditunjuk ke mentor di output:**

- `BEFORE` → jumlah history / token
- `COMPACTION` → `triggered=True`, `reason=idle_gap`
- `AFTER` → `summary_present=True`, token turun

Lalu (opsional):

```bash
$PY -m src.crm_sim scenario compaction-activity
$PY -m src.crm_sim scenario compaction-acceptance
```

**Kalimat Anda:**

> “Kalau percakapan panjang/idle, sistem merangkum history jadi summary supaya konteks tidak membengkak — tanpa menghapus jejak yang masih dibutuhkan.”

Jangan bilang “saya tunggu 2 hari live”. Scenario mempercepat kondisi itu.

---

## 4. Urutan 10 menit yang disarankan

```text
00:00  Buka 3 terminal + kalimat pembuka
01:00  CASE 1 — known (Lenovo / RTX / ASUS)
03:00  CASE 2 — gap cicilan → /escalations → /resolve
05:00  CASE 3 — /agent off → chat → /reply → /agent on
07:00  CASE 4 — multi-kontak (manual atau scenario multi-contact)
09:00  CASE 5 — scenario compaction-idle (tunjuk BEFORE/AFTER)
10:00  Tutup: “Ini perilaku eksternal CRM; Meta live belum dihubungkan.”
```

---

## 5. Extended 15 menit (opsional)

| Extra | Cara | Yang dilihat |
| --- | --- | --- |
| Banyak eskalasi 1 kontak | `scenario escalation-resolve` | 2 OPEN → resolve 1 → 1 masih OPEN |
| Compaction lebih lengkap | `compaction-mock` / `compaction-acceptance` | ringkasan token before/after |
| Concurrent 3 kontak | `scenario concurrent` | 6 pesan hampir bersamaan, history tetap per kontak |

```bash
$PY -m src.crm_sim scenario escalation-resolve
$PY -m src.crm_sim scenario compaction-mock
$PY -m src.crm_sim scenario concurrent
```

---

## 6. Bank kalimat customer (copy-paste)

### Known (harap dijawab agent)

```text
Ada Lenovo Legion?
Apakah ada laptop RTX 4090?
Apakah ada laptop ASUS?
Ada laptop gaming?
Ada laptop RTX 4070?
Yang Lenovo ada?
```

### Gap (harap eskalasi)

```text
Apakah bisa cicilan 24 bulan?
Bisa custom rakit laptop?
Bagaimana custom build?
```

### Agent-off test

```text
Berapa harganya?
```

---

## 7. Cheat-sheet operator

```text
/contacts                         → daftar semua kontak + jumlah eskalasi
/open 628111111111                → pilih kontak A
/open 628222222222                → kontak B
/open 628333333333                → kontak C
/history                          → lihat chat kontak aktif
/escalations                      → semua eskalasi
/agent off                        → matikan agent kontak aktif
/agent on                         → nyalakan lagi
/reply <teks>                     → balasan manusia
/resolve <id>                     → tutup satu eskalasi
```

---

## 8. Checklist “lulus demo” (eksternal)

Centang di depan mentor:

- [ ] Known → ada jawaban agent di customer/history  
- [ ] Gap → ada escalation OPEN, bukan jawaban bohong  
- [ ] Resolve → status resolved  
- [ ] Agent off → customer chat tanpa balasan agent; operator bisa `/reply`  
- [ ] Multi-kontak → 3 history terpisah  
- [ ] Compaction scenario → BEFORE/AFTER summary terlihat  

---

## 9. Kalau macet (aman)

| Masalah | Apa yang dilakukan |
| --- | --- |
| Port 8080 bentrok | `$PY -m src.crm_sim server --port 8081` lalu customer/operator `--base http://127.0.0.1:8081` |
| Customer tidak dapat balasan | Cek Terminal 1 masih hidup; coba `/inbox` di customer |
| Lupa escalation id | `/escalations` lagi, salin id |
| Multi-kontak ribet | Ganti ke `$PY -m src.crm_sim scenario multi-contact` |
| Compaction bingungkan | Hanya jalankan `compaction-idle`, tunjuk 3 blok BEFORE/COMPACTION/AFTER |

Jangan git commit/push/reset. Jangan buka SQL/payload rahasia.

---

## 10. Penutup (15 detik)

> “Yang barusan terlihat: pelanggan chat, agent jawab kalau tahu, escalate kalau tidak tahu, operator bisa ambil alih, multi-user tidak saling campur, dan history panjang bisa di-compact. Itu demo eksternal CRM lokal. Integrasi Meta produksi terpisah.”

---

## Lampiran — daftar scenario siap jalan

```bash
$PY -m src.crm_sim scenario
# (tanpa nama → daftar)

$PY -m src.crm_sim scenario multi-contact
$PY -m src.crm_sim scenario concurrent
$PY -m src.crm_sim scenario agent-off
$PY -m src.crm_sim scenario escalation-resolve
$PY -m src.crm_sim scenario compaction-idle
$PY -m src.crm_sim scenario compaction-activity
$PY -m src.crm_sim scenario compaction-mock
$PY -m src.crm_sim scenario compaction-acceptance
```

STOP. Fokus presentasi = hasil di terminal, bukan daleman sistem.
