# 📊 Automation Live Expense Dashboard (Local Setup)

End-to-end automation dashboard pengeluaran pribadi menggunakan:

- **n8n** (Automation Engine)
- **Google Sheets** (Raw Data Source)
- **Docker** (Local Container Runtime)
- **Supabase PostgreSQL** (Database)
- **Power BI (DirectQuery)** (Live Dashboard)

---

# 🎯 Tujuan Project

Project ini bertujuan untuk:

1. Mengotomatisasi pencatatan pengeluaran harian
2. Membersihkan & memvalidasi data secara otomatis
3. Menyimpan hanya data bersih ke database
4. Menampilkan dashboard real-time di Power BI
5. Membangun arsitektur data pipeline profesional skala personal

---

# 🏗️ Arsitektur Sistem

Google Sheets (Raw Data)
↓
n8n (Cleaning + Transform + Validation)
↓
Google Sheets (Cleaned_Data)
↓
Supabase PostgreSQL
↓
Power BI (DirectQuery Live Dashboard)


---

# 🐳 STEP 1 — Install & Jalankan n8n via Docker

## 1️⃣ Install Docker Desktop

Download:
https://www.docker.com/products/docker-desktop

Selesaikan instalasi hingga Docker berjalan normal.

---

## 2️⃣ Pull Image n8n

1. Buka Docker Desktop  
2. Klik **Images**  
3. Tab **Search** → ketik:

n8nio/n8n


4. Klik **Pull**

Tunggu sampai selesai.

---

## 3️⃣ Jalankan Container n8n

Klik ▶ Run lalu isi:

- **Container Name**: `n8n-expense`
- **Port**: `5678`
- **Host Path**: pilih folder lokal (untuk persistence)

Klik **Run**

Setelah container running:

Klik link:
5678:5678


Buka browser:

http://localhost:5678


Login dan selesaikan setup authentication.

---

# 📄 STEP 2 — Membuat Google Spreadsheet

Buat spreadsheet baru dengan 2 sheet:

- `Data`
- `Cleaned_Data`

## Struktur Kolom (Row 1)

| id | Date | Category | Description | Amount | Payment_Method | Mood |

### Penjelasan Kolom

- **id** → Primary key unik
- **Date** → Tanggal transaksi
- **Category** → transport, food, groceries, dll
- **Description** → detail transaksi
- **Amount** → jumlah uang
- **Payment_Method** → qris / cash / debit / e-wallet
- **Mood** → happy / tired / calm / bad / dll

Rename spreadsheet dari default “Untitled Spreadsheet”.

---

# ☁️ STEP 3 — Setup Google Cloud API

## 1️⃣ Buat Project

1. Login Google Cloud Console  
2. Klik **New Project**  
3. Isi nama project  
4. Parent Resource → No Organization  
5. Klik Create  

---

## 2️⃣ Enable API

Masuk ke:

APIs & Services → Enable APIs


Aktifkan:

- Google Sheets API
- Google Drive API

---

## 3️⃣ Buat OAuth Credential

1. APIs & Services → Credentials  
2. Klik **Create Credentials**  
3. Pilih **OAuth Client ID**  
4. Application Type → Web Application  
5. Klik Create  

---

## 4️⃣ Tambahkan Test User

Sidebar → Audience  
Scroll → Test Users  
Add email akun Google yang akan digunakan  
Klik Save  

---

# 🔐 STEP 4 — Koneksi Google Sheets ke n8n

## Buat Workflow Baru

Nama:
Expense Dashboard


---

## 1️⃣ Schedule Trigger

Node:
Schedule Trigger


Konfigurasi:

- Interval: Days  
- Days Between Triggers: 1  
- Hour: Midnight  
- Minute: 0  

---

## 2️⃣ Get Row(s) in Sheets

Tambah Node:
Get Row(s) in Sheets


Klik **Create New Credential**

Salin OAuth Redirect URL dari n8n:

http://localhost:5678/rest/oauth2-credential/callback


Masuk Google Cloud → Credentials → OAuth Client  
Tambahkan URL tersebut di **Authorized Redirect URLs**

Salin:
- Client ID
- Client Secret

Paste ke n8n.

Klik **Sign in with Google**  
Pilih akun Test User  
Centang semua permission  
Done.

---

Konfigurasi Node:

- Resource → Sheet Within Document  
- Operation → Get Row(s)  
- Document → pilih spreadsheet  
- Sheet → `Data`

---

# 🧹 STEP 5 — Cleaning & Transform Data

## Node: Edit Fields (Set)
Rename: `Cleaning Format`

Mode: Manual Mapping

Date = {{ $json.Date.toDateTime() }}
Category = {{ $json.Category.trim().toLowerCase() }}
Description = {{ $json.Description.trim().toLowerCase() }}
Amount = {{ parseInt($json.Amount) }}
Payment_Method = {{ $json["Payment_Method"].trim().toLowerCase() }}
Mood = {{ $json.Mood ? $json.Mood.trim().toLowerCase() : "unknown" }}


Toggle:
- Include Other Input Fields → ON  
- All Except → row_number  

---

## Update Row - Steps 1

Node:
Update Row


Sheet → Data  
Column to match → id  

---

## Convert Date Format

Node: Edit Fields (Set)  
Rename: `Convert DateTime to YYYY-MM-DD`

Date = {{ new Date($json.Date).toISOString().split('T')[0] }}


---

## Update Row - Steps 2

Copy node Update Row sebelumnya.

---

# 🔍 STEP 6 — Filtering Data

## Filter 1 → Amount > 0

Condition:

{{ $json.Amount.toNumber() }} > 0


---

## Filter 2 → Not Empty

Conditions:

- Category is not empty  
- Description is not empty  
- Payment_Method is not empty  

---

## Filter 3 → Does not contain "unknown"

Condition:

{{ $json.Mood }} does not contain unknown


---

# 📥 STEP 7 — Append ke Cleaned_Data

Node:
Append or Update Row


Rename:
Append to Cleaned_Data sheets


Sheet:
Cleaned_Data


Mapping:
- Map Automatically  
- Column to match → id  

---

# 🗄 STEP 8 — Setup Supabase PostgreSQL

## Install PostgreSQL

https://www.postgresql.org/

---

## Buat Project Supabase

1. Login Supabase  
2. Create Project  
3. Simpan password project  

---

## Ambil Connection String

Database → Connect  

Ubah:

- Type → PSQL  
- Method → Transaction Pooler  

Contoh:

psql -h host -p 6543 -d postgres -U user


Keterangan:
- `-h` → Host  
- `-p` → Port  
- `-d` → Database  
- `-U` → User  

---

# 🔌 STEP 9 — Connect Postgres ke n8n

Node:
Postgres


Create Credential:

Isi:
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


Mapping:
- Map Automatically  
- Columns to match → id  

---

# 🧱 STEP 10 — Buat Tabel di Supabase

Masuk:
Table Editor → Create Table


Nama tabel:
expense


Struktur:

| Column | Type | Constraint |
|--------|------|------------|
| id | int8 | Primary Key |
| date | date | NULL |
| category | varchar | NULL |
| description | text | NULL |
| amount | int8 | NULL |
| payment_method | varchar | NULL |
| mood | varchar | NULL |

Klik Create.

---

# 🔗 URUTAN WORKFLOW FINAL

1. Schedule Trigger  
2. Get Row(s) in Data  
3. Cleaning Format  
4. Update Row - Steps 1  
5. Convert DateTime to YYYY-MM-DD  
6. Update Row - Steps 2  
7. Amount > 0  
8. Not Empty  
9. Does not contain "unknown"  
10. Append to Cleaned_Data  
11. Insert or Update Postgres  

Publish workflow.

---

# 📊 STEP 11 — Connect Supabase ke Power BI (DirectQuery)

## Download SSL Certificate

Supabase → Database → Settings → SSL → Download

---

## Import SSL Certificate (Windows)

1. Win + R → ketik `mmc`  
2. File → Add/Remove Snap-in  
3. Add → Certificates  
4. Computer Account → Local Computer  
5. Trusted Root Certification Authorities  
6. Import file `.crt`

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

# 📅 STEP 12 — Data Modeling

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


---

# 📈 STEP 13 — DAX Measures

## Total Spending

Total Spending = SUM(expense[Amount])


## Avg Daily Spending

Avg Daily Spending =
DIVIDE(
[Total Spending],
DISTINCTCOUNT(expense[Date])
)


## MoM Growth

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

# 🎨 STEP 14 — Dashboard Layout

## KPI Area

- Total Spending  
- Avg Daily Spending  
- MoM Growth %  
- Total Transactions  

## Visual Insights

- Line Chart → Trend pengeluaran  
- Bar Chart → Top kategori  
- Donut Chart → Payment Method  
- Column Chart → Mood vs Spending  
- Bar Chart → Spending by Day  

Tambahkan slicer:
- Date
- Category
- Payment Method
- Mood

---

# 🔄 Auto Refresh

Enable:
Page Refresh → 10 seconds


Dashboard sekarang live.

---

# 🚀 Hasil Akhir

✔ Data otomatis dibersihkan  
✔ Tidak ada data kotor masuk database  
✔ PostgreSQL sinkron otomatis  
✔ Power BI real-time  
✔ Full automation pipeline  

---

# 📌 Kesimpulan

Project ini membangun:

- Automation pipeline
- Data cleaning layer
- Database persistence
- Real-time BI dashboard
- Arsitektur scalable

Dapat dikembangkan menjadi:
- Finance tracking system
- Small business reporting
- Real-time monitoring dashboard

---

END.
