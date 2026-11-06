# Blueprint Integrasi Pembayaran Pajak Kendaraan Bermotor (PKB)
**Pemerintah Provinsi DIY: Kominfo – Samsat – BPD DIY**
*Skema Optimal: Kominfo-as-Portal / Samsat-Orchestrated Chain (Versi Hardened)*

> **Status Dokumen**: Blueprint Teknis v1.0 — Rekomendasi Final
> **Tanggal**: November 2026
> **Berdasarkan**: Audit & Review Blueprint A (Chain Notification) + Blueprint B (Kominfo Direct)

---

## 1. Filosofi & Prinsip Desain

Skema optimal ini dibangun di atas **5 prinsip utama**:

| Prinsip | Deskripsi |
|---|---|
| **Kominfo = Portal, Bukan Orchestrator** | Kominfo hanya sebagai kanal UI/UX dan identity provider. Tidak menyentuh logika keuangan atau pembayaran secara langsung. |
| **Samsat = Single Source of Truth Pajak** | Semua status pajak (lunas/belum) **hanya sah** jika sudah tercatat di database Samsat. |
| **BPD = Pemilik Kebenaran Finansial** | Status finansial (uang sudah masuk/belum) hanya dipegang BPD. Tidak ada pihak lain yang men-declare "sudah bayar" tanpa konfirmasi BPD. |
| **Defense-in-Depth Security** | Setiap titik komunikasi antar-sistem diamankan berlapis: mTLS + HMAC Signature + IP Whitelist + Timestamp Anti-Replay. |
| **Self-Healing by Design** | Sistem harus mampu pulih otomatis dari kondisi partial failure tanpa intervensi manual untuk transaksi normal. |

---

## 2. Arsitektur Komponen & Aliran Data

```mermaid
flowchart LR
    WP["Wajib Pajak (User)"]

    subgraph Portal["🖥️ Front-Facing & Identity (Kominfo)"]
        JSS["JSS / Jogja Smart Service<br/>- SSO & Akun Warga<br/>- NIK Terverifikasi (Encrypted)<br/>- Portal UI / Mobile Apps<br/>- Webhook Consumer (read-only)"]
    end

    subgraph Biller["🏛️ Core Tax Authority (Samsat DIY)"]
        SAM["Samsat DIY<br/>- Core Tax Engine & Database<br/>- Billing & Lock Manager<br/>- QRIS Requester ke BPD<br/>- Penerbit e-SKKP / NTPN<br/>- Webhook Orchestrator"]
    end

    subgraph Bank["🏦 Payment Gateway & Acquirer (BPD DIY)"]
        BPD["BPD DIY<br/>- QRIS Dynamic Engine<br/>- Settlement & Core Banking<br/>- Webhook Publisher (ke Samsat)"]
    end

    subgraph Infra["⚙️ Shared Infrastructure"]
        MQ["Async Job Queue<br/>(Internal Samsat & Kominfo)<br/>- Retry Manager<br/>- DLQ Handler"]
        RECON["EOD Reconciliation Job<br/>(Nightly Batch)"]
    end

    WP <-->|"Login JSS / Scan QRIS"| JSS
    JSS <-->|"REST API Inquiry (mTLS)<br/>+ Webhook Consumer"| SAM
    SAM <-->|"REST API QRIS + Webhook<br/>(mTLS + HMAC)"| BPD
    SAM --> MQ
    JSS --> MQ
    BPD -.->|"EOD Report (SFTP/API)"| RECON
    RECON -.->|"Reconcile & Auto-Settle"| SAM
```

---

## 3. Alur Proses End-to-End (High-Level 6 Langkah)

```mermaid
flowchart TD
    subgraph Step1["1. 🔐 Otentikasi & Inquiry Kendaraan (Kominfo ➔ Samsat)"]
        S0["Login SSO JSS<br/>(NIK 347101******0002 diekstrak server-side)"] --> S1["mTLS Handshake & Validasi NIK di Samsat"]
        S1 --> S2["Samsat Return Daftar Kendaraan Terdaftar<br/>(Live Data: Status Tunggakan & Estimasi Tagihan)"]
    end

    subgraph Step2["2. 📋 Cek Tunggakan & Aturan FIFO (Pemisahan Belum vs Lunas)"]
        S2 --> S3a["Query Daftar Tahun Pajak per Nopol<br/>(GET /api/v2/pajak/tahun?nopol=...)"]
        S3a --> S3b["Penegakan Aturan FIFO Biller Samsat:<br/>- Atas: Card 1 (Tertunggak) + Card 2 (Berjalan)<br/>- Bawah: Riwayat Tahun Lunas (Arsip e-SKKP)"]
    end

    subgraph Step3["3. 🔒 Rincian Billing Resmi & Lock Database Samsat"]
        S3b --> S4["Inquiry Rincian Akumulasi Tagihan Resmi<br/>(PKB, SWDKLLJ Jasa Raharja, PNBP, Denda)"]
        S4 --> S5a["Kominfo POST /api/v2/billing/create"]
        S5a --> S5b["Samsat Lock Tagihan (BILLING_LOCKED)<br/>Mencegah Double-Payment dari Loket/ATM Lain"]
    end

    subgraph Step4["4. 📱 Penerbitan Dynamic QRIS ASPI (Samsat ⇄ BPD ➔ Kominfo)"]
        S5b --> S5c["Samsat Request QRIS ke BPD DIY<br/>(Samsat = Merchant Resmi Pemda DIY)"]
        S5c --> S5d["BPD DIY Terbitkan Dynamic QRIS ASPI (TTL: 15 Menit)"]
        S5d --> S5e["Samsat Return HTTP 200 OK ke Kominfo<br/>(Membawa billing_id, qr_string, expired_at)"]
        S5e --> S5f["Kominfo Forward ke Mobile App JSS<br/>Render Dynamic QR & Timer Hitung Mundur 15m"]
    end

    subgraph Step5["5. 💳 Pembayaran & Chain Webhook (User ➔ BPD ➔ Samsat ➔ Kominfo)"]
        S5f --> S6["User Scan & Bayar QRIS via M-Banking / e-Wallet"]
        S6 --> S7a["BPD Settlement Dana ➔ Webhook HMAC ke Samsat"]
        S7a --> S7b["Samsat Validasi Signature, Bukukan Lunas Akumulasi,<br/>Terbitkan NTPN Resmi & Terbitkan e-SKKP Sah"]
        S7b --> S7c["Samsat Kirim Webhook Berantai ke Kominfo (JSS)<br/>(Status SETTLED + NTPN + Dokumen e-SKKP URL)"]
        S7c --> S7d["Kominfo Push Notifikasi Sukses ke Smartphone Warga"]
    end

    subgraph Step6["6. 📜 Bukti Pelunasan & Penerbitan Dokumen Sah e-SKKP Digital"]
        S7d --> S8["Warga Buka Dokumen Sah e-SKKP di JSS<br/>(Kop Resmi Pemda DIY, Barcode Validasi Sah & NTPN Bank)<br/>Tanpa Perlu Datang Stempel Fisik ke Kantor Samsat"]
    end

    subgraph Fallback["🔄 Fallback & Self-Healing (Jaring Pengaman)"]
        S5f -.->|"Jika User Cek Status"| FB1["Status Check / Fallback Polling"]
        S5f -.->|"Jika Timeout 15 Menit"| FB2["Samsat Cron Job: Auto-Release Lock Tagihan"]
        S6 -.->|"Pukul 23:00 WIB"| FB3["EOD Reconciliation Batch (BPD ⇄ Samsat)"]
    end

    style Step1 fill:#002b4d,stroke:#0284c7,stroke-width:2px,color:#ffffff
    style Step2 fill:#003366,stroke:#38bdf8,stroke-width:2px,color:#ffffff
    style Step3 fill:#4a2800,stroke:#f59e0b,stroke-width:2px,color:#ffffff
    style Step4 fill:#451a03,stroke:#d97706,stroke-width:2px,color:#ffffff
    style Step5 fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#ffffff
    style Step6 fill:#065f46,stroke:#34d399,stroke-width:2px,color:#ffffff
    style Fallback fill:#312e81,stroke:#818cf8,stroke-width:2px,color:#ffffff
```

---

## 4. State Machine Resmi Status Transaksi

> [!IMPORTANT] Semua sistem (Kominfo, Samsat, BPD) WAJIB menggunakan state machine ini sebagai referensi tunggal. Transisi di luar diagram ini dianggap **invalid** dan harus ditolak.

```mermaid
stateDiagram-v2
    [*] --> INITIATED : User klik "Bayar QRIS"
    INITIATED --> BILLING_LOCKED : Kominfo request billing ke Samsat\n(Samsat lock tagihan)
    BILLING_LOCKED --> QRIS_GENERATED : Samsat request QRIS ke BPD\n(QR string diterima)
    QRIS_GENERATED --> PAID : BPD webhook sukses diterima Samsat\n(dana ter-settlement)
    PAID --> SETTLED : Samsat bukukan lunas\n& terbitkan NTPN + e-SKKP
    SETTLED --> [*] : Kominfo notifikasi user, tampil bukti bayar

    BILLING_LOCKED --> EXPIRED : TTL tagihan habis (otomatis)\n(Samsat release lock)
    QRIS_GENERATED --> EXPIRED : TTL QRIS habis (15 menit)\n(Samsat release lock)
    EXPIRED --> [*] : User bisa generate QRIS baru

    BILLING_LOCKED --> CANCELLED : User cancel atau\nsistem force-cancel
    QRIS_GENERATED --> CANCELLED : User cancel sebelum bayar
    CANCELLED --> [*] : Lock dilepas, tagihan bebas

    PAID --> PAID_PENDING_SETTLE : Samsat timeout/error\nsaat proses settlement
    PAID_PENDING_SETTLE --> SETTLED : Retry sukses / EOD Reconciliation
    PAID_PENDING_SETTLE --> MANUAL_REVIEW : Max retry habis (DLQ)\n→ Ops team notified
```

**Tabel Transisi & Actor:**

| Dari Status | Ke Status | Trigger Event | Actor Berwenang |
|---|---|---|---|
| `INITIATED` | `BILLING_LOCKED` | User klik bayar → Kominfo request → Samsat lock berhasil | Samsat |
| `BILLING_LOCKED` | `QRIS_GENERATED` | Samsat request ke BPD, QR string diterima | Samsat |
| `QRIS_GENERATED` | `PAID` | BPD webhook payment success diterima Samsat | BPD (via webhook) |
| `PAID` | `SETTLED` | Samsat sukses bukukan lunas & terbitkan e-SKKP | Samsat |
| `SETTLED` | *(akhir)* | Kominfo notifikasi user | Kominfo |
| `BILLING_LOCKED` | `EXPIRED` | Scheduler job Samsat (otomatis per TTL) | Samsat (cron) |
| `QRIS_GENERATED` | `EXPIRED` | QRIS TTL 15 menit habis (otomatis) | Samsat (cron) |
| `PAID` | `PAID_PENDING_SETTLE` | Samsat error saat proses settlement ke DB | Samsat |
| `PAID_PENDING_SETTLE` | `SETTLED` | Retry sukses atau EOD Reconciliation | Samsat (retry/batch) |
| `PAID_PENDING_SETTLE` | `MANUAL_REVIEW` | Max retry habis (masuk DLQ) | System → Ops |

---

## 5. Rincian Alur & Spesifikasi Interaksi

### 5.1. Tahap 0: Autentikasi & Identity — Proteksi Data NIK

> [!IMPORTANT] Prinsip UU PDP No. 27/2022
> NIK adalah **data pribadi sensitif**. NIK **tidak boleh** ada di JWT token yang diterima client/app. NIK hanya boleh ada di server-side session Kominfo dan hanya dikirim ke Samsat via backend-to-backend channel yang terenkripsi.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak (User)
    participant App as JSS App (Client)
    participant KOM as Kominfo Backend (JSS Server)
    participant DB as Session Store (Server-Side)

    WP->>App: Login ke JSS (username + password / SSO)
    App->>KOM: POST /auth/login (credentials)
    KOM->>KOM: Autentikasi & load profil user
    KOM->>DB: Simpan { session_id → { user_id, NIK_encrypted, nama } }
    note over DB: NIK disimpan encrypted (AES-256)<br/>di server-side session, BUKAN di JWT

    KOM-->>App: Set-Cookie: session_id=xxx (HttpOnly, Secure, SameSite=Strict)
    note over App: App HANYA memegang session cookie<br/>NIK tidak pernah terekspos ke client

    App-->>WP: Login berhasil, tampil profil (Nama saja, tanpa NIK)

    note over WP, KOM: Semua request selanjutnya menggunakan session cookie<br/>Kominfo Backend ekstrak NIK dari session server-side
```

---

### 5.2. Tahap 1–4: Inquiry Data Pajak & Penegakan Aturan FIFO (Kominfo → Samsat)

> [!IMPORTANT] Penegakan Aturan FIFO Biller Samsat & Akumulasi Multi-Tahun
> Berdasarkan regulasi perpajakan daerah DIY, wajib pajak **tidak diperkenankan melompati tahun pajak tertunggak**. Pembayaran wajib dilakukan secara berurutan (*First In, First Out / FIFO*). Jika terdapat tunggakan 2 tahun (misal: 2025 tertunggak + 2026 berjalan), tagihan disatukan dalam **satu kode billing akumulasi** dan dilunasi sekaligus via satu QRIS BPD DIY.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant App as JSS App (Client)
    participant KOM as Kominfo Backend
    participant SAM as Samsat DIY

    note over KOM, SAM: Semua komunikasi menggunakan mTLS (Port 8443)<br/>+ Header: X-API-Key + X-Request-ID (UUID)

    WP->>App: Buka menu "Pajak Kendaraan (PKB)"
    App->>KOM: GET /pajak/kendaraan (session cookie)
    KOM->>KOM: Ekstrak NIK dari server-side session (decrypt AES-256)

    %% Step 1: Inquiry Kendaraan by NIK
    KOM->>SAM: GET /api/v2/kendaraan?nik={NIK_MASKED}<br/>Header: X-API-Key, X-Request-ID, X-Timestamp
    SAM->>SAM: Validasi API Key & Timestamp (anti-replay < 5 menit)
    SAM-->>KOM: 200 OK: { total: 2, data_kendaraan: [ { nopol: "AB 2481 XZ", model: "Honda Vario 160 ABS", status_tagihan: "MENUNGGAK_2_TAHUN", total_tagihan: 750000 }, { nopol: "AB 1892 YQ", model: "Toyota Avanza 1.5 G MT", status_tagihan: "PERIODE_BERJALAN", total_tagihan: 2450000 } ] }
    KOM-->>App: Return daftar kendaraan (Live Data Samsat terenkripsi)
    App-->>WP: Screen 1: Render daftar kendaraan terdaftar di DIY

    %% Step 2: Inquiry Tahun Pajak & Aturan FIFO
    WP->>App: Klik kendaraan "AB 2481 XZ"
    App->>KOM: GET /pajak/nopol/AB2481XZ/tahun
    KOM->>SAM: GET /api/v2/pajak/tahun?nopol=AB2481XZ
    SAM-->>KOM: 200 OK: { unpaid_years: [ { card: 1, tahun: 2025, status: "TERTUNGGAK", pokok: 350000, denda: 50000, subtotal: 400000 }, { card: 2, tahun: 2026, status: "BERJALAN", pokok: 350000, denda: 0, subtotal: 350000 } ], total_akumulasi: 750000, paid_history: [ { tahun: 2024, ntpn: "NTPN-20241120-BPD6190", nominal: 335000, status: "LUNAS" }, { tahun: 2023, ntpn: "NTPN-20231118-BPD5102", nominal: 330000, status: "LUNAS" } ] }
    KOM-->>App: Return riwayat tagihan terstruktur
    App-->>WP: Screen 2: Render Card 1 (2025 Tertunggak) + Card 2 (2026 Berjalan)<br/>& List Bawah: Riwayat Lunas (Akses Arsip e-SKKP)

    %% Step 3: Rincian Akumulasi Tagihan Resmi
    WP->>App: Klik "Bayar Akumulasi 2 Tahun (Rp 750.000)"
    App->>KOM: GET /pajak/nopol/AB2481XZ/detail?mode=accumulation
    KOM->>SAM: GET /api/v2/pajak/detail?nopol=AB2481XZ&mode=accumulation
    SAM-->>KOM: 200 OK: { pkb_pokok: 630000, swdkllj_pokok: 70000, pnbp: 0, denda_akumulasi: 50000, total_akumulasi: 750000, masa_pajak_baru: "28 November 2027" }
    KOM-->>App: Return rincian resmi Samsat
    App-->>WP: Screen 3: Render Breakdown Resmi Biaya & Tombol "Bayar dengan QRIS BPD DIY"
```

---

### 5.3. Tahap 5: Generate Billing & Dynamic QRIS ASPI (Kominfo → Samsat → BPD)

> [!NOTE] Mengapa Samsat yang request ke BPD (bukan Kominfo)?
> Samsat adalah **Merchant resmi** BPD DIY berdasarkan perjanjian kerjasama perbankan rekening RKUD Pemda DIY. Kominfo bukan PJP (Penyelenggara Jasa Pembayaran) dan tidak memiliki izin BI untuk menerbitkan instrumen pembayaran. Skema ini patuh 100% pada regulasi Bank Indonesia.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant App as JSS App (Client)
    participant KOM as Kominfo Backend
    participant SAM as Samsat DIY
    participant BPD as BPD DIY

    WP->>App: Klik "Bayar dengan QRIS BPD DIY (Rp 750.000)"
    App->>KOM: POST /pajak/billing/create<br/>Body: { nopol: "AB 2481 XZ", periode: "2025 - 2026" }

    KOM->>KOM: 1. Generate trx_kominfo_id (UUID v4)<br/>2. Set status internal: INITIATED

    note over KOM, SAM: Timeout: 15 detik | Retry: 0 (non-idempotent create)
    KOM->>SAM: POST /api/v2/billing/create<br/>Body: { trx_kominfo_id, nopol: "AB 2481 XZ", periode: "2025 - 2026", total_nominal: 750000 }<br/>Header: X-API-Key, X-Request-ID: trx_kominfo_id, X-Timestamp

    SAM->>SAM: 1. Validasi API Key & Timestamp (Anti-Replay < 5 menit)<br/>2. Cek tidak ada billing aktif untuk nopol ini<br/>3. Lock tagihan: BILLING_LOCKED (Mencegah Double-Payment Loket/ATM)<br/>4. Generate billing_samsat_id: "BILL-DIY-XXXX"<br/>5. Simpan pemetaan: billing_samsat_id ↔ trx_kominfo_id

    note over SAM, BPD: Timeout: 10 detik | Retry: 3x (exp backoff)
    SAM->>BPD: POST /api/v1/qris/generate [Merchant: SAMSAT PROVINSI DIY]<br/>Body: { merchant_id: "BPD-DIY-SAMSAT-01", billing_samsat_id: "BILL-DIY-XXXX", amount: 750000, expired_minutes: 15 }<br/>Header: X-API-Key (Samsat), X-Idempotency-Key: billing_samsat_id

    BPD->>BPD: 1. Validasi Merchant ID Samsat<br/>2. Generate Dynamic QRIS string & Payload ASPI/BI standard
    BPD-->>SAM: 200 OK: { bpd_trx_id: "BPD-QR-XXXX", qr_string: "00020101021226600016ID.GO.DIY.BPD01...", expired_at: "2026-11-06T13:15:00Z" }

    SAM->>SAM: 1. Simpan: billing_samsat_id ↔ bpd_trx_id<br/>2. Update status: QRIS_GENERATED

    note over SAM, KOM: Balikan HTTP 200 OK dari Samsat ke Kominfo (Menutup Request Sinkron)
    SAM-->>KOM: 200 OK: { status: "BILLING_LOCKED", billing_samsat_id: "BILL-DIY-XXXX", trx_kominfo_id, nopol: "AB 2481 XZ", total_nominal: 750000, qr_string, expired_at, merchant: "SAMSAT PROVINSI D.I. YOGYAKARTA", pjp: "BANK BPD DIY" }

    KOM->>KOM: 1. Update status transaksi lokal: QRIS_GENERATED<br/>2. Simpan referensi billing_samsat_id
    KOM-->>App: Return { qr_string, billing_id, expired_at, nominal: 750000 }
    App-->>WP: Screen 4: Render Dynamic QRIS ASPI BPD DIY + Timer Countdown (15 Menit)
```

**Penanganan Kasus Error Generate Billing:**

| Kondisi | Respons Samsat | Aksi Kominfo |
|---|---|---|
| Sudah ada billing aktif untuk nopol+tahun | `409 Conflict: { existing_billing_id, expired_at }` | Tampilkan QRIS yang sudah ada (jika belum expired) |
| Tagihan tidak ditemukan / lunas | `404 Not Found` | Tampilkan pesan "Tagihan tidak ditemukan atau sudah lunas" |
| BPD timeout saat generate QRIS | `503 Service Unavailable` | Kominfo tampilkan "Coba lagi dalam beberapa saat" |
| NIK tidak cocok dengan nopol | `403 Forbidden` | Tampilkan "Kendaraan ini tidak terdaftar atas nama Anda" |

---

### 5.4. Tahap 6: Eksekusi Pembayaran & Chain Notification (BPD → Samsat → Kominfo)

> [!TIP] Ini adalah inti dari skema "Kominfo-as-Portal": Kominfo **hanya menerima notifikasi final yang sudah divalidasi** dari Samsat. Tidak perlu menyentuh logika settlement sama sekali.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant BPD as BPD DIY
    participant SAM as Samsat DIY
    participant KOM as Kominfo Backend
    participant App as JSS App (Client)

    WP->>BPD: Scan & Bayar QRIS (Mobile Banking / GoPay / OVO / dll)
    BPD->>BPD: 1. Validasi transaksi QRIS<br/>2. Proses settlement dana masuk<br/>3. Set status bpd_trx_id: SUCCESS

    note over BPD, SAM: ⬇️ STEP 1: BPD Notifikasi ke Samsat (Single Webhook)
    note over BPD, SAM: Security: mTLS + HMAC-SHA256 Signature + Timestamp

    BPD->>SAM: POST /webhook/v2/payment-notification<br/>Body: { bpd_trx_id, billing_samsat_id, amount, paid_at, ref_number }<br/>Header: X-BPD-Signature: HMAC-SHA256(secret, body+timestamp)<br/>         X-BPD-Timestamp: 1731123600<br/>         X-Idempotency-Key: bpd_trx_id

    note over SAM: Samsat Verifikasi Keamanan Webhook

    SAM->>SAM: ✅ 1. Verifikasi HMAC Signature (reject jika invalid)<br/>✅ 2. Verifikasi Timestamp (reject jika > 5 menit lalu)<br/>✅ 3. Cek Idempotency: sudah diproses? → return 200 langsung<br/>✅ 4. Validasi nominal cocok dengan billing<br/>✅ 5. Update status pajak: PAID<br/>✅ 6. Update status: SETTLED, Terbitkan NTPN & e-SKKP<br/>✅ 7. Simpan ref_number BPD sebagai bukti finansial

    SAM-->>BPD: HTTP 200 OK: { status: "RECEIVED", message: "Settlement sukses" }

    note over SAM, KOM: ⬇️ STEP 2: Samsat Notifikasi ke Kominfo (dengan Bukti Sah)
    note over SAM, KOM: Security: mTLS + HMAC-SHA256 + Timestamp (sama ketatnya)

    SAM->>KOM: POST /webhook/v2/payment-success<br/>Body: { billing_samsat_id, trx_kominfo_id, nopol, tahun, ntpn, status: "SETTLED", skkp_download_url, settled_at }<br/>Header: X-SAM-Signature: HMAC-SHA256(secret, body+timestamp)<br/>         X-Idempotency-Key: billing_samsat_id

    KOM->>KOM: ✅ 1. Verifikasi HMAC Signature Samsat<br/>✅ 2. Verifikasi Timestamp (anti-replay)<br/>✅ 3. Cek Idempotency (sudah diproses?)<br/>✅ 4. Update status transaksi user: SETTLED<br/>✅ 5. Simpan NTPN & skkp_download_url<br/>✅ 6. Trigger push notifikasi ke App

    KOM-->>SAM: HTTP 200 OK: { status: "ACK" }

    note over KOM, App: ⬇️ STEP 3: Kominfo Notifikasi ke User & Penerbitan e-SKKP
    KOM-->>App: WebSocket Push / FCM Push Notification:<br/>{ status: "SETTLED", nopol: "AB 2481 XZ", ntpn: "NTPN-20261106-BPD9812", total: 750000, skkp_url }
    App-->>WP: Screen 5: 🎉 Tampilkan "Pembayaran Berhasil!"<br/>(Ringkasan NTPN Resmi Bank BPD DIY & Masa Pajak Baru: 28 Nov 2027)
    WP->>App: Klik "Buka Dokumen e-SKKP Sah"
    App-->>WP: Screen 6: Render In-App Dokumen Digital e-SKKP Sah<br/>(Kop Resmi Pemda DIY, Barcode Kepolisian, Bebas Cap Fisik)
```

---

### 5.4.1. Spesifikasi Dokumen Sah Elektronik e-SKKP (In-App Digital Viewer)

> [!TIP] Efisiensi Pelayanan Publik (Paperless & Keabsahan Hukum Penuh)
> Surat Ketetapan Kewajiban Pembayaran Elektronik (**e-SKKP**) yang diterbitkan oleh Samsat DIY melalui JSS memiliki kekuatan hukum yang sah dan setara dengan SKKP cetak fisik berdasarkan UU ITE No. 1/2024 dan Peraturan Gubernur DIY:
> 1. **Kop Resmi**: Pemerintah Daerah D.I. Yogyakarta – Badan Pengelola Keuangan dan Aset Daerah (BPKAD) – Samsat DIY.
> 2. **Tervalidasi Bank Persepsi**: Tercantum Nomor Transaksi Pembayaran Pajak (**NTPN**) unik dari Bank BPD DIY (`NTPN-20261106-BPD9812`).
> 3. **Barcode QR Verifikasi Sah**: Dapat dipindai sewaktu-waktu oleh petugas kepolisian lalu lintas saat pemeriksaan di jalan tanpa memerlukan stempel cap basah fisik ke kantor Samsat.
> 4. **Arsip Digital Permanen**: Tersimpan permanen di akun JSS warga dan dapat diunduh (PDF) atau dibagikan kapan saja.

---

### 5.5. Spesifikasi Keamanan Webhook (HMAC Verification)

Setiap webhook yang masuk harus diverifikasi dengan pseudocode berikut:

```
// PENERIMA WEBHOOK (Samsat atau Kominfo)

function verifyWebhook(request):
    // 1. Ambil komponen dari header
    received_signature = request.headers["X-Signature"]
    received_timestamp = request.headers["X-Timestamp"]  // Unix timestamp (integer)
    
    // 2. Anti-Replay Attack: tolak jika timestamp > 5 menit lalu
    current_time = now_unix_timestamp()
    if abs(current_time - received_timestamp) > 300:  // 300 detik = 5 menit
        return HTTP 401 Unauthorized ("Timestamp expired")
    
    // 3. Reconstruct & Verify Signature
    raw_body = request.raw_body_string  // body sebelum di-parse JSON
    expected_signature = HMAC_SHA256(shared_secret_key, raw_body + received_timestamp)
    
    if not constant_time_equal(received_signature, expected_signature):
        return HTTP 401 Unauthorized ("Invalid signature")
    
    // 4. Idempotency Check
    idempotency_key = request.headers["X-Idempotency-Key"]
    if already_processed(idempotency_key):
        return HTTP 200 OK ("Already processed")  // return sukses, jangan error
    
    // 5. Lanjutkan proses bisnis
    process_payment_notification(request.body)
    mark_as_processed(idempotency_key)
    return HTTP 200 OK
```

---

### 5.6. Tahap 7: Status Check & Self-Healing Fallback

Mekanisme ini aktif ketika: (a) pengguna klik "Cek Status", atau (b) auto-polling dari app setiap 10 detik selama QRIS aktif.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant App as JSS App
    participant KOM as Kominfo Backend
    participant SAM as Samsat DIY
    participant BPD as BPD DIY

    WP->>App: Klik "Cek Status" / Auto-polling (tiap 10 detik)
    App->>KOM: GET /pajak/status?trx_id={trx_kominfo_id}

    note over KOM: Kominfo cek database internal dulu (cache-first)
    KOM->>KOM: Cek status di DB Kominfo

    alt KONDISI 1: Status di DB Kominfo sudah SETTLED
        note over KOM: Webhook sudah diterima sebelumnya, tidak perlu call Samsat
        KOM-->>App: 200 OK: { status: "SETTLED", ntpn, skkp_url }
        App-->>WP: Tampilkan "Lunas" + Bukti e-SKKP

    else KONDISI 2: Status di DB Kominfo masih PENDING / QRIS_GENERATED
        note over KOM, SAM: Fallback: Tanya ke Samsat (single-pintu, Kominfo tidak perlu tanya BPD langsung)
        KOM->>SAM: GET /api/v2/billing/status?billing_samsat_id={id}<br/>Timeout: 10 detik

        alt Sub-kondisi 2a: Samsat sudah SETTLED (webhook sebelumnya sudah masuk ke Samsat)
            SAM-->>KOM: 200 OK: { status: "SETTLED", ntpn, skkp_url }
            KOM->>KOM: Update status lokal ke SETTLED
            KOM-->>App: 200 OK: { status: "SETTLED", ntpn, skkp_url }
            App-->>WP: Tampilkan "Lunas" + Bukti e-SKKP

        else Sub-kondisi 2b: Samsat masih PENDING → Samsat proaktif tanya BPD
            SAM->>BPD: GET /api/v1/qris/status?bpd_trx_id={id}<br/>Timeout: 8 detik

            alt BPD: Sudah dibayar tapi webhook belum sampai ke Samsat
                BPD-->>SAM: 200 OK: { status: "PAID", paid_at, ref_number }
                SAM->>SAM: Auto-settle: Update DB LUNAS, Terbitkan NTPN & e-SKKP
                SAM->>KOM: POST /webhook/v2/payment-success (async, bukan dalam chain ini)
                SAM-->>KOM: 200 OK: { status: "SETTLED", ntpn, skkp_url }
                KOM-->>App: 200 OK: { status: "SETTLED", ntpn, skkp_url }
                App-->>WP: Tampilkan "Lunas" + Bukti e-SKKP

            else BPD: Belum dibayar / QRIS expired
                BPD-->>SAM: 200 OK: { status: "UNPAID" / "EXPIRED" }
                SAM-->>KOM: 200 OK: { status: "PENDING" / "EXPIRED", expired_at }
                KOM-->>App: 200 OK: { status: "PENDING" / "EXPIRED" }
                App-->>WP: "Belum dibayar / Waktu habis. Silakan generate QRIS baru."
            end
        end
    end
```

---

### 5.7. Tahap 8: Mekanisme Unlock & Cancel Billing

Ini adalah fitur **yang tidak ada di Blueprint A & B** namun kritis untuk menghindari tagihan terkunci permanen.

```mermaid
sequenceDiagram
    autonumber
    participant Scheduler as Samsat Cron Job
    participant SAM as Samsat DIY DB
    participant KOM as Kominfo Backend
    participant App as JSS App

    note over Scheduler: Berjalan setiap 1 menit

    Scheduler->>SAM: Query: SELECT * FROM billing<br/>WHERE status IN ('BILLING_LOCKED', 'QRIS_GENERATED')<br/>AND expired_at < NOW()

    alt Ada billing kadaluarsa
        SAM->>SAM: Update status semua billing expired → 'EXPIRED'<br/>(Melepas lock tagihan kendaraan)
        SAM->>KOM: POST /webhook/v2/billing-expired<br/>Body: { trx_kominfo_id, billing_samsat_id, nopol, tahun }
        KOM->>KOM: Update status lokal → EXPIRED
        KOM-->>App: Push notifikasi: "QRIS Anda telah kadaluarsa"
    end

    note over App: User bisa generate QRIS baru untuk tagihan yang sama
```

**Endpoint Cancel Manual (opsional, untuk user yang sadar ingin cancel):**

```
DELETE /api/v2/billing/{billing_samsat_id}
Pengirim: Kominfo
Penerima: Samsat

Response sukses: 200 OK { status: "CANCELLED", message: "Tagihan dibatalkan" }
Response jika sudah PAID: 409 Conflict { message: "Tagihan sudah dibayar, tidak bisa dibatalkan" }
```

---

### 5.8. Tahap 9: EOD Reconciliation (Jaring Pengaman Akhir)

> [!TIP] EOD Reconciliation adalah lapisan terakhir yang menjamin **tidak ada transaksi yang "jatuh ke celah"** antara sistem BPD, Samsat, dan Kominfo — bahkan jika semua retry sudah habis.

```mermaid
flowchart TD
    A["⏰ Jam 23:00 WIB — EOD Reconciliation Job Berjalan"] --> B["BPD Ekspor Laporan Harian<br/>(Semua transaksi settled hari ini)<br/>via Secure API atau SFTP Encrypted"]

    B --> C["Samsat Terima Laporan BPD"]
    C --> D{"Cocokkan dengan DB Samsat"}

    D -- "Semua Match (normal)" --> E["✅ Tidak ada aksi diperlukan<br/>Log: 'Rekonsiliasi bersih'"]

    D -- "Ada di BPD tapi BELUM SETTLED di Samsat<br/>(transaksi orphan)" --> F["🚨 Auto-settle transaksi orphan:<br/>1. Update status ke SETTLED<br/>2. Terbitkan NTPN & e-SKKP<br/>3. Kirim webhook ke Kominfo<br/>4. Log ke exception report"]

    D -- "Ada di Samsat tapi TIDAK ADA di BPD<br/>(ghost transaction)" --> G["🚨 Flag sebagai MANUAL_REVIEW<br/>Kirim alert ke tim ops Samsat & BPD<br/>Jangan auto-settle — perlu investigasi"]

    F --> H["📊 Kirim Exception Report ke:<br/>- Ops Samsat<br/>- Ops Kominfo<br/>- Ops BPD (jika ada anomali)"]
    G --> H
    E --> H
    H --> I["✅ Rekonsiliasi selesai. Simpan audit log."]

    style A fill:#1a237e,stroke:#7986cb,stroke-width:2px,color:#ffffff
    style E fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style F fill:#5c2d00,stroke:#ffb74d,stroke-width:2px,color:#ffffff
    style G fill:#5a1010,stroke:#ef5350,stroke-width:2px,color:#ffffff
    style H fill:#2d1a4e,stroke:#9575cd,stroke-width:2px,color:#ffffff
    style I fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
```

---

## 6. Ringkasan Desain API & Kontrak Data (Versi Final)

### 6.1. Daftar Endpoint Lengkap

| No | Endpoint | Versi | Pengirim | Penerima | Fungsi & Deskripsi Teknis |
|---|---|---|---|---|---|
| 1 | `GET /api/v2/wajib-pajak?nik={nik}` | v2 | Kominfo | Samsat | Verifikasi kepemilikan NIK terverifikasi di Samsat DIY |
| 2 | `GET /api/v2/kendaraan?nik={nik}` | v2 | Kominfo | Samsat | Daftar kendaraan aktif per NIK (status tunggakan & preview nominal) |
| 3 | `GET /api/v2/pajak/tahun?nopol={nopol}` | v2 | Kominfo | Samsat | Riwayat tahun pajak: `unpaid_years` (FIFO) & `paid_history` (Arsip) |
| 4 | `GET /api/v2/pajak/detail?nopol={nopol}&mode=accumulation` | v2 | Kominfo | Samsat | Rincian nominal resmi akumulasi Samsat (PKB, SWDKLLJ, Denda) |
| 5 | `POST /api/v2/billing/create` | v2 | Kominfo | Samsat | Request billing + lock tagihan akumulasi (Anti Double-Payment) |
| 6 | `GET /api/v2/billing/status?billing_samsat_id={id}` | v2 | Kominfo | Samsat | Cek status billing (fallback self-healing) |
| 7 | `DELETE /api/v2/billing/{billing_samsat_id}` | v2 | Kominfo | Samsat | Cancel billing & pelepasan lock sebelum bayar |
| 8 | `POST /api/v1/qris/generate` | v1 | Samsat | BPD DIY | Request QRIS dinamis ASPI oleh Samsat selaku Merchant Pemda DIY |
| 9 | `GET /api/v1/qris/status?bpd_trx_id={id}` | v1 | Samsat | BPD DIY | Inquiry status QRIS ke core banking BPD DIY |
| 10 | `POST /webhook/v2/payment-notification` | v2 | BPD DIY | Samsat | Webhook pelunasan dana BPD → Samsat (HMAC SHA-256) |
| 11 | `POST /webhook/v2/payment-success` | v2 | Samsat | Kominfo | Webhook berantai sukses Samsat → Kominfo (+ NTPN & e-SKKP) |
| 12 | `POST /webhook/v2/billing-expired` | v2 | Samsat | Kominfo | Notifikasi otomatis pelepasan lock tagihan (TTL 15 menit) |
| 13 | `GET /pajak/status?trx_id={trx_kominfo_id}` | v1 | App Client | Kominfo | Status check polling dari app user |
| 14 | *(Batch)* `GET /api/v1/eod-report?date={YYYY-MM-DD}` | v1 | Samsat | BPD DIY | Laporan rekonsiliasi harian (Nightly Batch) |

### 6.2. Canonical Idempotency Key per Tahap

> [!IMPORTANT] Semua pihak WAJIB menggunakan key yang sama untuk idempotency check.

| Tahap | Canonical Idempotency Key | Disimpan oleh |
|---|---|---|
| Generate Billing (Kominfo → Samsat) | `trx_kominfo_id` (UUID, dibuat Kominfo) | Samsat & Kominfo |
| Generate QRIS (Samsat → BPD) | `billing_samsat_id` (UUID, dibuat Samsat) | BPD & Samsat |
| Webhook BPD → Samsat | `bpd_trx_id` (dibuat BPD, source of truth finansial) | Samsat |
| Webhook Samsat → Kominfo | `billing_samsat_id` | Kominfo |
| Cancel Billing | `billing_samsat_id` | Samsat |

---

### 6.3. Standar Payload API Sesuai Prototype Implementasi

#### A. Response `GET /api/v2/kendaraan?nik=...` (Step 1)
```json
{
  "status": "SUCCESS",
  "nik": "347101******0002",
  "nama_pemilik": "RADEN MAS DANANG P.",
  "total_kendaraan": 2,
  "data_kendaraan": [
    {
      "nopol": "AB 2481 XZ",
      "merk": "Honda",
      "tipe": "Vario 160 ABS (2022)",
      "no_rangka": "MH1KF8123NK098124",
      "no_mesin": "KF81E1928371",
      "masa_pajak": "28 Nov 2026",
      "status_tagihan": "MENUNGGAK_2_TAHUN",
      "tagihan": {
        "periode_akumulasi": [2025, 2026],
        "total_nominal": 750000,
        "ada_denda": true
      }
    },
    {
      "nopol": "AB 1892 YQ",
      "merk": "Toyota",
      "tipe": "Avanza 1.5 G MT (2021)",
      "no_rangka": "MHFM2BA31NK019821",
      "no_mesin": "2NRE8192831",
      "masa_pajak": "15 Apr 2027",
      "status_tagihan": "PERIODE_BERJALAN",
      "tagihan": {
        "periode_akumulasi": [2027],
        "total_nominal": 2450000,
        "ada_denda": false
      }
    }
  ]
}
```

#### B. Response `GET /api/v2/pajak/tahun?nopol=AB2481XZ` (Step 2 - Aturan FIFO)
```json
{
  "nopol": "AB 2481 XZ",
  "unpaid_years": [
    {
      "card_num": 1,
      "tahun": 2025,
      "status": "TERTUNGGAK",
      "label": "[Card 1] Tahun Pajak 2025 (Tertunggak 12 Bulan)",
      "jatuh_tempo": "28 November 2025",
      "pokok_pkb_swdkllj": 350000,
      "denda": 50000,
      "subtotal": 400000
    },
    {
      "card_num": 2,
      "tahun": 2026,
      "status": "BERJALAN",
      "label": "[Card 2] Tahun Pajak 2026 (Periode Berjalan)",
      "jatuh_tempo": "28 November 2026",
      "pokok_pkb_swdkllj": 350000,
      "denda": 0,
      "subtotal": 350000
    }
  ],
  "total_akumulasi": 750000,
  "paid_history": [
    {
      "tahun": 2024,
      "nominal": 335000,
      "ntpn": "NTPN-20241120-BPD6190",
      "status": "LUNAS",
      "tgl_bayar": "20 November 2024"
    },
    {
      "tahun": 2023,
      "nominal": 330000,
      "ntpn": "NTPN-20231118-BPD5102",
      "status": "LUNAS",
      "tgl_bayar": "18 November 2023"
    }
  ]
}
```

#### C. Request & Response `POST /api/v2/billing/create` (Step 3 & 4)
**Request (Kominfo ➔ Samsat):**
```json
{
  "trx_kominfo_id": "trx-jss-1793943787788",
  "nopol": "AB 2481 XZ",
  "periode": "2025 - 2026",
  "total_nominal": 750000
}
```

**Response (Samsat ➔ Kominfo):**
```json
{
  "status": "BILLING_LOCKED",
  "billing_samsat_id": "BILL-DIY-20261106-9812",
  "trx_kominfo_id": "trx-jss-1793943787788",
  "nopol": "AB 2481 XZ",
  "periode": "2025 - 2026",
  "total_nominal": 750000,
  "qr_string": "00020101021226600016ID.GO.DIY.BPD01...",
  "expired_at": "2026-11-06T13:15:00Z",
  "merchant": "SAMSAT PROVINSI D.I. YOGYAKARTA",
  "pjp": "BANK BPD DIY"
}
```

#### D. Webhook Chain `POST /webhook/v2/payment-success` (Step 5 & 6)
**Samsat ➔ Kominfo:**
```json
{
  "billing_samsat_id": "BILL-DIY-20261106-9812",
  "nopol": "AB 2481 XZ",
  "periode": "2025 - 2026",
  "ntpn": "NTPN-20261106-BPD9812",
  "status": "SETTLED",
  "total_bayar": 750000,
  "masa_pajak_baru": "28 November 2027",
  "skkp_download_url": "https://samsat.jogjaprov.go.id/eskkp/download/NTPN-20261106-BPD9812"
}
```

---

## 7. Spesifikasi Keamanan Lengkap

### 7.1. Lapisan Keamanan Komunikasi Antar-Sistem

```
Lapisan 1 (Transport):  mTLS (Mutual TLS) — semua komunikasi server-to-server
Lapisan 2 (Auth):       API Key per pasangan institusi (Kominfo↔Samsat, Samsat↔BPD)
Lapisan 3 (Integrity):  HMAC-SHA256 Signature pada semua webhook body
Lapisan 4 (Anti-Replay): Timestamp validation (tolak jika > 5 menit)
Lapisan 5 (Network):    IP Whitelist di firewall/WAF masing-masing server
```

### 7.2. Spesifikasi Timeout per Endpoint

| Panggilan API | Timeout | Max Retry | Strategi Retry |
|---|---|---|---|
| Kominfo → Samsat (Inquiry) | 10 detik | 2x | Linear backoff (3 detik) |
| Kominfo → Samsat (Billing Create) | 15 detik | 0x | Non-idempotent, tidak diretry otomatis |
| Kominfo → Samsat (Status Check) | 10 detik | 2x | Linear backoff |
| Samsat → BPD (QRIS Generate) | 10 detik | 3x | Exponential backoff (2, 4, 8 detik) |
| Samsat → BPD (Status Inquiry) | 8 detik | 2x | Linear backoff |
| BPD Webhook → Samsat | 5 detik | 5x | Exponential backoff (2, 4, 8, 16, 32 detik) |
| Samsat Webhook → Kominfo | 5 detik | 5x | Exponential backoff (2, 4, 8, 16, 32 detik) |
| Samsat → Kominfo (Expired notif) | 5 detik | 3x | Exponential backoff |

### 7.3. Perlindungan Data NIK (Kepatuhan UU PDP No. 27/2022)

| Aspek | Ketentuan |
|---|---|
| **Penyimpanan di Kominfo** | NIK di-encrypt at-rest menggunakan AES-256. Disimpan hanya di server-side session, bukan di JWT yang dikirim ke client. |
| **Transmisi ke Samsat** | Hanya via backend-to-backend (mTLS). Tidak pernah via frontend atau URL parameter yang ter-log. |
| **Logging** | NIK **tidak boleh muncul** di application log dalam bentuk plaintext. Gunakan masking (e.g., `****1234`). |
| **Retensi** | Setelah sesi berakhir, NIK di-session store dihapus otomatis sesuai TTL session. |
| **Hak Akses** | Hanya service account spesifik (bukan semua developer) yang bisa akses data NIK di environment production. |

---

## 8. Arsitektur Ketahanan Sistem (Reliability Matrix)

| Skenario Failure | Dampak ke User | Mekanisme Self-Healing | Waktu Pemulihan Target |
|---|---|---|---|
| Kominfo **down** saat payment berlangsung | User tidak menerima notifikasi UI | BPD → Samsat tetap berjalan. Ketika Kominfo kembali online, Samsat retry webhook. User cek status saat buka app. | Saat Kominfo kembali online |
| Samsat **down** saat payment berlangsung | Dana masuk BPD, pajak belum ter-update | BPD retry webhook 5x. Jika semua gagal → masuk DLQ → EOD Reconciliation malam hari auto-settle. | Malam hari (EOD) |
| BPD webhook ke Samsat **delay** | User lihat status pending | User klik "Cek Status" → Samsat query ke BPD → auto-settle jika sudah bayar | < 1 menit (manual trigger) |
| **QRIS expired** (user tidak bayar dalam 15 menit) | QRIS tidak bisa discan | Samsat scheduler otomatis release lock. User generate QRIS baru. | Otomatis (cron 1 menit) |
| **Double webhook** dari BPD ke Samsat | Potensi double settle | Idempotency check di Samsat dengan `bpd_trx_id`. Request ke-2 diabaikan (return 200 OK). | Tidak ada dampak |
| Samsat webhook ke Kominfo **gagal terus** | Notifikasi tidak muncul di app | Masuk DLQ Samsat. Ops mendapat alert. User bisa cek status manual. | DLQ alert dalam < 30 menit |
| EOD Reconciliation menemukan **orphan** | Pajak user belum ter-update meski sudah bayar | Auto-settle oleh Reconciliation Job. Kominfo notifikasi user keesokan paginya. | Sebelum pukul 06:00 pagi |

---

## 9. API Versioning Policy

> [!NOTE] Semua endpoint menggunakan prefix `/v{n}/`. Versi saat ini adalah **v2** untuk Kominfo-Samsat dan **v1** untuk Samsat-BPD (mengikuti API BPD yang ada).

**Aturan Versioning:**
1. **Additive changes** (tambah field baru di response, tambah optional field di request) → **tidak butuh versi baru**, pastikan field bersifat optional.
2. **Breaking changes** (hapus field, ubah tipe data, ubah semantik) → **wajib naik versi** (`v2` → `v3`).
3. **Deprecation policy**: Versi lama tetap berjalan minimal **3 bulan** setelah versi baru tersedia. Tim consumer diberi notice H-30 hari sebelum sunset.
4. **Response envelope** standar untuk semua API:

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "request_id": "uuid-v4-dari-X-Request-ID",
    "timestamp": "2026-11-06T09:30:00+07:00",
    "api_version": "v2"
  },
  "error": null
}
```

---

## 10. Checklist Pre-Production (Siap Go-Live)

### 🔒 Security
- [ ] mTLS dikonfigurasi dan diuji di semua channel antar-server
- [ ] HMAC Secret Key di-generate per pasangan institusi (bukan satu key untuk semua)
- [ ] HMAC Secret Key disimpan di **Secrets Manager** (HashiCorp Vault / AWS Secrets Manager / Azure Key Vault) — bukan di file `.env` biasa
- [ ] IP Whitelist aktif di firewall/WAF semua server penerima webhook
- [ ] Penetration testing dilakukan oleh tim independen
- [ ] NIK tidak muncul di application log dan response yang dikirim ke client

### 🏗️ Reliability
- [ ] State Machine diimplementasikan dan semua transisi invalid ditolak dengan log error
- [ ] Scheduler job unlock billing kadaluarsa berjalan setiap 1 menit (uji dengan scenario)
- [ ] Retry mechanism + exponential backoff aktif sesuai tabel timeout
- [ ] DLQ terkonfigurasi dan alert ke tim ops aktif
- [ ] EOD Reconciliation Job terjadwal dan telah diuji dengan data simulasi
- [ ] Monitoring dashboard aktif (jumlah transaksi, status per state, alert anomali)

### ⚖️ Regulasi & Hukum
- [ ] Konfirmasi Samsat sebagai Merchant resmi BPD DIY (bukan Kominfo)
- [ ] PKS tiga pihak sudah mencakup klausul SLA, liability, dan data sharing
- [ ] Compliance review UU PDP No. 27/2022 untuk penyimpanan NIK di JSS
- [ ] Klausul DRP (Disaster Recovery Plan) tercantum di PKS

### 🧪 Testing
- [ ] Unit test: State machine (semua transisi valid & invalid)
- [ ] Integration test: End-to-end happy path (Inquiry → Generate → Pay → Notify → Download e-SKKP)
- [ ] Integration test: Webhook gagal 5x → masuk DLQ → alert ops
- [ ] Integration test: QRIS expired → lock dilepas → user generate QRIS baru
- [ ] Integration test: Double webhook → idempotency check sukses (tidak double settle)
- [ ] Load test: Kapasitas QRIS per detik (tentukan batas & throttle)
- [ ] Disaster Recovery test: Simulasi Samsat down 30 menit lalu kembali online
- [ ] UAT: Skenario pajak dengan denda, pajak multi-tahun, kendaraan mutasi provinsi

---

## Appendix: Perbandingan Tiga Blueprint

| Aspek | Blueprint A (Original) | Blueprint B (Kominfo Direct) | **Blueprint Optimal (Ini)** |
|---|---|---|---|
| Keamanan NIK | Tidak dibahas | Tidak dibahas | ✅ Full UU PDP compliance |
| Webhook Auth | Disebut, tidak di-spec | Tidak ada | ✅ HMAC + Timestamp + Idempotency |
| State Machine | Tidak ada | Tidak ada | ✅ Formal, semua pihak pakai |
| Unlock Billing | Tidak ada | Tidak ada | ✅ Scheduler + Manual endpoint |
| Idempotency Key | Tidak konsisten | Tidak konsisten | ✅ Canonical key per tahap |
| Timeout Spec | Tidak ada | Tidak ada | ✅ Per-endpoint dengan retry |
| EOD Reconciliation | Tidak ada | Tidak ada | ✅ Nightly batch + exception report |
| API Versioning | Disebut (v1) | Disebut (v1) | ✅ Policy formal + envelope standar |
| Risiko Regulasi BI | Aman | ⚠️ PJP issue | ✅ Aman (Samsat = Merchant) |
| Beban Kominfo | Ringan | Berat | ✅ Minimal (Portal only) |
| Reliability | Baik | Risiko tinggi | ✅ Self-healing by design |

---

*Blueprint ini merupakan evolusi dari Blueprint A dan Blueprint B, menggabungkan keunggulan arsitektur Chain Notification dengan tambahan spesifikasi teknis yang hardened untuk produksi.*
*Revisi selanjutnya hanya dilakukan setelah review bersama tim teknis Kominfo, Samsat, dan BPD DIY.*

