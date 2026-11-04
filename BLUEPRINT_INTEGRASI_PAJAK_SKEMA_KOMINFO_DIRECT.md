# Blueprint Integrasi Pembayaran Pajak Kendaraan Bermotor (PKB)
**Pemerintah Provinsi DIY: Kominfo – Samsat – BPD DIY**
*Skema Pembayaran: Kominfo Direct to Bank (Kominfo ➔ BPD Direct Integration)*

---

## 1. Ringkasan Eksekutif & Karakteristik Arsitektur

Dokumen ini menjelaskan rancangan cetak biru (*blueprint*) teknis untuk skema alternatif di mana **Kominfo (JSS)** bertindak sebagai agregator/orkestrator utama (*Hub / Gateway*) yang menghubungkan langsung kanal perpajakan **Samsat DIY** dengan kanal perbankan **BPD DIY**.

### Karakteristik Utama Skema Ini:
1. **Inquiry (Tahap 1–4)**: Jalur murni antara **Kominfo (JSS)** dan **Samsat DIY** untuk validasi identitas NIK JSS, daftar nopol kendaraan, daftar tahun pajak terutang, serta rincian nominal tagihan resmi.
2. **Create QRIS (Tahap 5)**: **Kominfo langsung menembak API BPD DIY** untuk generate Dynamic QRIS (tanpa melalui Samsat).
3. **Payment Notification (Tahap 6 - Opsi A)**: BPD DIY mengirim callback pelunasan ke **Kominfo**, kemudian **Kominfo yang meneruskan pelunasan ke Samsat DIY** untuk update status lunas dan meminta penerbitan bukti sah (e-SKKP / NTPN).
4. **Cek Status (Tahap 7)**: Kominfo melakukan sinkronisasi status ke **BPD DIY** terlebih dahulu untuk konfirmasi finansial, lalu jika terbukti lunas mengabari/settle ke **Samsat DIY**.

---

## 2. Arsitektur Komponen & Aliran Data

```mermaid
flowchart LR
    WP["Wajib Pajak (User)"] 
    
    subgraph Hub["Orchestrator & Front-Facing (Kominfo)"]
        JSS["JSS (Jogja Smart Service)<br/>- Akun & SSO User Terdaftar<br/>- Profil NIK Terverifikasi<br/>- Integrator Biller & PG"]
    end
    
    subgraph CoreBiller["Biller & Tax Authority"]
        SAM["Samsat DIY<br/>- Database Kendaraan & Pajak<br/>- Penerbit e-SKKP / NTPN"]
    end
    
    subgraph Banking["Acquirer & Payment Gateway"]
        BPD["BPD DIY<br/>- QRIS Dynamic Engine<br/>- Settlement Finansial"]
    end

    WP <-->|"1. Login JSS / Auto-load NIK<br/>2. Pilih Nopol & Scan QRIS"| JSS
    JSS <-->|"REST API (Inquiry Pajak & Settlement Lunas)"| SAM
    JSS <-->|"REST API (Generate QRIS, Webhook & Cek Status)"| BPD
```

---

## 3. Alur Proses End-to-End (High-Level Process)

```mermaid
flowchart TD
    subgraph Auth["Otentikasi & Profiling"]
        S0["0. Login Akun JSS<br/>(Ambil NIK otomatis dari Profil JSS)"]
    end

    subgraph Inquiry["Inquiry & Validasi Data (Kominfo ➔ Samsat)"]
        S0 --> S1["1. Get By NIK<br/>(Validasi Status Pemilik Pajak)"]
        S1 --> S2["2. Get List Nopol<br/>(Daftar Kendaraan Terdaftar)"]
        S2 --> S3["3. Get List Tahun by Nopol<br/>(Pilih Periode/Tahun Pajak)"]
        S3 --> S4["4. Get Detail Pajak Nopol<br/>(Rincian Nominal & Tagihan Sah)"]
    end

    subgraph Billing["Penerbitan QRIS (Kominfo ➔ BPD DIY)"]
        S4 --> S5["5. Create QRIS Dinamis<br/>(Kominfo Request Langsung ke BPD DIY)"]
    end

    subgraph PaymentFlow["Pembayaran & Settlement (BPD ➔ Kominfo ➔ Samsat)"]
        S5 --> S6["6. Payment Notification (Opsi A)<br/>BPD Callback ke Kominfo ➔ Kominfo Teruskan ke Samsat"]
    end

    subgraph StatusFlow["Pengecekan Status (Kominfo ➔ BPD ➔ Samsat)"]
        S6 --> S7["7. Cek Status Transaksi<br/>(Kominfo Inquiry BPD ➔ Update Samsat)"]
    end

    style S0 fill:#2d1a4e,stroke:#9575cd,stroke-width:2px,color:#ffffff
    style S1 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S2 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S3 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S4 fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style S5 fill:#5c2d00,stroke:#ffb74d,stroke-width:2px,color:#ffffff
    style S6 fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style S7 fill:#4a154b,stroke:#ba68c8,stroke-width:2px,color:#ffffff
```

---

## 4. Rincian Alur & Spesifikasi Interaksi (Sequence Diagrams)

### 4.1. Tahap 1 - 4: Inquiry dan Pemilihan Objek Pajak (Kominfo – Samsat)
User masuk via SSO JSS, NIK terverifikasi diambil langsung oleh server Kominfo untuk mengecek data kendaraan ke Samsat DIY.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak (User)
    participant JSSApp as JSS Client (App / Web)
    participant JSSServer as Kominfo (Backend JSS)
    participant SAM as Samsat DIY

    %% Inisialisasi
    note over WP, JSSServer: User Login via SSO JSS (Profil Terverifikasi)
    WP->>JSSApp: Buka Menu "Pajak Kendaraan (PKB)"
    JSSApp->>JSSServer: Request Layanan Pajak (Token JSS)
    JSSServer->>JSSServer: Ekstrak NIK Terverifikasi dari Akun JSS

    %% Step 1: Get By NIK
    note over JSSServer, SAM: 1. Get Data by NIK (Validasi Pemilik)
    JSSServer->>SAM: GET /api/v1/wajib-pajak?nik={nik_user_jss}
    SAM-->>JSSServer: 200 OK (Nama, Alamat, Status Wajib Pajak)

    %% Step 2: Get List Nopol
    note over JSSServer, SAM: 2. Get List Nopol Terdaftar
    JSSServer->>SAM: GET /api/v1/kendaraan?nik={nik_user_jss}
    SAM-->>JSSServer: 200 OK: List Kendaraan [Nopol, Merk, Tipe, Masa Pajak]
    JSSServer-->>JSSApp: Kirim List Nopol
    JSSApp-->>WP: Tampilkan Daftar Kendaraan Milik User

    %% Step 3: Get List Tahun by Nopol
    note over WP, SAM: 3. Get List Tahun by Nopol
    WP->>JSSApp: Pilih Salah Satu Nopol
    JSSApp->>JSSServer: Request Tagihan per Nopol { nopol }
    JSSServer->>SAM: GET /api/v1/pajak/tahun?nopol={nopol}
    SAM-->>JSSServer: 200 OK: List Tahun Tertunggak/Berjalan [2025, 2026]
    JSSServer-->>JSSApp: Kirim Opsi Tahun Pajak
    JSSApp-->>WP: Tampilkan Pilihan Periode/Tahun Pajak

    %% Step 4: Get Detail Pajak
    note over WP, SAM: 4. Get Detail Pajak untuk Nopol Terpilih
    WP->>JSSApp: Pilih Tahun Pajak & Konfirmasi
    JSSApp->>JSSServer: Request Rincian { nopol, tahun }
    JSSServer->>SAM: GET /api/v1/pajak/detail?nopol={nopol}&tahun={tahun}
    SAM-->>JSSServer: 200 OK: Rincian Pajak (PKB, SWDKLLJ, PNBP, Denda, Total)
    JSSServer-->>JSSApp: Return Rincian Tagihan
    JSSApp-->>WP: Tampilkan Ringkasan Tagihan & Tombol "Bayar QRIS"
```

---

### 4.2. Tahap 5: Create QRIS (Kominfo – BPD DIY Direct)
Pada skema ini, **Kominfo langsung menembak API BPD DIY** untuk pembuatan QRIS Dinamis menggunakan ID Referensi / Tagihan dari Kominfo.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant JSSApp as JSS Client (App / Web)
    participant JSSServer as Kominfo (Backend JSS)
    participant BPD as BPD DIY

    WP->>JSSApp: Klik "Bayar via QRIS"
    JSSApp->>JSSServer: POST /api/v1/transaction/create-qris<br/>Body: { nopol, tahun, nominal, user_id }
    
    JSSServer->>JSSServer: 1. Generate ID Transaksi JSS (trx_jss_id)<br/>2. Set status internal: PENDING
    
    note over JSSServer, BPD: Kominfo Request QRIS Langsung ke BPD DIY
    JSSServer->>BPD: POST /api/v1/qris/generate<br/>Body: { partner_trx_id: trx_jss_id, amount: nominal, expired_minutes: 15 }
    BPD->>BPD: Generate QR String & Payload Standar ASPI/BI
    BPD-->>JSSServer: 200 OK: { qr_string, qr_image_url, bpd_trx_id, expired_at }
    
    JSSServer->>JSSServer: Simpan referensi trx_jss_id <-> bpd_trx_id
    JSSServer-->>JSSApp: Return { qr_string, qr_image_url, expired_at }
    JSSApp-->>WP: Render Kode QRIS Dinamis & Timer Hitung Mundur
```

---

### 4.3. Tahap 6: Payment Execution (BPD ➔ Kominfo ➔ Samsat)
Alur ketika Wajib Pajak membayar QRIS, BPD mengabari Kominfo via webhook, lalu Kominfo melanjutkan instruksi pelunasan ke Samsat DIY.

```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak
    participant BPD as BPD DIY
    participant JSSServer as Kominfo (Backend JSS)
    participant SAM as Samsat DIY
    participant JSSApp as JSS Client (App / Web)

    WP->>BPD: Scan & Bayar QRIS (Mobile Banking / e-Wallet)
    BPD->>BPD: 1. Validasi transaksi & settlement dana masuk<br/>2. Set status: SUCCESS

    %% Callback Step 1: BPD -> Kominfo
    note over BPD, JSSServer: Step 1: BPD Callback ke Kominfo
    BPD->>JSSServer: POST /webhook/bpd/payment-notification<br/>Body: { partner_trx_id, bpd_trx_id, amount, paid_at, ref_number }
    
    JSSServer->>JSSServer: 1. Validasi signature & nominal<br/>2. Update status transaksi JSS: PAID
    JSSServer-->>BPD: HTTP 200 OK: { status: "RECEIVED" }

    %% Step 2: Kominfo -> Samsat
    note over JSSServer, SAM: Step 2: Kominfo Meneruskan Pelunasan ke Samsat
    JSSServer->>SAM: POST /api/v1/pajak/settle-payment<br/>Body: { nopol, tahun, amount, payment_ref: bpd_trx_id, paid_at }
    
    alt Sukses Bukukan di Samsat
        SAM->>SAM: 1. Update Database Pajak jadi LUNAS<br/>2. Terbitkan NTPN & Pengesahan e-SKKP
        SAM-->>JSSServer: 200 OK: { status: "SETTLED", ntpn: "...", skkp_download_url: "..." }
        JSSServer->>JSSServer: Simpan data NTPN & Dokumen e-SKKP
    else Gagal Bukukan di Samsat (e.g. Timeout/DB Lock)
        SAM-->>JSSServer: 500 / 504 Error
        JSSServer->>JSSServer: Masuk Antrean Retry / DLQ (Auto-Retry Settle ke Samsat)
    end

    %% Notifikasi ke User
    JSSServer-->>JSSApp: Push Notifikasi Pembayaran Berhasil
    JSSApp-->>WP: Tampilkan Notifikasi Lunas & Tombol Unduh e-SKKP
```

---

### 4.4. Tahap 7: Pengecekan Status Transaksi (Kominfo ➔ BPD ➔ Samsat)
Mekanisme ketika pengguna menekan tombol "Cek Status" di aplikasi JSS atau auto-polling berjalan saat webhook tertunda.

#### **Logika Bisnis Cek Status:**
1. **Kominfo (JSS)** mengecek terlebih dahulu ke **BPD DIY**: *"Apakah dana untuk QRIS ini sudah masuk/terbayar?"*
2. **Jika BPD menyatakan BELUM BAYAR / EXPIRED**:
   - Kominfo langsung mengembalikan status `PENDING` atau `EXPIRED` ke aplikasi user.
3. **Jika BPD menyatakan SUDAH BAYAR (PAID)**:
   - Kominfo memeriksa apakah data di **Samsat DIY** sudah lunas.
   - Jika Samsat belum lunas (belum ter-settle), Kominfo langsung memicu settlement ke Samsat DIY (`POST /api/v1/pajak/settle-payment`).
   - Samsat membukukan lunas, menerbitkan e-SKKP, dan Kominfo menampilkan status Lunas ke user.

#### **Flowchart Logika Cek Status:**
```mermaid
flowchart TD
    A["Kominfo (JSS) Eksekusi Cek Status"] --> B["Tanya ke BPD DIY:<br/>GET Status Transaksi QRIS"]
    
    B --> C{"Status dari BPD DIY"}
    
    C -- "BELUM BAYAR / EXPIRED" --> D["Kominfo Update Status:<br/>Tampilkan PENDING / EXPIRED ke User"]
    
    C -- "SUDAH BAYAR (PAID)" --> E{"Apakah Samsat<br/>Sudah Lunas?"}
    
    E -- "SUDAH LUNAS" --> F["Kominfo Ambil Dokumen e-SKKP<br/>➔ Tampilkan Status LUNAS"]
    
    E -- "BELUM LUNAS" --> G["Kominfo Panggil API Samsat:<br/>POST /pajak/settle-payment"]
    G --> H["Samsat Update LUNAS & Terbitkan e-SKKP"]
    H --> F

    style A fill:#003366,stroke:#4fc3f7,stroke-width:2px,color:#ffffff
    style B fill:#5c2d00,stroke:#ffb74d,stroke-width:2px,color:#ffffff
    style C fill:#4e3800,stroke:#ffd54f,stroke-width:2px,color:#ffffff
    style D fill:#5a1010,stroke:#ef5350,stroke-width:2px,color:#ffffff
    style E fill:#2d1a4e,stroke:#9575cd,stroke-width:2px,color:#ffffff
    style F fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
    style G fill:#004d40,stroke:#26a69a,stroke-width:2px,color:#ffffff
    style H fill:#0b3d20,stroke:#81c784,stroke-width:2px,color:#ffffff
```

#### **Sequence Diagram Interaksi API Cek Status:**
```mermaid
sequenceDiagram
    autonumber
    actor WP as Wajib Pajak (JSS App)
    participant JSSServer as Kominfo (Backend JSS)
    participant BPD as BPD DIY
    participant SAM as Samsat DIY

    WP->>JSSServer: Klik "Cek Status Pembayaran"
    
    %% Step 1: Kominfo cek ke BPD
    note over JSSServer, BPD: 1. Kominfo Cross-check ke BPD DIY
    JSSServer->>BPD: GET /api/v1/qris/status?partner_trx_id={trx_jss_id}
    
    alt Status BPD: Masih Belum Terbayar / Kadaluarsa
        BPD-->>JSSServer: 200 OK: { status: "UNPAID" / "EXPIRED" }
        JSSServer-->>WP: Tampilkan Info: Menunggu Pembayaran / Waktu Habis
        
    else Status BPD: SUDAH DIBAYAR (PAID)
        BPD-->>JSSServer: 200 OK: { status: "PAID", paid_at: "...", ref_number: "..." }
        
        %% Step 2: Kominfo cek/settle ke Samsat
        note over JSSServer, SAM: 2. Kominfo Sinkronisasi Pelunasan ke Samsat
        JSSServer->>SAM: GET /api/v1/pajak/detail?nopol={nopol}&tahun={tahun}
        
        alt Di Samsat Ternyata Sudah Lunas
            SAM-->>JSSServer: 200 OK: { status: "LUNAS", ntpn: "...", skkp_url: "..." }
        else Di Samsat Belum Tercatat Lunas
            JSSServer->>SAM: POST /api/v1/pajak/settle-payment<br/>Body: { nopol, tahun, amount, payment_ref: ref_number }
            SAM->>SAM: Update Database Lunas & Generate e-SKKP
            SAM-->>JSSServer: 200 OK: { status: "SETTLED", ntpn: "...", skkp_url: "..." }
        end
        
        JSSServer-->>WP: Tampilkan "Pembayaran Berhasil" & Tombol Unduh e-SKKP
    end
```

---

## 5. Ringkasan Desain API & Kontrak Data

| No | Endpoint | Pengirim | Penerima | Fungsi Utama |
|---|---|---|---|---|
| 1 | `GET /api/v1/wajib-pajak?nik={nik}` | Kominfo | Samsat | Verifikasi data kepemilikan NIK JSS |
| 2 | `GET /api/v1/kendaraan?nik={nik}` | Kominfo | Samsat | Mengambil daftar nopol yang terikat pada NIK |
| 3 | `GET /api/v1/pajak/tahun?nopol={nopol}` | Kominfo | Samsat | Mengambil daftar tahun pajak tertunggak/berjalan |
| 4 | `GET /api/v1/pajak/detail?nopol={nopol}&tahun={tahun}` | Kominfo | Samsat | Mengambil rincian kewajiban pajak resmi |
| 5 | `POST /api/v1/qris/generate` | Kominfo | BPD DIY | Permintaan pembuatan dynamic QRIS langsung ke bank |
| 6 | `POST /webhook/bpd/payment-notification` | BPD DIY | Kominfo | Webhook notifikasi pembayaran dari BPD ke Kominfo |
| 7 | `POST /api/v1/pajak/settle-payment` | Kominfo | Samsat | Kominfo membukukan pelunasan & meminta e-SKKP ke Samsat |
| 8 | `GET /api/v1/qris/status?partner_trx_id={id}` | Kominfo | BPD DIY | Inquiry status transaksi perbankan BPD DIY |

---

## 6. Pertimbangan Teknis & Mitigasi Risiko

1. **Kominfo Menjadi Single Point of Coordination**:
   - Karena Kominfo yang menghubungkan BPD dan Samsat, Kominfo **wajib mengimplementasikan mekanisme antrean (*Message Queue / Job Worker*)** untuk penyelesaian ke Samsat.
2. **Penanganan Kasus Gagal Settle ke Samsat**:
   - Jika uang sudah sukses dipotong di BPD DIY tetapi API Samsat sedang mengalami *timeout* atau *maintenance*, Kominfo mencatat status `PAID_PENDING_SETTLEMENT` dan otomatis melakukan retry berkala hingga Samsat mengonfirmasi penerbitan e-SKKP.
3. **Idempotensi Pelunasan di Samsat**:
   - Endpoint `POST /api/v1/pajak/settle-payment` di Samsat wajib bersifat *idempotent* berbasis `payment_ref` atau `bpd_trx_id` agar panggilan berulang tidak menimbulkan pembukuan ganda.

