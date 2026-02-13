Automation Live Expense Dashboard (Local Setup)
n8n + Google Sheets + Docker + Supabase PostgreSQL + Power BI (DirectQuery Real-Time)

🎯 Tujuan Project
Project ini bertujuan untuk:
Mengotomatisasi pencatatan pengeluaran pribadi
Membersihkan dan memvalidasi data secara otomatis
Menyimpan data bersih ke database PostgreSQL (Supabase)
Menampilkan dashboard interaktif di Power BI secara real-time (DirectQuery)
Membuat sistem end-to-end automation dari spreadsheet ke dashboard
Workflow ini memastikan:
Tidak ada data kotor masuk ke database
Dashboard selalu update otomatis
Tidak perlu input manual ke database

🏗️ Arsitektur Sistem
Google Sheets (Raw Data)
        ↓
n8n (Cleaning + Validation + Transform)
        ↓
Google Sheets (Cleaned_Data)
        ↓
Supabase PostgreSQL
        ↓
Power BI (DirectQuery Live Dashboard)


🐳 STEP 1 — Install & Jalankan n8n via Docker
1️⃣ Install Docker Desktop
Download dan install Docker Desktop dari:
https://www.docker.com/products/docker-desktop
Selesaikan instalasi sampai Docker berjalan normal.

2️⃣ Pull Image n8n
Buka Docker Desktop → Sidebar → Images
Pada tab Search ketik:
n8nio/n8n

Klik Pull
Tunggu hingga proses selesai.

3️⃣ Jalankan Container n8n
Klik tombol ▶ Run
Isi konfigurasi:
Container Name: n8n-expense
Port: 5678
Host Path: pilih folder lokal untuk persistent data
Klik Run
Setelah running:
Klik container → klik link 5678:5678
Buka di browser:
http://localhost:5678

Login dan selesaikan setup authentication.
n8n siap digunakan.

📄 STEP 2 — Membuat Spreadsheet
Buat Google Spreadsheet baru.
Buat 2 sheet:
Data
Cleaned_Data
Struktur Kolom (Row 1)
| id | Date | Category | Description | Amount | Payment_Method | Mood |
Penjelasan Kolom
id → unique identifier
Date → tanggal transaksi
Category → jenis pengeluaran
Description → detail transaksi
Amount → jumlah uang
Payment_Method → qris / cash / e-wallet / debit
Mood → happy / tired / calm / bad / dll
Rename spreadsheet dari default “Untitled Spreadsheet”.

☁️ STEP 3 — Setup Google Cloud API
1️⃣ Buat Project
Login ke Google Cloud Console
Klik New Project
Isi nama project
Parent Resource → No Organization
Klik Create

2️⃣ Enable API
Masuk ke:
APIs & Services → Enable APIs
Aktifkan:
Google Sheets API
Google Drive API

3️⃣ Buat Credential
Masuk ke:
APIs & Services → Credentials
Klik Create Credentials
Pilih OAuth Client ID
Application Type:
Web Application

Klik Create.

4️⃣ Tambahkan Test User
Sidebar → Audience
Scroll ke Test Users
Klik Add Users
Masukkan email Google Anda
Klik Save

🔐 STEP 4 — Koneksi Google Sheets ke n8n
Buat workflow baru:
Nama: Expense Dashboard

1️⃣ Tambah Trigger
Tambah Node:
Schedule Trigger

Konfigurasi:
Interval: Days
Days Between Triggers: 1
Hour: Midnight
Minute: 0
(Workflow jalan 1x sehari)

2️⃣ Tambah Google Sheets Node
Node:
Get Row(s) in Sheets

Klik Create New Credential
Salin OAuth Redirect URL:
http://localhost:5678/rest/oauth2-credential/callback

Masuk ke Google Cloud → Credentials → OAuth Client
Tambahkan URL tersebut ke:
Authorized Redirect URLs
Salin:
Client ID
Client Secret
Paste ke n8n
Klik Sign in with Google
Pilih akun yang sudah dimasukkan di Test Users
Centang semua permission → Done

Konfigurasi node:
Resource: Sheet Within Document
Operation: Get Row(s)
Document: Pilih spreadsheet
Sheet: Data

🧹 STEP 5 — Cleaning & Transform Data
Node: Edit Fields (Set)
Rename: Cleaning Format
Mode: Manual Mapping
Isi:
Date = {{ $json.Date.toDateTime() }}
Category = {{ $json.Category.trim().toLowerCase() }}
Description = {{ $json.Description.trim().toLowerCase() }}
Amount = {{ parseInt($json.Amount) }}
Payment_Method = {{ $json["Payment_Method"].trim().toLowerCase() }}
Mood = {{ $json.Mood ? $json.Mood.trim().toLowerCase() : "unknown" }}

Toggle:
Include Other Input Fields → ON
All Except → row_number

Update Row - Steps 1
Node: Update Row
Sheet: Data
Column to match: id

Convert Date Format
Node: Edit Fields (Set)
Rename: Convert DateTime to YYYY-MM-DD
Date = {{ new Date($json.Date).toISOString().split('T')[0] }}


Update Row - Steps 2
Copy node Update Row sebelumnya.

🔍 STEP 6 — Filtering Data
Filter 1 → Amount > 0
Condition:
{{ $json.Amount.toNumber() }} > 0


Filter 2 → Not Empty
Conditions:
Category is not empty
Description is not empty
Payment_Method is not empty

Filter 3 → Does not contain "unknown"
Condition:
{{ $json.Mood }} does not contain unknown


📥 STEP 7 — Append ke Cleaned_Data
Node:
Append or Update Row

Rename:
Append to Cleaned_Data sheets

Sheet:
Cleaned_Data

Mapping Mode:
Map Automatically
Column to match:
id

🗄 STEP 8 — Setup Supabase PostgreSQL
Install PostgreSQL
https://www.postgresql.org/

Buat Project Supabase
Masuk Supabase
Create Project
Simpan password project (PENTING)

Ambil Connection String
Database → Connect
Ubah:
Type → PSQL
Method → Transaction Pooler
Contoh:
psql -h host -p 6543 -d postgres -U user


🔌 STEP 9 — Connect Postgres ke n8n
Buat Node:
Postgres

Create Credential:
Isi:
Host
Database
User
Password
Port
Operation:
Insert or Update

Schema:
public

Table:
expense

Mapping:
Map Automatically
Match:
id

🧱 STEP 10 — Buat Tabel di Supabase
Masuk:
Table Editor → Create Table
Tabel: expense
Column
Type
Constraint
id
int8
Primary Key
date
date
NULL
category
varchar
NULL
description
text
NULL
amount
int8
NULL
payment_method
varchar
NULL
mood
varchar
NULL

Klik Create.

🔗 URUTAN WORKFLOW FINAL
Schedule Trigger
Get Row(s) in Data
Cleaning Format
Update Row - Steps 1
Convert DateTime to YYYY-MM-DD
Update Row - Steps 2
Amount > 0
Not Empty
Does not contain "unknown"
Append to Cleaned_Data
Insert or Update Postgres
Publish workflow.

📊 STEP 11 — Connect Supabase ke Power BI (DirectQuery)
Download SSL Certificate
Supabase → Database → Settings → SSL → Download

Import SSL Certificate
Win + R → ketik:
mmc

File → Add/Remove Snap-in
Add Certificates → Computer Account
Trusted Root Certification Authorities
Import .crt file

Connect di Power BI
Get Data → PostgreSQL
Gunakan:
DirectQuery

Masukkan:
Host
Database
Username
Password

📅 STEP 12 — Data Modeling
Buat Date Table
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

Buat Relationship:
DateTable[Date] → expense[Date]

📈 STEP 13 — DAX Measures
Total Spending
Total Spending = SUM(expense[Amount])

Avg Daily Spending
Avg Daily Spending =
DIVIDE(
    [Total Spending],
    DISTINCTCOUNT(expense[Date])
)

MoM Growth
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


🎨 STEP 14 — Dashboard Layout
🔷 KPI Area
Total Spending
Avg Daily Spending
MoM Growth %
Total Transactions
📈 Trend Line Chart
Axis → YearMonth
Value → Total Spending
📊 Category Bar Chart
Axis → Category
Value → Total Spending
🍩 Payment Method Donut
Legend → Payment_Method
Value → Total Spending
😊 Mood Analysis
Axis → Mood
Value → Total Spending
📅 Spending by Day
Axis → DayName
Value → Total Spending
Sort by WeekdayNumber

🔄 Auto Refresh
Power BI:
Enable Page Refresh
Set 10 seconds
Dashboard sekarang LIVE.

🚀 Hasil Akhir
✔ Data masuk dari Spreadsheet
✔ Dibersihkan otomatis
✔ Disimpan ke PostgreSQL
✔ Dashboard update real-time
✔ Full automation tanpa input manual

🏁 Conclusion
Project ini membangun sistem:
Data pipeline automation
Data validation layer
Database persistence
Real-time BI dashboard
Clean architecture
Sistem ini scalable dan bisa dikembangkan ke:
Finance tracking lebih kompleks
Personal KPI dashboard
Small business accounting automation
Real-time monitoring system

