# Airdrop Guide to AI Agent

> Panduan membangun dan menjalankan stack berburu airdrop yang dioperasikan AI agent — dari nol sampai siap jalan — dengan disiplin verifikasi, satu dompet farming, dan satu jalur browser.

Repo ini adalah **panduan**. Implementasi (script + dokumen perjalanan) tinggal di repo [`Rganthong/Agentic_wallet`](https://github.com/Rganthong/Agentic_wallet) dan di VM kerja (`~/workspace/`). Repo ini menjawab: stack ini apa, kenapa dibangun, kapan dipakai, dan bagaimana membangun + menjalankannya dari awal.

**Status:** stack sudah jalan dan terbukti. Tugas yang selesai penuh lewat stack ini: WL MemesFund, desk didapat, dan WL Cathereum, Registration ID didapat (keduanya 8 Nov 2026). Detailnya di [§4.11](#411-contoh-end-to-end-wl-memesfund-desk-didapat) dan [§4.12](#412-contoh-end-to-end-wl-cathereum-registration-id-didapat).

> ⚠️ Bukan saran keuangan. Tidak ada jaminan profit. Semua aksi dompet butuh preview + konfirmasi eksplisit sebelum eksekusi — tidak ada auto-claim.

---

## Daftar isi

- [1. Apa (What)](#1-apa-what)
- [2. Kenapa (Why)](#2-kenapa-why)
- [3. When / Where (Konteks)](#3-when--where-konteks)
- [4. Bagaimana (How)](#4-bagaimana-how)
- [5. Sumber tools](#5-sumber-tools)
- [6. Riwayat farming](#6-riwayat-farming)
- [7. Keamanan & rahasia](#7-keamanan--rahasia)

---

## 1. Apa (What)

### 1.1 Ringkasan

Stack ini adalah seperangkat tools + aturan operasi yang memungkinkan satu AI agent (Muse, di VM Linux) untuk mengerjakan tugas airdrop — whitelist, claim, dan program poin — atas nama user, dengan cara yang:

- **terverifikasi** (cek sumber primer + on-chain dulu, jangan percaya klaim situs),
- **aman** (satu dompet farming khusus, terpisah dari dompet utama; tidak ada private key yang dipegang agent),
- **manusiawi** (browser beneran, IP residensial rumah, tempo wajar, bukan burst),
- **tercatat** (setiap tugas punya preview di depan dan laporan hasil di belakang).

### 1.2 Komponen

| # | Komponen | Fungsi | Status |
|---|---|---|---|
| 1 | **Dompet Hunt (Privy)** — EVM `<alamat-dompet-hunt>`, Solana `<alamat-solana-hunt>` | Satu-satunya dompet farming. Dipakai untuk WL, claim, dan sebagai bridge signer. | ✅ Aktif, $0 (belum didanai) |
| 2 | **Bridge** (EIP-1193/6963 injected provider) | Menyuntikkan dompet Hunt ke dApp di browser farming, dengan allowlist origin + gerbang per-tugas (`arm.json`). | 🟡 Kode siap, setup ulang tertunda (lihat §4.7) |
| 3 | **Lane Telegram** (`tl.py`, Telethon) | Baca channel airdrop (read-only, allowlist keras 2 channel). Sumber tugas. | ✅ Aktif (akun Telegram) |
| 4 | **Lane Browser** (Nodriver + Chrome 157 headed) | Satu-satunya jalur kerja farming berbasis browser, lewat relay HP. | ✅ Aktif |
| 5 | **Relay HP** (Tailscale + microsocks + forwarder) | Egress IP residensial rumah user. | ✅ Aktif (Tailscale nyala terus) |
| 6 | **Akun X farming** (`@akun-X-farming`) | Akun X khusus farming (follow/like/repost/quote/OAuth read-only). | ✅ Aktif (sesi di profil browser farming) |
| 7 | **Audit tools** (`~/workspace/cyber-tools/bin/`) | Slither/Aderyn/Mythril/Heimdall/cast/forge/grype/syft + cek URL (urlscan/dnstwist). | 🟡 Sebagian terpasang (lihat §4.10) |
| 8 | **Aturan operasi** (SOUL.md §7 + workflow) | Preview + konfirmasi di depan, laporan hasil di belakang, stop-condition. | ✅ Disepakati & dipakai |

### 1.3 Alur kerja singkat

```
Telegram (Airdrop Finder / Tomket)
        │  baca post tugas (read-only)
        ▼
Verifikasi link + situs (sumber primer, urlscan/dnstwist)
        │
        ▼
PREVIEW lengkap ──► Konfirmasi eksplisit user
  (satu preview per langkah: aksi, origin/kontrak,
   fungsi, nilai, gas cap, apa yang di-approve/tidak)
        │  apa pun di luar daftar = BERHENTI + lapor
        ▼
Eksekusi via Nodriver + relay HP (X @akun-X-farming, dompet Hunt)
        │
        ▼
Laporan hasil (format baku) + catat di log farming
```

Format laporan hasil:

```
AIRDROP — [Proyek] — [tanggal, jam mulai–selesai WIB]
1. [aksi] → [hasil + bukti/link/tx]
2. [aksi] → [hasil + bukti/link/tx]
Keluar: [gas/biaya] | Masuk: [token/jumlah]
Status: [Selesai / Sebagian / Berhenti — alasan]
```

### 1.4 Yang BUKAN bagian dari stack ini
- **Dompet pribadi user** — penyimpanan jangka panjang, di luar stack, alamatnya tidak dicatat.

---

## 2. Kenapa (Why)

### 2.1 Masalah yang diselesaikan

| Masalah | Dampak kalau tanpa stack | Jawaban stack |
|---|---|---|
| Link airdrop palsu / drainer | Sekali klik connect wallet, dana habis | Semua link dicek dulu (urlscan + dnstwist + sumber primer); klaim hanya lewat Bridge di origin allowlist |
| Salah dompet (dompet utama ikut farming) | Dompet utama terekspos ke kontrak berisiko | Satu dompet Hunt khusus; BAW + dompet pribadi terpisah by design |
| IP datacenter kena flag | Akun X / situs memblokir atau menandai bot | Semua kerja browser lewat relay HP (IP residensial rumah) |
| Aksi terlalu cepat / burst | Akun kena rate limit / shadowban | Human behaviour: satu tugas sampai selesai, jeda bervariasi, tanpa burst massal |
| Tidak ada jejak | Tidak bisa audit, tidak tahu kenapa gagal | Preview di depan + laporan hasil + log farming per tugas |
| Terlalu banyak dompet | Bingung mana buat apa, risiko salah kirim | Wallet minimalism: SATU peran farming (Privy EVM + Solana) |
| Approval nyangkut | Approval jadi pintu drainer | Setelah tugas selesai, approval yang dibuat tugas di-revoke |

### 2.2 Prinsip desain

1. **Verifikasi sebelum percaya.** Counter situs, klaim "gratis", dan saldo tampilan pernah terbukti bohong. Yang berlaku: sumber primer + data on-chain.
2. **Preview gate.** Tidak ada preview (approve/mint/swap/claim/sign) sebelum (a) source kontrak dibaca — terverifikasi di explorer, atau analisis opcode/storage kalau tidak terverifikasi — DAN (b) audit tool yang relevan dijalankan dan hasilnya dilaporkan. Kalau tidak bisa, berhenti di laporan gap.
3. **Satu jalur, satu IP.** Semua kerja farming berbasis browser wajib lewat Nodriver + relay HP. Kalau rantai mati di tengah kerja: BERHENTI TOTAL. Jangan pindah IP, jangan lompat ke egress datacenter VM.
4. **Konfirmasi hanya untuk daftar langkah itu.** Konfirmasi berlaku HANYA untuk langkah yang di-preview. Signature/approval tambahan, jumlah/kontrak beda, CAPTCHA, error = BERHENTI + lapor.
5. **Read-only dulu.** Lane Telegram tidak bisa menulis apa pun by construction. Situs dibaca dulu sebelum ada klik.
6. **Agent tidak pegang private key.** Kunci dompet Hunt ada di enclave Privy. Agent hanya minta tanda tangan lewat API / device flow, tidak pernah menyimpan seed/private key.

### 2.3 Keuntungan

- **Konsisten:** tugas yang sama dikerjakan dengan langkah yang sama, tempo yang sama, dan jejak yang sama setiap kali.
- **Aman:** pemisahan dompet + allowlist + gerbang per-tugas mempersempit permukaan serangan ke satu dompet khusus yang saldonya terkontrol.
- **Cepat di jendela sempit:** kuota WL bisa habis dalam hitungan menit; preview di depan + eksekusi langsung setelah konfirmasi membuat tugas selesai dalam satu sesi.
- **Bisa diaudit:** preview, laporan, dan log memungkinkan user memeriksa ulang apa yang terjadi — termasuk apa yang TIDAK dilakukan (mis. follow/like yang hanya klik-klik-back).

### 2.4 Batasan (jujur)

- Stack ini **bukan auto-claim**. Setiap tugas butuh konfirmasi user.
- **Tidak menjamin profit.** Token gratis belum tentu bisa dijual; likuiditas dan vesting harus dicek per proyek.
- **CAPTCHA = berhenti.** Tidak ada solver, tidak ada tebak-tebakan.
- **Bergantung pada HP + Tailscale.** Kalau relay mati, semua kerja farming berhenti.
- **Verifikasi sosial bersifat klik-klik-back** (cara channel): tugas X ditandai selesai tanpa benar-benar follow/like/repost. Kalau proyek memverifikasi ulang, desk/spot bisa berisiko.

---

## 3. When / Where (Konteks)

### 3.1 Kapan sebaiknya dipakai

- **Whitelist (WL) proyek baru** — daftar early access / NFT spot (contoh: MemesFund).
- **Claim token gratis** — yang menarik dan gratis (hanya gas yang dipertaruhkan), walau cashability belum pasti.
- **Program poin / tugas sosial** — follow, like, repost, quote, join channel, submit alamat.
- **Jendela waktu sempit** — kuota cepat habis, pendaftaran dibuka singkat.
- **Riset alpha** — membaca channel Telegram allowlist untuk kandidat tugas berikutnya.

### 3.2 Kapan JANGAN dipakai

- **Paid mint / pembelian** tanpa keputusan eksplisit user (dengan hitungan skenario + batas rugi).
- **Proyek di chain lain** di luar yang didukung dompet Hunt, tanpa keputusan baru.
- **Origin di luar allowlist** Bridge — berhenti dan lapor.
- **CAPTCHA / challenge keamanan** — berhenti, jangan ditebak. Jalan keluar yang sah: user menyelesaikan sendiri di perangkatnya, lalu tugas dilanjutkan.
- **Situs mati / kontrak tutup** — parkir, jangan dipaksa (contoh: Tzero, Alchemist saat `mintOpen=false`).
- **Di luar jam / kondisi relay mati** — jangan pindah jalur.

### 3.3 Lingkungan (environment)

| Aspek | Nilai |
|---|---|
| Mesin | VM Linux (home `~` persisten; proses di luar `~` bisa hilang saat VM diganti) |
| Browser | Chrome 157.0.8103.38 (headed via Xvfb `:98`, 1280x900) — headless DITOLAK situs (403) |
| Egress | Relay HP via Tailscale (VM tag `tag:muse`; HP `<IP-tailscale-HP>`, microsocks `:1080`; forwarder `127.0.0.1:1180`) |
| IP saat jalan | IP residensial rumah (contoh saat MemesFund: `<IP-rumah>`) — verifikasi ulang tiap sesi |
| Python | venv Nodriver (`~/workspace/chip404/nodriver-venv`, nodriver 0.50.8); venv Telegram (`~/workspace/telegram-mcp/.venv`, telethon 1.45.0) |
| Node | v24.22.1, npm 10.9.4 (`~/workspace/privy-tools/`) |
| Akun | Telegram (akun pribadi); X farming `@akun-X-farming`; Privy App ID `<privy-app-id>` |

**Catatan VM:** VM bisa diganti kapan saja. Yang selamat: semua file di `~/workspace/` dan `~`. Yang hilang: proses berjalan (Xvfb, forwarder, relay), Chrome `/usr/bin` (stock), dan direktori yang belum sempat disalin ke repo. Karena itu semua script + dokumen penting harus ada salinannya di repo, dan setiap sesi dimulai dengan verifikasi ulang (relay, IP, sesi).

---

## 4. Bagaimana (How)

Panduan ini disusun kronologis: dari VM kosong sampai siap jalan. Ikuti berurutan; setiap langkah berakhir dengan cara verifikasi.

### 4.1 Prasyarat

**Akun & layanan (siapkan dulu):**

| Butuh | Keterangan |
|---|---|
| Akun Telegram | Untuk lane baca channel. Buat API ID + API Hash di https://my.telegram.org (gratis). |
| Akun X farming | Akun X khusus farming (di sini: `@akun-X-farming`). Jangan pakai akun utama / akun niche lain. |
| Akun Privy | App di dashboard Privy (App ID publik; App Secret HANYA transient, tidak pernah disimpan). |
| HP Android | Termux + aplikasi Tailscale + microsocks (lihat §4.4). |
| VM Linux | Dengan Node 24, Python 3.12, dan akses internet. |

**Aturan rahasia (berlaku di semua langkah):** tidak ada App Secret, private key, seed phrase, password, file sesi, atau token apa pun di repo — termasuk di git history. Kredensial hanya hidup transient di env/stdin saat dipakai, atau di file mode `0600` di VM.

### 4.2 Langkah 0 — Struktur direktori

Semua yang persisten tinggal di `~/workspace/`:

```
~/workspace/
├── chip404/
│   ├── chrome/                  # Chrome 157 hasil ekstrak .deb (survive VM reset)
│   │   └── chrome.deb           # installer tersimpan (sumber install ulang)
│   └── nodriver-venv/           # venv Nodriver (nodriver 0.50.8)
├── x-login/
│   ├── run.sh                   # wrapper: pastikan Xvfb :98 + forwarder, lalu jalan
│   ├── cookies.json             # sesi X @akun-X-niche-lain (0600) — BUKAN akun farming
│   └── profile/                 # profil Chrome @akun-X-niche-lain
├── telegram-mcp/
│   ├── .venv/                   # venv Telethon 1.45.0
│   ├── .env                     # (0600) TELEGRAM_API_ID/HASH/SESSION_STRING
│   └── scripts/
│       ├── tg_forwarder.py      # forwarder SOCKS 127.0.0.1:1180
│       └── tg_watchdog.sh       # cek + restart forwarder
├── telegram-lane/
│   ├── tl.py                    # CLI read-only (whoami/dialogs/read/search)
│   ├── session_from_mcp.py      # konverter sesi MCP → file sesi tl.session
│   ├── session/tl.session       # sesi Telegram (dir 700, file 600)
│   ├── allowlist.json           # [Tomket, Airdrop Finder] — TIDAK di-commit
│   ├── README.md / RUNBOOK.md
├── privy-tools/
│   ├── create-wallet.js         # buat 1 dompet per run + verifikasi personal_sign
│   ├── sign-test.js             # (jangan dipakai untuk create — bikin dompet baru tiap run)
│   ├── privy-solana-send-test.js
│   └── package.json             # @privy-io/node 0.38.0, viem, @solana/web3.js 1.98.4
├── memesfund/                   # contoh tugas #1 (driver + log + screenshot)
│   ├── probe1.py / step2.py / step3b.py / step5.py / step6.py / step7b.py / step8.py
│   ├── cookies-x-farming.json   # sesi X farming (0600)
│   ├── profile/                 # profil Chrome farming (sesi X + situs)
│   └── run-log.md               # log tugas
├── cyber-tools/bin/             # aderyn, anvil, cast, forge, grype, heimdall, syft, ...
└── farming-log.md               # (rencana) log semua farming — lihat §4.9
```

Repo pendamping (GitHub):

| Repo | Isi |
|---|---|
| `Agentic_wallet` | Dokumen perjalanan (`perjalanan-lengkap.md`, `privy-tutorial.md`, `kendala-saat-ini.md`, `tools-inventory.md`) + `scripts/` (telegram-lane, bridge-poc, okx-relay, farming-tools) |
| `GUIDE_AIRDROP_AGENT` | Repo ini — panduan publik |

### 4.3 Langkah 1 — Chrome + Xvfb + Nodriver

Kenapa ribet: Chrome stock di `/usr/bin` hilang saat VM diganti; build Chromium bawaan menolak start standalone; mode headless kena tembok bot (X 403); headed butuh display X virtual yang segar.

```bash
# 1. Chrome 157: ekstrak .deb tersimpan (atau unduh ulang, lihat §5)
mkdir -p ~/workspace/chip404/chrome
dpkg-deb -x ~/workspace/chip404/chrome/chrome.deb ~/workspace/chip404/chrome/
# biner: ~/workspace/chip404/chrome/opt/google/chrome/chrome

# 2. Venv Nodriver
python3 -m venv ~/workspace/chip404/nodriver-venv
~/workspace/chip404/nodriver-venv/bin/pip install nodriver==0.50.8

# 3. Xvfb (sudah ada di /usr/bin/Xvfb; kalau belum: apt install xvfb)
# 4. Wrapper run.sh — pastikan Xvfb :98 + forwarder jalan sebelum script apa pun:
cat ~/workspace/x-login/run.sh
```

Resep Nodriver yang terbukti jalan (dipakai semua script farming):

```python
import asyncio, os
for k in list(os.environ):
    if 'proxy' in k.lower():
        del os.environ[k]          # jangan biarkan proxy runtime ikut campur
import nodriver as uc

browser = await uc.start(
    browser_executable_path="/home/hatch/workspace/chip404/chrome/opt/google/chrome/chrome",
    headless=False, sandbox=False,
    browser_args=[
        "--proxy-server=socks5://127.0.0.1:1180",
        "--disable-dev-shm-usage",
        "--disable-gpu",            # WAJIB: tanpa ini headed Chrome hang di GPU init
        "--no-first-run",
        "--no-default-browser-check",
        "--window-size=1280,900",
        "--lang=id-ID,id",
    ],
    user_data_dir="/home/hatch/workspace/memesfund/profile",  # profil per tugas/akun
)
```

**Gotcha yang sudah terbukti:**

- Profil Chrome **baru** lambat saat first-run dan bisa melewati timeout connect Nodriver ("Failed to connect to browser"). Perbaiki: panaskan profil sekali manual (`chrome ... --remote-debugging-port=9222 about:blank`, cek `http://127.0.0.1:9222/json/version`, lalu matikan), baru jalankan Nodriver.
- Display Xvfb yang basi bikin headed Chrome hang. `run.sh` memakai `:98` segar; kalau `:98` basi, matikan dan buat display baru.
- Jangan `pkill -f <pola>` yang muncul di baris perintahmu sendiri — pakai trik kurung (`pkill -f "tg_forwarder[.]py"`).

**Verifikasi:** jalankan script probe yang membuka `https://api.ipify.org` di browser dan membaca IP-nya — harus IP relay HP (lihat §4.4), bukan IP datacenter VM.

### 4.4 Langkah 2 — Tailscale + relay HP (IP residensial)

Rantai egress: `Chrome → --proxy-server=socks5://127.0.0.1:1180 → tg_forwarder.py → Tailscale → HP (microsocks :1080) → internet rumah`.

**Di HP (Android):**

1. Pasang aplikasi **Tailscale**, login, biarkan nyala.
2. Pasang **Termux**, lalu di Termux:
   ```bash
   pkg install microsocks
   microsocks -i 0.0.0.0 -p 1080
   ```
   (Jalankan ulang tiap Termux restart — kalau tidak, forwarder timeout.)
3. Pastikan Tailscale di HP nyala **terus** selama sesi farming.

**Di VM:**

```bash
# Tailscale: pakai client bawaan, JANGAN install upstream / jalankan tailscaled sendiri
/opt/hatch/bin/tailscale up     # pertama kali muncul link approval (hanya user yang bisa approve)
/opt/hatch/bin/tailscale status # VM harus Connected; JANGAN jalankan `tailscale down`
                                # (down menghapus identitas Tailscale VM)

# Forwarder (mati diam-diam saat VM diganti — hidupkan via watchdog):
~/workspace/telegram-mcp/scripts/tg_watchdog.sh
```

ACL Tailscale yang dipakai: VM di-tag `tag:muse`; HP (`<IP-tailscale-HP>`) tidak di-tag; `tag:muse` boleh mencapai HP port `1080`. Exit node: `None` (benar).

**Verifikasi (wajib tiap sesi, sebelum tugas apa pun):**

```bash
curl --socks5-hostname 127.0.0.1:1180 https://api.ipify.org
# harus mengembalikan IP rumah (contoh: <IP-rumah>)
# JANGAN pakai --socks5 biasa: DNS lokal bisa mengembalikan 198.18.x.x
# yang tidak bisa dijangkau HP.
```

**Diagnosis cepat:**

| Gejala | Artinya | Perbaiki |
|---|---|---|
| SOCKS error 4 | microsocks jalan, tapi HP tidak ada internet | Cek internet HP / WiFi |
| Connect timeout setelah Termux restart | microsocks belum dijalankan ulang | Jalankan `microsocks -i 0.0.0.0 -p 1080` di HP |
| IP berubah di tengah sesi | IP seluler berotasi (CGNAT) | Verifikasi ulang; jangan lanjutkan sesi dengan IP berbeda |
| Watchdog lolos tapi egress gagal | Watchdog hanya membuktikan handshake forwarder+microsocks, BUKAN egress | Selalu cek `curl` di atas |

**Aturan keras:** kalau rantai mati di tengah kerja — BERHENTI TOTAL. Jangan lanjut dengan IP lain, jangan lompat ke egress datacenter VM, jangan ganti jalur diam-diam. Lapor ke user dan tunggu rantai hidup lagi.

### 4.5 Langkah 3 — Lane Telegram (sumber tugas)

Lane ini **read-only by construction**: `tl.py` hanya memanggil operasi baca (`whoami`, `dialogs`, `read`, `search`). Tidak ada kode tulis — tidak bisa post/edit/delete/join/react, dan membaca tidak menandai pesan sebagai terbaca.

```bash
# 1. Ambil kode (sumber: repo Agentic_wallet scripts/telegram-lane/)
#    File: tl.py, README.md, RUNBOOK.md, requirements.lock, setup.sh
#    Salin ke ~/workspace/telegram-lane/

# 2. Venv + dependensi (pin persis)
python3 -m venv ~/workspace/telegram-mcp/.venv
~/workspace/telegram-mcp/.venv/bin/pip install telethon==1.45.0 python-socks==3.2.1 qrcode==8.2
# atau: jalankan setup.sh dari repo (rebuild idempoten dari requirements.lock)

# 3. Kredensial — HANYA dari env saat runtime, jangan pernah tulis ke file:
export TELEGRAM_API_ID=...        # dari https://my.telegram.org
export TELEGRAM_API_HASH=...

# 4. Login sekali (pilih salah satu):
~/workspace/telegram-mcp/.venv/bin/python login_qr.py   # scan QR dari HP
# atau: login_otp.py (kode OTP dibaca dari stdin; 2FA → exit 3, helper menolak password)
# atau (yang dipakai 8 Nov 2026): konversi dari session string telegram-mcp
#      lewat session_from_mcp.py — lihat gotcha di bawah.

# 5. Verifikasi + bangun allowlist (whoami/dialogs TIDAK ter-gerbang):
TL_PROXY=socks5://127.0.0.1:1180 ~/workspace/telegram-mcp/.venv/bin/python tl.py whoami
TL_PROXY=socks5://127.0.0.1:1180 ~/workspace/telegram-mcp/.venv/bin/python tl.py dialogs
# tulis ID channel yang diizinkan ke allowlist.json, contoh:
# [<id-channel-1>, <id-channel-2>]   # Tomket Lovers Airdrop, Airdrop Finder

# 6. Pakai:
TL_PROXY=socks5://127.0.0.1:1180 ~/workspace/telegram-mcp/.venv/bin/python tl.py search <id-channel-2> memesfund
```

**Aturan allowlist:** file tidak ada = tolak semua; chat di luar daftar = ditolak. `allowlist.json` asli (berisi ID asli) **tidak di-commit**; di repo hanya `allowlist.template.json`.

**Gotcha sesi (penting, ditemukan 8 Nov 2026):** `MTProtoSender` menangkap `session.auth_key` **saat `TelegramClient` dibuat**. Jadi file sesi harus terisi penuh (dc + auth_key) **SEBELUM** `TelegramClient(...)` dipanggil — kalau tidak, sender mulai dengan kunci `None` dan membuat kunci baru yang tidak terdaftar saat `connect()`, lalu `is_user_authorized()` jadi `False`. Pola benar: isi `SQLiteSession` dulu (`set_dc` → `auth_key` → `save()`), baru buat client dari objek sesi itu. Lihat `session_from_mcp.py`.

**Volume:** baca dengan tempo manusia, jangan brutal. Kalau muncul respons flood/limit, berhenti dan lapor.

### 4.6 Langkah 4 — Dompet Hunt (Privy)

Satu peran farming = dompet Privy EVM + Solana. BAW dan dompet pribadi tetap terpisah.

```bash
# 1. Buat App di dashboard Privy (https://dashboard.privy.io)
#    Catat App ID (publik). App Secret HANYA dipakai transient per sesi.

# 2. Tooling (sumber: npm — lihat §5; kode lokal ~/workspace/privy-tools/)
cd ~/workspace/privy-tools
npm install   # @privy-io/node@0.38.0, viem, @solana/web3.js@1.98.4 (pin 1.x wajib)

# 3. Buat dompet (SATU dompet per run; jangan pakai sign-test.js untuk create
#    karena ia membuat dompet baru setiap run):
PRIVY_APP_ID=<privy-app-id> PRIVY_APP_SECRET='<transient, jangan disimpan>' \
  node create-wallet.js
# Hasil (8 Nov 2026):
#   EVM:    <alamat-dompet-hunt>  (id <id-dompet-hunt-evm>)
#   Solana: <alamat-solana-hunt> (id <id-dompet-hunt-solana>)
# Verifikasi: personal_sign EVM lolos. Saldo $0, belum didanai, tidak ada dana bergerak.

# 4. Cek App Secret TIDAK tersimpan di mana pun setelah dipakai.
```

**Jebakan API Privy yang sudah terverifikasi:**

- Konstruktor: `new PrivyClient({ appId, appSecret })` (huruf `d` kecil).
- Panggil RPC: `rpc(walletId, {...})`.
- `eth_signTransaction` tidak memakai `caip2`/`chain_type` di input tx (chain dari `transaction.chain_id`, field snake_case).
- Tx Solana diverifikasi via HTTP `getSignatureStatus` (websocket gagal lewat proxy VM).
- `api.privy.io` bisa dijangkau langsung dari VM.

**Kebijakan (policy) — pakai:** policy bisa dipasang ke dompet yang sudah ada via `wallets().update(id, { policy_ids })`. Policy dievaluasi di TEE dan default DENY — jadi policy HARUS mencakup aturan untuk SETIAP method RPC yang dipakai (`personal_sign`, `eth_signTransaction`, `eth_sendTransaction`, `eth_signTypedData_v4`) atau call ditolak. Catatan: policy hanya benar-benar mengikat kalau agent memegang authorization key sebagai SIGNER sementara user memegang App Secret sebagai owner (agent yang memegang App Secret bisa menghapus policy-nya sendiri).

**Jalan yang direkomendasikan (perlu verifikasi dulu): Privy Agent CLI** (`@privy-io/agent-wallet-cli`) — user approve sekali via browser device flow, agent tidak pernah memegang App Secret/private key, ephemeral key per tx, sesi 30 hari auto-rotate, pantau/revoke di `agents.privy.io`. Yang harus diverifikasi dulu: apakah wallet-nya hidup di sandbox app Privy (bukan app kita) dan di mana sesi tersimpan di VM headless (kemungkinan fallback file terenkripsi `~/.privy/session.json`).

**Quick win (pakai):** `sendTransaction` (sign + broadcast satu call, gas/nonce otomatis) untuk kirim EVM biasa; **idempotency key** di setiap retry call yang mengubah state; untuk Solana, dummy `recentBlockhash` `11111111111111111111111111111111` (Privy yang mengisi aslinya).

**Yang di-skip (sengaja):** gas sponsorship (arahnya salah), key quorum (overkill solo), MPP, OpenClaw (kredensial plaintext di `~/.openclaw/openclaw.json` — melanggar aturan no-store), Bankr, Hyperliquid, AgentCore (kredensial ke AWS), auto.exchange, Virtuals EconomyOS, wallet automations (konflik dengan preview gate).

**Keputusan yang masih terbuka (user):** pendanaan dompet Hunt (jumlah + waktu); pasang policy farming atau tidak; Agent CLI vs authorization-key-as-signer; hapus repo supply lama (App Secret masih ada di git history-nya).

### 4.7 Langkah 5 — Bridge (dompet Hunt ke dApp)

Bridge adalah provider EIP-1193/6963 yang disuntikkan ke browser farming, sehingga dApp mengira ada wallet — padahal yang menandatangani adalah dompet Hunt (Privy) di belakang, dengan gerbang per-tugas.

**Desain:**

- **Allowlist origin** (`policy.json`): daftar origin yang boleh memakai Bridge. Origin di luar daftar = tolak.
- **Gerbang per-tugas** (`arm.json`): ditulis HANYA setelah preview disetujui user. Berisi origin + method yang diizinkan + batas nilai + TTL. Di luar itu = tolak.
- **Signer:** `BRIDGE_SIGNER=privy` (dompet Hunt). Signer burner PoC lama (`0x7843…fE556`) sudah **dipensiunkan**.
- **Terbukti jalan:** di `chainlist.org` + transaksi Sepolia dengan signer burner lama (didanai 0.05 SepETH oleh user).

**Setup (status: kode ada di repo, setup ulang tertunda):**

```bash
# 1. Ambil kode dari repo Agentic_wallet: scripts/bridge-poc/
# 2. Arahkan signer ke dompet Hunt baru (BRIDGE_SIGNER=privy)
# 3. Jalankan ulang matriks uji gerbang (gate test)
#    - Signing test butuh App Secret transient lagi per sesi
# 4. TODO: WalletConnect client untuk situs yang hanya support WC
```

**Rekomendasi (sudah disampaikan ke user):** jangan tulis ulang dari nol — pakai ulang kode yang sudah terbukti, dengan deployment baru + signer Privy baru + uji gerbang ulang. Satu-satunya bagian yang layak ditulis baru adalah WalletConnect client.

**Batasan:** klaim di situs HANYA lewat Bridge "Hunt Wallet" di origin allowlist. BAW bukan dompet farming.

### 4.8 Langkah 6 — Akun X farming (`@akun-X-farming`)

Akun X khusus farming. Dipakai untuk: OAuth read-only ke situs WL, dan follow/like/repost/quote/reply (klik-klik-back atau beneran — sesuai preview per tugas).

```bash
# 1. Sesi disimpan sebagai cookies (auth_token + ct0), file mode 0600:
#    ~/workspace/memesfund/cookies-x-farming.json
#    Pola ambil token (sudah disepakati): user menaruh 2 nilai sesi di README
#    repo private → agent membaca → memakai → MENGHAPUS dari README.
#    JANGAN pernah kirim nilai token di chat, dan jangan catat nilainya di memori/file biasa.

# 2. Verifikasi handle SEBELUM authorize apa pun (wajib):
#    - Pasang cookies di profil browser farming
#    - Buka https://x.com/home, baca link profil (data-testid="AppTabBar_Profile_Link")
#    - Harus /<akun-X-farming>. Kalau bukan, BERHENTI — jangan authorize.

# 3. OAuth ke situs: periksa scope di URL authorize
#    (contoh MemesFund: scope=users.read+tweet.read, redirect ke situs itu).
#    Tolak kalau ada izin post/DM/follow/like/manage.
```

### 4.9 Langkah 7 — Aturan operasi (workflow per tugas)

Ini bagian terpenting. Tools tanpa aturan ini berbahaya.

**Sebelum tugas:**

1. Baca tugas dari channel allowlist (lane Telegram).
2. Verifikasi link: urlscan + dnstwist untuk URL publik (jangan kirim data wallet/pribadi ke pihak ketiga), lalu verifikasi sumber primer langsung (buka situsnya, baca isinya). Kalau dnstwist belum terpasang, kompensasi = urlscan + verifikasi sumber primer.
3. Buat **preview lengkap di depan** — satu preview per langkah: aksi, origin/kontrak + alamat, fungsi, nilai dikirim, gas cap, apa yang di-approve / tidak di-approve.
4. Minta **konfirmasi eksplisit** user. Konfirmasi berlaku HANYA untuk daftar langkah itu.

**Saat eksekusi:**

- Kerjakan satu tugas sampai selesai, tempo wajar — baca instruksi/konten dulu sebelum like, repost, join, atau submit form.
- Jeda bervariasi antar-task; jangan burst massal; variasikan urutan task antar sesi.
- Interaksi kontekstual sesuai task, bukan copy-paste spam yang sama persis.
- Hormati rate limit, CAPTCHA, dan aturan platform.
- Klik elemen yang benar-benar terlihat, setelah halaman dimuat dan kontennya dibaca. Scroll ke elemen sebelum mengklik. Ketik per karakter dengan kecepatan wajar. Satu aksi tuntas dulu (halaman settle, hasil terlihat) sebelum aksi berikutnya.
- **Stop-condition (apa pun yang terjadi, BERHENTI + lapor):** signature/approval tambahan, jumlah atau kontrak beda, CAPTCHA, error, origin di luar allowlist, permintaan wallet connect / signature / pembayaran yang tidak ada di preview.

**Setelah tugas:**

- Kirim **laporan hasil saja** (format di §1.3). Jangan kirim log mentah.
- Catat di log farming: link yang dipakai, waktu, hasil (sukses/gagal), reward yang ditargetkan. Jangan catat private key/seed/password.
- Revoke approval yang dibuat oleh tugas (kecuali allowance sudah verifikasi 0). Jangan auto sign-out BAW — user yang sign out sendiri.
- Kalau sesi akun dikompromikan, laporkan ke user.

**Preview gate (§4 kontrak) — berlaku untuk semua klaim yang butuh signature/transaksi:**
Jangan buat preview apa pun sebelum (a) source kontrak didapat dan dibaca — terverifikasi di explorer, atau analisis opcode/storage kalau tidak terverifikasi — DAN (b) audit tool yang relevan sudah dijalankan dan hasilnya dilaporkan. Kalau source tidak bisa didapat atau tool tidak bisa jalan, BERHENTI di laporan gap: tidak ada preview, sampai user secara eksplisit memerintahkan lanjut untuk kasus itu, dengan gap tersebut ditulis di preview itu sendiri.

**Aturan cek token (sebelum buy/sell apa pun):** cek kontraknya sendiri, jangan percaya pair/volume/jumlah holder/aktivitas buy-sell saja. Cek harus mencakup: source terverifikasi (atau analisis opcode/storage), `balanceOf` vs storage balance-mapping asli, simulasi transfer/swap (bukan cuma quote router), slot tax + wallet tax, dan blacklist/restriksi trading — lalu laporkan hasilnya sebelum preview/eksekusi.

**Strategi klaim gratis:** klaim token walau belum yakin bisa dicairkan, selama gratis dan menarik — ketidakpastian likuiditas saja tidak menghentikan klaim gratis, karena hanya gas yang dipertaruhkan.

### 4.10 Langkah 8 — Audit & cek link

**Audit kontrak (sebelum preview, lihat §4.9):**

```bash
export PATH="$HOME/workspace/cyber-tools/bin:$PATH"
# Alat yang terverifikasi ada di ~/workspace/cyber-tools/bin/:
# aderyn, anvil, cast, chisel, forge, grype, heimdall, syft
# (slither / mythril: verifikasi keberadaannya sebelum dipakai.
#  dnstwist + urlscan-check SUDAH terpasang 8 Nov 2026 — lihat §5;
#  SBOM venv tercatat di ~/workspace/cyber-tools/url-tools-sbom.json)

# Contoh pemakaian:
cast sig "claimReward(uint256[])"          # hitung selector
forge build                                 # lalu slither --compile-force-framework Foundry
aderyn --help
python3 -m mythril analyze -a <alamat> --rpc bsc-dataseed2.binance.org:443 --rpctls true \
  --execution-timeout 600 --max-depth 22 --outform json
heimdall decompile <alamat> --rpc-url https://bsc-dataseed.binance.org/
```

Aturan decompile: untuk kontrak yang source-nya NULL di explorer, decompile langsung sebagai langkah standar (Heimdall dulu, Dedaub hanya kalau Heimdall tidak jelas atau keputusannya kritis). Decompile menaikkan KEJELASAN saja, tidak pernah menaikkan skor — skor tetap capped rendah dan preview tetap ter-gate.

**Cek link (sebelum dipakai):** urlscan + dnstwist untuk URL publik saja. Jangan kirim data wallet/pribadi ke pihak ketiga. Cek juga: domain meniru (typosquatting), IOC tanpa data wallet, dan SBOM sebelum install tool baru.

**Catatan RPC:** baca on-chain via public BSC RPC (`https://bsc-dataseed.binance.org/`, fallback dataseed2). RPC publik kadang flaky — rotasi endpoint + retry. `eth_getLogs` publik sering gagal "limit exceeded" walau range 500 blok. Return kosong dari `getPair` bukan bukti tidak ada pair (quirk RPC yang sudah terbukti).

### 4.11 Contoh end-to-end: WL MemesFund (desk didapat)

Tugas pertama yang selesai penuh lewat stack ini (8 Nov 2026, 11:00–11:20 WIB).

**Tugas:** dari channel Airdrop Finder (post 7 Nov 15:34 UTC): "WL Memesfund → Register: https://memesfund.xyz/r/<kode-referral> → Task X gausah dikerjain, cukup klik klik, back → Submit EVM Address → Done".

**Yang dicek dulu:** situs `memesfund.xyz` dibaca langsung — game idle (hedge fund memecoin, $MFUND), chain Elysium buka 9 Nov, desk = early access + spot NFT. Link cocok dengan post channel. (Gap yang diungkap ke user: urlscan tidak jalan — tidak ada API key di VM.)

**Preview (disetujui user: "Lanjut + referral"):**

1. Nodriver lewat relay HP (IP <IP-rumah>) buka link referral → klik Continue.
2. Ikuti 4 tugas bareng Dex (klik-klik sesuai instruksi channel).
3. Task X: tidak dikerjakan beneran — cukup klik, back. Kalau situs minta connect X beneran → BERHENTI + lapor.
4. Submit EVM address `<alamat-dompet-hunt>` (dompet Hunt Privy). Hanya isi form, bukan signature.
5. Verifikasi desk didapat → laporan hasil.

**Yang terjadi (sesuai workflow):**

| Tahap | Hasil |
|---|---|
| Probe halaman | Chat Dex terkunci (`disabled`, placeholder "Sign in to chat") sampai X sign-in |
| Klik "Sign in with X" | Ke halaman OAuth X (`scope=users.read+tweet.read`, read-only). Back tanpa authorize → tetap terkunci → **BERHENTI sesuai preview, lapor ke user** |
| Keputusan user | "Lanjut pakai @akun-X-farming" |
| Sesi X | Sesi lama hilang saat VM diganti; token sesi diambil dari README repo private, handle diverifikasi `@akun-X-farming` di browser farming sebelum authorize |
| OAuth | Guard lolos (scope + redirect + handle + tanpa izin post/DM) → klik "Authorize app" |
| Tugas Dex (klik-klik-back) | Follow ✓, Like ✓, Repost ✓, Reply ✓, Quote ✓ — intent X dibuka lalu ditutup TANPA aksi di X |
| Submit EVM | Alamat diketik per karakter, nilai dibaca balik (42 karakter cocok), lalu Send |
| Hasil | "WELCOME TO THE FLOOR / Desk <nomor-desk> is reserved for you." — verifikasi fresh load: "DESK LOCKED / Desk <nomor-desk> is yours." Trader digambar dari PFP |
| Bersih-bersih | Token sesi dihapus dari README repo private (read-back terverifikasi; token tetap ada di git history) |

**Pelajaran dari tugas ini (masuk ke stack):**

- Profil Chrome baru lambat di first-run — panaskan manual sebelum Nodriver (lihat §4.3).
- Cookie yang dipasang via CDP belum tentu langsung tersimpan ke profil — satu sesi browser yang sama harus dipakai untuk login + OAuth (atau pasang ulang tiap run).
- Jangan pipe output script Nodriver ke `head` — pipe tertutup bisa membunuh script sebelum selesai (SIGPIPE). Tulis ke file log, lalu baca log-nya.
- Link tugas yang sudah selesai ditandai ✓ — driver harus memilih link yang BELUM ✓.

### 4.12 Contoh end-to-end: WL Cathereum (Registration ID didapat)

Tugas kedua yang selesai penuh lewat stack ini (8 Nov 2026, 11:32–12:37 WIB).

**Tugas:** dari channel Airdrop Finder (post 7 Nov 15:24 UTC; Tomket membagikan link yang sama hari itu): "Cathereum 8888 DN404 Cats Whitelist — Complete All Task: https://cathereum.tech/". WL = 6 langkah: 01 Connect Wallet (Elysium Testnet), 02 Connect X, 03 Follow @CathereumTech, 04 Like + Comment post resmi, 05 Quote post resmi, 06 Submit Registration. Setiap registrasi melewati Sybil check. Mint 4.444, harga TBA, tanggal TBA. Fase: GTD maks 2/wallet, FCFS maks 1/wallet, Public maks 2/wallet, masing-masing 30 menit.

**Yang dicek dulu:** `urlscan` search `cathereum.tech` → 0 laporan publik. `dnstwist --registered` → 5.631 permutasi, 0 terdaftar. Situs dibaca langsung di browser farming (IP <IP-rumah>) — isinya cocok dengan post channel. Pencarian web tidak menemukan apa pun yang relevan. Tidak ada alamat kontrak di situs — ini registrasi WL saja, tanpa mint/pembayaran.

**Preview (disetujui user, urutan: "Aksi 1 dulu aja. Kalo udah berhasil aksi 1 baru lanjut aksi 2"):**

1. **Aksi 1 — uji gerbang Bridge** (tidak ada aksi di situs): ambil ulang kode Bridge dari repo, policy allowlist = testdapp + chainlist.org + cathereum.tech, arm = `wallet_switchEthereumChain` + `wallet_addEthereumChain` saja, cap 0, tanpa `personal_sign`/`send`. Gate test 11/11 lolos (substansi).
2. **Aksi 2 — eksekusi WL:** wallet Hunt auto-connect → Connect X dalam sesi (OAuth read-only) → Follow, Like + reply, Quote dikerjakan beneran di X → klik ulang tombol step agar situs memverifikasi → centang Terms → SUBMIT → CHECK ALLOCATION.

**Yang terjadi (sesuai workflow):**

| Tahap | Hasil |
|---|---|
| Cek link | urlscan 0 laporan; dnstwist 0 dari 5.631 permutasi; situs cocok dengan post channel |
| Aksi 1 — gate test Bridge | 11/11 lolos: connect sebagai Hunt (Sepolia 0xaa36a7); sign/typed/send DIBLOKIR tanpa arm; dengan arm, sign + typed-data OK dan recovery cocok ke alamat Hunt; disarm memblokir lagi; origin asing diblokir; addChain terbukti via HTTP langsung |
| Wallet + X | Wallet auto-connect saat load (provider terdeteksi). X OAuth dalam sesi (scope=users.read+tweet.read, redirect cathereum.tech, PKCE S256). Link X TIDAK bertahan antar launch browser — OAuth harus diulang tiap sesi, di sesi yang sama dengan submit |
| Follow / Like + reply / Quote | Dikerjakan beneran di X via `@akun-X-farming` — Follow = Following ✓, reply live (tautan reply (dihapus di versi publik)), quote live (tautan quote (dihapus di versi publik)) |
| Verifikasi step di situs (run final2) | Klik ulang tombol FOLLOW/LIKE/QUOTE satu-satu → semua step tampil VERIFIED. Submit masih gagal — penyebab ketemu: checkbox "I agree to the Terms of Service and Privacy Policy, and I am 18 or older" belum dicentang |
| Submit — run 10 | Layanan restart di tengah run (mati setelah wallet + X lolos). Run ulang gagal "Failed to connect to browser" — lock profil basi (`SingletonLock`/`SingletonSocket`/`SingletonCookie`) → hapus lock + panaskan profil manual → browser jalan lagi |
| Submit — run 11 | Klik re-verifikasi step gagal total: FOLLOW tidak ketemu (selector button-only), LIKE/QUOTE diklik tapi step tidak berubah VERIFIED (verifikasi situs flaky) |
| Submit — run 12 | Klik step via JS (teks persis, elemen apa pun) + retry sampai VERIFIED → semua VERIFIED di percobaan pertama. Checkbox Terms dicentang atas konfirmasi user (12:27 WIB). SUBMIT diklik → halaman: "REGISTRATION CONFIRMED / YOU'RE ON THE LIST. / REGISTRATION ID <registration-id>" — tapi driver salah lapor gagal karena cek keyword case-sensitive (`confirmed` vs `CONFIRMED`) |
| Verifikasi akhir | Load baru: REGISTRATION CONFIRMED + ID <registration-id> bertahan ✓. CHECK ALLOCATION masih kosong ("YOUR MINT TICKET APPEARS HERE") — tiket keluar setelah Sybil check proyek (batch 1 sedang diproses) |
| Biaya | $0 — tanpa transaksi, tanpa signature |

**Pelajaran dari tugas ini (masuk ke stack):**

- Layanan/VM bisa restart di tengah run — sebelum menjalankan ulang, BACA dulu `run-log-*.json` + log driver untuk melihat tahap apa yang sudah selesai. Jangan run ulang buta.
- Setelah restart, Chrome bisa gagal start karena lock profil basi — hapus `SingletonLock`/`SingletonSocket`/`SingletonCookie` di profil, lalu panaskan profil manual (`chrome --no-sandbox --disable-gpu --user-data-dir=<profil> --remote-debugging-port=9222 about:blank`, cek `/json/version` merespons, lalu matikan).
- Jangan `pkill -f <pola>` kalau pola itu muncul di baris perintahmu sendiri — itu membunuh shell-mu sendiri. Pakai trik bracket (`pkill -f "nama[.]py"`) atau bunuh berdasarkan PID.
- Tombol step di situs belum tentu `<button>`/`<a>` — klik via JS berdasarkan teks persis (elemen apa pun). Dan verifikasi situs bisa flaky: retry klik sampai step tampil VERIFIED (maks 3x, jeda 15–20 detik), jangan klik sekali lalu lanjut.
- Cek konfirmasi submit harus case-insensitive — situs menulis `REGISTRATION CONFIRMED` (kapital); keyword huruf kecil melewatkannya dan driver salah menyimpulkan gagal. Selalu baca teks halaman setelah submit sebelum menyimpulkan.
- Checkbox Terms + 18+ adalah atestasi umur tentang user — TANYA user dulu sebelum mencentang; jangan jadikan bagian rutin.
- Step situs tidak tampil selesai sampai tombolnya diklik — verifikasi dipicu oleh klik, bukan otomatis. Setelah aksi nyata selesai, klik ulang tiap tombol step.
- Cek status like di halaman post X bisa mengembalikan `no-btn` (tombol belum ter-render) — verifikasi situs adalah sumber kebenaran, bukan DOM halaman X.
- Registrasi ≠ tiket: CHECK ALLOCATION bisa tetap kosong setelah registrasi terkonfirmasi — tiket keluar setelah Sybil check proyek (batch).

### 4.13 Operasional harian & troubleshooting

**Checklist tiap sesi farming:**

```bash
# 1. Relay + IP
~/workspace/telegram-mcp/scripts/tg_watchdog.sh
curl --socks5-hostname 127.0.0.1:1180 https://api.ipify.org   # catat IP-nya

# 2. Lane Telegram
TL_PROXY=socks5://127.0.0.1:1180 ~/workspace/telegram-mcp/.venv/bin/python ~/workspace/telegram-lane/tl.py whoami

# 3. Browser (probe IP di dalam browser sebelum tugas)
#    lihat ~/workspace/memesfund/probe1.py sebagai contoh
```

**Troubleshooting:**

| Gejala | Penyebab umum | Perbaiki |
|---|---|---|
| Nodriver "Failed to connect to browser" | Profil baru belum dipanaskan; atau GPU hang | Panaskan profil manual; pastikan flag `--disable-gpu --no-first-run --no-default-browser-check` |
| Headed Chrome hang, tanpa DevTools listener | Display Xvfb basi | Display Xvfb baru yang segar |
| X 403 | Mode headless | Wajib headed via Xvfb |
| Telethon `AuthKeyUnregisteredError` / `is_user_authorized() = False` | Sesi file diisi SETELAH client dibuat | Isi `SQLiteSession` dulu, baru buat `TelegramClient` (lihat §4.5) |
| Forwarder mati diam-diam | VM diganti | `tg_watchdog.sh` |
| SOCKS error 4 | Microsocks HP jalan, HP tidak ada internet | Cek internet/WiFi HP |
| Timeout setelah Termux restart | microsocks belum dijalankan ulang | Jalankan `microsocks` di HP |
| OAuth X minta login padahal sudah login | Cookie belum tersimpan ke profil | Pasang cookie + login dalam SATU sesi browser yang sama |
| Dex/input tidak bisa diketik | `tab.select()` mengambil input tersembunyi | Cari input lewat placeholder (`input[placeholder*="0x"]`) + `focus()` |
| Nodriver "Failed to connect to browser" SETELAH restart layanan/VM | Lock profil basi (`SingletonLock`/`SingletonSocket`/`SingletonCookie`) — Chrome mati di tengah run | Hapus file lock di profil, lalu panaskan profil manual (lihat §4.12) |
| Driver mati di tengah run (restart layanan/VM) | Proses ikut mati; browser + bridge ikut hilang | BACA dulu `run-log-*.json` + log driver — lihat tahap apa yang sudah selesai — baru jalankan ulang; jangan run ulang buta |
| Tombol step situs tidak ketemu / klik tidak mengubah status | Tombol bukan `<button>`/`<a>`; atau verifikasi situs flaky | Klik via JS berdasarkan teks persis (elemen apa pun) + retry sampai step tampil VERIFIED (maks 3x, jeda 15–20 dtk) |
| Submit diklik tapi driver bilang gagal | Cek konfirmasi case-sensitive; situs menulis `REGISTRATION CONFIRMED` (kapital) | Cek konfirmasi case-insensitive; selalu baca teks halaman setelah submit sebelum menyimpulkan |
| Submit tidak bisa dipencet / minta centang Terms | Checkbox "I agree to the Terms … and I am 18 or older" belum dicentang | Itu atestasi umur tentang user — TANYA user dulu sebelum mencentang; jangan centang sebagai bagian rutin |
| Step situs tidak pernah tampil selesai | Verifikasi baru jalan saat tombol step diklik (bukan otomatis) | Setelah aksi nyata selesai, klik ulang tiap tombol step agar situs memverifikasi ulang |
| Cek like di halaman post X: `no-btn` | Tombol like belum ter-render saat dicek | Verifikasi situs adalah sumber kebenaran, bukan DOM halaman X; beri jeda lebih lama |

**Setelah VM diganti (recovery):** `~/workspace/` selamat; yang harus dihidupkan ulang: Xvfb (`run.sh`), forwarder (`tg_watchdog.sh`), Tailscale (cek status — jangan `down`), lalu verifikasi IP + `tl.py whoami` + probe browser. Direktori yang hilang (pernah terjadi: `privy-tools/`, `okx-relay/`, `farming-tools/`, `telegram-lane/`) diambil ulang dari repo.

---

## 5. Sumber tools

Setiap baris: dari mana tool diambil, versi pin yang dipakai, dan untuk apa. Kalau sebuah tool belum terpasang, pasang dari sumber yang tercantum — jangan dari sumber lain yang tidak dikenal.

| Tool / komponen | Versi pin | Sumber (ambil dari sini) | Dipakai untuk |
|---|---|---|---|
| Chrome (Linux) | 157.0.8103.38 | `https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb` — salinan tersimpan: `~/workspace/chip404/chrome/chrome.deb` | Browser farming (headed) |
| Nodriver | 0.50.8 | PyPI: `pip install nodriver==0.50.8` — https://pypi.org/project/nodriver/ | Otomasi Chrome farming |
| Xvfb | sistem | `apt install xvfb` (sudah ada di `/usr/bin/Xvfb`) | Display virtual untuk headed Chrome |
| Telethon | 1.45.0 | PyPI: `pip install telethon==1.45.0` — https://pypi.org/project/Telethon/ | Lane Telegram |
| python-socks | 3.2.1 | PyPI: `pip install python-socks==3.2.1` | Proxy SOCKS Telethon |
| qrcode | 8.2 | PyPI: `pip install qrcode==8.2` | Login QR Telegram (cadangan) |
| telegram-mcp | v1.30.0 | GitHub `chigwell/telegram-mcp` (juga sumber `.env` + session string awal) | MCP Telegram (pencarian alpha; lane utama tetap `tl.py`) |
| Privy Node SDK | 0.38.0 | npm: `@privy-io/node` — https://www.npmjs.com/package/@privy-io/node ; docs https://docs.privy.io/ | Buat + operasikan dompet Hunt |
| viem | 2.57.16 | npm: `viem` | Utilitas EVM di privy-tools |
| @solana/web3.js | 1.98.4 (pin 1.x) | npm: `@solana/web3.js` | Utilitas Solana di privy-tools |
| Privy Agent CLI | — (verifikasi dulu) | npm: `@privy-io/agent-wallet-cli` ; sandbox `agents.privy.io` | Jalur signing tanpa App Secret (direkomendasikan, belum dipakai) |
| Node.js | v24.22.1 | https://nodejs.org/ (sudah terpasang di VM) | Runtime privy-tools |
| Tailscale | client bawaan | https://tailscale.com/download (aplikasi HP + client VM di `/opt/hatch/bin/tailscale`) | Jaringan privat VM ↔ HP |
| microsocks | Termux | Di HP: `pkg install microsocks` (Termux) ; upstream https://github.com/rofl0r/microsocks | SOCKS5 di HP |
| Foundry (forge/cast/anvil/chisel) | — | https://getfoundry.sh/ (`curl -L https://foundry.paradigm.xyz \| bash`) | Build + `cast` (selector, call on-chain) |
| Aderyn | — | Cyfrin Aderyn — rilis resmi Cyfrin (binary sudah ada di `~/workspace/cyber-tools/bin/`) | Audit statis Solidity |
| Slither | — | PyPI: `pip install slither-analyzer` (verifikasi keberadaan sebelum dipakai) | Audit statis Solidity |
| Mythril | — | PyPI: `pip install mythril` ; jalan via `python3 -m mythril` (verifikasi keberadaan sebelum dipakai) | Analisis simbolik |
| Heimdall | 0.9.2 | Heimdall-rs — rilis resmi (binary sudah ada di `~/workspace/cyber-tools/bin/`) | Decompile bytecode |
| Grype / Syft | — | Anchore — rilis resmi (binary sudah ada di `~/workspace/cyber-tools/bin/`) | Scan kerentanan + SBOM |
| dnstwist | 20250130 | `~/workspace/cyber-tools/url-tools-venv/bin/dnstwist` (PyPI, terpasang 8 Nov 2026; install ulang via `scripts/cyber-tools-install.sh` di repo Agentic_wallet) | Cek typosquatting domain |
| urlscan | — | Wrapper `~/workspace/cyber-tools/bin/urlscan-check` (sumber: `scripts/cyber-tools/urlscan-check` di repo Agentic_wallet); search/result publik tanpa key; submit butuh `URLSCAN_API_KEY` (daftar gratis di urlscan.io) | Cek URL sebelum dipakai |
| Bridge PoC | — | Repo `Rganthong/Agentic_wallet`, `scripts/bridge-poc/` | Provider EIP-1193/6963 + gerbang per-tugas |
| Telegram lane | — | Repo `Rganthong/Agentic_wallet`, `scripts/telegram-lane/` | `tl.py`, RUNBOOK, template allowlist |
| Farming Chrome stack | — | Repo `Rganthong/Agentic_wallet`, `scripts/farming-tools/` (+ runbook di `kendala-saat-ini.md` Bagian B) | Venv Nodriver, smoke test, Chrome .deb |
| Dokumen perjalanan | — | Repo `Rganthong/Agentic_wallet`: `perjalanan-lengkap.md`, `privy-tutorial.md`, `kendala-saat-ini.md`, `tools-inventory.md` | Riwayat keputusan + runbook recovery |
| Repo ini | — | Repo `Rganthong/GUIDE_AIRDROP_AGENT` (README ini) | Panduan |

---

## 6. Riwayat farming

| Proyek | Tanggal | Hasil | Catatan |
|---|---|---|---|
| HiMan | 7 Nov 2026 | ✅ Selesai | 5 aksi X terverifikasi via `@akun-X-farming` |
| Tzero | 7 Nov 2026 | ⏸ Parkir | Situs mati (Vercel 402 `DEPLOYMENT_DISABLED`); persiapan tersimpan |
| Octops | 7 Nov 2026 | ❌ Dibatalkan user | Pola dev-farming (link X mengarah ke proyek lain + template sama + hitung mundur + kumpul alamat massal) |
| Elyon | 7 Nov 2026 | ⏸ Parkir | CAPTCHA "SECURITY CHECK" saat connect wallet — tidak ditebak; jalan keluar: user menyelesaikan sendiri di perangkatnya |
| **MemesFund WL** | **8 Nov 2026** | **✅ Selesai — desk didapat** | **Tugas pertama penuh via stack ini (lihat §4.11)** |
| **Cathereum WL** | **8 Nov 2026** | **✅ Selesai — Registration ID didapat** | **Tugas kedua penuh via stack ini (lihat §4.12); tiket mint menunggu Sybil check** |

---

## 7. Keamanan & rahasia

- **Rule #1 (repo):** tidak ada App Secret, private key, seed phrase, password, file sesi, atau kredensial apa pun di repo — termasuk di git history. Sekali masuk history, menghapusnya butuh rewrite history; lebih aman tidak pernah memasukkan.
- **File sesi di VM:** mode `0600` (file) / `0700` (direktori). Jangan pernah kirim nilainya di chat, jangan catat di memori/file biasa.
- **Pola token X:** user menaruh `auth_token` + `ct0` di README repo private → agent membaca → memakai → **menghapus dari README** setelah selesai. Nilai token tidak pernah dicetak/di-catat di mana pun.
- **App Secret Privy:** hanya transient per sesi (env var proses), tidak pernah disimpan. Jalur vault konektor ditolak untuk kasus ini (satu konektor = satu secret, Privy butuh appId+appSecret per request).
- **Setelah tugas:** revoke approval yang dibuat tugas; tanyakan dulu sebelum menghapus file tugas; jangan sign out BAW untuk user.
- **Kalau ada yang janggal:** berhenti dan lapor dulu — jangan lanjut sampai user konfirmasi. Ini berlaku untuk kontrak aneh, situs aneh, izin aneh, dan angka yang tidak cocok.
- **Repo lama:** ada satu repo supply lama yang masih menyimpan App Secret di git history-nya (README sudah dihapus dari working tree, tapi history tidak ikut bersih). Menghapus repo itu adalah keputusan user yang masih terbuka.

---

*Versi publik dari panduan privat (8 Nov 2026). Nama akun, alamat dompet, IP, ID registrasi, dan nama repo privat sudah diganti placeholder — isinya hanya panduan global.*

*Dokumen ini ditulis 8 Nov 2026, dari pekerjaan nyata membangun stack ini (bukan dari teori). Kalau ada langkah yang sudah berubah, perbaiki dokumen ini dulu sebelum mengikuti versi lamanya — dokumen yang kedaluwarsa lebih berbahaya daripada tidak ada dokumen.*
