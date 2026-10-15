# Rencana: Fitur Broadcast Siska (pemicu × isi × penerima)

## Context

Tim ingin Siska (agent dealer, `src/agents/dealer/`) bisa broadcast ke grup WhatsApp dengan banyak pola, bukan satu kasus saja. Tiga kasus awal (keputusan user):

1. **Barang masuk** → grup **dealer**. RI cabang Surabaya → semua grup dealer (nasional). RI cabang lain → hanya grup dealer cabang itu.
2. **Rotasi brand** → grup **dealer**, hari kerja jam **09:00 & 14:00**. Tiap slot **satu brand**, bergilir sampai semua brand tersapu lalu mulai lagi. Isinya **top 5 SKU menurut ABC nasional** (`abc_category_all`) dalam **1 kolase gambar** + caption nama/harga.
3. **Request sales lewat Nexus** ("broadcast item A,B,C,D 30 hari kerja ke depan") → grup **dealer** atau **grup internal** (= `DEALER_INTERNAL_GROUP` + `DEALER_BRANCH_GROUPS`). Dikirim di slot **09:00 & 14:00**, **hari kerja saja**. Nexus memanggil Siska lewat MCP.

Yang sudah ada: `src/agents/dealer/broadcast.py` (kind `receive_item`, teks saja, ke SEMUA grup dealer, `POST /dealer/broadcast` ber-HMAC, dedup `dealer_broadcasts`). Portal belum pernah memanggilnya. Belum ada penjadwal, kirim gambar, penargetan cabang, grup internal sebagai tujuan, atau jalur Nexus. wa-gateway `origin/main` sudah mendukung `type:"image"` + `mediaBase64` + `caption` dan membuang gema pesan kiriman API. Nexus sudah memanggil agent lain lewat MCP (preseden Valerica/`distribusi`).

## Desain inti

Satu broadcast = satu **aturan** (rule) yang menggabungkan tiga bagian yang bisa dipasang bebas. Kind baru nanti cukup menambah satu builder isi.

| Bagian | v1 |
|---|---|
| **Pemicu** | `event` (`receive_item`) atau `slots` (slot global `BROADCAST_SLOTS=09:00,14:00`, hari kerja saja, opsional `ends_on`) |
| **Isi** | `receive_item` (SKU dari event), `brand_rotation` (top N ABC per brand, bergilir), `items` (SKU tetap, harga/stok diambil segar tiap kirim), `text` (teks bebas) |
| **Penerima** | `groups: dealer | internal` × `branches: all | [cabang...] | origin` (+ `national_from` untuk `origin`) |

Tiga kasus sebagai baris aturan:
- Barang masuk: `trigger={event:"receive_item"}`, `content={kind:"receive_item"}`, `audience={groups:"dealer", branches:"origin", national_from:["surabaya"]}`.
- Rotasi brand: `trigger={slots:true}`, `content={kind:"brand_rotation", limit:5}`, `audience={groups:"dealer", branches:"all"}`, `state={last_brand_id}`.
- Request sales: `trigger={slots:true, ends_on:<hari kerja ke-30>}`, `content={kind:"items", skus:[...]}`, `audience={groups:"dealer"|"internal", branches:...}`, `created_by=<label token Nexus>`.

**Hari kerja** = bukan Minggu dan bukan libur HRIS (`https://hris-panel.dbklik.co.id/api/v1/kegiatan/hari-libur`, publik; sama dengan `WorkingTimeHelper::isWorkingDay` portal). Hasil sukses di-cache 24 jam. Kalau gagal, hanya Minggu yang dianggap libur, warning ditulis, dan dicoba lagi setelah 15 menit.

**Harga**: `receive_item` tetap mengumumkan barangnya, tetapi baris harga hanya muncul kalau `price_status == OK`. Ini konsisten dengan aturan Siska yang tidak menyebut harga bermasalah. Barang baru sering belum punya harga web. `brand_rotation` dan `items` melewati SKU yang harganya tidak OK atau stoknya kosong. `receive_item` tidak difilter stok, karena stok portal baru terisi lewat webhook Omni `/update-stock` setelah RI.

## Repo 1: dbklik-web-agent (worktree ini)

### Paket `src/agents/dealer/broadcast/` (menggantikan `broadcast.py`)
`DealerBroadcast`/`PostgresBroadcastStore` lama dihapus. Test dan skrip ditulis ulang, tidak dipertahankan lewat re-export. Fungsi yang masih berguna dipindah: `format_rupiah`, `render_receive_item`, `_HEADER`, `_price`, parser body lama.

- `rules.py`: dataclass `Rule`/`Trigger`/`Content`/`Audience` + parse/validasi dari dict (dipakai DB, MCP, CLI). Cabang ∈ `surabaya,jakarta,semarang,jogja,malang,bali`. `items` 1–10 SKU. Request maks. 90 hari kerja.
- `calendar.py`: `is_workday`, `nth_workday(start, n)` (untuk `ends_on`), `due_slots(rule, now, last_runs)`. Slot yang telat > `BROADCAST_SLOT_GRACE_MINUTES` (30) dilewati, jadi restart jam 13:00 tidak mengirim slot 09:00.
- `audience.py`: `resolve(audience, origin_branch, groups_with_subject, settings) -> list[jid]`.
  - `dealer`: grup sesi minus grup internal/cabang (logika `target_groups` lama; tetap patuh `DEALER_ONLY_GROUPS`, tetapi subject grup tetap diambil). Cabang grup dari `branch_for(subject, parse_branch_groups(...))`. `BranchGroup` cabang X = yang kata kuncinya memuat nama cabang portal (`SEMARANG`, ...). Grup dealer tanpa cabang hanya menerima broadcast nasional.
  - `internal`: `all` → `DEALER_INTERNAL_GROUP`; per cabang → JID `DEALER_BRANCH_GROUPS` cabang itu.
  - `origin`: cabang asal ∈ `national_from` → semua grup dealer; selain itu → grup dealer cabang asal.
  - Cek saat startup: tiap cabang cocok dengan tepat satu `BranchGroup`. Kalau tidak, tulis error di log.
- `content.py`: builder per kind → `list[OutMessage(text, image: bytes | None)]` lewat `PortalClient` (`src/agents/dealer/catalog.py`).
  - Pengayaan per SKU memakai `_enrich` tanpa `_attach_alternatives`. `get_products` selalu menambah alternatif (`catalog.py:430`), jadi diekstrak jadi fungsi publik `fetch_products(skus)`.
  - `receive_item`: body lama `{items:[{name, price}]}` tetap diterima. Body baru `{sku, qty}` diperkaya (nama, harga offline + `assess_price`).
  - `brand_rotation`: satu panggilan `abc.top(after_brand_id=state.last_brand_id, limit=5)` → brand berikutnya yang punya SKU layak + daftar SKU-nya. Lalu `fetch_products` untuk harga/stok, buang `is_hidden_variant`, `item.images.get` untuk foto, kolase. `last_brand_id` disimpan setelah run tercatat.
  - `items`: SKU tetap, harga/stok segar, kolase. Semua SKU tidak layak → run `skipped`.
  - `text`: teks apa adanya.
  - `abc.top` dan `item.images.get` TIDAK dimasukkan ke `PORTAL_TOOLS` (startup check), supaya agent tidak error sebelum portal deploy. Kegagalan saat run dicatat di ringkasan run.
- `collage.py`: Pillow (dependensi baru). Unduh foto (timeout, batas ukuran), grid 2–3 kolom, nomor 1..N sesuai caption, ubin placeholder berisi nama kalau foto tidak ada, JPEG lebar ~1080 px < 1 MB. Tanpa foto sama sekali → teks saja.
- `store.py`: Protocol + `InMemory*` + `Postgres*` (pola `escalation.py`).
  - `claim_run(rule_id, slot_key)` (ON CONFLICT DO NOTHING) = dedup slot dan dedup event (`slot_key` = `source_id` RI).
  - Run menyimpan teks + `bytea` kolase supaya kirim ulang tidak membangun ulang isi.
  - Delivery per grup: `pending → sent | failed`, `attempts`, `last_error`.
- `dispatch.py`: kirim per grup berurutan dengan jeda `BROADCAST_SEND_INTERVAL_SECONDS` (default 3). wa-gateway memakai satu antrean per nomor dengan concurrency 1, jadi tanpa jeda balasan Siska ke dealer tertahan di belakang broadcast. "sent" berarti diterima gateway (202). `BROADCAST_LOG_ONLY` (default true) hanya menulis log. Sebelum kirim ke grup dealer, panggil `DealerProcessor.note_outgoing(jid, text_or_caption)`, metode publik baru dengan pola `_relay` (`processor.py:321-332`). Ini mencatat pesan sebagai `role="agent"` (caption di-strip) supaya gema `fromMe` tidak jadi pesan "team", yang akan menutup eskalasi terbuka grup itu lewat `_resolve_escalations`, dan supaya Siska tahu isi broadcast saat dealer bertanya "yang nomor 2 masih ada?". Dilewati di mode log-only. GatewayError dicoba lagi di tick berikutnya, maks. 3 kali.
- `scheduler.py`: `run_worker(stop)`, loop asyncio tiap 30 detik (pola `run_reminders`/`compact.run_worker`), dimulai di `start_dealer` dan dihentikan di `stop_dealer`. Urutannya:
  1. kirim delivery yang masih `pending`/`failed` (termasuk dari event);
  2. untuk tiap aturan `slots` aktif, cek slot jatuh tempo → `claim_run` → bangun isi → buat delivery;
  3. aturan yang lewat `ends_on` → `finished`.

  Dalam satu slot: rotasi brand dulu, lalu request sales sesuai urutan dibuat. Batas `BROADCAST_MAX_PER_GROUP_PER_SLOT` (4); kelebihannya dilewati dengan warning, dan sudah diperingatkan di preview.

### Gateway (`src/channels/whatsapp/gateway.py`)
- `send_image(to, image: bytes, caption, mimetype="image/jpeg")` → `POST /sessions/{sid}/send` `{type:"image", mediaBase64, mimetype, caption}`.
- `list_groups_with_subject() -> list[tuple[jid, subject]]` (respons `GET /groups` sudah memuat `subject`).

### Database (Alembic, `down_revision="c5f2a8d41e90"`, model di `src/storage/models.py`)
- `dealer_broadcast_rules`: id, name, trigger/content/audience/state JSONB, status (`active|cancelled|finished`), created_by, source (`system|nexus|cli`), created_at, updated_at.
- `dealer_broadcast_runs`: id, rule_id FK, slot_key, status (`pending|sent|partial|skipped|failed`), text, image BYTEA NULL, summary JSONB, created_at; `uq_dealer_broadcast_runs_rule_slot`.
- `dealer_broadcast_deliveries`: id, run_id FK, group_jid, status, attempts, last_error, sent_at; unique (run_id, group_jid).
- Seed literal di migrasi (tanpa membaca settings): aturan "Barang Masuk" dan "Rotasi Brand Top 5", `active`. Pengaman kirimnya adalah `BROADCAST_LOG_ONLY=true` default.
- Tabel `dealer_broadcasts` tidak dipakai lagi. Aman, karena portal belum pernah memanggil endpoint ini. Penghapusannya di PR pembersihan terpisah.

### Endpoint event `POST /dealer/broadcast` (`dealer_webhook.py:273`)
- Secret baru `DEALER_BROADCAST_SECRET`, tidak lagi `DEALER_WEBHOOK_SECRET`. Secret webhook wa-gateway tidak dibagi ke portal: portal punya backdoor aktif, dan siapa pun yang memegang secret itu bisa memalsukan pesan dealer.
- Body baru `{kind:"receive_item", source_id, branch, items:[{sku, qty}]}`. Body lama tetap diterima; tanpa `branch` = nasional.
- Endpoint hanya memvalidasi, `claim_run` untuk tiap aturan aktif dengan `trigger.event == kind`, membangun isi, membuat delivery `pending`, lalu membalas **202** `{ok, duplicate, run_ids, groups}`. Pengiriman dikerjakan worker, sehingga portal tidak menunggu.
- `scripts/broadcast_receive_item.py` ditulis ulang (opsi `--sku`, `--branch`, `--name`, `--price`).

### Server MCP Siska untuk Nexus (`src/channels/mcp/siska_server.py`)
- Factory `build_siska_mcp()` (bukan singleton modul, karena `session_manager.run()` hanya boleh sekali per instance): `FastMCP("siska", stateless_http=True, streamable_http_path="/siska", transport_security=TransportSecuritySettings(enable_dns_rebinding_protection=False))`. Tanpa opsi terakhir, host produksi ditolak 421. Dipasang `app.mount("/mcp", ...)` → URL `/mcp/siska`. `session_manager.run()` dijalankan di lifespan `src/app.py`. Naikkan `requirements.txt` ke `mcp>=1.29,<2` (venv: 1.29.0).
- Auth: middleware ASGI di depan app mount. Token dicek dari `SISKA_MCP_TOKENS=label1:token1,...` (token per user Nexus → label `created_by`, pola Valerica) dan `SISKA_MCP_SERVICE_TOKEN`; token salah → 401. `(label, kind)` disimpan di `scope["state"]`. Tool membacanya lewat `ctx.request_context.request.state`. Service token hanya untuk `tools/list` (Nexus memakainya saat boot); `tools/call` dengan service token ditolak.
- Tool, semuanya dengan `ToolAnnotations` eksplisit. Tanpa anotasi, `_write_metadata` Nexus menganggap tool read-only dan tidak meminta approval.
  - `broadcast.preview` (`readOnlyHint=True`): terima nama bebas atau SKU, cocokkan via `item.search` portal. Mengembalikan kandidat untuk nama yang ambigu, SKU final, harga/stok, jumlah dan nama grup tujuan, slot, `ends_on`, peringatan batas per slot, dan contoh teks. Tidak menulis apa pun.
  - `broadcast.create` (`readOnlyHint=False`): hanya SKU persis; membuat aturan `items` atau `text`; `ends_on` = hari kerja ke-N dari besok.
  - `broadcast.list` (`readOnlyHint=True`): aturan aktif + hasil run terakhir (terkirim/gagal per grup).
  - `broadcast.cancel` (`readOnlyHint=False, destructiveHint=True`).
- Docs: ganti bagian "Broadcast Barang Masuk" di `docs/dealer-agent.md` dengan "Broadcast", lalu perbarui `.env.example`.

### Settings baru (`src/settings.py`)
`BROADCAST_LOG_ONLY` (true), `BROADCAST_SLOTS` (`09:00,14:00`), `BROADCAST_SLOT_GRACE_MINUTES` (30), `BROADCAST_SEND_INTERVAL_SECONDS` (3), `BROADCAST_MAX_PER_GROUP_PER_SLOT` (4), `DEALER_BROADCAST_SECRET`, `HRIS_HOLIDAYS_URL`, `SISKA_MCP_TOKENS`, `SISKA_MCP_SERVICE_TOKEN`.

## Repo 2: portal_produk_dbklik (worktree `broadcast-feature-50198f`)

1. **Tool MCP `abc.top`** (read-only), dengan pola tool portal:
   - File: `app/Mcp/Tools/AbcTopTool.php` + adapter `app/Mcp/Web/AbcTopTool.php` (`#[IsReadOnly]`), didaftarkan di `$tools` `app/Mcp/Servers/PortalServer.php`.
   - Input: `after_brand_id` (opsional), `limit` (default 5, maks. 20), `only_in_stock` (default true).
   - Perilaku: mengembalikan brand pertama dengan `id > after_brand_id` yang punya SKU layak (berputar ke awal kalau habis), plus top N-nya. Urutan: `abc_category_all` (A++ > A+ > A > B > C > D > E > F), lalu `wma_all` desc. Item suspend/terhapus dikecualikan, `ScopedQuery` diterapkan.
   - Output: `{brand:{id,nama}, items:[{sku, abc_category_all}]}`. `ColumnPolicy::filter` membuang kolom tanpa izin `abc-column-*`, jadi hanya kolom yang aman.
   - Izin `abc-list` sudah di-seed (`McpAbcPermissionSeeder`), tinggal diberikan ke user token Siska.
   - Test `tests/Feature/Mcp/McpAbcTopTest.php` (pola `McpItemAgingTest`).
2. **Hook barang masuk** di `POFinalReceiveAPIController::updatePoStatusAfterRi`:
   - Kumpulkan per SKU di dalam loop, hanya kalau `ri_data['action'] === 'create'` dan `qty_change[$i] > 0`. `quantity_ready` adalah qty invoice, bukan qty diterima. Cabang = `$request->warehouse[$i]`, yang sudah nama cabang portal.
   - Setelah `DB::commit()` (~505): `dispatch(fn () => app(DealerBroadcastClient::class)->receiveItem(...))->afterResponse()`, satu POST per cabang.
   - `app/Services/DealerBroadcastClient.php`: HMAC-SHA256 `X-Wa-Signature` dengan `services.dealer_broadcast.secret`, timeout 5 detik, `source_id = transaction_number:cabang`. Gagal hanya ditulis ke log channel `ri`; RI tidak pernah gagal karenanya.
   - `PoFinalController::updateStatus` tidak membuat RI, jadi tidak perlu disentuh.
3. Config `services.dealer_broadcast.{url,secret}` + `.env.example`.
4. `docs/features/broadcast-dealer.md` + entri `docs/changelog/` (aturan `CLAUDE.md` portal).

## Repo 3: nexus-agent (di luar folder sesi ini, PR/sesi terpisah)
- Tambah server `siska` di `src/tools/mcp/servers.json` (`SISKA_MCP_URL`, `user_token_key:"siska_mcp_token"`, `SISKA_MCP_SERVICE_TOKEN`, `streamable_http`).
- Tambah kolom `siska_mcp_token`, mengikuti pola `hris_mcp_token` di `db/models.py`, `repositories/user.py`, `routes/users.py`, `services/user.py`, `services/profile.py`, plus migrasi Alembic.
- Grant `siska__broadcast_*` per role di admin panel.
- Catatan skill/prompt: selalu `broadcast_preview` dulu, tunjukkan ke user, baru `broadcast_create`.

## Urutan pengerjaan
1. Agent: paket broadcast (rules, calendar, audience, content `receive_item`/`items`/`text`, store, dispatch, scheduler, migrasi + seed), endpoint event 202, `note_outgoing`, gateway `send_image`/subject. Test.
2. Portal: hook RI + `abc.top`. Test.
3. Agent: `brand_rotation` + kolase. Test.
4. Agent: server MCP Siska. Test.
5. Nexus: sesi/PR terpisah.

## Verifikasi
- Agent: `venv/Scripts/python -m pytest` (venv repo utama `E:\DBKlik Projects\dbklik-web-agent\venv`; pasang Pillow + `mcp>=1.29`), `ruff check src tests scripts`, `ruff format --check`, `mypy src tests scripts`. Cek dulu `tests/test_app.py`: daftar route-nya belum memuat `/dealer/broadcast`, jadi kemungkinan sudah gagal sebelum perubahan; perbarui untuk `/mcp/siska`. Test baru tanpa jaringan/DB (InMemory store, `httpx.MockTransport`, jam palsu):
  - routing penerima: RI surabaya → semua grup dealer; RI semarang → hanya grup ber-kata kunci SMG; grup tanpa cabang hanya menerima nasional; `internal` → JID grup internal/cabang;
  - `due_slots`: Minggu/libur HRIS dilewati, telat > grace dilewati, lewat `ends_on` → `finished`; `nth_workday`;
  - rotasi: `last_brand_id` maju hanya setelah run tercatat;
  - dedup: `claim_run` dua kali → sekali kirim; endpoint 202 + delivery pending; retry delivery gagal;
  - harga: `receive_item` tanpa baris harga kalau status ≠ OK; `items` melewati SKU tidak layak;
  - `note_outgoing`: gema caption tidak menjadi pesan "team";
  - kolase: ukuran, jumlah ubin, placeholder;
  - MCP lewat `TestClient`: token user bisa call, service token hanya list, token salah 401, anotasi tool benar, preview tidak menulis.
- Portal: `php artisan test tests/Feature/Mcp/McpAbcTopTest.php` + test hook RI dengan `Http::fake` (create → POST per cabang; update/delete/qty 0 → tidak ada POST; HTTP gagal → RI tetap sukses).
- Uji WA sungguhan:
  1. Set `DEALER_ONLY_GROUPS=<satu grup uji>` dan `BROADCAST_LOG_ONLY=false`.
  2. Jalankan `python -m scripts.broadcast_receive_item --sku ... --branch semarang`.
  3. Jalankan `python -m scripts.broadcast_run_slot --rule <id>` (CLI baru: jalankan satu slot sekarang).
  4. Panggil `broadcast.preview`/`create` dari klien MCP lokal.
  5. Periksa di grup: kolase muncul, caption benar, dan dealer yang membalas "nomor 2 masih ada?" dijawab Siska dengan konteks yang benar.

## Di luar fitur ini (mendesak)
`config/time.php` portal berisi backdoor upload file aktif (field `seopjpUplaoder`), masuk lewat commit "Cyber Panel Commit perubahan sebelum pull" 2025-08-24. Harus ditangani terpisah:
- hapus filenya;
- periksa `public_html` untuk file yang sudah terunggah;
- rotasi secret dan token portal.
