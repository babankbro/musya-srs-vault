---
title: "SRS §6 — Data Requirements (ข้อกำหนดด้านข้อมูล)"
tags: [MUSYA, SRS, data, database]
created: 2026-07-18
---

# 6. ข้อกำหนดด้านข้อมูล (Data Requirements)

← [[05 - Non-Functional Requirements]] · → กลับสู่ [[00 - Actors & Use-Case Map]]

## 6.1 แหล่งจัดเก็บข้อมูลถาวร (Persistent stores)

ระบบใช้แหล่งจัดเก็บ 3 ส่วนหลักเพื่อแบ่งเบาภาระและแยกประเภทข้อมูลตามโครงสร้างที่เหมาะสม:

| แหล่งเก็บข้อมูล (Store) | เนื้อหาที่จัดเก็บ (Content) | หมายเหตุ / เวอร์ชัน (Notes) |
|---|---|---|
| PostgreSQL `musyadata` | ตารางการทำงานทั่วไปของแอป (App tables), ตารางวิเคราะห์อุบัติเหตุ (Star schema), และดัชนีเนื้อหา Obsidian สำหรับ RAG | ใช้อิมเมจ `pgvector/pgvector:pg16` เพื่อรองรับ Vector Search |
| MinIO | ไฟล์ที่ผู้ใช้อัปโหลดดิบ (`fileapa`), ห้องสมุด PDF (`pdf-library`), และเอกสารโครงสร้าง APA | เก็บไฟล์ไบนารี โดยให้ PostgreSQL เก็บพาธของไฟล์ไว้โยงกัน |
| Redis | ประวัติการแชทรอประมวลผล (เก็บ 6 ข้อความล่าสุด, ลบทิ้งใน 24 ชั่วโมง), แคชชั่วคราวของไปป์ไลน์ | ไม่ใช่ฐานข้อมูลถาวร เป็นเพียงทางผ่าน (not source of truth) |

## 6.2 เอนทิตีระดับแอปพลิเคชัน (Application entities: `DR-APP-*`)

### `accounts` — บัญชีผู้ใช้งาน (DR-APP-01)
ตารางนี้ใช้เก็บข้อมูลยืนยันตัวตน, บทบาทสิทธิ์, สถานะอนุมัติ, ข้อมูลส่วนตัวโปรไฟล์, และรหัสผ่านที่ถูกรีเซ็ต

| ชื่อคอลัมน์ (Column) | ชนิดข้อมูล (Type) | คำอธิบาย (Notes) |
|---|---|---|
| id | UUID PK | กุญแจหลัก ใช้คำสั่ง `gen_random_uuid()` |
| name | varchar | ชื่อเต็ม (จำเป็นต้องมี) |
| email | varchar UNIQUE | อีเมล (ใช้ล็อกอินและห้ามซ้ำ) |
| password_hash | varchar | รหัสผ่านที่เข้ารหัสแบบ bcrypt(12) |
| role | varchar | สิทธิ์ใช้งาน: `user` \| `admin` \| `adminsuper` (ค่าตั้งต้นคือ `user`) |
| status | varchar | สถานะ: `pending` \| `approved` \| `rejected` (ค่าตั้งต้นคือ `pending`) |
| approved_by | UUID FK→accounts.id | โยงไปหา ID แอดมินผู้กดยืนยันให้ |
| approved_at | timestamptz | วัน-เวลาที่ได้รับการอนุมัติ |
| prefix, organization, position, department, phone, province, health_zone, parent_organization, org_code, address, website | varchar/text | ชุดข้อมูลโปรไฟล์ (คำนำหน้า, องค์กร, ตำแหน่ง ฯลฯ) |
| reset_token, reset_token_expires | varchar / timestamptz | โทเค็นและเวลาหมดอายุสำหรับกระบวนการรีเซ็ตรหัสผ่าน |
| created_at, updated_at | timestamptz | วันที่สร้าง/แก้ไข (`updated_at` ทำงานผ่านระบบ Trigger) |

> [!warning] คำเตือนเชิงนโยบาย (Governance)
> โค้ดในไฟล์ `schema.sql` ปัจจุบันมีการตั้งรหัสผ่านดิบไว้สำหรับบัญชีตัวอย่าง (เช่น `musya@gmail.com`, `supermusya@gmail.com`, `musya01–50`) ซึ่งข้อมูลเหล่านี้ **ต้อง (must)** ถูกลบหรือเปลี่ยนรหัสผ่านใหม่ก่อนนำขึ้นใช้งานจริง (Production)

### `chat_sessions` — บันทึกเซสชันการแชท (DR-APP-02)

| ชื่อคอลัมน์ (Column) | ชนิดข้อมูล (Type) | คำอธิบาย (Notes) |
|---|---|---|
| id | UUID PK | กุญแจหลัก |
| session_id | text UNIQUE | รหัสอ้างอิงเซสชันที่ฝั่งแอปสร้างให้ (App-level session key) |
| user_id | UUID FK→accounts.id | ลิ้งก์เจ้าของ (กรณีลบบัญชี จะตั้งค่าเป็น NULL แทน `ON DELETE SET NULL`) |
| status | varchar | สถานะปัจจุบัน: `idle` \| `running` \| `done` \| `error` |
| last_user_prompt | text | คำถามล่าสุดที่ผู้ใช้พิมพ์ |
| messages_json | jsonb | ก้อนข้อมูลประวัติข้อความแบบ Array (User/Assistant) |
| created_at, updated_at | timestamptz | ใช้ `updated_at DESC` เพื่อเรียงรายการแถบเมนูด้านซ้าย |

### `journal_reports` — คลังรายงานนโยบาย (DR-APP-03)

| ชื่อคอลัมน์ (Column) | ชนิดข้อมูล (Type) | คำอธิบาย (Notes) |
|---|---|---|
| id | UUID PK | กุญแจหลัก |
| user_id | UUID FK→accounts.id | ลิ้งก์เจ้าของ หากลบบัญชีรายงานนี้จะหายไปตามกฎ `ON DELETE CASCADE` |
| title, query | text | หัวเรื่อง และ คำสั่งต้นทางที่ใช้สร้าง |
| doc_type | varchar | ชนิดเอกสาร `policy` \| `plan` \| `workplan` |
| article_count | int | จำนวนแหล่งที่มาที่ระบบอ้างอิง |
| topic_plan | text | โครงร่างแบบสั้น (Outline) ของรายงาน |
| html_content | text | เนื้อหาเอกสารตัวเต็มรูปแบบ HTML |
| created_at, updated_at | timestamptz | วันและเวลาที่จัดทำ |

## 6.3 ฐานข้อมูลการวิเคราะห์อุบัติเหตุแบบสตาร์คีมา (Accident analytics — star schema: `DR-ACC-*`)

เป็นโมเดลข้อมูลระดับอ่านอย่างเดียว (Read-only) ที่ถูกสืบค้นโดยท่อ Accident pipelines (d1)
- **ข้อมูลมุมมองมิติ (Dimensions):** `dim_geography` (จังหวัด→อำเภอ→ตำบล + พิกัดละติจูด/ลองจิจูด), `dim_time` (ปี 2020–2030 ที่เตรียมไว้), `dim_source`, `dim_road_segment`
- **ข้อมูลแฟคท์ตารางความจริง (Facts):** `fact_accident_event` (ตารางระดับเหตุการณ์ เก็บจำนวน, ความรุนแรง, สาเหตุ, สภาพอากาศ, ปี `csv_year`), และตาราง `fact_accident_person` (**ปัจจุบันข้อมูลเป็นความว่างเปล่า** จึงไม่มีการวิเคราะห์เรื่องเพศ อายุ หมวกกันน็อก หรือเข็มขัด)
- **ข้อมูลวิเคราะห์พร้อมใช้งาน (Marts):** `mart_accident_summary`, `mart_accident_hotspot`, `mart_province_year`, และ `mart_province_road` (ตารางหลักสำหรับ Policy Briefs แต่ตารางรายชื่อถนน `road_name` ส่วนใหญ่มักเป็นค่าความว่างเปล่า)
- **ข้อมูลมุมมอง (Views):** `v_province_year_summary`, `v_province_road_year`

> [!note] ตรรกะของปี พ.ศ. และ ค.ศ. (Year semantics — DR-ACC-YEAR)
> การจัดเก็บในปีตารางเป็นปี **ค.ศ. (ปี CE 2021–2026)** หากผู้ใช้ถามมาด้วยปี พ.ศ. (เช่น 2568) ระบบจะแปลงเป็น ค.ศ. (ลบด้วย 543) ก่อนลงไปสืบค้นตาราง และค่อยใส่ป้าย พ.ศ. คืนกลับมาในกระบวนการสรุปผลคำตอบ

## 6.4 ดัชนีโครงสร้างค้นหา RAG ของ Obsidian (`DR-KB-*`)

| ชื่อตาราง (Table) | ระดับข้อมูล (Grain) | วัตถุประสงค์ (Purpose) |
|---|---|---|
| `obsidian_vaults` | ตัวคลัง (vault) | ลงทะเบียนแฟ้มคลังความรู้ เช่น `health_region_10` |
| `obsidian_notes` | หน้าโน้ต (note) | เนื้อหา พร้อมพ่วงแท็ก จังหวัด/อำเภอ/ชนิด/ปี; ใช้ Trigram Index ครอบคลุมชื่อและเนื้อหาดิบที่ปอกแท็กออกแล้ว |
| `obsidian_note_chunks` | ส่วนย่อย (section) | เนื้อหาที่ถูกหั่นชิ้น (Chunking) เพื่อใช้วิ่งระบบค้นหา Hybrid Search ในเฟสแรก |
| `obsidian_note_links` | ความสัมพันธ์ (edge) | เชื่อมกราฟโยงใย (Wikilink graph) เอาไว้ขยายผลคำตอบ |
| `obsidian_pdf_assets` | ไฟล์เอกสาร (file) | เก็บ URL เช็คกับ MinIO โยงไปหาไฟล์ PDF |

## 6.5 โครงสร้างการจัดเก็บระดับ Object (Object storage: `DR-OBJ-*`)

| ถังข้อมูล (Bucket) | สิ่งที่จัดเก็บ (Content) |
|---|---|
| `fileapa` | ไฟล์ดิบผู้ใช้อัปโหลด, เอกสารต้นทางของการทำ APA, และชิ้นส่วนไฟล์รีพอร์ต (Report artefacts) |
| `pdf-library` | เอกสาร PDF ที่นำเข้ามาเพื่อรอดูดเข้ากระบวนการวิเคราะห์ในคลัง Vault |

ส่วนไฟล์ประเภท CSV (ที่ใช้ในโดเมน `d2–d4`) จะถูกเซฟโดยใช้ชื่อไฟล์เป็นรหัสตัวเลข (Numeric IDs) ส่วนชื่อไฟล์จริงๆ จะถูกแนบเป็น Metadata และใช้ระบบการค้นหาแบบคลุมเครือ (Fuzzy resolution) ในการจับคู่ (อาจมีโอกาสพลาดได้หากชื่อไฟล์มีความกำกวมเกินไป)

## 6.6 ระยะเวลาเก็บรักษาและวงจรชีวิต (Retention & lifecycle: `DR-LIFE-*`)

| รหัสอ้างอิง | กฎหมาย/ระยะเวลาการคงอยู่ (Rule) |
|---|---|
| DR-LIFE-01 | ประวัติแชทใน Redis จะหมดอายุสูญสลายใน **24 ชั่วโมง**; ส่วนในตาราง PostgreSQL `chat_sessions` จะอยู่ถาวร |
| DR-LIFE-02 | เมื่อบัญชีผู้ใช้ใดถูกลบออกไป ประวัติเซสชันของเขาจะไม่หายไปแต่ถูกป้ายเป็นไร้เจ้าของ (NULL) แทน แต่ข้อมูลที่อยู่ในฐานตาราง **journal reports จะถูกลบถาวรแบบลูกโซ่ (Cascades)** |
| DR-LIFE-03 | โทเค็นรีเซ็ตรหัสผ่านจะมีอายุอยู่จนกว่าจะถึงเวลาที่กำหนดในช่อง `reset_token_expires` และใช้งานได้เพียงครั้งเดียว |
| DR-LIFE-04 (ช่องโหว่) | ปัจจุบันข้อมูล Error Agent ในโฟลเดอร์ `error_logs/` จะถูกสะสมเพิ่มเป็นแบบรายวัน โดยที่ **ยังไม่มีนโยบายจำกัดวันหมดอายุ (No retention policy)** ซึ่งในอนาคตควรสร้างคำสั่งทำความสะอาดแฟ้มนี้ |
