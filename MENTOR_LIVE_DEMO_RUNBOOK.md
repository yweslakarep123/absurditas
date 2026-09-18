# Mentor Live Demo Runbook

**Untuk:** Anda (presenter manusia)  
**Bukan untuk:** Cursor / agent otomatis menjalankan demo  
**Durasi:** 10 menit (utama) / 15 menit (opsional)  
**Status validasi:** D13 `PRODUCTION READY — VALIDATION COMPLETE`

---

## 0. Tiga mode — jangan dicampur

| Mode | Apa artinya | Cara demo di laptop |
| --- | --- | --- |
| **SIMULATOR** | `python -m src.crm_sim` → in-memory inbound ledger + Mock WhatsApp + CRM inline (`process_inline=True`) | **Jalur live utama** (Terminal 1–3) |
| **LOCAL PRODUCTION-STYLE** | Postgres + durable D2–D9 + `process_inline=False` + worker claim/fencing | **Tidak ada CLI server/worker terpisah di repo.** Buktikan lewat **pytest** (+ SQL read-only jika Postgres hidup) |
| **REAL META** | Live Cloud API credentials + traffic nyata | **NOT AVAILABLE — DO NOT DEMONSTRATE** kecuali mentor menyediakan credential & Anda sengaja siap |

Jangan pernah mengatakan output Mock WhatsApp = perilaku Meta produksi.

---

## 1. Arsitektur (60 detik — hafalkan)

```text
Webhook
  ↓
D2 Durable Inbound Ledger
  ↓
D3 Lease + Fencing
  ↓
D4 Ordering
  ↓
D5 Durable CRM Checkpoint
  ↓
D10 Context Budget + Compaction
  ↓
CRM Agent
  ├── Known → LLM
  └── Gap   → Escalation
  ↓
D6 Durable Outbox
  ↓
D7 Delivery
  ↓
D8 Callback / Reconciliation
  ↓
D11 Observability
```

Satu kalimat per lapisan:

| # | Kalimat |
| --- | --- |
| D2 | Setiap pesan provider disimpan durable dengan identitas unik sebelum ACK. |
| D3 | Worker mengklaim event dengan lease; worker basi tidak boleh commit. |
| D4 | Urutan per conversation dijaga di DB, bukan dari urutan HTTP. |
| D5 | State percakapan CRM punya checkpoint durable yang dikorelasi ke event. |
| D10 | Context dipotong/ditolak sebelum LLM jika melebihi budget. |
| CRM | Known → jawab lewat retrieval+LLM; gap → tidak mengarang, escalate. |
| D6 | Intent balasan ditulis ke outbox sebelum network call. |
| D7 | Delivery di luar transaksi DB; timeout → AMBIGUOUS, bukan “gagal pasti”. |
| D8 | Callback idempotent; tidak menjalankan CRM lagi. |
| D11 | Correlation tanpa PII/secret di log. |

---

## 2. Topology terminal (minimum)

Repo **tidak** menyediakan worker CLI production-style. Untuk live mentor, pakai **3 terminal simulator**:

```text
Terminal 1 — Application / Webhook (SIMULATOR)     [biarkan hidup]
Terminal 2 — Customer CLI                         [biarkan hidup]
Terminal 3 — Operator CLI                         [biarkan hidup]
Terminal 4 — Presenter (curl / pytest / SQL)      [opsional; boleh sama dengan Terminal 3 saat idle]
```

**Stop:** `Ctrl+C` di masing-masing terminal.

---

## 3. Prerequisites (jalankan sebelum mentor datang)

Semua perintah dari **root repo**:

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
```

### 3.1 Python

Di mesin validasi D13 dipakai:

```bash
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
$PY -c "import src; print('ok')"
```

Jika Anda punya venv sendiri di laptop presentasi, ganti `PY` ke interpreter yang sama yang biasa dipakai `pytest` / `src.crm_sim`.

### 3.2 Git safety (jangan mutate)

```bash
git status --short
```

**Expected (pola, bukan exact file list):**

- tepat **46** baris yang diawali `D ` (staged deletions handover)
- `src/llm/provider.py` masih modified (62+/3−)
- **Jangan** `git add` / `commit` / `push` / `restore` / dll. selama demo

### 3.3 Knowledge fixture (opsional, cepat)

```bash
$PY scripts/seed_crm_local.py --local
```

Ini memvalidasi fixture lokal; **bukan** write ke PostgreSQL produksi.

### 3.4 PostgreSQL (hanya untuk bukti durable / SQL)

Jika container D9 masih ada:

```bash
docker ps --format '{{.Names}} {{.Ports}}' | grep dbklik
```

Contoh URL yang dipakai validasi D13 (sesuaikan port jika berbeda):

```bash
export CRM_TEST_DATABASE_URL='postgresql://postgres:d9test@127.0.0.1:32773/dbklik_d9_test'
```

Migrasi (hanya jika DB fresh / mentor bertanya “apakah schema ada”):

```bash
DATABASE_URL="$CRM_TEST_DATABASE_URL" $PY -m alembic upgrade head
$PY -m alembic heads
# expected head: d9a3c8e2f510
```

Jika Postgres **tidak** hidup: **SKIP SQL live** — pakai pytest sebagai bukti durable (bagian Emergency Fallback).

### 3.5 Start simulator (wajib untuk main path)

**Terminal 1:**

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
$PY -m src.crm_sim server
```

**Expected:** banner port (default `8080`), webhook `POST /webhooks/whatsapp`, “Waiting for messages…”.

**Terminal 2:**

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
$PY -m src.crm_sim customer 628111111111
```

**Terminal 3:**

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
$PY -m src.crm_sim operator
```

Default base URL: `http://127.0.0.1:8080` (`CRM_SIM_HOST` / `CRM_SIM_PORT`).

---

## 4. Main 10-minute demo

### STEP A — Architecture (00:00–01:00)

**COMMAND:** *(tidak ada — bicara dari diagram di §1)*

**EXPECTED:** Mentor mengerti alur Webhook → … → D11.

**WHAT TO EXPLAIN:**

> “Ini CRM WhatsApp production-style: inbound durable, worker ber-lease, ordering per conversation, checkpoint, budget konteks, lalu outbox sebelum delivery. Simulator yang kita jalankan sekarang menunjukkan perilaku CRM; bukti durable Postgres sudah diverifikasi D2–D13.”

**PROVES:** Anda paham arsitektur, bukan hanya UI chat.

---

### STEP B — Health / readiness (01:00–02:00)

**COMMAND (Terminal 4 atau Terminal 3 idle):**

```bash
curl -s http://127.0.0.1:8080/health | python -m json.tool
curl -s http://127.0.0.1:8080/ready | python -m json.tool
```

**EXPECTED (SIMULATOR):**

```json
{
  "ok": true,
  "liveness": true,
  "runtime_mode": "simulator",
  "process_inline": true,
  ...
}
```

```json
{
  "ready": true,
  "durable_inbound": false,
  "harness": true
}
```

**WHAT TO EXPLAIN:**

> “`/health` = process hidup. `/ready` di simulator bilang ready tapi `durable_inbound=false` karena ledger in-memory. Di production-style Postgres, readiness mensyaratkan DB — itu yang diverifikasi D12/D13.”

**PROVES:** Anda membedakan liveness vs readiness vs durable dependency.

---

### STEP C — Known knowledge (02:00–04:00)

Fixture aktual dari `src/crm_sim/scenarios.py` / knowledge lokal:

**COMMAND (Terminal 2 — customer):**

Ketik:

```text
Ada Lenovo Legion?
```

**EXPECTED:**

- Terminal 1 log: `[CRM INBOUND]` / `[CRM RETRIEVAL]` / `[CRM OUTCOME]` (nama log aktual di banner server)
- Terminal 2: balasan mock inbox muncul (Mock WhatsApp)
- Terminal 3: `/history` atau buka contact menunjukkan percakapan agent

**Operator (Terminal 3), contoh:**

```text
/contacts
/open 628111111111
/history
```

**WHAT TO EXPLAIN:**

> “Retrieval menemukan knowledge. LLM dipanggil untuk merangkai jawaban. Ini jalur known — bukan escalation.”

**PROVES:** Jalur known knowledge end-to-end di simulator.

**Catatan jujur:** Outbox/delivery durable Postgres **tidak** terlihat di simulator in-memory. Untuk outbox durable → STEP bukti pytest di bawah / extended SQL.

---

### STEP D — Knowledge gap + escalation (04:00–06:00)

Pertanyaan gap aktual dari concurrent fixture (tidak ada di knowledge cicilan 24 bulan):

**COMMAND (Terminal 2):**

```text
Apakah bisa cicilan 24 bulan?
```

**EXPECTED:**

- Tidak “mengarang” jawaban inventif sebagai fakta katalog
- Escalation OPEN muncul di operator

**Terminal 3:**

```text
/escalations
```

**EXPECTED fields (aman):** `id`, `reason` (~`knowledge_gap`), `status`=`open`, `contact_id`

**Resolve (demonstrasi operator explicit):**

```text
/resolve <escalation_id>
```

atau:

```bash
curl -s -X POST "http://127.0.0.1:8080/operator/escalations/<ESCALATION_ID>/resolve" \
  -H 'Content-Type: application/json' \
  -d '{"resolved_by":"operator"}' | python -m json.tool
```

**EXPECTED:** `status=resolved`, `resolved_by=operator`, `resolved_at` terisi.

**WHAT TO EXPLAIN:**

> “Kalau knowledge tidak tersedia, sistem tidak mengarang lewat LLM sebagai fakta. Sistem membuat escalation dan menyiapkan jalur aman. Escalation hanya selesai lewat aksi operator eksplisit — `/reply` tidak auto-resolve.”

**PROVES:** Gap → escalation → resolve eksplisit (sesuai D1/D9).

---

### STEP E — Duplicate webhook (06:00–07:00)

Customer CLI **selalu** membuat `message_id` baru — untuk duplicate wajib **curl dengan ID sama**.

**COMMAND:**

```bash
BASE=http://127.0.0.1:8080
MID=wamid.demo-dup-001
BODY1=$(cat <<EOF
{"object":"whatsapp_business_account","entry":[{"id":"mock-waba","changes":[{"field":"messages","value":{"messaging_product":"whatsapp","metadata":{"display_phone_number":"15550001111","phone_number_id":"mock-phone-number-id"},"contacts":[{"wa_id":"628111111111","profile":{"name":"628111111111"}}],"messages":[{"from":"628111111111","id":"$MID","timestamp":"1760000000","type":"text","text":{"body":"Ada laptop ASUS?"}}]}}]}]}
EOF
)

echo "REQUEST 1"; curl -s -o /tmp/wa1.json -w "%{http_code}\n" -X POST "$BASE/webhooks/whatsapp" \
  -H 'Content-Type: application/json' -d "$BODY1"
cat /tmp/wa1.json; echo

echo "REQUEST 2 (identical)"; curl -s -o /tmp/wa2.json -w "%{http_code}\n" -X POST "$BASE/webhooks/whatsapp" \
  -H 'Content-Type: application/json' -d "$BODY1"
cat /tmp/wa2.json; echo
```

**EXPECTED:**

- HTTP 200 keduanya (simulator ACK)
- Response JSON: request 1 memproses; request 2 menunjukkan disposition duplicate / tidak menambah side effect CRM kedua (lihat log Terminal 1 + history tidak dobel balasan untuk event yang sama)

**WHAT TO EXPLAIN:**

> “Provider bisa retry. Duplicate webhook tidak boleh menghasilkan side effect kedua.”

**PROVES:** Idempotency identitas provider message (D2 semantics; durable uniqueness dibuktikan pytest).

---

### STEP F — Ordering (07:00–08:00)

**Live interactive ordering di production-style worker:**  
`NOT AVAILABLE — DO NOT DEMONSTRATE` sebagai CLI terpisah (tidak ada worker entrypoint).

**Ganti dengan bukti D13/D4 yang aman:**

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
export CRM_TEST_DATABASE_URL='postgresql://postgres:d9test@127.0.0.1:32773/dbklik_d9_test'
export PYTHONDONTWRITEBYTECODE=1

$PY -m pytest -p no:cacheprovider \
  tests/test_crm_final_release_gate.py::test_d13_ordering_gap \
  tests/test_crm_conversation_ordering.py::test_d4_c1_single_conversation_ordering \
  -q --tb=line
```

**EXPECTED:** `2 passed` (jika Postgres hidup). Jika DB down → Emergency Fallback.

**WHAT TO EXPLAIN:**

> “Ordering tidak bergantung pada urutan HTTP. Sequence 2 yang diklaim lebih dulu mendapat `WAITING_FOR_PREDECESSOR` sampai sequence 1 selesai.”

**PROVES:** D4 durable ordering.

---

### STEP G — Outbox + callback (08:00–09:00)

**Live outbox/callback CLI:** tidak ada perintah presenter satu-baris untuk mengisi outbox Postgres dari simulator in-memory.

**Demonstrasikan bukti D13:**

```bash
$PY -m pytest -p no:cacheprovider \
  tests/test_crm_final_release_gate.py::test_d13_e2e_known_knowledge \
  tests/test_crm_final_release_gate.py::test_d13_outbound_timeout_and_callback \
  tests/test_crm_delivery_callback.py::test_d8_c2_duplicate_delivered \
  -q --tb=line
```

**WHAT TO EXPLAIN:**

> “Response intent disimpan di outbox sebelum network. Timeout setelah request = AMBIGUOUS, bukan gagal pasti. Callback idempotent dan tidak menjalankan CRM lagi. Ini bukan Meta live.”

**PROVES:** D6–D8 lifecycle pada Postgres + mock/timeout transport.

---

### STEP H — PostgreSQL metadata + conclusion (09:00–10:00)

**Hanya jika** `CRM_TEST_DATABASE_URL` reachable:

```bash
docker exec -it dbklik-d9-postgres \
  psql -U postgres -d dbklik_d9_test
```

Lalu jalankan SQL di §6 (metadata only).

**Jika tidak:**

```bash
$PY -m pytest -p no:cacheprovider tests/test_crm_final_release_gate.py -q --tb=line
```

**EXPECTED D13:** `14 passed`

**WHAT TO EXPLAIN:**

> “D13 adalah release gate final: 415 full pytest PASS pada validasi; hari ini kita tunjukkan subset + simulator perilaku CRM. Meta live masih external dependency.”

**PROVES:** Validation complete tanpa mengklaim Meta E2E.

---

## 5. Optional extended demo (menuju 15 menit)

### OPT-1 — Agent OFF

**COMMAND (Terminal 3):**

```text
/open 628111111111
```

lalu di operator CLI (lihat help saat start): toggle agent off — perintah aktual:

```text
/agent off
```

(Implementasi: `POST /operator/conversations/{contact_id}/agent` dengan `{"enabled":false}`.)

**curl:**

```bash
curl -s -X POST "http://127.0.0.1:8080/operator/conversations/628111111111/agent" \
  -H 'Content-Type: application/json' \
  -d '{"enabled":false}' | python -m json.tool
```

**COMMAND (Terminal 2):** kirim `halo agent off demo`

**EXPECTED:** pesan tercatat; tidak ada balasan agent (mock inbox kosong / log agent disabled).

**Restore:**

```bash
curl -s -X POST "http://127.0.0.1:8080/operator/conversations/628111111111/agent" \
  -H 'Content-Type: application/json' \
  -d '{"enabled":true}' | python -m json.tool
```

Atau scenario non-server:

```bash
$PY -m src.crm_sim scenario agent-off
```

**Mark:** praktis untuk live — **lakukan jika ada waktu**.

---

### OPT-2 — Conflict duplicate

**OPTIONAL — SHOW ONLY IF TIME ALLOWS**

```bash
BASE=http://127.0.0.1:8080
MID=wamid.demo-conflict-001

# canonical
curl -s -X POST "$BASE/webhooks/whatsapp" -H 'Content-Type: application/json' -d "{\"object\":\"whatsapp_business_account\",\"entry\":[{\"id\":\"mock-waba\",\"changes\":[{\"field\":\"messages\",\"value\":{\"messaging_product\":\"whatsapp\",\"metadata\":{\"display_phone_number\":\"15550001111\",\"phone_number_id\":\"mock-phone-number-id\"},\"contacts\":[{\"wa_id\":\"628111111111\",\"profile\":{\"name\":\"628111111111\"}}],\"messages\":[{\"from\":\"628111111111\",\"id\":\"$MID\",\"timestamp\":\"1760000001\",\"type\":\"text\",\"text\":{\"body\":\"asli\"}}]}}]}]}"

# conflicting payload, same id
curl -s -X POST "$BASE/webhooks/whatsapp" -H 'Content-Type: application/json' -d "{\"object\":\"whatsapp_business_account\",\"entry\":[{\"id\":\"mock-waba\",\"changes\":[{\"field\":\"messages\",\"value\":{\"messaging_product\":\"whatsapp\",\"metadata\":{\"display_phone_number\":\"15550001111\",\"phone_number_id\":\"mock-phone-number-id\"},\"contacts\":[{\"wa_id\":\"628111111111\",\"profile\":{\"name\":\"628111111111\"}}],\"messages\":[{\"from\":\"628111111111\",\"id\":\"$MID\",\"timestamp\":\"1760000001\",\"type\":\"text\",\"text\":{\"body\":\"BERBEDA\"}}]}}]}]}"
```

Durable conflict proof (lebih kuat):

```bash
$PY -m pytest -p no:cacheprovider \
  tests/test_crm_final_release_gate.py::test_d13_duplicate_and_conflict -q --tb=line
```

---

### OPT-3 — Delivery timeout

**Tidak ada** tombol demo timeout di `src.crm_sim` CLI.

```bash
$PY -m pytest -p no:cacheprovider \
  tests/test_crm_final_release_gate.py::test_d13_outbound_timeout_and_callback -q --tb=line
```

**Explain:** timeout → `AMBIGUOUS` (bukan Meta live).

---

### OPT-4 — Worker crash / recovery

**OPTIONAL — DO NOT RISK LIVE DEMO**

Terlalu rumit untuk live (lease expiry timing). Tunjukkan:

```bash
$PY -m pytest -p no:cacheprovider \
  tests/test_crm_final_release_gate.py::test_d13_crash_after_outbox_no_duplicate \
  tests/test_crm_inbound_worker.py -q --tb=line
```

---

## 6. PostgreSQL read-only queries

Jalankan hanya di DB test lokal. **Jangan** `SELECT` `processing_payload`, `body_text`, `question_text`, secrets.

### Inbound

```sql
SELECT id, provider, provider_message_id, conversation_id, sequence, status,
       correlation_id, attempt_count, lease_owner, lease_token,
       identity_conflict_count, received_at, completed_at
FROM crm_inbound_events
ORDER BY received_at DESC
LIMIT 10;
```

### Checkpoint

```sql
SELECT conversation_id, source_event_id, source_sequence, checkpoint_id,
       writer_lease_owner, writer_lease_token, updated_at
FROM crm_checkpoint_heads
ORDER BY updated_at DESC
LIMIT 10;
```

### Escalation (tanpa teks pertanyaan)

```sql
SELECT escalation_id, conversation_id, contact_id, reason, status,
       source_event_id, source_message_id, best_score,
       resolved_by, resolved_at, created_at
FROM crm_escalations
ORDER BY created_at DESC
LIMIT 10;
```

### Outbox (tanpa body)

```sql
SELECT id AS outbox_id, source_event_id, conversation_id, source_sequence,
       status, outcome_kind, escalation_id, correlation_id,
       idempotency_key, provider_message_id, attempt_count,
       created_at, updated_at
FROM crm_outbox_messages
ORDER BY created_at DESC
LIMIT 10;
```

### Callback

```sql
SELECT id, provider, provider_message_id, callback_event_id, status,
       disposition, outbox_id, received_at
FROM crm_delivery_callbacks
ORDER BY received_at DESC
LIMIT 10;
```

---

## 7. Observability demonstration

Observability D11 emit ke logger sebagai baris `[CRM_OBS] ...` (tanpa body/PII).

**Live:** perhatikan Terminal 1 saat known/gap.

**Bukti test (aman & deterministik):**

```bash
$PY -m pytest -p no:cacheprovider tests/test_crm_observability.py -q --tb=line
```

**Yang ditunjuk:** `correlation_id`, `source_event_id`, `conversation_id`, `sequence`, `attempt`, `outbox_id`, `escalation_id`, `provider_message_id` (saat ada) — **bukan** nomor telepon / isi pesan / token.

**Kalimat:**

> “Correlation ID memungkinkan mengikuti satu lifecycle inbound→delivery tanpa memasukkan PII atau secret ke log.”

---

## 8. Final proof commands

Jalankan dari root repo. Set `CRM_TEST_DATABASE_URL` jika menjalankan tes Postgres.

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
export CRM_TEST_DATABASE_URL='postgresql://postgres:d9test@127.0.0.1:32773/dbklik_d9_test'
export PYTHONDONTWRITEBYTECODE=1

# D13 gate
$PY -m pytest -p no:cacheprovider tests/test_crm_final_release_gate.py -q --tb=line

# CRM + WhatsApp suite
$PY -m pytest -p no:cacheprovider tests/test_crm*.py tests/test_whatsapp*.py -q --tb=line

# Full suite (lama ~1 menit)
$PY -m pytest -p no:cacheprovider -q --tb=line

# Static
$PY -m ruff check src tests alembic/env.py
$PY -m ruff format --check src tests alembic/env.py
$PY -m mypy src tests
# expected residual: src/llm/provider.py:54 only

git diff --check
```

**Baseline D13 (referensi, bukan hard assert live):** full pytest = **415 PASS**.  
Jika angka berbeda di hari presentasi, **laporkan angka aktual** dari output.

---

## 9. Mentor Q&A (singkat)

**Q1 — Webhook masuk dua kali?**  
Identitas provider unik di D2; duplicate → disposition duplicate, tidak ada lifecycle CRM/outbox kedua.

**Q2 — Worker mati?**  
Lease D3 expire → reclaim; fencing menolak worker basi. Setelah outbox ada, reclaim reuse outbox (D6/D13).

**Q3 — Dua message bersamaan?**  
Per conversation: D4 memaksa urutan sequence. Conversation berbeda boleh parallel.

**Q4 — Kenapa outbox?**  
Intent balasan durable sebelum network. Crash setelah commit transaksi tidak menghilangkan intent; delivery bisa di-retry aman.

**Q5 — Knowledge tidak ada?**  
Retrieval gap → **tanpa** mengarang fakta via LLM → escalation OPEN + outcome aman + outbox bila perlu.

**Q6 — LLM gagal?**  
Gagal ditangani sebagai outcome/error lifecycle (retryable/terminal sesuai jalur); tidak menghapus inbound durable. Detail: lihat tes CRM/outbox LLM failure.

**Q7 — Exactly-once?**  
**Tidak diklaim ke Meta.** Jaminan repo: **at-least-once processing + idempotent durable side effects**.

**Q8 — Sudah terhubung Meta?**  
**Belum diverifikasi live.** Adapter + HMAC + wiring ada; E2E Meta = external dependency.

**Q9 — Kenapa PostgreSQL?**  
Sumber kebenaran durable bersama untuk inbound, lease, ordering, checkpoint head, outbox, callback, escalation — bukan state di memori proses.

**Q10 — Apa yang dibuktikan D2–D13?**  
Keamanan operasional jalur CRM/WhatsApp durable di repo (config fail-closed, migrasi, smoke E2E, regresi 415, static checks) — **bukan** deployment cloud / Meta live.

---

## 10. Troubleshooting (aman)

| Gejala | Cek |
| --- | --- |
| Postgres not running | `docker ps \| grep dbklik` — jika mati, **jangan drop DB**; skip SQL, pakai pytest fallback atau start ulang container test yang sudah dikenal |
| Port 8080 dipakai | `ss -ltn \| grep 8080` atau jalankan server dengan `$PY -m src.crm_sim server --port 8081` lalu customer/operator `--base http://127.0.0.1:8081` |
| Migration mismatch | `DATABASE_URL=... $PY -m alembic heads` / `upgrade head` — **jangan** downgrade destruktif di depan mentor |
| Worker not processing | Di **simulator** tidak ada worker terpisah (`process_inline=True`). Jangan bilang “worker macet” — itu harness inline |
| Webhook 403 | Signature required / header salah — simulator default biasanya tanpa wajib signature kecuali `--require-signature` |
| Webhook 503 | Ingest durable gagal (production-style Postgres). Di simulator jarang; cek log Terminal 1 |
| pytest skip CRM_TEST_DATABASE_URL | Export URL yang benar; tanpa itu tes PG di-skip |

**Jangan:** `git reset`, `git restore`, drop database produksi, hapus file repo, commit/push.

---

## 11. Emergency demo fallback

Jika live simulator / DB rusak, ucapkan:

> “Saya akan tunjukkan evidence test dari validation suite yang sama (D13), supaya kita tidak melakukan recovery berisiko di depan demo.”

Lalu **hanya**:

```bash
cd /home/daffa/Documents/dbklik/dbklik-web-agent
export PY=/home/daffa/Documents/dbklik/DBKL/bin/python
export CRM_TEST_DATABASE_URL='postgresql://postgres:d9test@127.0.0.1:32773/dbklik_d9_test'
export PYTHONDONTWRITEBYTECODE=1
$PY -m pytest -p no:cacheprovider tests/test_crm_final_release_gate.py -q --tb=line
```

Tanpa Git operations. Tanpa mengubah kode.

---

## 12. Presenter script (≈30 detik / langkah)

### Architecture

- **Saya melakukan:** menjelaskan diagram.  
- **Saya menunjukkan:** slide/teks D2–D11.  
- **Yang terjadi:** —  
- **Penting:** satu pipeline, satu sumber kebenaran.

### Health

- **Saya melakukan:** `curl /health` dan `/ready`.  
- **Saya menunjukkan:** `durable_inbound=false` di simulator.  
- **Yang terjadi:** process hidup; durable belum diklaim di harness ini.  
- **Penting:** bedakan liveness vs readiness.

### Known

- **Saya melakukan:** customer kirim “Ada Lenovo Legion?”.  
- **Saya menunjukkan:** balasan + log retrieval.  
- **Yang terjadi:** known → LLM.  
- **Penting:** jawaban berbasis knowledge.

### Gap

- **Saya melakukan:** “Apakah bisa cicilan 24 bulan?”.  
- **Saya menunjukkan:** `/escalations` lalu `/resolve`.  
- **Yang terjadi:** gap → escalation; resolve eksplisit.  
- **Penting:** tidak mengarang.

### Duplicate

- **Saya melakukan:** dua POST webhook identitas sama.  
- **Saya menunjukkan:** request kedua tidak menambah efek.  
- **Yang terjadi:** idempotent ingest.  
- **Penting:** provider retry aman.

### Ordering / outbox

- **Saya melakukan:** jalankan pytest D13 ordering + timeout/callback.  
- **Saya menunjukkan:** PASS.  
- **Yang terjadi:** durable semantics terbukti di Postgres.  
- **Penting:** live CLI worker belum ada; bukti validasi cukup untuk arsitektur.

### Closing

- **Saya melakukan:** sebut D13 PASS / 415 tests / Meta masih external.  
- **Saya menunjukkan:** `git status` 46 staged D, tanpa commit.  
- **Yang terjadi:** repo aman.  
- **Penting:** production-ready di level validasi repo.

---

## 13. What NOT to demonstrate

- Git commit / push / merge / restore / reset  
- Production deployment / DNS / secrets manager  
- Real customer data / live Meta traffic  
- Secret values, Authorization headers  
- `SELECT` payload / body / question_text  
- Menjalankan seluruh 415 test kecuali diminta  
- Mengklaim Mock WhatsApp = Meta  
- Mengklaim exactly-once delivery ke Meta  

---

## 14. Ringkasan kejujuran teknis (penting)

| Fitur | Live CLI demo? | Bukti alternatif |
| --- | --- | --- |
| Chat known/gap/agent-off/resolve | YES — `src.crm_sim` | — |
| Health/ready | YES — curl | — |
| Durable inbound/outbox/ordering/callback | NO standalone worker/server CLI | pytest D13/D2–D12 + SQL |
| Real Meta | NO | external |

Ini **bukan** kekurangan presentasi jika Anda nyatakan dengan jelas.

---

*Dokumen ini hanya instruksi. Cursor tidak menjalankan demonstrasi. Tidak ada D14.*
