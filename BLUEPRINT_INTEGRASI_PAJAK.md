# Blueprint Integrasi Pembayaran Pajak Kendaraan Bermotor (PKB)
**Pemerintah Provinsi DIY: Kominfo – Samsat – BPD DIY**
*Skema Pembayaran: Komparasi Opsi A (Split Broadcast) & Opsi B (Chain Notification)*

---

## 1. Ringkasan Eksekutif & Arsitektur Solusi

Dokumen ini menjelaskan rancangan cetak biru (*blueprint*) teknis untuk alur integrasi pembayaran pajak kendaraan bermotor. Integrasi melibatkan entitas utama:
1. **Wajib Pajak (Pengguna JSS)**: Warga yang telah memiliki akun terverifikasi di **Jogja Smart Service (JSS)**.
2. **Dinas Komunikasi dan Informatika (Kominfo / Portal JSS)**: Penyedia kanal layanan publik (SuperApp JSS / SSO Identity Provider & App Server) yang menyimpan profil kependudukan terverifikasi (termasuk NIK).
3. **Samsat DIY**: Otoritas pengelola data perpajakan kendaraan bermotor dan pencetakan bukti bayar/e-SKKP/e-TBPKP (*Core Biller*).
4. **BPD DIY**: Institusi perbankan persepsi dan penyedia gerbang pembayaran QRIS (*Payment Gateway / Acquirer*).

---

## 2. Arsitektur Komponen & Aliran Data

```mermaid
flowchart LR
    WP["Wajib Pajak (User)"] 
    
    subgraph Frontend["Front-Facing & Identity (Kominfo)"]
        JSS["JSS (Jogja Smart Service)<br/>- Akun & SSO User<br/>- Profil NIK Terverifikasi<br/>- Portal / Mobile Apps"]
    end
    
    subgraph CoreBiller["Biller & Tax Authority"]
        SAM["Samsat DIY<br/>(Core Tax Engine & Database)"]
    end
    
    subgraph Banking["Banking & Payment Gateway"]
        BPD["BPD DIY<br/>(QRIS Engine & Core Banking)"]
    end

    WP <-->|"1. Login JSS / Auto-load NIK<br/>2. Pilih Nopol & Scan QRIS"| JSS
    JSS <-->|"REST API (Inquiry by NIK, Billing, Webhook)"| SAM
    SAM <-->|"REST / ISO 8583 / JSON Webhook"| BPD
```

---

## 3. Alur Proses End-to-End (High-Level Process)

```mermaid
flowchart TD
    subgraph Auth["Otentikasi & Profiling"]
        S0["0. Login Akun JSS<br/>(Ambil NIK otomatis dari Profil JSS)"]
    end

    subgraph Inquiry["Inquiry & Validasi Data"]
        S0 --> S1["1. Get By NIK<br/>(Kominfo kirim NIK JSS ke Samsat)"]
        S1 --> S2["2. Get List Nopol<br/>(Daftar Kendaraan terdaftar atas NIK)"]
        S2 --> S3["3. Get List Tahun by Nopol<br/>(Pilih Periode/Tahun)"]
        S3 --> S4["4. Get Detail Pajak Nopol<br/>(Rincian Nominal & Tagihan)"]
    end

    subgraph Billing["Penagihan & Pembayaran"]
        S4 --> S5["5. Generate QRIS<br/>(Kominfo JSS ➔ Samsat ➔ BPD DIY)"]
        S5 --> S6{"6. Payment Notification"}
        S6 -->|Opsi A| OptA["Opsi A: BPD Split Broadcast<br/>Kabari Samsat & Kominfo Sekaligus"]
        S6 -->|Opsi B| OptB["Opsi B: Chain Notification (Direkomendasikan)<br/>BPD ➔ Samsat ➔ Kominfo"]
    end

    subgraph StatusCheck["Verifikasi & Rekonsiliasi"]
        OptA --> S7["7. Cek Status Transaksi<br/>(Polling / Fallback / Inquiry Status)"]
        OptB --> S7
    end

    style S0 fill:#2d1a4e,stroke:#9575cd,stroke-width:2px,color:#ffffff
    style S1 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S2 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S3 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S4 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S5 fill:#5c2d00,stroke:#ffb74d,stroke-width:2px,color:#ffffff
    style OptA fill:#4e3800,stroke:#ffd54f,stroke-width:2px,color:#ffffff
    style OptB fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style S7 fill:#4a154b,stroke:#ba68c8,stroke-width:2px,color:#ffffff
```

---

## 4. Rincian Alur & Spesifikasi Interaksi (Sequence Diagrams)

### 4.1. Tahap 1 - 4: Inquiry dan Pemilihan Objek Pajak (Berbasis Akun JSS)
Karena pengguna telah terotentikasi di aplikasi JSS (Jogja Smart Service), **pengguna tidak perlu lagi mengetik NIK secara manual**. NIK langsung diambil dari sesi/token profil akun JSS terverifikasi dan diteruskan ke Samsat.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak (User)
    participant JSSApp as JSS Client (App / Web)
    participant JSSServer as Kominfo (Backend JSS)
    participant SAM as Samsat DIY

    %% Inisialisasi dari JSS
    note over WP, JSSServer: User Login via SSO JSS (Profil Terverifikasi)
    WP->>JSSApp: Buka Menu "Pajak Kendaraan (PKB)"
    JSSApp->>JSSServer: Request Layanan Pajak (Bearer Token JSS)
    JSSServer->>JSSServer: Extract Data User & NIK dari Akun JSS terdaftar

    %% Step 1: Get By NIK
    note over JSSServer, SAM: 1. Get Data by NIK (Validasi Wajib Pajak)
    JSSServer->>SAM: GET /api/v1/wajib-pajak?nik={nik_user_jss}
    SAM-->>JSSServer: Response: Data Wajib Pajak (Status NIK Valid & Profil Samsat)

    %% Step 2: Get List Nopol
    note over JSSServer, SAM: 2. Get List Nopol Berdasarkan NIK JSS
    JSSServer->>SAM: GET /api/v1/kendaraan?nik={nik_user_jss}
    SAM-->>JSSServer: List Kendaraan Terdaftar (Nopol, Merk, Tipe, Masa Berlaku)
    JSSServer-->>JSSApp: Kirim List Kendaraan Terdaftar
    JSSApp-->>WP: Tampilkan Daftar Nopol Kendaraan Milik User

    %% Step 3: Get List Tahun by Nopol
    note over WP, SAM: 3. Get List Tahun by Nopol
    WP->>JSSApp: Pilih Salah Satu Nopol
    JSSApp->>JSSServer: Request Tagihan per Nopol { nopol }
    JSSServer->>SAM: GET /api/v1/pajak/tahun?nopol={nopol}
    SAM-->>JSSServer: List Tahun/Masa Pajak yang Belum Lunas (e.g. [2025, 2026])
    JSSServer-->>JSSApp: Kirim Opsi Tahun Pajak
    JSSApp-->>WP: Tampilkan Pilihan Periode/Tahun Pajak

    %% Step 4: Get Detail Pajak
    note over WP, SAM: 4. Get Detail Pajak untuk Nopol
    WP->>JSSApp: Pilih Tahun Pajak & Konfirmasi
    JSSApp->>JSSServer: Request Rincian { nopol, tahun }
    JSSServer->>SAM: GET /api/v1/pajak/detail?nopol={nopol}&tahun={tahun}
    SAM-->>JSSServer: Rincian Tagihan Resmi (PKB, SWDKLLJ, PNBP, Denda, Total)
    JSSServer-->>JSSApp: Return Rincian Tagihan
    JSSApp-->>WP: Tampilkan Billing Summary & Tombol "Bayar QRIS"
```

---

### 4.2. Tahap 5: Pembuatan Billing & QRIS Dinamis
Kominfo meminta kode billing ke Samsat, dan Samsat meneruskan parameter nominal ke BPD DIY untuk pembuatan *Dynamic QRIS* (NMID BPD DIY).

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant KOM as Kominfo (Portal)
    participant SAM as Samsat DIY
    participant BPD as BPD DIY

    WP->>KOM: Klik "Bayar via QRIS"
    KOM->>SAM: POST /api/v1/billing/qris<br/>Body: { nik, nopol, tahun, nominal }
    
    SAM->>SAM: 1. Kunci Tagihan (Lock State)<br/>2. Generate Kode Billing Samsat
    
    SAM->>BPD: POST /api/v1/qris/generate<br/>Body: { billing_id, nominal, expired_minutes: 15 }
    BPD->>BPD: Generate QR String & Payload ASPI/BI standard
    BPD-->>SAM: Response: { qr_string, qr_image_url, trx_id, expired_at }
    
    SAM->>SAM: Simpan relasi: billing_id <-> trx_id BPD
    SAM-->>KOM: Response: { billing_id, qr_string, qr_image_url, expired_at, nominal }
    KOM-->>WP: Render QRIS Dinamis & Timer Hitung Mundur
```

---

### 4.3. Tahap 6: Eksekusi Pembayaran (Payment Execution)

#### **Opsi A: BPD Split Callback (BPD Mengabari Samsat & Kominfo Secara Terpisah)**

> [!WARNING] Catatan Evaluasi Opsi A:
> - **Kelebihan**: Notifikasi ke Kominfo datang secara langsung dari BPD sehingga respons UI bisa lebih cepat.
> - **Kekurangan / Risiko**: 
>   1. **Risiko Inkonsistensi Data (Race Condition)**: Kominfo menerima info SUKSES padahal Samsat bisa jadi mengalami kegagalan saat proses *commit database* atau penerbitan nomor pengesahan (NTPN/e-SKKP).
>   2. **Beban BPD**: BPD DIY harus mengelola 2 URL endpoint callback terpisah dan menangani retry terpisah untuk tiap institusi.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant BPD as BPD DIY
    participant SAM as Samsat DIY
    participant KOM as Kominfo (Portal)

    note over WP, KOM: Opsi A: BPD Split Callback (Broadcast ke 2 Pihak)

    WP->>BPD: Scan & Bayar QRIS (Mobile Banking / e-Wallet)
    BPD->>BPD: Settlement & Validasi Pembayaran

    par Webhook ke Samsat DIY
        note over BPD, SAM: Callback Jalur 1 (Update Core Biller)
        BPD->>SAM: POST /webhook/bpd/samsat-notify<br/>Body: { trx_id, billing_id, amount, status: "SUCCESS" }
        SAM->>SAM: Update Status DB Pajak (LUNAS) & Generate e-SKKP / NTPN
        SAM-->>BPD: HTTP 200 OK (Acknowledged)
    and Webhook ke Kominfo
        note over BPD, KOM: Callback Jalur 2 (Update Portal Warga)
        BPD->>KOM: POST /webhook/bpd/kominfo-notify<br/>Body: { trx_id, billing_id, amount, status: "SUCCESS" }
        KOM->>KOM: Update Status Transaksi UI (PAID)
        KOM-->>BPD: HTTP 200 OK (Acknowledged)
    end

    note over SAM, KOM: Catatan: Kominfo belum memegang nomor dokumen resmi (e-SKKP) dari Samsat<br/>sehingga butuh API call tambahan ke Samsat untuk ambil bukti sah.

    KOM-->>WP: Tampilkan Notifikasi Pembayaran Berhasil
```

---

#### **Opsi B: Chain Notification (BPD ➔ Samsat ➔ Kominfo) — [DIREKOMENDASIKAN]**

> [!TIP] Catatan Evaluasi Opsi B:
> - **Single Source of Truth**: Samsat memvalidasi dan membukukan pelunasan terlebih dahulu serta langsung melampirkan nomor NTPN dan tautan unduh e-SKKP sebelum mengabari Kominfo.
> - **Integritas Terjamin**: Kominfo dipastikan hanya menampilkan status lunas jika sistem legalitas Samsat telah sukses menerbitkan bukti sah.
> - **Arsitektur Rapi**: BPD hanya berkomunikasi dengan Samsat (standar B2B Biller Bank).

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant BPD as BPD DIY
    participant SAM as Samsat DIY
    participant KOM as Kominfo (Portal)

    note over WP, KOM: Opsi B: Chain Notification (Single Source of Truth)

    WP->>BPD: Scan & Bayar QRIS (Mobile Banking / e-Wallet mana saja)
    BPD->>BPD: 1. Proses settlement dana<br/>2. Tandai transaksi: SUCCESS

    %% Callback Step 1: BPD -> Samsat
    note over BPD, SAM: Step 1: Bank Melaporkan Pelunasan ke Biller
    BPD->>SAM: POST /webhook/bpd/payment-notification<br/>Body: { trx_id, billing_id, amount, paid_at, ref_number }
    
    alt Verifikasi & Pelunasan Samsat
        SAM->>SAM: 1. Validasi nominal & billing_id<br/>2. Update Status Database Pajak: LUNAS<br/>3. Terbitkan e-SKKP / NTPN / No. Pengesahan
        SAM-->>BPD: HTTP 200 OK: { status: "RECEIVED", message: "Settled" }
    else Validasi Gagal / Billing Expired
        SAM-->>BPD: HTTP 400 Bad Request / 409 Conflict
    end

    %% Callback Step 2: Samsat -> Kominfo
    note over SAM, KOM: Step 2: Biller Mengirim Notifikasi + Bukti Sah ke Portal Warga
    SAM->>KOM: POST /webhook/samsat/payment-success<br/>Body: { billing_id, nopol, ntpn, status: "PAID", download_url_skkp }
    KOM->>KOM: 1. Update status transaksi user<br/>2. Simpan referensi dokumen sah
    KOM-->>SAM: HTTP 200 OK: { status: "ACK" }

    %% Notifikasi ke User
    KOM-->>WP: Notifikasi Sukses / WebSocket Push / Tampilkan Bukti Pelunasan (e-SKKP)
```

---

### 4.4. Tahap 7: Pengecekan Status Transaksi (Fallback / Polling / Auto-Reconcile)

Mekanisme ini bekerja sebagai penjamin konsistensi (*self-healing*) jika webhook mengalami delay/gangguan jaringan, atau saat pengguna menekan tombol "Cek Status" di aplikasi JSS.

#### **Logika Bisnis Pengecekan Status:**
1. **Kominfo (JSS)** hanya bertanya ke satu pintu: **Samsat DIY**.
2. **Jika database Samsat sudah Lunas**: Samsat langsung membalas ke Kominfo (tanpa beban request ke BPD).
3. **Jika database Samsat masih Belum Lunas (Pending)**: Samsat proaktif melakukan cross-check ke BPD DIY untuk memastikan status QRIS mutakhir:
   - Jika BPD menyatakan **Sudah Bayar**: Samsat seketika melakukan pelunasan otomatis (*auto-settle*), menerbitkan e-SKKP, dan mengembalikan status **PAID** ke Kominfo.
   - Jika BPD menyatakan **Belum Bayar / Expired**: Samsat mengembalikan status **PENDING / EXPIRED** ke Kominfo.

#### **Flowchart Logika Keputusan Cek Status:**
```mermaid
flowchart TD
    A["Kominfo (JSS) Tanya Status ke Samsat"] --> B{"Cek Database Samsat:<br/>Sudah Lunas?"}
    
    B -- "YA (Sudah Lunas)" --> C["Samsat LANGSUNG respon ke Kominfo:<br/>Status: LUNAS + e-SKKP"]
    
    B -- "BELUM (Masih Pending)" --> D["Samsat CEK ke BPD DIY:<br/>'Apakah QRIS ini sudah dibayar?'"]
    
    D --> E{"Respon dari BPD DIY"}
    E -- "SUDAH BAYAR di Bank" --> F["1. Samsat update status jadi LUNAS<br/>2. Terbitkan NTPN / e-SKKP"]
    F --> C
    
    E -- "BELUM BAYAR / EXPIRED" --> G["Samsat kirim ke Kominfo:<br/>Status: MENUNGGU PEMBAYARAN / EXPIRED"]

    style A fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style B fill:#2d1a4e,stroke:#9575cd,stroke-width:2px,color:#ffffff
    style C fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style D fill:#5c2d00,stroke:#ffb74d,stroke-width:2px,color:#ffffff
    style E fill:#4e3800,stroke:#ffd54f,stroke-width:2px,color:#ffffff
    style F fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style G fill:#5a1010,stroke:#ef5350,stroke-width:2px,color:#ffffff
```

#### **Sequence Diagram Interaksi API Cek Status:**
```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak (JSS App)
    participant KOM as Kominfo (JSS Backend)
    participant SAM as Samsat DIY
    participant BPD as BPD DIY

    WP->>KOM: Klik "Cek Status Pembayaran" / Polling Interval
    KOM->>SAM: GET /api/v1/payment/status?billing_id={id}

    alt KONDISI 1: Di Database Samsat Sudah Lunas
        note over SAM: Webhook BPD sudah diproses sebelumnya
        SAM-->>KOM: Return 200 OK: { status: "PAID", ntpn: "...", skkp_url: "..." }
        KOM-->>WP: Tampilkan Status Lunas & Bukti Sah (e-SKKP)
    else KONDISI 2: Di Database Samsat Masih Belum Terbayar (Pending)
        note over SAM, BPD: Samsat proaktif cross-check ke BPD DIY
        SAM->>BPD: GET /api/v1/qris/status?billing_id={id}
        
        alt Respon BPD: Ternyata Sudah Dibayar di Bank
            BPD-->>SAM: Response 200 OK: { status: "PAID", paid_at: "...", ref_no: "..." }
            SAM->>SAM: 1. Auto-settlement di database pajak (LUNAS)<br/>2. Generate nomor NTPN & e-SKKP
            SAM-->>KOM: Return 200 OK: { status: "PAID", ntpn: "...", skkp_url: "..." }
            KOM-->>WP: Tampilkan Notifikasi Pembayaran Berhasil & e-SKKP
        else Respon BPD: Memang Belum Dibayar / Waktu Habis
            BPD-->>SAM: Response 200 OK: { status: "UNPAID" / "EXPIRED" }
            SAM-->>KOM: Return 200 OK: { status: "PENDING" / "EXPIRED" }
            KOM-->>WP: Tampilkan Info: Silakan selesaikan pembayaran sebelum timer habis
        end
    end
```

---

## 5. Ringkasan Desain API & Kontrak Data

| No | Endpoint | Pengirim | Penerima | Skema | Fungsi Utama |
|---|---|---|---|---|---|
| 1 | `GET /api/v1/wajib-pajak?nik={nik}` | Kominfo | Samsat | Semua | Verifikasi data kepemilikan NIK |
| 2 | `GET /api/v1/kendaraan?nik={nik}` | Kominfo | Samsat | Semua | Mengambil daftar nopol yang terikat pada NIK |
| 3 | `GET /api/v1/pajak/tahun?nopol={nopol}` | Kominfo | Samsat | Semua | Mengambil daftar tahun pajak tertunggak/berjalan |
| 4 | `GET /api/v1/pajak/detail?nopol={nopol}&tahun={tahun}` | Kominfo | Samsat | Semua | Menghitung rincian kewajiban pembayaran |
| 5 | `POST /api/v1/billing/qris` | Kominfo | Samsat | Semua | Permintaan pembuatan billing dan QRIS |
| 6 | `POST /api/v1/qris/generate` | Samsat | BPD DIY | Semua | Request penerbitan QR string resmi |
| 7a | `POST /webhook/bpd/samsat-notify` | BPD DIY | Samsat | Opsi A | Webhook notifikasi bayar dari BPD ke Samsat |
| 7b | `POST /webhook/bpd/kominfo-notify` | BPD DIY | Kominfo | Opsi A | Webhook notifikasi bayar dari BPD ke Kominfo |
| 8a | `POST /webhook/bpd/payment-notification` | BPD DIY | Samsat | Opsi B | Notifikasi pelunasan tunggal dari BPD ke Biller |
| 8b | `POST /webhook/samsat/payment-success` | Samsat | Kominfo | Opsi B | Notifikasi berantai dari Samsat ke Portal Kominfo (+ e-SKKP) |
| 9 | `GET /api/v1/payment/status?billing_id={id}` | Kominfo | Samsat | Semua | Verifikasi status pembayaran berkala/fallback |
| 10 | `GET /api/v1/qris/status?trx_id={trx_id}` | Samsat | BPD DIY | Semua | Konfirmasi status ke core bank |

---

## 6. Aspek Keamanan & Ketahanan Sistem (Reliability)

1. **Keamanan Transaksi**:
   - Komunikasi antar-server menggunakan **mTLS** atau **HTTPS + Signature Token (HMAC SHA-256)**.
   - Pemanfaatan *IP Whitelisting* untuk akses endpoint webhook.
2. **Idempotensi (Idempotency Key)**:
   - Webhook harus bersifat idempotent menggunakan `billing_id` atau `trx_id` guna mengantisipasi pengiriman ulang (*duplicate/retry callback*).
3. **Masa Berlaku Tagihan & Billing Timeout**:
   - Dynamic QRIS memiliki TTL (Time-To-Live) terbatas (misal: 15–30 menit).
   - Jika waktu habis tanpa pelunasan, Samsat melepaskan *lock* pada tagihan kendaraan.
4. **Retry Mechanism & Dead-Letter Queue (DLQ)**:
   - Jika sistem penerima webhook mengalami *downtime*, pengirim webhook menerapkan mekanisme antrean coba-ulang (*exponential backoff*).


