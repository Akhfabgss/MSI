# 📘 TECHNICAL README & SYSTEM MAINTENANCE GUIDE

**MSI Atlas Adjusting System (Loss Adjusting & Claims Management)**

Dokumen petunjuk teknis dan pemetaan berkas kode ini disusun sebagai panduan pengembang (*developer guide*) maupun administrator sistem saat melakukan modifikasi, kustomisasi logika, konfigurasi database, pengelolaan folder Google Drive, serta penyesuaian aturan SLA/Status pada sistem **MSI Atlas Adjusting**.

## 📑 DAFTAR ISI

 1. [Arsitektur & Struktur Berkas Portal](#1-arsitektur--struktur-berkas-portal)

 2. [Struktur Firestore (Database User & Otentikasi)](#2-struktur-firestore-database-user--otentikasi)

 3. [Google Spreadsheet & Pemetaan Kolom Data](#3-google-spreadsheet--pemetaan-kolom-data)

 4. [Manajemen Google Drive & Template PDF](#4-manajemen-google-drive--template-pdf)

 5. [Sistem Status, Multi-Step Stage & Lifecycle](#5-sistem-status-multi-step-stage--lifecycle)

 6. [Aturan SLA, Warning Badge (H-3/H-2/H-1) & Overdue](#6-aturan-sla-warning-badge-h-3h-2h-1--overdue)

 7. [Generasi & Pengiriman PDF (`generateMastercasePDF`)](#7-generasi--pengiriman-pdf-generatemastercasepdf)

 8. [Fitur Ekspor Excel Bergaya Kustom](#8-fitur-ekspor-excel-bergaya-kustom)

 9. [Pusat Notifikasi (In-App Dropdown & Email Digest)](#9-pusat-notifikasi-in-app-dropdown--email-digest)

10. [Panduan Deployment Apps Script](#10-panduan-deployment-apps-script)

---

## 1. ARSITEKTUR & STRUKTUR BERKAS PORTAL

Sistem MSI Atlas dibangun menggunakan arsitektur gabungan **Google Apps Script (GAS)** untuk *backend*, **Firebase (Auth & Firestore)** untuk otentikasi user, serta **Tailwind CSS + HTML5/JS (SPA Router)** untuk *frontend UI*.

Setelah proses refactoring, struktur direktori proyek terisolasi secara modular sesuai *Single Responsibility Principle*:

```text
appscript/
├── .clasp.json
├── appsscript.json
├── backend/
│   ├── Config.js                  // Konstanta terpusat (Spreadsheet ID, Drive Folder, SLA Limits, Enums)
│   ├── Kode.js                    // Entry point doGet() & helper include()
│   ├── Mastercase_Crud.js         // CRUD Mastercase Sheet, Milestones Sync, Folder Generator
│   ├── Mastercase_Pdf.js          // Docs Template Processor, PDF Generator & Email Attachment
│   ├── Notification_SLA.js        // SLA Engine, In-App Notifications Sheet & Read Status Management
│   ├── Email_Service.js           // Daily Digest Cronjob, HTML Email Templating & Recipient Routing
│   └── User_Profile.js            // Upload Foto Profil ke Google Drive & Sync PIC Table
├── views/
│   ├── Main.html                  // Layout Induk SPA & Scriptlet Include Manager
│   ├── Auth.html                  // Halaman Autentikasi Login & Reset Password
│   ├── Profile_SetUp.html         // Form Setup Profil Pengguna Pertama Kali
│   ├── Dashboard.html             // Tampilan Ringkasan Overview & Chart
│   ├── JobDetail.html             // Tampilan Utama Manajemen Pekerjaan & Lifecycle
│   ├── Finance.html               // Tampilan Billing, Faktur & Rekapitulasi Keuangan
│   ├── Mastercase_Create.html     // Multi-Step Form Registrasi Kasus Baru
│   ├── Mastercase_Edit.html       // Multi-Step Form Pembaruan Kasus
│   ├── Mastercase_Detail.html     // Modal Tampilan Detail Ringkas Pekerjaan
│   ├── Setting.html               // Halaman Pengaturan Akun Pengguna
│   └── FullNotifications.html     // Halaman Pusat Notifikasi & Filter
└── scripts/
    ├── Utils_JS.html              // Helper Terpusat (Format Rupiah, Parsing Tanggal, Ekstraksi Nilai)
    ├── Main_JS.html               // Router SPA switchPage(), Toggle Sidebar, Export Excel Utama
    ├── Notification_JS.html       // Dropdown Header Notifikasi, Filter Notifikasi & Handler Klik
    ├── JobDetail_Lifecycle_JS.html// Kalkulasi Badge SLA/Overdue, Audit Trail & Pipeline Stepper
    ├── JobDetail_Table_JS.html    // Data Table Job Detail, Pagination, Filter Tanggal & Status
    ├── Finance_JS.html            // KPI Keuangan, Tabel Billing, Expected Net Payment & Export Excel
    ├── Dashboard_JS.html          // KPI Overview, Chart.js Visualisasi Tren & Distribusi LOB
    ├── Mastercase_Wizard_JS.html  // Multi-step Stepper Form (Stage 1-5) & Logika Navigasi
    ├── Mastercase_Form_JS.html    // Handler Submit Form Create/Edit, Auto-Formula & Cetak PDF
    ├── Setting_JS.html            // Sinkronisasi Firestore Profil, Preview Foto & Statistik Proyek
    └── Auth_JS.html               // Firebase Client Auth, Session Persistence & Auth State Router
```

---

## 2. STRUKTUR FIRESTORE (DATABASE USER & OTENTIKASI)

Firestore digunakan untuk mengelola profil user, role hak akses, dan foto profil.

### 📍 Lokasi Berkas Kode

* **Frontend Authentication & Session Handler:** `scripts/Auth_JS.html`
* **Setting & Profil Sync:** `scripts/Setting_JS.html`
* **Backend Role Lookup:** `backend/Notification_SLA.js`

### 🔧 Petunjuk Modifikasi

1. **Mengubah Konfigurasi Firebase App:**
   * **Berkas:** `scripts/Auth_JS.html` & `scripts/Setting_JS.html`
   * **Objek:** `firebaseConfig`
   ```javascript
   const firebaseConfig = {
     apiKey: "...",
     authDomain: "...",
     projectId: "...",
     storageBucket: "...",
     messagingSenderId: "...",
     appId: "..."
   };
   ```

2. **Skema Koleksi Firestore (`users`):**
   * **Nama Collection:** `users`
   * **Document ID:** `user.uid`
   * **Atribut Field:**
     * `name` (String): Nama lengkap pengguna.
     * `phone` (String): Nomor telepon/WhatsApp.
     * `picCode` (String): Kode inisial PIC Adjuster (contoh: `"AZ"`, `"RM"`) yang terhubung ke Sheet `"PIC"`.
     * `role` (String): Level hak akses (`"Adjuster"`, `"Manager"`, `"Finance"`, `"Admin"`).
     * `photoURL` (String): URL publik foto profil dari Google Drive (dilengkapi query cache buster `?t=timestamp`).

---

## 3. GOOGLE SPREADSHEET & PEMETAAN KOLOM DATA

Database utama operasional *Mastercase* tersimpan di **Google Sheets**.

### 📍 Lokasi Berkas Kode

* **Konstanta Terpusat ID:** `backend/Config.js`
* **Backend Operasi CRUD:** `backend/Mastercase_Crud.js` & `backend/Kode.js`
* **Sync Master PIC:** `backend/User_Profile.js`

### 🔧 Petunjuk Modifikasi

1. **Mengganti ID Spreadsheet Utama:**
   * **Berkas:** `backend/Config.js`
   * **Perintah:** Ganti nilai variabel `SPREADSHEET_ID = "ID_SPREADSHEET_BARU"`.

2. **Penamaan Tab Sheet:**
   * Sheet Register Pekerjaan: `"Mastercase"`
   * Sheet Master PIC Adjuster: `"PIC"`
   * Sheet Log Notifikasi: `"Notification_Logs"`

3. **Penambahan Kolom Data Baru:**
   Jika ada kolom baru pada Spreadsheet, sesuaikan indeks/pembacaan array pada:
   * `backend/Mastercase_Crud.js` (Fungsi `createMastercase()` & `updateMastercase()`)
   * `scripts/Utils_JS.html` (Fungsi `getVal(item, possibleKeys, defaultVal)`)

---

## 4. MANAJEMEN GOOGLE DRIVE & TEMPLATE PDF

Sistem memanfaatkan Google Drive API untuk membuat struktur folder penugasan secara otomatis, menyimpan foto profil, serta membuat salinan berkas PDF.

### 📍 Lokasi Berkas Kode

* **Folder Mastercase & Job Drive:** `backend/Mastercase_Crud.js` (Fungsi `getOrCreateMastercaseFolder()`)
* **Folder Foto Profil:** `backend/User_Profile.js` (Fungsi `uploadProfilePhotoToDrive()`)
* **Template Google Docs / Engine PDF:** `backend/Mastercase_Pdf.js`

### 🔧 Petunjuk Modifikasi

* **Folder Induk Mastercase Drive:**
  Edit `backend/Config.js` pada variabel `PARENT_DRIVE_FOLDER_ID`.
* **Folder Foto Profil User:**
  Edit `backend/Config.js` pada variabel `PROFILE_PHOTOS_FOLDER_NAME` (Default: `"MSI_Profile_Photos"`).
* **Template Google Docs (PDF Mastercase):**
  Edit `backend/Config.js` pada variabel `TEMPLATE_DOC_ID`.

---

## 5. SISTEM STATUS, MULTI-STEP STAGE & LIFECYCLE

Alur pekerjaan terbagi dalam 5 Stage Utama yang mengontrol *Job Status*.

### 🔄 Matriks Alur Stage vs Status Otomatis

| Stage # | Nama Stage | Target SLA | Status Job Terkait | 
| :--- | :--- | :--- | :--- | 
| **Stage 1** | *Instruction & Client Appoint* | Hari H (0 Hari) | `On Process` | 
| **Stage 2** | *Site Survey & Risk Details* | Instruction Date + 3 Hari | `In Site Survey` | 
| **Stage 3** | *Interim & Reporting* | Survey Finish + 5 Hari | `Drafting Report` / `Report Sent` | 
| **Stage 4** | *Fee & Expenses (Adjusting)* | Report Sent + 3 Hari | `Awaiting Invoice` | 
| **Stage 5** | *Invoicing & Settlement* | Invoice Date + 7 Hari | `Invoice Sent` / `Invoice Paid` | 

### 🔧 Petunjuk Mengubah Target Hari SLA

Pengaturan target hari SLA terdapat pada **Backend** dan **Frontend**. Jika durasi SLA diubah, pastikan kedua bagian diperbarui agar perhitungan sistem tetap konsisten.

#### 1. Backend
File:
`backend/Config.js`

Ubah nilai pada `SLA_LIMITS`:

```javascript
var SLA_LIMITS = {
  SURVEY_DAYS: 3,
  REPORT_DAYS: 5,
  LOP_DAYS: 3,
  SETTLEMENT_DAYS: 7
};
```

#### 2. Frontend
Pada fungsi renderExpandAuditTrail(item), sesuaikan nilai hari pada t1Date sampai t5Date agar target tanggal pada Lifecycle Audit Trail sesuai dengan konfigurasi backend.

File:
scripts/JobDetail_Lifecycle_JS.html

```javascript
var t2Date = isValidDateStr(instDate) ? addDaysToDate(instDate, 3) : '-';
var t3Date = isValidDateStr(cleanS2Finish) ? addDaysToDate(cleanS2Finish, 5) : '-';
var t4Date = isValidDateStr(cleanS3Finish) ? addDaysToDate(cleanS3Finish, 3) : '-';
var t5Date = isValidDateStr(cleanS5Start) ? addDaysToDate(cleanS5Start, 7) : '-';
```

### 📍 Petunjuk Modifikasi Logic

1. **Default Status Per Stage:**
   Edit `scripts/Mastercase_Form_JS.html` pada objek `STAGE_DEFAULT_STATUS`:
   ```javascript
   var STAGE_DEFAULT_STATUS = {
     1: 'On Process',
     2: 'In Site Survey',
     3: 'Drafting Report',
     4: 'Awaiting Invoice',
     5: 'Invoice Sent'
   };
   ```

2. **Kalkulasi Status Dinamis dari Tanggal Input:**
   Edit `scripts/JobDetail_Lifecycle_JS.html` pada fungsi `getCalculatedJobStatus(item)`.

3. **Aturan Pembukaan Stage pada Wizard Form:**
   Edit `scripts/Mastercase_Wizard_JS.html` pada fungsi `calculateMaxStageFromForm()` & `determineMaxUnlockedStage()`.

---

## 6. ATURAN SLA, WARNING BADGE (H-3/H-2/H-1) & OVERDUE

Sistem secara otomatis melacak batas waktu penuntasan tahap pekerjaan berdasarkan kalkulasi selisih hari dari *target date* ke hari ini ($00:00:00$).

### ⏰ Aturan Perhitungan SLA

1. **Target Survey:** `Instruction Date` $+ 3 \text{ Hari}$
2. **Target Report Issued:** `Survey Finish Date` $+ 5 \text{ Hari}$ (*Di Audit Trail*) / $+ 7 \text{ Hari}$ (*Di Badge Warning*)
3. **Target LOP / Fee Approval:** `Report Issued Date` $+ 3 \text{ Hari}$
4. **Target Settlement:** `Invoice Date` $+ 7 \text{ Hari}$

### 📍 Berkas & Kode Warning/Overdue

1. **Tampilan Warning Badge & Lifecycle Audit Trail:**
   * **Berkas:** `scripts/JobDetail_Lifecycle_JS.html`
   * **Fungsi:** `getJobWarningBadge(item)` & `renderExpandAuditTrail(item)`
   * **Kondisi Display Badge:**
     * `daysLeft < 0`: Tampil Badge Merah `Overdue X Hari` (animasi pulse).
     * `daysLeft <= 3`: Tampil Badge Kuning `H-X Target`.

2. **Backend SLA Notifier Engine:**
   * **Berkas:** `backend/Notification_SLA.js` & `backend/Email_Service.js`
   * **Fungsi:** `checkAndGenerateNotifications()`

---

## 7. GENERASI & PENGIRIMAN PDF (`generateMastercasePDF`)

Fungsi ini membaca Google Docs Template, mengganti placeholder `{{tag}}` dengan data dari form, mengonversinya ke bentuk Blob PDF, menyimpannya di Drive, dan mengirimkannya ke email user.

### 📍 Lokasi Berkas Kode

* **Backend Apps Script:** `backend/Mastercase_Pdf.js`
* **Pemicu Frontend:** `scripts/Mastercase_Form_JS.html` (Fungsi `printJobToPDF(jobNo)`)

### 🔧 Petunjuk Modifikasi Placeholder PDF

1. Buka berkas `backend/Mastercase_Pdf.js`.
2. Sesuaikan pemetaan `replaceText` pada fungsi `generateMastercasePDF(payload)`:
   ```javascript
   body.replaceText("{{JOB_NO}}", payload.jobNo || "-");
   body.replaceText("{{CLIENT}}", payload.client || "-");
   body.replaceText("{{FEE}}", formatRupiah(payload.fee));
   ```
3. Pastikan tag yang tertulis di dalam **Google Docs Template** persis sama (menggunakan kurung kurawal ganda `{{...}}`).

---

## 8. FITUR EKSPOR EXCEL BERGAYA KUSTOM

Ekspor Excel menggunakan library `XLSX-JS-Style` untuk menghasilkan file `.xlsx` lengkap dengan gaya visual.

### 📍 Lokasi Berkas Kode

* **Berkas:** `scripts/Main_JS.html` & `scripts/Finance_JS.html`
* **Fungsi Utama:** `exportMastercaseUnified()`, `processExecuteMastercaseExport()`, & `exportFinanceToExcel()`

### 🔧 Petunjuk Modifikasi Excel

* **Mengubah Warna Header Excel (Default Gelap `#1E232A`):**
  Edit pada `scripts/Main_JS.html`:
  ```javascript
  worksheet[cell_address].s = {
    fill: { fgColor: { rgb: "1E232A" } }, // Kode Hex Warna tanpa simbol '#'
    font: { name: "Arial", sz: 10, bold: true, color: { rgb: "FFFFFF" } }
  };
  ```
* **Menambah/Mengurangi Kolom Ekspor:**
  Edit struktur pemetaan objek `excelRows` di dalam fungsi `processExecuteMastercaseExport()`.

---

## 9. PUSAT NOTIFIKASI (IN-APP DROPDOWN & EMAIL DIGEST)

Sistem notifikasi terintegrasi dalam dua bentuk: **Dropdown / Full Notification Center** (In-App) dan **Automated Email Digest**.

### 📍 Lokasi Berkas Kode

* **Dropdown Header & Routing UI:** `scripts/Notification_JS.html`
* **Tampilan Halaman Full Notification:** `views/FullNotifications.html`
* **Backend Logging & State Read/Unread:** `backend/Notification_SLA.js`
* **Engine Email Digest Harian:** `backend/Email_Service.js`

### 🔧 Petunjuk Modifikasi

1. **Mengaktifkan / Mematikan Notifikasi Email Digest:**
   Buka `backend/Config.js` atau `backend/Email_Service.js`, ganti variabel toggle:
   ```javascript
   var ENABLE_EMAIL_NOTIFICATIONS = true; // Set ke false untuk mematikan pengiriman email
   ```

2. **Pengambilan Email Penerima:**
   Penerima notifikasi (*PIC Adjuster* & *Manager*) ditarik secara otomatis dari **Firestore Collection `users`** berdasarkan kueri role/picCode. Jika tidak ditemukan, sistem melakukan *fallback* pencarian ke tab Spreadsheet `"PIC"`.

---

## 10. PANDUAN DEPLOYMENT APPS SCRIPT

Setiap kali terjadi perubahan kode pada berkas di direktori `backend/`, `views/`, maupun `scripts/`:

### 🛠️ Opsi A: Deployment via Clasp CLI (Direkomendasikan)

1. Jalankan perintah push dari terminal proyek:
   ```bash
   clasp push
   ```
2. Rilis deployment baru:
   ```bash
   clasp deploy --description "Pembaruan Fitur Modular"
   ```

### 🌐 Opsi B: Deployment Manual via Apps Script Editor

1. Buka Google Apps Script Editor.
2. Klik tombol **Deploy** di pojok kanan atas > pilih **New Deployment**.
3. Pilih Jenis Deployment: **Web app**.
4. Set Konfigurasi:
   * **Execute as:** *Me (Email Pemilik Skrip)*
   * **Who has access:** *Anyone* (Agar dapat diakses portal web).
5. Klik **Deploy** dan salin URL Web App yang baru jika ada pembaruan URL.

---

*© 2026 PT Atlas Adjusting Indonesia — MSI Atlas Adjusting System.*
