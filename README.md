# 📊 Automation Live Expense Dashboard (Local Setup)

Project ini adalah sistem otomatis untuk mencatat, membersihkan, menyimpan, dan menampilkan data pengeluaran secara **real-time** menggunakan:

- 🐳 Docker (untuk menjalankan n8n secara lokal)
- ⚙️ n8n (automation workflow)
- 📄 Google Sheets (tempat input data)
- 🗄 Supabase PostgreSQL (database)
- 📊 Power BI (dashboard live dengan DirectQuery)

Semua berjalan otomatis dari input → cleaning → database → dashboard.

---

# 🎯 Tujuan Project Ini

Project ini dibuat supaya:

- Kamu cukup input pengeluaran di Google Sheets
- Data otomatis dibersihkan & divalidasi
- Data yang sudah bersih masuk ke database
- Dashboard Power BI update otomatis
- Tidak perlu input manual ke database
- Tidak ada data kotor yang masuk sistem

Singkatnya: ini adalah mini data pipeline profesional untuk personal finance.

---

# 🏗️ Gambaran Alur Sistem

Google Sheets (Data mentah)
↓
n8n (Cleaning + Validasi + Transform)
↓
Google Sheets (Cleaned_Data)
↓
Supabase PostgreSQL
↓
Power BI (Dashboard Live)


---

# 🐳 STEP 1 — Install & Jalankan n8n dengan Docker

## 1️⃣ Install Docker Desktop

Download di:
https://www.docker.com/products/docker-desktop

Install sampai selesai dan pastikan Docker dalam keadaan running.

---

## 2️⃣ Download Image n8n

1. Buka Docker Desktop
2. Masuk ke menu **Images**
3. Pada tab Search ketik:

n8nio/n8n


4. Klik **Pull**
5. Tunggu sampai selesai

---

## 3️⃣ Jalankan n8n

Klik tombol ▶ Run

Isi konfigurasi berikut:

- **Container Name:** `n8n-expense`
- **Port:** `5678`
- **Host Path:** pilih folder lokal (agar data tidak hilang saat restart)

Klik **Run**

Setelah container berjalan:

Klik link:
5678:5678


Buka di browser:
http://localhost:5678


Lakukan login awal dan selesai.

Sekarang n8n sudah siap dipakai.

---

# 📄 STEP 2 — Buat Google Spreadsheet
Akses Spreadsheet yang sudah dibuat: https://docs.google.com/spreadsheets/d/1aGhMRQGWnxdecq0F4E7g_4-REdpb-CHO7VTnBwA82iA/edit?usp=sharing
Buat spreadsheet baru.

Buat 2 sheet:

- `Data`
- `Cleaned_Data`

---

## Struktur Kolom (WAJIB di baris pertama)

| id | Date | Category | Description | Amount | Payment_Method | Mood |

### Penjelasan Kolom

- **id** → nomor unik setiap transaksi
- **Date** → tanggal transaksi
- **Category** → food, transport, entertainment, dll
- **Description** → keterangan transaksi
- **Amount** → jumlah uang
- **Payment_Method** → qris / cash / debit / e-wallet
- **Mood** → happy / tired / calm / bad / dll

Rename spreadsheet supaya tidak “Untitled Spreadsheet”.

---

# ☁️ STEP 3 — Setup Google Cloud (Supaya n8n Bisa Akses Spreadsheet)

## 1️⃣ Buat Project Baru

1. Login Google Cloud Console
2. Klik **New Project**
3. Isi nama project
4. Parent Resource → No Organization
5. Klik Create

---

## 2️⃣ Aktifkan API

Masuk ke:

APIs & Services → Enable APIs


Aktifkan:

- Google Sheets API
- Google Drive API

Ini penting supaya n8n bisa baca file spreadsheet kamu.

---

## 3️⃣ Buat OAuth Credential

Masuk:

APIs & Services → Credentials


Klik:
Create Credentials → OAuth Client ID


Pilih:
Web Application


Klik Create.

---

## 4️⃣ Tambahkan Test User

Masuk ke menu:
Audience → Test Users


Tambahkan email Google kamu.

Ini supaya akun kamu boleh mengakses API.

---

# 🔐 STEP 4 — Hubungkan Google Sheets ke n8n

## Buat Workflow Baru

Nama:
Expense Dashboard


---

## 1️⃣ Tambahkan Schedule Trigger

Tambahkan node:
Schedule Trigger


Atur seperti ini:

- Interval: Days
- Days Between Triggers: 1
- Hour: Midnight
- Minute: 0

Artinya workflow jalan otomatis 1x sehari jam 00:00.

---

## 2️⃣ Tambahkan Node Get Row(s)

Tambah node:
Get Row(s) in Sheets


Klik **Create New Credential**

Salin URL dari n8n:
http://localhost:5678/rest/oauth2-credential/callback


Masuk Google Cloud → Credentials → OAuth Client

Tempel URL tadi ke:
Authorized Redirect URLs


Salin:
- Client ID
- Client Secret

Paste ke n8n.

Klik:
Sign in with Google


Pilih akun yang tadi dimasukkan ke Test User.

Berikan izin → selesai.

---

Konfigurasi node:

- Resource → Sheet Within Document
- Operation → Get Row(s)
- Document → pilih spreadsheet
- Sheet → `Data`

---

# 🧹 STEP 5 — Cleaning Data

## Tambahkan Node Edit Fields (Set)

Rename jadi:
Cleaning Format


Mode:
Manual Mapping


Isi mapping:

Date = {{ $json.Date.toDateTime() }}
Category = {{ $json.Category.trim().toLowerCase() }}
Description = {{ $json.Description.trim().toLowerCase() }}
Amount = {{ parseInt($json.Amount) }}
Payment_Method = {{ $json["Payment_Method"].trim().toLowerCase() }}
Mood = {{ $json.Mood ? $json.Mood.trim().toLowerCase() : "unknown" }}


Aktifkan:
- Include Other Input Fields → ON
- All Except → row_number

Tujuannya:
- Hilangkan spasi
- Samakan huruf kecil
- Pastikan amount angka
- Mood kosong jadi "unknown"

---

## Update Row - Steps 1

Tambahkan node:
Update Row


Sheet:
Data


Match:
id


Ini untuk update format hasil cleaning.

---

## Convert Date ke Format YYYY-MM-DD

Tambah node Edit Fields lagi.

Rename:
Convert DateTime to YYYY-MM-DD


Isi:

Date = {{ new Date($json.Date).toISOString().split('T')[0] }}


Tujuannya supaya database bisa membaca format date dengan benar.

---

## Update Row - Steps 2

Copy node Update Row sebelumnya.

---

# 🔍 STEP 6 — Filter Data Supaya Bersih

## Filter 1 — Amount > 0

Condition:
{{ $json.Amount.toNumber() }} > 0


---

## Filter 2 — Tidak Kosong

Pastikan:

- Category tidak kosong
- Description tidak kosong
- Payment_Method tidak kosong

---

## Filter 3 — Mood bukan "unknown"

Condition:
{{ $json.Mood }} does not contain unknown


---

# 📥 STEP 7 — Masukkan ke Cleaned_Data

Node:
Append or Update Row


Rename:
Append to Cleaned_Data sheets


Sheet:
Cleaned_Data


Mapping:
- Map Automatically
- Match → id

Sekarang sheet Cleaned_Data hanya berisi data yang benar-benar bersih.

---

NOTE: untuk mengakses database yang sudah ready boleh email ke nicholasnelson71@gmail.com ya 
untuk bisa invite akses ke organization dari Supabase yang saya miliki
# 🗄 STEP 8 — Setup Supabase Database

## 1️⃣ Install PostgreSQL

https://www.postgresql.org/

---

## 2️⃣ Buat Project Supabase

- Login Supabase
- Create Project
- Simpan password project

---

## 3️⃣ Ambil Connection String

Masuk:
Database → Connect


Ubah:
- Type → PSQL
- Method → Transaction Pooler

---

# 🔌 STEP 9 — Hubungkan Postgres ke n8n

Tambahkan node:
Postgres


Create Credential:

Isi sesuai dari Supabase:
- Host
- Database
- User
- Password
- Port

Operation:
Insert or Update


Schema:
public


Table:
expense


Match:
id


---

# 🧱 STEP 10 — Buat Tabel di Supabase

Masuk:
Table Editor → Create Table


Nama:
expense


Struktur:

| Column | Type |
|--------|------|
| id | int8 (Primary Key) |
| Date | date |
| Category | varchar |
| Description | text |
| Amount | int8 |
| Payment_method | varchar |
| Mood | varchar |
| created_at | timestampz |

Klik Create.

---

# 🔗 Urutan Node Workflow

1. Schedule Trigger  
2. Get Row(s)  
3. Cleaning Format  
4. Update Row - Steps 1  
5. Convert Date  
6. Update Row - Steps 2  
7. Amount > 0  
8. Not Empty  
9. Does not contain unknown  
10. Append to Cleaned_Data  
11. Insert or Update Postgres  

Publish workflow.

---

# 📊 STEP 11 — Hubungkan ke Power BI (DirectQuery)
Docs Power BI dapat Direct Query Menggunakan PostgreSQL:
https://learn.microsoft.com/id-id/power-bi/report-server/data-sources

Detailnya dapat dilihat pada link berikut:
https://dev.to/jenniekibiri/how-to-connect-supabase-to-microsoft-power-bi-50p8

## Download SSL Certificate

Supabase → Database → Settings → SSL → Download

---

## Import Certificate di Windows

1. Win + R → ketik `mmc`
2. Add Snap-in → Certificates
3. Computer Account
4. Trusted Root Certification Authorities
5. Import file .crt

---

## Connect di Power BI

Get Data → PostgreSQL  
Gunakan:
DirectQuery


Masukkan:
- Host
- Database
- Username
- Password

---

# 📅 STEP 12 — Data Modeling di Power BI

## Buat Date Table

Modeling → New Table:

DateTable =
ADDCOLUMNS(
CALENDAR(MIN('expense'[Date]), MAX('expense'[Date])),
"Year", YEAR([Date]),
"Month", FORMAT([Date], "MMM"),
"MonthNumber", MONTH([Date]),
"YearMonth", FORMAT([Date], "YYYY-MM"),
"Day", DAY([Date]),
"DayName", FORMAT([Date], "dddd"),
"WeekdayNumber", WEEKDAY([Date])
)


Hubungkan:
DateTable[Date] → expense[Date]


---

# 📈 DAX Measures

Total Spending = SUM(expense[Amount])

Avg Daily Spending =
DIVIDE(
[Total Spending],
DISTINCTCOUNT(expense[Date])
)

Previous Month Spending =
CALCULATE(
[Total Spending],
DATEADD(DateTable[Date], -1, MONTH)
)

MoM Growth % =
DIVIDE(
[Total Spending] - [Previous Month Spending],
[Previous Month Spending]
)


---

# 🎨 Struktur Dashboard

KPI:
- Total Spending
- Avg Daily Spending
- MoM Growth
- Total Transaction

Visual:
- Trend Line
- Category Bar
- Payment Donut
- Mood Analysis
- Spending by Day

Tambahkan slicer:
- Date
- Category
- Payment Method
- Mood

---

# 🔄 Auto Refresh

Aktifkan:
Page Refresh → 10 seconds


Dashboard sekarang live.

---

# 🚀 Hasil Akhir

✔ Input cukup di Google Sheets  
✔ Data otomatis dibersihkan  
✔ Database otomatis update  
✔ Dashboard real-time  
✔ Sistem automation end-to-end  

---

# 📌 Kesimpulan

Ini bukan cuma dashboard biasa.

Ini adalah mini data engineering pipeline versi personal.

Bisa dikembangkan jadi:
- Business reporting
- Finance system UMKM
- Monitoring system real-time

---

SELESAI.
