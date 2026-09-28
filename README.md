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




## 1. ARSITEKTUR & STRUKTUR BERKAS PORTAL

Sistem MSI Atlas dibangun menggunakan arsitektur gabungan **Google Apps Script (GAS)** untuk *backend*, **Firebase (Auth & Firestore)** untuk otentikasi user, serta **Tailwind CSS + HTML5/JS (SPA Router)** untuk *frontend UI*.

### 📂 Modul Backend (Server-Side `.js` / `.gs`)

* `Kode.js` : Main routing `doGet()`, helper `include()`, dan koneksi Spreadsheet.

* `Mastercase.js` : Logic CRUD Mastercase, pembuat folder Drive otomatis, dan inisialisasi milestone.

* `Mastercase_Pdf.js` : Engine pencetak PDF dari Google Docs template dan pengirim lampiran email.

* `Notification.js` : Engine kalkulasi SLA, pembuatan notifikasi in-app, dan penarikan role penerima.

* `EmailService.js` : Engine pengirim email digest harian per PIC & Manager.

* `Profile.js` : Handler upload foto profil ke Google Drive.

* `Settings.js` : Sinkronisasi data PIC ke Sheet `"PIC"`.

### 💻 Modul Frontend & Views (`.html`)

* `Main.html` & `Main_JS.html` : Framework layout utama, SPA router `switchPage()`, header sync, dan export Excel.

* `Dashboard.html` & `Dashboard_JS.html` : Overview KPI cards, grafik Chart.js (Line & Doughnut), dan daftar job terbaru.

* `JobDetail_2.html` & `JobDetail_JS.html` : Register detail pekerjaan, expandable audit trail, dan badge SLA warning.

* `Finance.html` & `Finance_JS.html` : Register keuangan, billing gross, cash-in, outstanding, dan settlement.

* `Mastercase_Create.html`, `Mastercase_Edit.html`, `Mastercase_Detail.html` & `Mastercase_JS.html` : Wizard multi-stage form (Create/Edit) dan halaman tampilan detail job.

* `Setting.html` & `Setting_JS.html` : Halaman profil akun, ganti foto, ganti password, dan statistik user.

* `Auth.html` & `Auth_JS.html` : Halaman login & persitensi sesi Firebase Auth.

* `Profile_SetUp.html` : Setup awal profil saat registrasi pertama kali.

* `FullNotifications.html` : Halaman pusat notifikasi penuh (*Notification Center*).

* `Utils_JS.html` : Helper terpusat format Rupiah, tanggal, dan accessor `getVal()`.





## 2. STRUKTUR FIRESTORE (DATABASE USER & OTENTIKASI)

Firestore digunakan untuk mengelola profil user, role hak akses, dan foto profil.

### 📍 Lokasi Berkas Kode

* **Frontend Authentication & Session Handler:** `Auth_JS.html`

* **Setting & Profil Sync:** `Setting_JS.html`

* **Backend Role Lookup:** `Notification.js`

### 🔧 Petunjuk Modifikasi

1. **Mengubah Konfigurasi Firebase App:**

   * **Berkas:** `Auth_JS.html` & `Setting_JS.html`

   * **Objek:** `firebaseConfig`

   ```
   const firebaseConfig = {
     apiKey: "....",
     authDomain:"....",,
     projectId: "....",,
     storageBucket: "....",,
     messagingSenderId: "....",
     appId: "....",
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





## 3. GOOGLE SPREADSHEET & PEMETAAN KOLOM DATA

Database utama operasional *Mastercase* tersimpan di **Google Sheets**.

### 📍 Lokasi Berkas Kode

* **Backend Apps Script:** `Mastercase.js` & `Kode.js`

* **Master Input Setting:** `Settings.js`

### 🔧 Petunjuk Modifikasi

1. **Mengganti ID Spreadsheet Utama:**

   * **Berkas:** `Kode.js`, `Mastercase.js`, `Settings.js`, `Notification.js`, `EmailService.js`

   * **Perintah:** Ganti parameter string pada `SpreadsheetApp.openById("ID_SPREADSHEET_BARU")`.

2. **Penamaan Tab Sheet:**

   * Sheet Register Pekerjaan: `"Mastercase"`

   * Sheet Master PIC Adjuster: `"PIC"`

   * Sheet Log Notifikasi: `"Notification_Logs"`

3. **Penambahan Kolom Data Baru:**
   Jika ada kolom baru pada Spreadsheet, sesuaikan indeks/pembacaan array pada:

   * `Mastercase.js` (Fungsi `createMastercase()` & `updateMastercase()`)

   * `Utils_JS.html` (Fungsi `getVal(item, possibleKeys, defaultVal)`)







## 4. MANAJEMEN GOOGLE DRIVE & TEMPLATE PDF

Sistem memanfaatkan Google Drive API untuk membuat struktur folder penugasan secara otomatis, menyimpan foto profil, serta membuat salinan berkas PDF.

### 📍 Lokasi Berkas Kode

* **Folder Mastercase & Job Drive:** `Mastercase.js` (Fungsi `getOrCreateMastercaseFolder()`)

* **Folder Foto Profil:** `Profile.js` (Fungsi `uploadProfilePhotoToDrive()`)

* **Template Google Docs / Engine PDF:** `Mastercase_Pdf.js`

### 🔧 Petunjuk Modifikasi

* **Folder Induk Mastercase Drive:**
  Edit `Mastercase.js` pada fungsi `getOrCreateMastercaseFolder()`. Ganti ID folder pada:

  ```
  DriveApp.getFolderById("ID_PARENT_FOLDER_DRIVE_MASTERCASE");
  
  ```

* **Folder Foto Profil User:**
  Edit `Profile.js`. Folder otomatis dibuat/dicari dengan nama `"MSI_Profile_Photos"`.

* **Template Google Docs (PDF Mastercase):**
  Edit `Mastercase_Pdf.js`. Ganti variabel template ID pada:

  ```
  var TEMPLATE_DOC_ID = "ID_GOOGLE_DOCS_TEMPLATE_KAMU";
  
  ```






## 5. SISTEM STATUS, MULTI-STEP STAGE & LIFECYCLE

Alur pekerjaan terbagi dalam 5 Stage Utama yang mengontrol *Job Status*.

### 🔄 Matriks Alur Stage vs Status Otomatis

| Stage # | Nama Stage | Target SLA | Status Job Terkait | 
 | ----- | ----- | ----- | ----- | 
| **Stage 1** | *Instruction & Client Appoint* | Hari H (0 Hari) | `On Process` | 
| **Stage 2** | *Site Survey & Risk Details* | Instruction Date + 3 Hari | `In Site Survey` | 
| **Stage 3** | *Interim & Reporting* | Survey Finish + 5 Hari | `Drafting Report` / `Report Sent` | 
| **Stage 4** | *Fee & Expenses (Adjusting)* | Report Sent + 3 Hari | `Awaiting Invoice` | 
| **Stage 5** | *Invoicing & Settlement* | Invoice Date + 7 Hari | `Invoice Sent` / `Invoice Paid` | 

### 📍 Petunjuk Modifikasi Logic

1. **Default Status Per Stage:**
   Edit `Mastercase_JS.html` pada objek `STAGE_DEFAULT_STATUS`:

   ```
   var STAGE_DEFAULT_STATUS = {
     1: 'On Process',
     2: 'In Site Survey',
     3: 'Drafting Report',
     4: 'Awaiting Invoice',
     5: 'Invoice Sent'
   };
   
   ```

2. **Kalkulasi Status Dinamis dari Tanggal Input:**
   Edit `JobDetail_JS.html` pada fungsi `getCalculatedJobStatus(item)`.

3. **Aturan Pembukaan Stage pada Wizard Form:**
   Edit `Mastercase_JS.html` pada fungsi `calculateMaxStageFromForm()` & `determineMaxUnlockedStage()`.







## 6. ATURAN SLA, WARNING BADGE (H-3/H-2/H-1) & OVERDUE

Sistem secara otomatis melacak batas waktu penuntasan tahap pekerjaan berdasarkan kalkulasi selisih hari dari *target date* ke hari ini ($00:00:00$).

### ⏰ Aturan Perhitungan SLA

1. **Target Survey:** `Instruction Date` $+ 3 \text{ Hari}$

2. **Target Report Issued:** `Survey Finish Date` $+ 5 \text{ Hari}$ (*Di Audit Trail*) / $+ 7 \text{ Hari}$ (*Di Badge Warning*)

3. **Target LOP / Fee Approval:** `Report Issued Date` $+ 3 \text{ Hari}$

4. **Target Settlement:** `Invoice Date` $+ 7 \text{ Hari}$

### 📍 Berkas & Kode Warning/Overdue

1. **Tampilan Warning Badge di Tabel Job Register:**

   * **Berkas:** `JobDetail_JS.html`

   * **Fungsi:** `getJobWarningBadge(item)`

   * **Kondisi Display:**

     * `daysLeft < 0`: Tampil Badge Merah `Overdue X Hari` (animasi pulse).

     * `daysLeft <= 3`: Tampil Badge Kuning `H-X Target`.

2. **Kalkulasi Audit Trail Lifecycle:**

   * **Berkas:** `JobDetail_JS.html`

   * **Fungsi:** `renderExpandAuditTrail(item)`

   * **Variabel Target Date:**

     ```
     var t2Date = isValidDateStr(instDate) ? addDaysToDate(instDate, 3) : '-'; // Survey: 3 Hari
     var t3Date = isValidDateStr(cleanS2Finish) ? addDaysToDate(cleanS2Finish, 5) : '-'; // Report: 5 Hari
     var t4Date = isValidDateStr(cleanS3Finish) ? addDaysToDate(cleanS3Finish, 3) : '-'; // LOP: 3 Hari
     var t5Date = isValidDateStr(cleanS5Start) ? addDaysToDate(cleanS5Start, 7) : '-'; // Invoice: 7 Hari
     
     ```

3. **Backend SLA Notifier Engine:**

   * **Berkas:** `Notification.js` & `EmailService.js`

   * **Fungsi:** `checkAndGenerateNotifications()`







## 7. GENERASI & PENGIRIMAN PDF (`generateMastercasePDF`)

Fungsi ini membaca Google Docs Template, mengganti placeholder `{{tag}}` dengan data dari form, mengonversinya ke bentuk Blob PDF, menyimpannya di Drive, dan mengirimkannya ke email user.

### 📍 Lokasi Berkas Kode

* **Backend Apps Script:** `Mastercase_Pdf.js`

* **Pemicu Frontend:** `Mastercase_JS.html` (Fungsi `printJobToPDF(jobNo)`)

### 🔧 Petunjuk Modifikasi Placeholder PDF

1. Buka berkas `Mastercase_Pdf.js`.

2. Sesuaikan pemetaan `replaceText` pada fungsi `generateMastercasePDF(payload)`:

   ```
   body.replaceText("{{JOB_NO}}", payload.jobNo || "-");
   body.replaceText("{{CLIENT}}", payload.client || "-");
   body.replaceText("{{FEE}}", formatRupiah(payload.fee));
   
   ```

3. Pastikan tag yang tertulis di dalam **Google Docs Template** persis sama (menggunakan kurung kurawal ganda `{{...}}`).






## 8. FITUR EKSPOR EXCEL BERGAYA KUSTOM

Ekspor Excel tidak menggunakan CSV biasa, melainkan library `XLSX-JS-Style` untuk menghasilkan file `.xlsx` lengkap dengan gaya visual.

### 📍 Lokasi Berkas Kode

* **Berkas:** `Main_JS.html`

* **Fungsi Utama:** `exportMastercaseUnified()` & `processExecuteMastercaseExport()`

### 🔧 Petunjuk Modifikasi Excel

* **Mengubah Warna Header Excel (Default Gelap `#1E232A`):**
  Edit pada `Main_JS.html`:

  ```
  worksheet[cell_address].s = {
    fill: { fgColor: { rgb: "1E232A" } }, // Kode Hex Warna tanpa simbol '#'
    font: { name: "Arial", sz: 10, bold: true, color: { rgb: "FFFFFF" } }
  };
  
  ```

* **Menambah/Mengurangi Kolom Ekspor:**
  Edit struktur pemetaan objek `excelRows` di dalam fungsi `processExecuteMastercaseExport()`.







## 9. PUSAT NOTIFIKASI (IN-APP DROPDOWN & EMAIL DIGEST)

Sistem notifikasi terintegrasi dalam dua bentuk: **Dropdown / Full Notification Center** (In-App) dan **Automated Email Digest**.

### 📍 Lokasi Berkas Kode

* **Dropdown Header & Routing SPA:** `Main_JS.html`

* **Tampilan Halaman Full Notification:** `FullNotifications.html`

* **Backend Logging & State Read/Unread:** `Notification.js`

* **Engine Email Digest Harian:** `EmailService.js`

### 🔧 Petunjuk Modifikasi

1. **Mengaktifkan / Mematikan Notifikasi Email Digest:**
   Buka `EmailService.js`, ganti variabel toggle:

   ```
   var ENABLE_EMAIL_NOTIFICATIONS = true; // Set ke false untuk mematikan pengiriman email
   
   ```

2. **Pengambilan Email Penerima:**
   Penerima notifikasi (*PIC Adjuster* & *Manager*) ditarik secara otomatis dari **Firestore Collection `users`** berdasarkan kueri role/picCode. Jika tidak ditemukan, sistem melakukan *fallback* pencarian ke tab Spreadsheet `"PIC"`.




   

## 10. PANDUAN DEPLOYMENT APPS SCRIPT

Setiap kali terjadi perubahan kode pada berkas `.js` / `.gs` maupun `.html`:

1. Buka Google Apps Script Editor.

2. Klik tombol **Deploy** di pojok kanan atas > pilih **New Deployment**.

3. Pilih Jenis Deployment: **Web app**.

4. Set Konfigurasi:

   * **Execute as:** *Me (Email Pemilik Skrip)*

   * **Who has access:** *Anyone* (Agar dapat diakses portal web).

5. Klik **Deploy** dan salin URL Web App yang baru jika ada pembaruan URL.

*© 2026 PT Atlas Adjusting Indonesia — MSI Atlas Adjusting System.*
