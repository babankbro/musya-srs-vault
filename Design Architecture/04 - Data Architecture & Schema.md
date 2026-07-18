---
title: "Design — Data Architecture & Database Schema (การออกแบบสถาปัตยกรรมข้อมูลและฐานข้อมูล)"
tags: [MUSYA, design, data, database, schema, erd]
created: 2026-07-18
---

# 🗄️ การออกแบบสถาปัตยกรรมข้อมูลและโครงสร้างสคีมา (Data Architecture & Schema)

← [[03 - Runtime Views]] · กลับสู่ภาพรวม → [[00 - Architecture Overview]] · อ้างอิงข้อกำหนดเดิม: [[06 - Data Requirements]]

> เอกสารชุดนี้เป็น **มุมมองเชิงการออกแบบข้อมูล (Data Design View)** ซึ่งแกะแบบมาจากไฟล์ DDL ต้นฉบับที่รันจริงในระบบ
> ทั้งในฝั่ง `chatapi.python/database/` (เช่นไฟล์ `schema.sql`, และซีรีส์ `025`–`027` ของระบบ obsidian) รวมถึง
> ฝั่ง `chatappandpython/database/schema.sql` — รับรองได้ว่านี่คือ "แผนผังของจริงที่ระบบกำลังใช้อยู่" ไม่ใช่แค่แบบร่าง

---

## 1. หลักการออกแบบระบบ — "หนึ่งฐานข้อมูล สามอาณาจักร" (One DB, three subdomains)

โครงสร้างทั้งหมดใช้ขุมพลังของ **PostgreSQL เพียงก้อนเดียว** (ชื่อฐานข้อมูล `musyadata`, รันบนอิมเมจ `pgvector/pgvector:pg16`)
แต่ภายในถูกจัดสรรแบ่งโซนออกเป็น 3 อาณาจักรที่ระดับความละเอียดของข้อมูล (grain) และวงจรชีวิตแตกต่างกันอย่างชัดเจน โดยทำงานผสานกับ
แหล่งเก็บข้อมูลภายนอกอีก 2 แหล่งเสริมความแกร่ง:

```mermaid
flowchart TB
    subgraph PG["ดาต้าเบส PostgreSQL: musyadata (ใช้ปลั๊กอิน pgvector + pg_trgm)"]
        APP["① ตารางหลักของแอปพลิเคชัน (Application Tables)<br/>accounts (บัญชี) · chat_sessions (แชท) · journal_reports (คลังรายงาน)"]
        ACC["② ตารางสถิติอุบัติเหตุแบบดาวกระจาย (Accident Star Schema)<br/>dim_* (มิติ) · fact_* (ตารางความจริง) · mart_* (สรุปยอด) · views (มุมมอง)"]
        RAG["③ ตารางคลังความรู้ Obsidian RAG Index<br/>vaults · notes · chunks · links · pdf_assets"]
    end
    MinIO[("ระบบเก็บไฟล์ MinIO<br/>แบ่งเป็นถัง fileapa / pdf-library")]
    Redis[("หน่วยความจำ Redis<br/>เก็บประวัติแชทระยะสั้น 24 ชม. · และทำแคช")]

    FE["ฝั่งหน้าบ้าน Frontend / BFF"] --> APP
    BE["ฝั่งหลังบ้าน Backend python-ai"] --> ACC
    BE --> RAG
    BE --> MinIO
    BE --> Redis
    FE --> MinIO
    RAG -. อ้างอิงลิงก์ minio_url .-> MinIO
```

| ชื่ออาณาจักร (Subdomain) | เจ้าของหลักดูแล | ระดับความละเอียด/ธรรมชาติของข้อมูล | ใช้ทำหน้าที่ตอบโจทย์อะไร |
|---|---|---|---|
| ① ระบบ Application | ฝั่ง Frontend (BFF) | เชิงปฏิบัติการ (มีข้อมูลแยกตามผู้ใช้/เซสชันแชท/เอกสารรายงาน) | ระบบล็อกอิน, ประวัติการแชทเก่า, และคลังเก็บผลงานรายงาน |
| ② ระบบ Accident Star | ฝั่ง Backend | เชิงวิเคราะห์ (Aggregate สรุปยอดรวมแยกตามพิกัด/เวลา/เส้นถนน) | ถาม-ตอบสถิติอุบัติเหตุของเขตสุขภาพที่ 10 |
| ③ ระบบ RAG Index | ฝั่ง Backend | เชิงสืบค้น (แบ่งเป็นตัวโน้ต/การหั่นย่อหน้า chunk/กราฟเชื่อมโยง) | ถาม-ตอบจากทฤษฎีในคลังความรู้ |
| แหล่งไฟล์ MinIO | ใช้งานร่วมกันทั้งคู่ | เป็นชิ้นไฟล์ไบนารี (Binary object) | เก็บไฟล์ดิบ CSV/PDF/DOCX + จัดการ APA + เซฟรายงาน |
| หน่วยจำ Redis | ฝั่ง Backend | เกิดดับรวดเร็ว (มีอายุขัย TTL 24 ชม.) | ความจำสั้นของแชทระหว่างเซสชัน + ทำระบบแคช |

> [!tip] เคล็ดลับการจำหลักคิดที่สำคัญ
> **"MinIO มีไว้เก็บ ตัวไฟล์อ้วนๆ (binary), ส่วน PostgreSQL มีไว้เก็บ ที่อยู่และป้ายกำกับ (path/URL/metadata)"**
> — ยกตัวอย่างเช่น ตาราง `obsidian_pdf_assets` จะมีคอลัมน์ `minio_path` เพื่อทำหน้าที่เป็นลายแทงชี้เป้าไปยังไฟล์ต้นฉบับใน MinIO

---

## 2. อาณาจักร ① — ผังความสัมพันธ์ตารางแอปพลิเคชัน (Application ERD)

```mermaid
erDiagram
    ACCOUNTS ||--o{ CHAT_SESSIONS  : "ถือครอง (owns)"
    ACCOUNTS ||--o{ JOURNAL_REPORTS : "ถือครอง (owns)"
    ACCOUNTS ||--o{ ACCOUNTS        : "เป็นผู้อนุมัติ (approves)"

    ACCOUNTS {
        uuid id PK "กุญแจหลัก"
        varchar email UK "อีเมลห้ามซ้ำ"
        varchar password_hash "รหัสผ่านที่ถูกแฮช"
        varchar role "สถานะ user|admin|adminsuper"
        varchar status "รอ pending|ผ่าน approved|ตก reject"
        uuid approved_by FK "ใครอนุมัติ"
        varchar reset_token "โทเค็นลืมรหัส"
        timestamptz created_at "เวลาลงทะเบียน"
    }
    CHAT_SESSIONS {
        uuid id PK
        text session_id UK "รหัสอ้างอิงตอนแชท"
        uuid user_id FK "ถ้าลบคน ให้เป็น ON DELETE SET NULL"
        varchar status "สถานะ idle|running|done|error"
        jsonb messages_json "ก้อนประวัติแชท"
        timestamptz updated_at
    }
    JOURNAL_REPORTS {
        uuid id PK
        uuid user_id FK "ถ้าลบคน ให้เป็น ON DELETE CASCADE"
        text title "ชื่อรายงาน"
        varchar doc_type "ชนิด policy|plan|workplan"
        int article_count "จำนวนแหล่งอ้างอิง"
        text html_content "เนื้อหาโค้ด HTML"
        timestamptz created_at
    }
```

**ประเด็นน่าสนใจในการออกแบบชั้นนี้:**
- ระบบเลือกใช้กุญแจหลักเป็น **UUID** ทั้งหมด (เจนเนอเรตผ่านคำสั่ง `gen_random_uuid()` จากปลั๊กอิน `pgcrypto`) เพื่อความปลอดภัย
- ช่อง **`updated_at` จะเด้งอัปเดตอัตโนมัติ** ผ่านระบบฟังก์ชัน `update_updated_at()` ที่ผูก Trigger ไว้กับทุกตาราง
- กฎการลบบัญชีมีความแตกต่างกันโดยเจตนา: หากผู้ใช้ถูกลบ → ข้อมูลแชทของเขาจะใช้สิทธิ์ `SET NULL` (เพื่อเก็บ Log ไว้ดูเล่น) แต่เอกสารรายงานของเขาจะใช้สิทธิ์ `CASCADE` (สลายตามไปเลย) อ่านเพิ่มที่ [[06 - Data Requirements|DR-LIFE-02]]
- ฟิลด์ **`messages_json` เลือกใช้ชนิด JSONB** — เอาไว้แพ็กก้อนบทสนทนายัดใส่แถวเดียวเลย (ไม่เสียเวลากระจายแตกตารางทีละประโยคข้อความ)
- ระบบปุ่มกดอนุมัติคนใช้ โยงกลับเข้าหาตัวเอง (Self-reference) `approved_by → วิ่งไปหา accounts.id`

> [!warning] คำเตือนเรื่องช่องโหว่ความปลอดภัยเบื้องต้น (Governance)
> ภายในสคริปต์ไฟล์ `schema.sql` ทีมพัฒนาได้แอบ **หยอดบัญชีทดสอบพร้อมรหัสผ่านที่ตั้งง่ายๆ เอาไว้** (เช่น `musya@gmail.com`, `supermusya@gmail.com`, และ `musya01`–`musya50`) ดังนั้นตอนขึ้น Production จริงๆ ต้องอย่าลืมกวาดล้างลบทิ้งหรือบังคับเปลี่ยนรหัสผ่านทันที ([[05 - Non-Functional Requirements|NFR-SEC-08]])

---

## 3. อาณาจักร ② — ผังดาวกระจายสถิติอุบัติเหตุ (Accident Star Schema ERD)

```mermaid
erDiagram
    DIM_GEOGRAPHY   ||--o{ DIM_ROAD_SEGMENT    : "พิกัดที่ตั้ง"
    DIM_GEOGRAPHY   ||--o{ FACT_ACCIDENT_EVENT : "เกิดเหตุในพื้นที่"
    DIM_ROAD_SEGMENT||--o{ FACT_ACCIDENT_EVENT : "เกิดบนสายทาง"
    DIM_SOURCE      ||--o{ FACT_ACCIDENT_EVENT : "อิงข้อมูลจากแหล่ง"
    FACT_ACCIDENT_EVENT ||--o{ FACT_ACCIDENT_PERSON : "บุคคลที่เกี่ยวข้อง"
    DIM_GEOGRAPHY   ||--o{ MART_PROVINCE_ROAD  : "จับรวมสถิติ"
    DIM_ROAD_SEGMENT||--o{ MART_PROVINCE_ROAD  : "จับรวมสถิติ"

    DIM_GEOGRAPHY {
        bigserial geography_id PK
        varchar province_name "ชื่อจังหวัด"
        varchar district_name "ชื่ออำเภอ"
        varchar subdistrict_name "ชื่อตำบล"
        decimal latitude "พิกัดละติจูด"
        decimal longitude "พิกัดลองจิจูด"
    }
    DIM_ROAD_SEGMENT {
        bigserial road_segment_id PK
        varchar road_name "ชื่อเส้นทาง"
        varchar road_code "รหัสเส้นทาง"
        bigint geography_id FK
        boolean risk_flag "เป็นทางเสี่ยงหรือไม่"
    }
    DIM_SOURCE {
        bigserial source_id PK
        varchar source_name "ชื่อหน่วยงานอ้างอิง"
        varchar quality_level "ระดับคุณภาพเดต้า"
    }
    FACT_ACCIDENT_EVENT {
        bigserial accident_id PK
        timestamp event_datetime "เวลาเกิดเหตุ"
        bigint geography_id FK
        bigint road_segment_id FK
        varchar severity_level "ความรุนแรง"
        int injured_count "ยอดเจ็บ"
        int death_count "ยอดตาย"
        int csv_year "ปีที่เกิด (ค.ศ.)"
        bigint source_id FK
    }
    FACT_ACCIDENT_PERSON {
        bigserial person_event_id PK
        bigint accident_id FK
        int age "อายุ"
        varchar sex "เพศ"
        boolean helmet_used "สวมหมวกไหม"
        boolean seatbelt_used "รัดเข็มขัดไหม"
    }
    MART_PROVINCE_ROAD {
        bigserial id PK
        int year_no
        bigint geography_id FK
        varchar province_name
        varchar district_name
        bigint road_segment_id FK
        int accident_count
        int death_count
        decimal hotspot_score "คะแนนความเสี่ยงวิกฤต"
    }
```

**ตารางสายมิติ (Dimensions):** `dim_geography` (ซอยย่อยลงจาก จังหวัด→อำเภอ→ตำบล + พิกัดพิน), `dim_time` (หยอดข้อมูลรอไว้ตั้งแต่ปี 2020–2030), `dim_source`, `dim_road_segment`
**ตารางสายความจริงข้อมูลดิบ (Facts):** `fact_accident_event` (นับยอดเป็นรายเหตุการณ์ที่เกิดขึ้น), `fact_accident_person` (นับยอดระดับตัวบุคคล)
**ตารางสายสรุปรวบยอด (Marts):** `mart_accident_summary`, `mart_accident_hotspot`, `mart_province_year`, `mart_province_road`
**ตารางสายมุมมองเสริม (Views):** `v_province_year_summary`, `v_province_road_year`

> [!danger] ระวัง! ข้อจำกัดทางข้อมูลที่จะส่งผลกระทบต่อคำตอบของ AI
> - **ตาราง `fact_accident_person` ปัจจุบันมีแต่ความว่างเปล่า** → ทำให้ AI ไม่สามารถตอบเรื่องสถิติหมวกกันน็อค/การคาดเข็มขัดนิรภัย/แยกอายุหรือเพศได้เลย → ไปป์ไลน์จึงจำเป็นต้องห้อยท้ายเชิงอรรถข้อจำกัดข้อมูลเอาไว้กันเหนียว
> - **คอลัมน์ชื่อถนน `road_name` ในตาราง `mart_province_road` ส่วนใหญ่มักเป็นค่าความว่างเปล่า (NULL)**
> - **ข้อมูลปีทั้งหมดถูกเก็บเป็น ค.ศ. (`csv_year` / `year_no` = 2021–2026)** → หากผู้ใช้ดันถามมาเป็นระบบ พ.ศ. โค้ดจะต้องจัดการจับตัวเลขไปลบออก −543 ก่อนส่งไปทำคิวรี แล้วค่อยบวกกลับ +543 ตอนจะแสดงผลหน้าจอให้คนดู ([[03 - Functional Requirements|FR-CHAT-09]])
> - ตัวตาราง `dim_time` ถึงจะมีการเตรียมข้อมูลไว้ แต่ก็ **ไม่ได้ถูกจับผูกเส้น FK** โยงจากตารางข้อเท็จจริงโดยตรง (ตารางตัว event หันไปยึดติดกับคอลัมน์ `event_datetime` แทน)

**การรีเฟรชข้อมูลในตาราง mart:** เนื่องจากตารางกลุ่ม mart เป็นก้อนผลรวม aggregate — จึงต้องคอยมีช่วงเวลาอัปเดตข้อมูลเข้าไปให้สดใหม่ (ตัวอย่างเช่นรัน `030_populate_mart_province_road.sql`) (ในอนาคตควรเขียน Job ตั้งเวลาและตีตรา timestamp กำกับความสดไว้ด้วย)

---

## 4. อาณาจักร ③ — ผังดัชนีคลังความรู้ RAG (Obsidian RAG ERD)

```mermaid
erDiagram
    OBSIDIAN_VAULTS ||--o{ OBSIDIAN_NOTES        : "เก็บบรรจุโน้ต"
    OBSIDIAN_NOTES  ||--o{ OBSIDIAN_NOTE_CHUNKS  : "สับหั่นเป็นท่อน"
    OBSIDIAN_NOTES  ||--o{ OBSIDIAN_NOTE_LINKS   : "เป็นต้นทางปล่อยลิงก์"
    OBSIDIAN_NOTES  ||--o{ OBSIDIAN_NOTE_LINKS   : "เป็นปลายทางรับลิงก์"
    OBSIDIAN_NOTES  ||--o{ OBSIDIAN_PDF_ASSETS   : "ผูกอ้างอิงกับไฟล์ PDF"

    OBSIDIAN_VAULTS {
        text vault_id PK
        text name "ชื่อคลัง"
        text vault_path "พาธบนเซิร์ฟ"
        int note_count "ยอดรวม"
    }
    OBSIDIAN_NOTES {
        text note_id PK "รูปแบบ vault_id::relative_path"
        text vault_id FK
        text province "ป้ายจังหวัด"
        text district "ป้ายอำเภอ"
        text note_type "ชนิด MOC|district|report|research|policy"
        text_array tags "แท็กที่แปะ"
        int year "ปี"
        text content_stripped "เนื้อหาถูกปลอกเปลือกแท็กออก"
    }
    OBSIDIAN_NOTE_CHUNKS {
        text chunk_id PK "รูปแบบ note_id::index"
        text note_id FK
        int chunk_index "เบอร์ 0 = บทนำ intro"
        text header "หัวข้อชิ้นส่วน"
        text chunk_content "เนื้อหาเฉพาะส่วนนั้น"
    }
    OBSIDIAN_NOTE_LINKS {
        text source_note_id PK,FK
        text target_note_id PK,FK
        text link_text "คำอธิบายลิงก์"
    }
    OBSIDIAN_PDF_ASSETS {
        serial id PK
        text province
        text note_id FK
        text minio_path UK
        text minio_url "ลิ้งก์กดดู"
    }
```

**ดีไซน์การค้นหาข้อมูลแบบลูกผสม (Hybrid Search ด้วย pg_trgm):**
- อาศัยเปิดส่วนเสริม **`pg_trgm`** — ซึ่งเป็นท่าไม้ตายในการจับ **ค้นหาภาษาไทยแบบตรงบ้างไม่ตรงบ้าง (partial-match)** โดยไม่ต้องแคร์การตัดคำ tokenizer
- สร้างดัชนีแบบ **GIN trigram** คลุมทับฟิลด์ `content_stripped`, `title`, และ `chunk_content` เอาไว้
- **ระดับที่ 1 (หาแบบแม่นๆ):** เล็งค้นหาลงไประดับก้อนชิ้นส่วน **chunk** (ที่โดนซอยแยกตามหัวข้อ `##`) + ผสานการขยายปริมณฑลตามเส้นใย **wikilink graph** (โดยให้วิ่งค้นตาราง `obsidian_note_links` — ถอยหน้าและถอยหลังไป 1 ก้าว hop)
- **ระดับที่ 2 (ท่าสำรองยามฉุกเฉิน):** หากสแกนหาระดับ chunk ไม่รอด ก็จะหันไปค้นเหวี่ยงแหด้วยคำสั่ง `LIKE` คลุมยาวทั้งโน้ต
- การตั้งรหัสเชื่อมโยง: `note_id` จะเกิดจาก `vault_id::relative_path`, ส่วนรหัส `chunk_id` ก็จะเกิดจาก `note_id::chunk_index`
- ตัวคลังข้อมูล Vault เริ่มต้น ถูกติดตั้งเตรียมไว้ล่วงหน้าแล้วในชื่อ: `health_region_10`
- กฎการล้างบางทิ้ง (Cascade delete): ถ้าทุบคลัง vault ทิ้ง → โน้ตทั้งหมดปลิว → ท่อน chunks/links สลายตัว (แต่ว่าลิงก์บน `pdf_assets.note_id` จะแค่กลายเป็น `SET NULL` เพื่อกันเหนียวรักษาตัวไฟล์ PDF เอาไว้)

---

## 5. แหล่งจัดเก็บนอกตัวฐาน — MinIO & Redis

| ชื่อถัง (MinIO bucket) | เนื้อหาที่กักเก็บ | ตัวแปรตั้งค่าที่เชื่อม |
|---|---|---|
| `fileapa` | ไฟล์สารพัดที่ผู้ใช้อัปโหลด, ตัวเอกสารโครงสร้าง APA, และก้อนร่างรายงาน | ฝั่งหน้าเว็บ (`MINIO_BUCKET`) |
| `pdf-library` | แฟ้ม PDF ไว้กองเป็นห้องสมุด + เพื่อเตรียมทะลวงยัดเข้าคลัง vault | ตัวแปร (`PDF_BUCKET`) |

- พวกไฟล์ตาราง CSV (หมวด `d2–d4`) จะถูกตั้งชื่อเซฟเป็น **รหัสตัวเลข (id)** ส่วนชื่อไฟล์จริงๆ จะถูกแนบเนียนเก็บไว้ในหมวดออบเจกต์ Metadata → ดังนั้นเวลา AI จะค้นหาไฟล์ต้องพึ่งวิทยายุทธ์ **การกะเดาหาคำคล้าย (fuzzy matching)** (ซึ่งก็มีสิทธิ์พลาดได้ ถ้าชื่อไฟล์ตั้งมาแบบชวนสับสน)
- **Redis** (`redis://redis:6379/0`) — ถือครองประวัติการแชทล่าสุด **6 เทิร์น, อายุขัยตั้งไว้ที่ TTL 24 ชม.**, รวมถึงรับฝากแคชผลสรุป PDF งานวิจัยของสายท่อ ThaiJo (ใช้คีย์ = `thaijo_pdf:{sha256(url)}`)

---

## 6. เส้นทางการไหลเวียนข้อมูลเข้าสู่ระบบ (Ingestion / write paths)

```mermaid
flowchart LR
    subgraph Ingest["ต้นทางจุดป้อนข้อมูล (Ingest)"]
        A["ไฟล์ accident.sql / หรือแก๊ง 030_*"] -->|ลงเข็มปูพรม + อัดข้อมูล| ACC[(สถิติ Accident Star)]
        B["สคริปต์ index_obsidian.py"] -->|ตระเวนอ่านไฟล์ .md → ดันข้อมูลเข้า (upsert)| RAG[(คลังความรู้ RAG Index)]
        C["สคริปต์ sync_obsidian_pdfs.py"] -->|โยนอัปโหลด PDF| MO[(เข้าโกดัง MinIO)]
        C -->|จดบันทึกเส้นทางพาธ path/url| RAG
        D["จุดที่ผู้ใช้งานกดอัปโหลดเอง /fileapa"] --> MO
        E["เมื่อสร้างรายงาน report wizard เสร็จและกดบันทึก"] --> JR[(ลงตาราง journal_reports)]
    end
```

| แหล่งกำเนิด | ชื่อสคริปต์/จุดรันคำสั่งเส้นทาง | จุดหมายปลายทาง |
|---|---|---|
| ข้อมูลสถิติอุบัติเหตุ | `accident.sql`, `030_populate_mart_province_road.sql` | ② เขต star schema |
| คลังองค์ความรู้ | `src/scripts/index_obsidian.py` | ③ โยนลงตาราง notes/chunks/links |
| ไฟล์สแกน PDF ดิบ | `src/scripts/sync_obsidian_pdfs.py` | โกดัง MinIO + ผูก `obsidian_pdf_assets` |
| อัปโหลดของผู้ใช้/APA | โยนผ่านด่าน BFF `/api/files/*`, `/api/generate-apa` | โกดัง MinIO ถัง `fileapa` |
| ตัวรายงานที่สั่งเซฟเก็บ | โยนผ่านด่าน BFF `/api/journal-reports` | โยนลงตาราง `journal_reports` |

---

## 7. วัฏจักรวงจรชีวิตและธรรมาภิบาลข้อมูล (Lifecycle & governance)

| ประเด็นเรื่อง | กฎกติกาและมารยาท/ความเสี่ยง | อ้างอิง |
|---|---|---|
| ประวัติแชทความจำสั้น | แช่ใน Redis มีอายุสูญสลายข้ามวัน (24 ชม.); ส่วนประวัติในตาราง `chat_sessions` คือที่เก็บถาวรตัวจริง | DR-LIFE-01 |
| การสั่งประหารลบผู้ใช้ | แชทเก่าเขาจะโชว์ว่าไร้ตัวตน `SET NULL`, แต่งานรายงานเก่าของเขาจะปลิวสลาย `CASCADE` ทันที | DR-LIFE-02 |
| โทเค็นรีเซ็ตรหัส | จำกัดการใช้แค่ครั้งเดียว และจะหมดคุณค่าตามตารางเวลา `reset_token_expires` | DR-LIFE-03 |
| ชุดบัญชีรหัสผ่านตั้งต้น | ในโค้ด `schema.sql` มียัดบัญชีทิ้งไว้แบบแฮชรหัสที่รู้กัน — ต้องสั่งล้างบางก่อนขึ้นรัน Prod เด็ดขาด | NFR-SEC-08 |
| สถานะแบบ Enum ในช่อง VARCHAR | ในฟิลด์พวก `role`/`status`/`doc_type` ไม่ได้มีการเขียน CHECK constraint กันหลงไว้แต่แรก — แนะนำในอนาคตควรแก้ใส่เพื่อกันข้อมูลขยะ | — |
| การตั้งปลุกข้อมูล mart | ควรต้องสร้างวงจร Job ขึ้นมาสั่งปลุกอัปเดตข้อมูลก้อน mart ให้ตามตาราง fact ทันเสมอ + ประทับตรา timestamp | DR-ACC |
| การเสิร์ชค้นหาแบบ trigram | ถือว่าเวิร์กมากกับภาษาไทยเพราะหาคำโดดๆ ได้ แต่ยังไม่ใช่สเต็ปของการเข้าใจบริบท (semantic vector) (ถ้าอนาคตอยากทำ ต้องเสริม embedding column + ลง vector index เพิ่ม) | — |
| แฟ้มบ่นปัญหา error_logs | โดนพอกพูนถมเป็นภูเขารายวัน **ไร้นโยบายเคลียร์ทิ้ง (ไม่มี retention policy)** — ถึงจุดนึงควรเปิดระบบตั้งลบอัตโนมัติ | DR-LIFE-04 |

---

## 8. สรุปแมปโครงสร้างชิ้นส่วนองค์ประกอบ → จับคู่กับตารางที่รองรับ (Component → Tables)

| ส่วนองค์ประกอบการทำงาน | ฝากข้อมูลไว้ที่ตาราง/ที่เก็บไหน | โยงไปอ้างอิงยูสเคส |
|---|---|---|
| ฝ่ายประตูล็อกอิน / ตรวจอนุมัติสิทธิ์คนเข้าแอป | ลงตาราง `accounts` | ยูสเคส [[01 - Account & Access]] |
| ฝ่ายแชทพูดคุย / หน้าต่างประวัติความหลัง | ลงตาราง `chat_sessions` + พึ่งความจำ Redis | ยูสเคส [[02 - Chat & Domain Analysis]] |
| ฝ่ายร่างนโยบาย/เซฟรวบรวมรายงาน | ลงตาราง `journal_reports` + พึ่งที่ฝากไฟล์ MinIO | ยูสเคส [[03 - Report Generation]] |
| ฝ่ายสถิติอุบัติเหตุ | ลงตารางดาวกระจาย `fact_*`, `mart_*`, `dim_*`, รวมถึงพวก views | ยูสเคส [[02 - Chat & Domain Analysis]] |
| ฝ่ายสืบค้นวิชาการ-ค้นคลังความรู้องค์กร | ลงตาราง `obsidian_notes/chunks/links` + และพ่วง `pdf_assets` | ยูสเคส [[04 - Knowledge, Files & Admin]] |
| ฝ่ายจัดการจัดการไฟล์ & ทำ APA อัตโนมัติ | ฝากโยนเข้า MinIO ถัง `fileapa` | ยูสเคส [[04 - Knowledge, Files & Admin]] |

*เอกสารฉบับนี้แกะซอร์สโค้ดตารางฐานข้อมูล DDL ณ วันที่ 2026-07-18 — หากตารางโครงสร้างถูกโปรแกรมเมอร์แอบเข้าไปไขปรับเปลี่ยนภายหลัง ขอให้รีครอสเช็คกับบรรดาไฟล์โค้ด
`database/*.sql` อีกครั้งเสมอเพื่อความชัวร์*
