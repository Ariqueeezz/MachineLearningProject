<div align="center">

# Otomatisasi Penyusunan Dokumen RPHUA Terintegrasi JIRA (N8N + Google Apps Script)

</div>

## 1. Judul dan Deskripsi Proyek

**Otomatisasi Penyusunan Dokumen Rencana, Pelaksanaan, dan Hasil Uji Aplikasi (RPHUA) Terintegrasi JIRA menggunakan N8N dan Google Apps Script**

Workflow ini (bernama `RPHUAxN8N`) dipicu secara otomatis melalui webhook JIRA setiap kali status sebuah tiket berubah dari **"UAT"** menjadi **"Ready for Released"**. Begitu terpicu, sistem akan:

1. Mengambil data skenario pengujian dari Google Sheets yang tertaut pada tiket JIRA (melalui custom field).
2. Mendeteksi format spreadsheet secara otomatis (profile detection) dan memetakan datanya ke struktur standar.
3. Menentukan template dokumen RPHUA yang sesuai berdasarkan epic induk tiket.
4. Menduplikasi template ke Google Drive dan mengisinya (tabel identifikasi, tabel deskripsi, serta screenshot bukti pengujian dari lampiran JIRA) melalui Google Apps Script.
5. Mengunggah kembali tautan dokumen RPHUA yang telah jadi sebagai *remote link* ke tiket JIRA asal, lengkap dengan mekanisme pencegahan duplikasi tautan.

Dengan alur ini, waktu penyusunan dokumen RPHUA yang semula membutuhkan peninjauan rekaman UAT selama 1–8 jam dapat dipangkas menjadi rata-rata **±33,8 detik per dokumen** (hasil pengujian 12 eksekusi, tingkat keberhasilan 100%).

## 2. Fitur Utama (Features)

* Pemicu otomatis (trigger) berbasis **webhook JIRA** saat status tiket berubah menjadi "Ready for Released".
* Integrasi dengan **Google Sheets API** untuk pengambilan data skenario pengujian secara dinamis (ID spreadsheet diekstrak langsung dari URL pada custom field JIRA).
* **Deteksi profil spreadsheet otomatis** (mendukung format "Antrian Online" dan "JMO Skenario") beserta mekanisme *remap header* untuk format kolom yang tidak terbaca standar.
* **Ekstraksi multi-blok** skenario pengujian (blok JMO dan blok SMILE) dalam satu baris spreadsheet.
* **Routing template dokumen otomatis** berdasarkan nama epic induk tiket JIRA.
* Duplikasi template & pengisian dokumen **Google Docs** secara programatik (tabel identifikasi, tabel deskripsi, dan penyisipan gambar bukti pengujian) via **Google Apps Script**.
* Pemformatan border tabel otomatis melalui **Google Docs REST API** (`batchUpdate`).
* Pengunggahan tautan dokumen RPHUA sebagai **remote link JIRA** melalui JIRA REST API, dengan pengecekan agar tautan tidak terunggah dobel (duplicate check).

## 3. Prasyarat (Prerequisites)

* **n8n** dijalankan melalui Docker (image `docker.n8n.io/n8nio/n8n`), dengan mode tunnel bawaan aktif.
* **Docker** terpasang di mesin/server (workflow ini diuji di atas Ubuntu 22.04 LTS).
* **Ngrok** (terintegrasi otomatis lewat opsi `start --tunnel` pada n8n) untuk membuat endpoint webhook publik yang dapat diakses oleh JIRA Cloud.
* Akun **JIRA Cloud** dengan izin membuat webhook serta akses ke JIRA REST API v3 (endpoint issue & remotelink).
* Akun **Google Workspace** dengan akses ke:
  * Google Sheets API (baca data skenario pengujian).
  * Google Drive API (duplikasi template dokumen).
  * Google Docs API / Google Apps Script (pengisian dan pemformatan dokumen).
* **Google Apps Script** yang sudah di-deploy sebagai *Web App* (menerima request `doPost`) dan dapat diakses oleh n8n.
* Template dokumen **RPHUA** di Google Docs (satu atau lebih, tergantung jumlah tim/epic yang dilayani).

## 4. Variabel Lingkungan & Kredensial (Environment Variables / Credentials)

> ⚠️ Jangan pernah memasukkan token, secret, atau kredensial asli ke dalam repositori ini. Seluruh kredensial dikonfigurasi langsung melalui menu **Credentials** di n8n.

| Nama | Tipe / Digunakan Pada | Keterangan |
|---|---|---|
| `jiraSoftwareCloudApi` | Kredensial n8n (JIRA) | Autentikasi ke JIRA REST API, dipakai pada node *Get Remote Link JIRA* dan *Paste Link*. Opsi `allowUnauthorizedCerts: true` diaktifkan untuk mengakomodasi jaringan internal. |
| `googleDocsOAuth2Api` | Kredensial n8n (Google) | Autentikasi OAuth2 untuk memanggil endpoint Google Apps Script (node *Post Content*) serta node Google Sheets/Drive bawaan n8n. |
| `JIRA_API_TOKEN` (dipakai di sisi GAS) | Google Apps Script | Digunakan pada `UrlFetchApp.fetch()` dengan skema Basic Auth untuk mengunduh screenshot bukti pengujian dari lampiran JIRA. |
| `WEBHOOK_PATH` | Node Webhook n8n | Path endpoint webhook, contoh: `rphua-trigger` → URL publik menjadi `https://<subdomain>.ngrok-free.app/webhook/rphua-trigger`. Path ini yang didaftarkan sebagai webhook di pengaturan JIRA Cloud. |
| `DRIVE_FOLDER_ID` | Node *Copy Template RPHUA* | ID folder Google Drive tujuan hasil duplikasi template dokumen. |
| `TEMPLATE_ID_JMO` / `TEMPLATE_ID_DEFAULT` | Node *Code in JavaScript* (routing template) | ID Google Docs template RPHUA yang dipilih otomatis berdasarkan nama epic induk tiket (misalnya template khusus tim JMO vs. template Antrian Online/lainnya). |
| `GAS_WEB_APP_URL` | Node *Post Content* (HTTP Request) | URL endpoint Web App hasil deploy Google Apps Script yang menerima payload `docId` dan `payload` melalui `doPost`. |

## 5. Cara Instalasi dan Impor (Setup & Installation)

1. **Jalankan n8n via Docker** dengan volume persisten agar workflow dan kredensial tidak hilang saat container di-restart:
   ```bash
   docker run -it --rm --name n8n -p 5678:5678 \
     -v n8n_data:/home/node/.n8n \
     docker.n8n.io/n8nio/n8n start --tunnel
   ```
   Opsi `start --tunnel` akan otomatis membuat URL publik HTTPS (via ngrok) yang meneruskan permintaan ke n8n di `localhost:5678`.
2. Buka dashboard n8n di `http://localhost:5678`.
3. Buat workflow baru, klik menu di kanan atas, lalu pilih **Import from File**.
4. Pilih file `workflow.json` (ekspor workflow `RPHUAxN8N`) yang sudah diunduh dari repositori ini.
5. Hubungkan ulang kredensial pada node yang bertanda seru/eror, minimal:
   * `jiraSoftwareCloudApi` pada node *Get Remote Link JIRA* dan *Paste Link*.
   * `googleDocsOAuth2Api` pada node *Get Sheet Metadata*, *Get row(s) in sheet*, *Copy Template RPHUA*, dan *Post Content*.
6. Deploy skrip **Google Apps Script** sebagai Web App, salin URL-nya ke node *Post Content* (HTTP Request).
7. Salin URL publik ngrok yang dihasilkan n8n, lalu daftarkan sebagai webhook di pengaturan JIRA Cloud dengan path `/webhook/rphua-trigger`, dipicu pada event perubahan status tiket menjadi "Ready for Released".
8. Aktifkan (Activate) workflow di n8n, lalu lakukan uji coba dengan mengubah status salah satu tiket JIRA untuk memverifikasi seluruh alur berjalan hingga tautan dokumen RPHUA muncul di tiket tersebut.

---

*README ini disusun berdasarkan Laporan Kerja Praktik "Otomatisasi Penyusunan Dokumen Rencana, Pelaksanaan, dan Hasil Uji Aplikasi (RPHUA) Terintegrasi JIRA Menggunakan N8N dan Google Apps Script" — Raihan Ariq Muzakki, Program Studi Informatika, Universitas Bhayangkara Jakarta Raya.*
