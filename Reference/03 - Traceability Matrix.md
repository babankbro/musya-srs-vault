---
title: "Reference — Traceability Matrix (ตารางสอบย้อนกลับระบบ)"
tags: [MUSYA, reference, traceability, coverage]
created: 2026-07-18
---

# 🔗 ข้อมูลอ้างอิง: ตารางสอบย้อนกลับสถานะ (Traceability Matrix)

← กลับไป [[02 - Prompt Strategy & Anti-Hallucination]] · กลับสู่หน้าหลัก → [[index]]

> โน้ตฉบับนี้คือใยแมงมุมที่ "เชื่อมโยงโครงสร้างทั้งวอลต์เข้าด้วยกัน" ตั้งแต่หัวจรดหาง: **เอกสารข้อกำหนดความต้องการ (FR/NFR) ↔ เชื่อมสู่สถานการณ์ผู้ใช้ (UC) ↔ เชื่อมการออกแบบ/เวิร์กโฟลว์ ↔ มัดรวมกับเอเจนต์และโค้ด ↔ ก่อนจะสรุปด้วย "สถานะการทดสอบ (Test Status)"**
> วัตถุประสงค์เพื่อใช้เป็นแผงหน้าปัดตรวจจับว่า "ความต้องการข้อไหนถูกเอาไปเขียนโค้ดทำแล้ว" และที่สำคัญที่สุด **"จุดไหนบ้างที่แอบมีรูรั่ว/ยังไม่มีการตรวจทดสอบ/หรือเป็นช่องว่างในระบบ"**
> *คำอธิบายป้ายสถานะทดสอบ:* ✅ = ผ่านการยืนยันระบบจริงแล้ว · 🧪 = **มีชุดทดสอบอัตโนมัติ (pytest) คุมจริงแล้ว** · ⚠️ = โค้ดมีแล้ว แต่ยังไร้ชุดทดสอบอัตโนมัติ (Automated Test) มารองรับ · ❌ = ยังเป็นวุ้น/ยังไม่ได้ลงมือทำ/เป็นช่องโหว่

> [!success] 🆕 อัปเดตสำคัญ — วอลต์นี้เคยบอกว่า "ไม่มีเทสต์อัตโนมัติเลยสักตัว" ซึ่ง**ไม่จริงอีกต่อไปแล้ว**
> ตอนนี้ฝั่ง backend มีชุดทดสอบ **pytest จำนวน 43 เทสต์ ผ่านหมดทั้งชุด** อยู่ในโฟลเดอร์ `tests/` (ประกาศ dependency ไว้ใน `requirements.txt` + คอนฟิกที่ `pytest.ini`) แบ่งเป็น 4 ไฟล์:
> - `test_obsidian_fullcontext.py` — ตัวใหญ่สุด คุม anti-leak guard, streaming guard, follow-up extractor, `_clean_doc_title` และ **golden-set** ที่จำลอง Gemini ตอบเนื้อหาดิบหลุดแล้วยืนยันว่า retry/cleanup เอาอยู่
> - `test_analyze_report_gather.py` — คุมการประกอบอ้างอิงของ report-gather (`_obsidian_notes_to_articles_text`) รวมถึงล็อกบั๊กเรื่อง path สัมพัทธ์ไม่กลายเป็นลิงก์
> - `test_question_resolver.py` — คุมพรอมต์ + guard ของ Memory Agent
> - `test_thaijo_linkify.py` — คุม `_linkify_bare_urls` รวมถึงบั๊กวงเล็บเหลี่ยมถูกกลืนเข้าไปใน URL
>
> **ข้อควรรู้:** เทสต์เหล่านี้ **ไม่ได้ถูก COPY เข้า Docker image** (Dockerfile หยิบแค่ `main.py` กับ `src/`) จึงรันด้วย `docker compose exec` ตรง ๆ ไม่ได้ ต้อง mount โฟลเดอร์รีโพเข้าไปในคอนเทนเนอร์ชั่วคราวแทน — ดูคำสั่งเต็มใน [[00 - Setup & Onboarding Guide]]
>
> ⚠️ ฝั่ง **frontend ยังไม่มีเทสต์อัตโนมัติเลยแม้แต่ตัวเดียว** (ไม่มี jest/vitest/playwright ใน `package.json`) — ช่องโหว่ข้อนี้ยังเปิดอยู่เต็ม ๆ

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
| FR-CHAT-03 | [[02 - Chat & Domain Analysis\|UC-12]] | [[01 - Shared Agents (Memory & Router)]] | เอเจนต์หมอความจำ: `question_resolver` (Memory Agent) | 🧪 `tests/test_question_resolver.py` |
| FR-CHAT-04 | UC-06 ถึง 08 | [[01 - Shared Agents (Memory & Router)]] | เอเจนต์สับราง: `router.py` (Router/Classifier) | ⚠️ |
| FR-CHAT-05 | [[02 - Chat & Domain Analysis\|UC-07]] | [[02 - Statistics Workflow]] | ทีม: Accident 2-agent + อาวุธ: `accident_chat_sql` | ⚠️ |
| FR-CHAT-06 | [[02 - Chat & Domain Analysis\|UC-08]] | [[02 - Statistics Workflow]] | ทีม: แก๊งทุบ CSV 6-agent + อาวุธรันโค้ด: `execute_python_code` | ⚠️ |
| FR-CHAT-08 | UC-07 และ 13 | [[02 - Statistics Workflow]], [[06 - Report Generation Workflow]] | การ์ดด่านนอกเขต: out-of-zone guard + สมุดจด: `missing_data_logger` | ⚠️ |
| FR-CHAT-09 | UC-07 | [[04 - Data Architecture & Schema\|DR-ACC-YEAR]] | กลไกแปลง: พ.ศ.↔ค.ศ. แอบฝังใน `analyze.py` | ⚠️ |
| FR-CHAT-10 | [[02 - Chat & Domain Analysis\|UC-10]] | [[04 - Research Workflow]] | กลไกรันงานวิจัยคู่ขนาน: ThaiJo+PubMed parallel + 🆕 `_linkify_bare_urls` (บังคับห่อ URL เป็น `<a>`) | 🧪 `tests/test_thaijo_linkify.py` (เฉพาะส่วน linkify) |
| FR-CHAT-11 | [[02 - Chat & Domain Analysis\|UC-11]] | [[05 - Web Search Workflow]] | ทีมหาข้อมูลเน็ต: `tavily_pipeline` (จัดทีม 2-agent) | ⚠️ |
| FR-CHAT-12 | [[02 - Chat & Domain Analysis\|UC-09]] | [[03 - Knowledge Vault Workflow]] | ท่อคลังความรู้แบบตู้มเดียว: `obsidian_fullcontext` + 🆕 anti-leak guard/retry, สตรีมสด, follow-up แบบ JSON block, dedupe อ้างอิงรายเอกสาร | 🧪 `tests/test_obsidian_fullcontext.py` (รวม golden-set) |
| FR-CHAT-13 | [[02 - Chat & Domain Analysis\|UC-12]] | [[07 - Auth & Session Workflow]] | โค้ดจำอดีต: `history.py` (ฝาก Redis) + โค้ดจดแชท: `chat_sessions` | ⚠️ |
| FR-CHAT-14 | UC-06 | [[01 - SSE Event Protocol]] ส่วนที่ 6 | ภารโรงตามจดตอนเน็ตหลุด: `persistFallbackCompletion` (ท่า tee) | ⚠️ |
| FR-CHAT-15 | UC-06 | [[05 - Non-Functional Requirements\|NFR-PERF-02]] | วาล์วจำกัดคิว: `_AI_SEMAPHORE(5)` | ✅ (ตรวจเจอว่าใส่โค้ดไว้จริง) |
| 🆕 FR-CHAT-17 (กันเนื้อหาดิบหลุด) | [[02 - Chat & Domain Analysis\|UC-09]] | [[02 - Prompt Strategy & Anti-Hallucination]] ส่วนที่ 3.1 | ด่าน 3 ชั้นใน `obsidian_fullcontext.py`: พรอมต์ → เบรกมือระหว่างสตรีม → `_contains_leak`/retry/`_strip_leaked_blocks` | 🧪 `tests/test_obsidian_fullcontext.py` (รวม golden-set) |
| 🆕 FR-CHAT-18 (สตรีมคำตอบสด) | UC-09 | [[01 - SSE Event Protocol]] | `_call_gemini(on_delta=...)` → SSE `obsidian_stream_start` + `obsidian_chunk` | 🧪 (เฉพาะส่วน streaming guard) |
| 🆕 FR-CHAT-19 (เตือนว่าเป็นตัวเลขจากรายงาน ไม่ใช่ DB สด) | [[02 - Chat & Domain Analysis\|UC-07]] | [[03 - Knowledge Vault Workflow]] | `accident_redirected_to_obsidian` + ข้อความ notice นำหน้าคำตอบใน `analyze.py` | ⚠️ |
| 🆕 FR-CHAT-20 (relevance gate รายไฟล์) | [[02 - Chat & Domain Analysis\|UC-08]] | [[02 - Stats Tool Agents]] ส่วน B1.5 | `csv_pipeline._verify_file_relevance` + `history.get_verified_file_ids`/`mark_file_verified` (Redis) | ⚠️ |
| 🆕 FR-CHAT-21 (บรรณานุกรม APA ของอ้างอิงคลังความรู้) | [[02 - Chat & Domain Analysis\|UC-09]] | [[02 - Frontend Design]] ส่วนที่ 8 | `app/chat/obsidianApa.ts` (ตรรกะแกะชื่อไฟล์) + `app/component/chat/ApaReferences.tsx` (แสดงผล) + `LeftPane.tsx` | ⚠️ (ตรวจด้วยการรันจริง 10 เคสรวมเคสขอบ แต่ยังไม่มีเทสต์อัตโนมัติในรีโพ) |
| 🆕 FR-CHAT-22 (ปุ่มคัดลอกบรรณานุกรมพร้อม URL เต็ม) | UC-09 | [[02 - Frontend Design]] ส่วนที่ 8 | `formatApa()` + `toAbsoluteUrl()` ใน `obsidianApa.ts` | ⚠️ |

## 3. หมวดจัดทำรายงานวิชาการ (Report & Journal — `FR-REPORT-*`)

| รหัสข้อกำหนด (FR) | อ้างอิง Use Case | เวิร์กโฟลว์/การออกแบบ | เป้าหมายโค้ด/เอเจนต์ | สถานะ (Test) |
|---|---|---|---|---|
| FR-REPORT-01/02/03 | [[03 - Report Generation\|UC-13]] | [[06 - Report Generation Workflow]] | ทีมโจรสลัดปล้น 5 แหล่ง: report-gather 5-source + ตัวช่วย wizard | ⚠️ (ตัว pipeline ยังไม่มีเทสต์) |
| 🆕 FR-REPORT-09 (เลือกหัวข้อ + ชนิดเอกสาร **ก่อน** ระดมข้อมูล) | UC-13 | [[06 - Report Generation Workflow]] | ฝั่งหน้าจอ: `preGatherTopicsStore.ts`, ฝั่งหลังบ้าน: ฟิลด์ `doc_type` + `report_title` ใน `AnalyzeRequest` แล้ว echo กลับใน `final` | ⚠️ |
| 🆕 FR-REPORT-10 (ป้ายสถานะรายแหล่ง + ปุ่ม "ลองใหม่" รายแหล่ง) | UC-13 | [[06 - Report Generation Workflow]], [[01 - SSE Event Protocol]] | event `report_source_status`, โหมด `report-gather-retry`, ฝั่งหน้าจอ: `reportSourceStore.ts` + `ReportSourceBadges.tsx` + `reportRetry.ts` | ⚠️ |
| 🆕 FR-REPORT-11 (กู้คืนความคืบหน้า wizard + ปุ่มรายงานที่เซฟแล้ว หลัง reload) | UC-13, UC-14 | [[02 - Frontend Design]] | `wizardPersist.ts`, `reportSavePersist.ts` (ยัดลง `messages_json` ของ `chat_sessions`) | ⚠️ |
| FR-REPORT-04 | UC-13 | [[04 - Research Workflow]] | ทีมวิจัย: ThaiJo Planner/Generator ([[06 - Report & Policy Agents]]) | ⚠️ |
| FR-REPORT-05/06/07 | [[03 - Report Generation\|UC-14]] | [[06 - Report Generation Workflow]] | ตารางเก็บตู้เซฟ: `journal-reports`, โค้ดจัดหน้าตา: `journal-template/*` | ⚠️ |
| FR-REPORT-08 | UC-13 | [[02 - Prompt Strategy & Anti-Hallucination]] | กลไกห่ออ้างอิงแยกแหล่ง: citation แยกแหล่ง (ใน `analyze.py`) — 🆕 ตอนนี้เอกสารคลังความรู้ (Obsidian) ไหลเข้าอ้างอิงด้วยแล้วผ่าน `_obsidian_notes_to_articles_text()` + `PUBLIC_APP_URL` | 🧪 `tests/test_analyze_report_gather.py` |

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
> 1. **ชุดทดสอบอัตโนมัติ — ดีขึ้นแล้วแต่ยังไม่พอ (อัปเดต)** — ฝั่ง backend มี pytest 43 เทสต์ผ่านหมดแล้ว (ดูกล่อง success ด้านบน) แต่ยัง**กระจุกอยู่แค่ 4 จุด**คือ obsidian guard / report-gather citation / question resolver / thaijo linkify เท่านั้น ส่วนที่ยัง**ไม่มีเทสต์เลย**ได้แก่ Router/Classifier, Accident SQL 2-agent, แก๊ง CSV 6-agent, ท่อ report-gather ตัวจริง, ระบบ auth ทั้งชุด และ **ฝั่ง frontend ทั้งโปรเจกต์** (ยังไม่มี jest/vitest/playwright) → Epic E13 ยังเปิดอยู่ แค่คืบไปหนึ่งก้าว
> 2. **ความสวิงของผลลัพธ์ (Determinism) — คืบไปบางส่วน (อัปเดต)** — ท่อ CSV ล็อก `temperature=0.1` ไปแล้วที่ `_get_llm()` (ทั้ง folder nav / relevance verifier / schema / code-gen / insight) และท่อ Obsidian ใช้ `temperature=0.2` — **แต่ยังไม่ใช่ 0 เป๊ะ และยังไม่มีการล็อก seed หรือทำแคชคำตอบ** โค้ดเองก็ยอมรับตรง ๆ ว่ายังต้องมี safety net ถามซ้ำ 2 ครั้งใน relevance gate เพราะ "ยังไม่ deterministic พอ" เชิญไปอ่านต่อที่ [[02 - Prompt Strategy & Anti-Hallucination]] ส่วนที่ 6 (โปรเจกต์ Epic E1)
> 3. **ระเบิดเวลา! รหัสผี (seed passwords + dev secrets)** กองพะเนินอยู่ในซอร์สโค้ด (ขัดกับ NFR-SEC-08 เต็มๆ) — เป็นกฎเหล็กที่ต้องสั่งประหารลบทิ้งให้เหี้ยน (❌) ก่อนจะยกเซิร์ฟเวอร์เปิดให้ชาวบ้านใช้จริง (Production)
> 4. **ตะแกรงร่อนกฎหมายคุ้มครองข้อมูลส่วนบุคคล (PDPA screening)** สำหรับสอดส่องไฟล์เอกสารที่ผู้ใช้อัปโหลด — โปรเจกต์นี้ก็ยังไม่ได้เริ่มต้นเขียนเลยแม้แต่บรรทัดเดียว ([[05 - Non-Functional Requirements|NFR-SEC-08]])
> 5. **ไม่มีนโยบายการกำจัดขยะทิ้งในถัง `error_logs/` (No retention policy)** — ขยะจะกองพูนขึ้นเรื่อยๆ ไปชั่วกัลปาวสาน ([[06 - Data Requirements|DR-LIFE-04]])
> 6. **หนี้กรรมทางเทคนิคที่กองสุม (Technical Debt)** — ไม่ว่าจะเป็น การสร้างเอเจนต์ออกมาวิ่งชนทับซ้อนหน้าที่กัน (`obsidian_agent` ไปซ้ำกับ `obsidian_fullcontext`), หรือพวกตระกูล hardcode ชื่อโดเมน (domain codes), รวมถึงระบบโชว์แถบสถานะดุ๊กดิ๊ก `progress.py` ที่ยังเขียนแบบ hardcode ไว้เช่นกัน (เชิญไปตามเก็บได้ที่ [[00 - Architecture Overview]] ส่วนที่ 6)

## 7. คู่มือการนำตารางฉบับนี้ไปใช้งาน
- **ใช้เช็คสุขภาพความครบถ้วน (Completeness Check):** กฎคือ ข้อกำหนด FR ทุกข้อบนโลก จะต้องมีเป้าหมาย Design อย่างน้อย 1 อัน + ต้องชี้พิกัดไปถึง 1 โค้ด เสมอ — ถ้าช่องไหนโบ๋ นั่นแหละคือช่องโหว่ (Gap) จากการออกแบบ
- **ใช้กางแผนรบฝั่ง QA (Test Planning):** จับตาดูแถวสัญลักษณ์ ⚠️ หรือ ❌ ให้ดี — พวกมันคือเป้าหมายสำคัญลำดับแรกสุดสำหรับทีมทดสอบ (QA) ที่จะต้องลงไปขุดรันเทสต์
- **ใช้เป็นโล่กำบังตอนส่งงาน (Audit / Handover):** เอาตารางกางโชว์ให้ผู้ประเมินระบบดูเป็นหลักฐาน เพื่อโชว์ความครบถ้วนอลังการว่าระบบได้ตอบโจทย์ความต้องการครบทุกมิติแล้ว

*วาร์ปเอกสารเชื่อมโยง: ไปอ่านสเปคข้อกำหนดได้ที่ [[03 - Functional Requirements]] · กับ [[05 - Non-Functional Requirements]] · หรือดูแผนผังตัวละครใช้ระบบ [[00 - Actors & Use-Case Map]] · และสามารถเข้าไปตามเผือกโครงการเฟส 2 (Epics) ลับๆ ได้ที่วอลต์เพื่อนบ้าน `musya-obsidian-document`*
