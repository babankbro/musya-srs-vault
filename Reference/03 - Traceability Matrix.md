---
title: "Reference — Traceability Matrix (ตารางสอบย้อนกลับระบบ)"
tags: [MUSYA, reference, traceability, coverage]
created: 2026-07-18
---

# 🔗 ข้อมูลอ้างอิง: ตารางสอบย้อนกลับสถานะ (Traceability Matrix)

← กลับไป [[02 - Prompt Strategy & Anti-Hallucination]] · กลับสู่หน้าหลัก → [[index]]

> โน้ตฉบับนี้คือใยแมงมุมที่ "เชื่อมโยงโครงสร้างทั้งวอลต์เข้าด้วยกัน" ตั้งแต่หัวจรดหาง: **เอกสารข้อกำหนดความต้องการ (FR/NFR) ↔ เชื่อมสู่สถานการณ์ผู้ใช้ (UC) ↔ เชื่อมการออกแบบ/เวิร์กโฟลว์ ↔ มัดรวมกับเอเจนต์และโค้ด ↔ ก่อนจะสรุปด้วย "สถานะการทดสอบ (Test Status)"**
> วัตถุประสงค์เพื่อใช้เป็นแผงหน้าปัดตรวจจับว่า "ความต้องการข้อไหนถูกเอาไปเขียนโค้ดทำแล้ว" และที่สำคัญที่สุด **"จุดไหนบ้างที่แอบมีรูรั่ว/ยังไม่มีการตรวจทดสอบ/หรือเป็นช่องว่างในระบบ"**
> *คำอธิบายป้ายสถานะทดสอบ:* ✅ = ผ่านการยืนยันระบบจริงแล้ว · ⚠️ = โค้ดมีแล้ว แต่ยังไร้ชุดทดสอบอัตโนมัติ (Automated Test) มารองรับ · ❌ = ยังเป็นวุ้น/ยังไม่ได้ลงมือทำ/เป็นช่องโหว่

---

## 1. หมวดความปลอดภัย ระบบบัญชี & สิทธิ (Auth & Account — `FR-AUTH-*`, `FR-ADMIN-*`)

| รหัสข้อกำหนด (FR) | อ้างอิง Use Case | เวิร์กโฟลว์/การออกแบบ | เป้าหมายโค้ด/เอเจนต์ | สถานะ (Test) |
|---|---|---|---|---|
| FR-AUTH-01/02/03 | [[01 - Account & Access\|UC-01]] | [[07 - Auth & Session Workflow]] | API: `app/api/auth/register`, และ `lib/auth.ts` | ⚠️ |
| FR-AUTH-04/05 | [[01 - Account & Access\|UC-02]] | [[02 - Frontend Design]] ส่วนที่ 3 | API: `auth/login`, เครื่องปั่น: `signToken` | ✅ (ทดสอบ login รันได้จริง) |
| FR-AUTH-07 | [[01 - Account & Access\|UC-03]] | [[07 - Auth & Session Workflow]] | API: `forgot/reset-password` | ⚠️ |
| FR-AUTH-08 | [[01 - Account & Access\|UC-04]] | — | API: `auth/me` (ใช้เมธอด PATCH) | ⚠️ |
| FR-AUTH-09 | สอดแทรกทุก UC | [[02 - Frontend Design]] | การ์ดหน้าด่าน: `requireAuth()` (ระบบนี้ไม่มี `middleware.ts`) | ✅ |
| FR-ADMIN-01/02/03 | [[01 - Account & Access\|UC-05]] | [[07 - Auth & Session Workflow]] | API: `auth/users` (สิทธิระดับ adminsuper) | ⚠️ |

## 2. หมวดพูดคุย & ทีมวิเคราะห์ (Chat & Analysis — `FR-CHAT-*`)

| รหัสข้อกำหนด (FR) | อ้างอิง Use Case | เวิร์กโฟลว์/การออกแบบ | เป้าหมายโค้ด/เอเจนต์ | สถานะ (Test) |
|---|---|---|---|---|
| FR-CHAT-01/07 | [[02 - Chat & Domain Analysis\|UC-06]] | [[01 - Chat Normal Workflow]] | โค้ดหลัก: `analyze.py`, โปรโตคอล: SSE ([[01 - SSE Event Protocol]]) | ✅ (stack เทสต์ระบบรันได้จริง) |
| FR-CHAT-02 | UC-06 ถึง 11 | [[02 - Frontend Design]] ส่วนที่ 5 | โค้ดคำนวณโหมด: `ChatInput` ฟังก์ชัน `getEffectiveMode` | ✅ |
| FR-CHAT-03 | [[02 - Chat & Domain Analysis\|UC-12]] | [[01 - Shared Agents (Memory & Router)]] | เอเจนต์หมอความจำ: `question_resolver` (Memory Agent) | ⚠️ |
| FR-CHAT-04 | UC-06 ถึง 08 | [[01 - Shared Agents (Memory & Router)]] | เอเจนต์สับราง: `router.py` (Router/Classifier) | ⚠️ |
| FR-CHAT-05 | [[02 - Chat & Domain Analysis\|UC-07]] | [[02 - Statistics Workflow]] | ทีม: Accident 2-agent + อาวุธ: `accident_chat_sql` | ⚠️ |
| FR-CHAT-06 | [[02 - Chat & Domain Analysis\|UC-08]] | [[02 - Statistics Workflow]] | ทีม: แก๊งทุบ CSV 6-agent + อาวุธรันโค้ด: `execute_python_code` | ⚠️ |
| FR-CHAT-08 | UC-07 และ 13 | [[02 - Statistics Workflow]], [[06 - Report Generation Workflow]] | การ์ดด่านนอกเขต: out-of-zone guard + สมุดจด: `missing_data_logger` | ⚠️ |
| FR-CHAT-09 | UC-07 | [[04 - Data Architecture & Schema\|DR-ACC-YEAR]] | กลไกแปลง: พ.ศ.↔ค.ศ. แอบฝังใน `analyze.py` | ⚠️ |
| FR-CHAT-10 | [[02 - Chat & Domain Analysis\|UC-10]] | [[04 - Research Workflow]] | กลไกรันงานวิจัยคู่ขนาน: ThaiJo+PubMed parallel | ⚠️ |
| FR-CHAT-11 | [[02 - Chat & Domain Analysis\|UC-11]] | [[05 - Web Search Workflow]] | ทีมหาข้อมูลเน็ต: `tavily_pipeline` (จัดทีม 2-agent) | ⚠️ |
| FR-CHAT-12 | [[02 - Chat & Domain Analysis\|UC-09]] | [[03 - Knowledge Vault Workflow]] | ท่อคลังความรู้แบบตู้มเดียว: `obsidian_fullcontext` | ⚠️ |
| FR-CHAT-13 | [[02 - Chat & Domain Analysis\|UC-12]] | [[07 - Auth & Session Workflow]] | โค้ดจำอดีต: `history.py` (ฝาก Redis) + โค้ดจดแชท: `chat_sessions` | ⚠️ |
| FR-CHAT-14 | UC-06 | [[01 - SSE Event Protocol]] ส่วนที่ 6 | ภารโรงตามจดตอนเน็ตหลุด: `persistFallbackCompletion` (ท่า tee) | ⚠️ |
| FR-CHAT-15 | UC-06 | [[05 - Non-Functional Requirements\|NFR-PERF-02]] | วาล์วจำกัดคิว: `_AI_SEMAPHORE(5)` | ✅ (ตรวจเจอว่าใส่โค้ดไว้จริง) |

## 3. หมวดจัดทำรายงานวิชาการ (Report & Journal — `FR-REPORT-*`)

| รหัสข้อกำหนด (FR) | อ้างอิง Use Case | เวิร์กโฟลว์/การออกแบบ | เป้าหมายโค้ด/เอเจนต์ | สถานะ (Test) |
|---|---|---|---|---|
| FR-REPORT-01/02/03 | [[03 - Report Generation\|UC-13]] | [[06 - Report Generation Workflow]] | ทีมโจรสลัดปล้น 5 แหล่ง: report-gather 5-source + ตัวช่วย wizard | ⚠️ |
| FR-REPORT-04 | UC-13 | [[04 - Research Workflow]] | ทีมวิจัย: ThaiJo Planner/Generator ([[06 - Report & Policy Agents]]) | ⚠️ |
| FR-REPORT-05/06/07 | [[03 - Report Generation\|UC-14]] | [[06 - Report Generation Workflow]] | ตารางเก็บตู้เซฟ: `journal-reports`, โค้ดจัดหน้าตา: `journal-template/*` | ⚠️ |
| FR-REPORT-08 | UC-13 | [[02 - Prompt Strategy & Anti-Hallucination]] | กลไกห่ออ้างอิงแยกแหล่ง: citation แยกแหล่ง (ใน `analyze.py`) | ⚠️ |

## 4. หมวดห้องสมุด / แฟ้มเอกสาร / และงานแอดมิน (Knowledge / Files / Ops — `FR-KB-*`, `FR-FILE-*`, `FR-OPS-*`)

| รหัสข้อกำหนด (FR) | อ้างอิง Use Case | เวิร์กโฟลว์/การออกแบบ | เป้าหมายโค้ด/เอเจนต์ | สถานะ (Test) |
|---|---|---|---|---|
| FR-KB-01 | [[04 - Knowledge, Files & Admin\|UC-16]] | [[03 - Knowledge Vault Workflow]] | API ค้นหา: `/api/obsidian/search` (ใช้วิชา pg_trgm) | ⚠️ |
| FR-KB-03/04 | UC-16 | [[03 - Knowledge Vault Workflow]] | ภารโรงจัดไฟล์: `index_obsidian.py`, และ `sync_obsidian_pdfs.py` | ✅ (ตรวจพบสคริปต์ทำงานจริง) |
| FR-FILE-01/02/03 | [[04 - Knowledge, Files & Admin\|UC-15]] | [[07 - API Reference — Frontend (BFF)]] ส่วนที่ 3 | API อัปไฟล์: `files/*`, คนปั้นบรรณานุกรม: `generate-apa`, เครื่องมือ: `lib/apa.ts` | ⚠️ |
| FR-OPS-01 | [[04 - Knowledge, Files & Admin\|UC-17]] | [[06 - API Reference — Backend]] ส่วนที่ 8 | API แอบส่องเบส: `db_explorer.py` (validate) | ⚠️ |
| FR-OPS-02/03 | UC-18 | [[06 - API Reference — Backend]] ส่วนที่ 9 | สมุดพกพฤติกรรม: `error_logger`, ยามเฝ้าระวัง: `error_monitor_agent` | ⚠️ |
| FR-OPS-04 | — | [[00 - Setup & Onboarding Guide]] | จุดเช็คชีพจร: API `/health`, คู่มืออัตโนมัติ: `/docs` | ✅ |

## 5. หมวดระบบเชิงสมรรถนะ (Non-Functional — `NFR-*`)

| รหัสสเปคระบบ (NFR) | การออกแบบรองรับ | เป้าหมายโค้ด | สถานะ (Test) |
|---|---|---|---|
| NFR-PERF-02 (กั้นคิวรับคนไม่เกิน semaphore≤5) | [[01 - Backend Design]] ส่วนที่ 7 | ท่อกั้น: `_AI_SEMAPHORE` | ✅ |
| NFR-PERF-03 (ระบบดักรอ timeout ห้ามเกิน 10m) | [[02 - Frontend Design]] ส่วนที่ 4 | ฝั่ง BFF: `chat/route.ts` ใช้งาน AbortSignal | ✅ |
| NFR-PERF-05 (ท่าหน่วงจังหวะปล่อยตัว stagger 1.5s) | [[06 - Report Generation Workflow]] | ตัวปล่อยตัวใน: `analyze.py` บล็อก report-gather | ⚠️ |
| NFR-REL-01 (ระบบตั้งการ์ดสู้ error 429 backoff) | [[00 - Agent Catalogue]] | ยันต์กันแครช: `agent_defaults.py` | ⚠️ |
| NFR-REL-03 (ระบบเซฟเซสชันปลอดภัยตัดตอนเน็ตหลุด) | [[01 - SSE Event Protocol]] | โค้ดไม้ตาย: `persistFallbackCompletion` | ⚠️ |
| NFR-SEC-01/02/03 (บังคับรักษาความปลอดภัย JWT/bcrypt) | [[02 - Frontend Design]] ส่วนที่ 3 | โค้ดเข้ารหัส: `lib/auth.ts` | ✅ (ระบบตั้งค่าปิดตาย fail-closed) |
| NFR-SEC-04 (ใช้กุญแจหลังบ้าน internal key) | [[00 - Architecture Overview]] | การ์ดผ่านทาง: `internalFetch.ts` | ✅ |
| NFR-SEC-05 (ระบบแบ่งชั้นวรรณะสิทธิ RBAC) | [[01 - Account & Access\|UC-05]] | จุดสกัด: middleware/route สำหรับเช็ค role | ⚠️ |
| NFR-SEC-07 (DB บังคับให้เป็นโหมดอ่านอย่างเดียว read-only) | [[06 - API Reference — Backend]] ส่วนที่ 8 | จุดตรวจสอบ: `db_explorer` validate | ⚠️ |
| NFR-SEC-08 (มาตรการโรเททสลับรหัสลับ rotate secrets) | [[00 - Setup & Onboarding Guide]] ส่วนที่ 5 | ⚠️ ตรวจพบรหัสผี seed passwords ปลูกทิ้งไว้เพียบใน `schema.sql` | ❌ (ยังไม่ได้กวาดล้าง) |

---

## 6. 🔥 สรุปรายการช่องโหว่และจุดสลบ (Coverage Gaps) — กองงานที่ดองรอไว้ให้ลุยต่อ

> [!warning] ระวัง: นี่คือพิกัดจุดที่ยังบอดสนิท (ไม่มีการทดสอบ) หรือยังคงเป็นช่องโหว่เรื้อรัง
> 1. **อาการสาหัส! โลกนี้ไร้ซึ่งชุดทดสอบอัตโนมัติ (unit/integration/E2E)** — หากสังเกตตาราง จะเห็นได้ว่าเกือบทุกความต้องการ (FR) ถูกปั๊มตราสัญลักษณ์ ⚠️ เต็มไปหมด → วาระซ่อนเร้นคือ เราต้องลุกขึ้นมาเขียน test suite ให้ระบบโดยด่วน (ลากไปเชื่อมกับโปรเจกต์กู้ชาติ Epic E13)
> 2. **ความสวิงของผลลัพธ์ (Determinism)** — AI ยังอารมณ์ศิลปิน ตอบสองรอบได้ข้อความไม่นิ่ง (เพราะเรายังไม่ยอมตั้งค่า temp=0 / ล็อก seed / หรือขุดบ่อทำแคช) เชิญไปอ่านบ่นได้ที่ [[02 - Prompt Strategy & Anti-Hallucination]] ส่วนที่ 6 (โปรเจกต์ Epic E1)
> 3. **ระเบิดเวลา! รหัสผี (seed passwords + dev secrets)** กองพะเนินอยู่ในซอร์สโค้ด (ขัดกับ NFR-SEC-08 เต็มๆ) — เป็นกฎเหล็กที่ต้องสั่งประหารลบทิ้งให้เหี้ยน (❌) ก่อนจะยกเซิร์ฟเวอร์เปิดให้ชาวบ้านใช้จริง (Production)
> 4. **ตะแกรงร่อนกฎหมายคุ้มครองข้อมูลส่วนบุคคล (PDPA screening)** สำหรับสอดส่องไฟล์เอกสารที่ผู้ใช้อัปโหลด — โปรเจกต์นี้ก็ยังไม่ได้เริ่มต้นเขียนเลยแม้แต่บรรทัดเดียว ([[05 - Non-Functional Requirements|NFR-SEC-08]])
> 5. **ไม่มีนโยบายการกำจัดขยะทิ้งในถัง `error_logs/` (No retention policy)** — ขยะจะกองพูนขึ้นเรื่อยๆ ไปชั่วกัลปาวสาน ([[06 - Data Requirements|DR-LIFE-04]])
> 6. **หนี้กรรมทางเทคนิคที่กองสุม (Technical Debt)** — ไม่ว่าจะเป็น การสร้างเอเจนต์ออกมาวิ่งชนทับซ้อนหน้าที่กัน (`obsidian_agent` ไปซ้ำกับ `obsidian_fullcontext`), หรือพวกตระกูล hardcode ชื่อโดเมน (domain codes), รวมถึงระบบโชว์แถบสถานะดุ๊กดิ๊ก `progress.py` ที่ยังเขียนแบบ hardcode ไว้เช่นกัน (เชิญไปตามเก็บได้ที่ [[00 - Architecture Overview]] ส่วนที่ 6)

## 7. คู่มือการนำตารางฉบับนี้ไปใช้งาน
- **ใช้เช็คสุขภาพความครบถ้วน (Completeness Check):** กฎคือ ข้อกำหนด FR ทุกข้อบนโลก จะต้องมีเป้าหมาย Design อย่างน้อย 1 อัน + ต้องชี้พิกัดไปถึง 1 โค้ด เสมอ — ถ้าช่องไหนโบ๋ นั่นแหละคือช่องโหว่ (Gap) จากการออกแบบ
- **ใช้กางแผนรบฝั่ง QA (Test Planning):** จับตาดูแถวสัญลักษณ์ ⚠️ หรือ ❌ ให้ดี — พวกมันคือเป้าหมายสำคัญลำดับแรกสุดสำหรับทีมทดสอบ (QA) ที่จะต้องลงไปขุดรันเทสต์
- **ใช้เป็นโล่กำบังตอนส่งงาน (Audit / Handover):** เอาตารางกางโชว์ให้ผู้ประเมินระบบดูเป็นหลักฐาน เพื่อโชว์ความครบถ้วนอลังการว่าระบบได้ตอบโจทย์ความต้องการครบทุกมิติแล้ว

*วาร์ปเอกสารเชื่อมโยง: ไปอ่านสเปคข้อกำหนดได้ที่ [[03 - Functional Requirements]] · กับ [[05 - Non-Functional Requirements]] · หรือดูแผนผังตัวละครใช้ระบบ [[00 - Actors & Use-Case Map]] · และสามารถเข้าไปตามเผือกโครงการเฟส 2 (Epics) ลับๆ ได้ที่วอลต์เพื่อนบ้าน `musya-obsidian-document`*
