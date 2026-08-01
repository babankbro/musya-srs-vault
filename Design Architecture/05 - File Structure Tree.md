---
title: "Design — File Structure Tree (ผังโครงสร้างต้นไม้ของไฟล์ระบบ Frontend & Backend)"
tags: [MUSYA, design, structure, files, tree]
created: 2026-07-18
---

# 🌲 แผนผังโครงสร้างไฟล์และไดเรกทอรีของระบบ (File Structure Tree)

← [[04 - Data Architecture & Schema]] · กลับสู่ภาพรวม → [[00 - Architecture Overview]]

> นี่คือแผนผังที่ถูกสกัดภาพจำลองแบบย่อส่วนมาจากโครงสร้างไฟล์ของโปรเจกต์ของจริงในโฟลเดอร์ `new_version_musya/` 
> (โดยทำการตัดทอนส่วนขยะ หรือโฟลเดอร์รกๆ ออก เช่น `node_modules`, `.next`, `__pycache__`, และแฟ้มเอกสาร `.md` ของ Vault จำนวนมหาศาล เพื่อให้กวาดสายตาจับใจความได้ง่าย)
> หากคุณต้องการเจาะลึกการอธิบายเบื้องหลังการทำงานแต่ละส่วนโดยละเอียด ให้ติดตามอ่านได้ที่หน้า [[01 - Backend Design]] และ [[02 - Frontend Design]]

---

## 1. ⚙️ แดนหลังบ้าน Backend — โปรเจกต์ `chatapi.python` (FastAPI + CrewAI)

```
chatapi.python/
├── main.py                     ← 🚀 จุดสตาร์ทจุดแรก: ใช้ปั้นแอป FastAPI ขึ้นมา, สั่งลงทะเบียนสายทาง 10 routers, ยึดพื้นที่เมาท์โฟลเดอร์ /static, และปล่อยหน้าทดสอบที่ /ui
├── requirements.txt            ← 📦 รายการไลบรารี Python หลัก (มี fastapi, crewai, litellm, psycopg2, minio, redis และ 🆕 pytest)
├── pytest.ini                  ← 🆕 ⚙️ คอนฟิกชุดทดสอบ
├── tests/                      ← 🆕 🧪 ชุดทดสอบ pytest **43 เทสต์ ผ่านหมด** (⚠️ ไม่ถูก COPY เข้า image — Dockerfile หยิบแค่ main.py กับ src/)
│   ├── test_obsidian_fullcontext.py  ← ตัวใหญ่สุด: anti-leak guard, streaming guard, follow-up extractor, _clean_doc_title + golden-set
│   ├── test_analyze_report_gather.py ← การประกอบอ้างอิงของ report-gather (_obsidian_notes_to_articles_text)
│   ├── test_question_resolver.py     ← พรอมต์ + guard ของ Memory Agent
│   └── test_thaijo_linkify.py        ← _linkify_bare_urls (รวมบั๊กวงเล็บเหลี่ยมถูกกลืนเข้า URL)
├── Dockerfile                  ← 🐳 สูตรผสมเตรียมสร้างอิมเมจ (อิงพื้นฐาน python:3.12-slim)
├── docker-compose.yml          ← ไฟล์ compose สำหรับเปิดรันรันแบบฉายเดี่ยว Standalone (สำหรับฝั่ง Dev เอาไว้เทสเดี่ยวๆ)
├── AGENTS.md                   ← 📄 แผ่นพับแคตตาล็อกแนะนำตัวละครเอเจนต์ทั้งหมด (ไฟล์เอกสาร)
├── STRUCTURE.md                ← 📄 เอกสารชี้แจงโครงสร้างโปรเจกต์ (ไฟล์เอกสาร)
├── AI_CSV_Data_Analyst_System.md ← 📄 เอกสารเผยดีไซน์เคล็ดลับ pipeline ที่ใช้วิเคราะห์ CSV ผ่าน 6 ด่านอรหันต์ (ไฟล์เอกสาร)
├── check_obsidian.py           ← 🔧 สคริปต์คุณหมอช่วยวินิจฉัยตรวจสอบตารางระบบ Obsidian
├── check_paths.py              ← 🔧 สคริปต์ตรวจความถูกต้องของเส้นทางไฟล์ path
├── list_models.py              ← 🔧 สคริปต์สแกนตรวจสอบว่ามีโมเดล LLM ตัวไหนเปิดใช้งานได้บ้าง
├── db_check_out.txt            ← ซากผลลัพธ์ dump จากคำสั่งตรวจสุขภาพฐานข้อมูล
├── musya_*.dump                ← ไฟล์แฟ้มกู้วิกฤตสำรองฐานข้อมูล (ที่ได้จากการทำ pg_dump)
├── data/                       ← โกดังพักเก็บข้อมูลดิบ
├── database/                   ← 🗄️ กรุคำสั่ง SQL สำหรับปั้น Schema + เครื่องมือ migration ย้ายฐาน (เจาะลึกที่ [[04 - Data Architecture & Schema]])
│   ├── schema.sql              ← ตัวสั่งเนรมิต star schema แดนอุบัติเหตุ + ตารางฝั่งแอป (accounts/sessions/reports)
│   ├── accident.sql            ← ก้อนกระสอบข้อมูลสถิติอุบัติเหตุหนักๆ (ขนาดใหญ่โตมโหฬารร่วม ~74,000 บรรทัด)
│   ├── 025_obsidian_knowledge.sql  ← ตัวเนรมิตตาราง vault + notes (พร้อมฝัง pg_trgm)
│   ├── 026_obsidian_hybrid.sql     ← ตัวเนรมิต chunks ส่วนย่อย + เส้นทางใยแมงมุม wikilink graph
│   ├── 027_obsidian_pdf_assets.sql ← ตัวใช้ผูกพันธะ โยงหน้า note ↔ ให้คู่กับไฟล์ PDF ใน MinIO
│   ├── 028_md_to_db.sql        ← ตัวสูบ md เข้า db
│   ├── 030_populate_mart_province_road.sql ← สูตรคำนวณเติมอัปเดตข้อมูลฝั่ง mart
│   ├── 031_llm_settings.sql    ← 🆕 ตารางเก็บค่าย LLM ที่ผู้ดูแลตั้งได้ (key/model/enabled + updated_by)
│   ├── 032_csv_data_dict.sql   ← 🆕 พจนานุกรมข้อมูลของแต่ละ CSV (ความหมายคอลัมน์ · ขอบเขต · ข้อควรระวัง)
│   ├── 033_hdc_import.sql      ← 🆕 ทะเบียนของที่นำเข้าจาก HDC (ตาราง · ปีที่ใช้ได้ · เวลาซิงก์ · date_com)
│   ├── 034_data_dict_definition.sql ← 🆕 เพิ่มช่องนิยาม/ตัวตั้ง/ตัวหารจากหน้ารายงาน HDC
│   ├── 035_kpi_target.sql      ← 🆕 เพิ่มช่องค่าเป้าหมายตัวชี้วัด
│   ├── obsidian_schema.sql     ← สคีมาเฉพาะกิจ
│   └── migrate.py              ← 🔧 ตัวรันสคริปต์ migration สั่งอัปเดตสคีมา
└── src/
    ├── config.py               ← ⚙️ กล่องดวงใจ Settings (ผ่านอิง Pydantic) ที่สั่งคอยสูบค่าจากไฟล์แอบซ่อน .env + สั่งยึด cache แบบ lru_cache
    ├── domains.py              ← นิยามกำหนดรายชื่อหน้าตาโดเมนทั้งหลาย เช่น d0–d4, dt, หรือแก๊ง obsidian
    ├── history.py              ← สมุดจดประวัติแชทระยะสั้นบน Redis (จำได้ 6 ก้าว, มีเวลาสลายตัว 24 ชม.)
    ├── db/
    │   └── pool.py             ← 🚰 แหล่งน้ำบ่อจัดการท่อ PostgreSQL ThreadedConnectionPool (กติกาคือเปิดท่อล่อต่ำสุด 2 / ลิมิตทะลุห้ามเกิน 20 เส้น)
    ├── routers/                🚪 13 กรุ๊ปสายทางพนักงานรับแขก Endpoint (ตรวจกับ `main.py` 1 ส.ค. 2569)
    │   ├── analyze.py          ← ★ ประตูใหญ่บานหลัก: ระบบ Memory→สับราง Router→เข้าสายพาน pipeline, บรอดแคสต์ SSE, พร้อมตู้ล็อก semaphore ≤ 5 งาน
    │   ├── accident_chat.py    ← ระบบโต๊ะสนทนาเจาะลึกอุบัติเหตุ (ใช้หลักจับคู่ 2-agent ช่วยกัน) + มีทางด่วน /quick ให้
    │   ├── accident_policy.py  ← ระบบวิเคราะห์เขียน Policy Brief รายเขตสุขภาพ 10 (เข้าเป้า /zone10)
    │   ├── obsidian.py         ← ศูนย์อำนวยการค้น/ซักถาม/ปั้นดัชนี/อัปเดต sync ภายในถัง vault
    │   ├── thaijo.py           ← ศูนย์ล้วงงานวิจัย ThaiJo → เอามามิกซ์สังเคราะห์ใหม่
    │   ├── pubmed.py           ← ศูนย์ควานหาวรรณกรรมแพทย์สากล PubMed
    │   ├── tools_router.py     ← โถงแจกจ่ายทางเดินเครื่องมือวิเคราะห์ เช่น /compare /report /workplan /database
    │   ├── db_explorer.py      ← ประตูช่องเล็กสำหรับนักสำรวจ DB แบบแตะตาดูได้อย่างเดียว
    │   ├── pdf_ingest.py       ← ประตูป้อนยัดกระดาษ PDF กินกลืนเป็นก้อนเล็กๆ chunk
    │   ├── error_log.py        ← ตู้เปิดดูประวัติ/ให้สรุปอาการแผลงๆ error log
    │   ├── llm_config.py       ← 🆕 ห้องควบคุมค่าย AI (เฉพาะ `adminsuper` · มีปุ่มยิงทดสอบจริง)
    │   ├── hdc_import.py       ← 🆕 ด่านศุลกากรนำเข้าข้อมูล HDC (preview → import → refresh · 409 กันข้อมูลหาย)
    │   └── data_dict.py        ← 🆕 ป้ายอธิบายความหมายข้างตู้ข้อมูล (404 = ไฟล์นี้ไม่มีพจนานุกรม ไม่ใช่ error)
    ├── agents/                 ← 🤖 สมองกล AI กว่า ~23 ตัว (ถูกสร้างเป็นตี้ CrewAI crews) — แวะไปดูโฉมหน้าได้ที่ [[01 - Backend Design]]
    │   ├── router.py           ← เด็กชี้เป้าจัดเส้นทางโดเมน
    │   ├── question_resolver.py← หมอความจำ Memory Agent (ผู้ช่วยขยายประโยคต่อยอด follow-up ให้เป็นเรื่องราว)
    │   ├── progress.py         ← โฆษกคอยประกาศส่ง progress event ถ่ายทอดผ่านสาย SSE
    │   ├── agent_defaults.py   ← ศูนย์ตั้งค่า CrewAI + วิชาโกงความตาย 429 backoff (ใช้วิชา monkey-patch สอดแทรก)
    │   ├── prompt_profile.py   ← แบบหล่อพิมพ์ prompt มาตรฐานแยกตามโดเมน
    │   ├── csv_pipeline.py         ← สายพานกะเทาะ CSV แบบแผ่นเดียว (อาศัยพลัง 6 agents ช่วยกัน)
    │   ├── multi_csv_pipeline.py   ← สายพานควานหา CSV ข้ามโดเมนมั่วโฟลเดอร์ (นักผจญภัย folder navigator)
    │   ├── compare_agent.py / report_agent.py / database_agent.py / workplan_agent.py ← แก๊งตัวปั้นรายงาน
    │   ├── accident_chat_orchestrator.py / accident_policy_agent.py / accident_policy_orchestrator.py / analyst_accident.py ← แก๊งสายตรวจอุบัติเหตุ
    │   ├── thaijo_agent.py / thaijo_prompts.py / pubmed_agent.py ← แก๊งสายวิชาการ
    │   ├── obsidian_fullcontext.py / obsidian_agent.py / obsidian_progress.py ← แก๊งขุนคลัง
    │   ├── tavily_pipeline.py  ← แก๊งนักเซิร์ฟเน็ต
    │   ├── error_monitor_agent.py ← ตำรวจเฝ้าระวังข้อผิดพลาด
    │   ├── llm_provider.py     ← 🆕 นายทะเบียนกลางค่าย LLM (gemini/chatgpt/claude) · ลำดับ DB > env > ค่าตั้งต้น
    │   │                         + แปล error เป็นภาษาคนได้ 5 แบบ (`friendly_llm_error()`)
    │   └── text_utils.py       ← ตัวช่วยตัดคำเกลาข้อความ
    ├── tools/                  ← 🔧 คลังอาวุธ 15 ชิ้น สำหรับ CrewAI ให้หยิบใช้ (คลาส @tool)
    │   ├── accident_chat_sql.py← ดาบ SQL 15+ เล่ม กระซวกหาคำตอบแชทอุบัติเหตุ
    │   ├── zone10_accident.py  ← ดาบ SQL 7 เล่ม เฉพาะกิจสำหรับพิมพ์ Policy Brief
    │   ├── minio.py            ← ผู้ดูแลคลังไฟล์ + กระบะทรายรันงัด execute_python_code (subprocess)
    │   ├── obsidian.py         ← ปรมาจารย์ค้น vault มี 2 กระบวนท่า (ตีแตกเป็น chunk+trgm / ท่าไม้ตายสุดท้ายงัด LIKE)
    │   ├── vault_rag.py        ← เครื่องโหลดข้อมูล context vault แยกตามพิกัดจังหวัด
    │   ├── tavily_search.py    ← กล้องส่องค้นเว็บ
    │   ├── weather_tool.py     ← เสาอากาศดึงเป้า Open-Meteo
    │   ├── thaijo_cache.py     ← กระปุกออมสินเก็บแคช PDF ค้างไว้บน Redis
    │   ├── error_logger.py     ← สมุดปกดำจดแจ้ง log/จัดประเภท error
    │   ├── missing_data_logger.py ← สมุดปกแดงจดประจานพวกถามนอกเขต/หรือไม่มีข้อมูลจะให้
    │   ├── hdc_opendata.py     ← 🆕 ทูตเจรจากับคลังกลางกระทรวง — ลองซ้ำเมื่อโดน Cloudflare 403,
    │   │                         กรองเขต 10, ล้าง HTML แบบไม่กินเกณฑ์ตัวเลข (`< 100 mg%`)
    │   ├── data_dict.py        ← 🆕 ช่างปั้นพจนานุกรมข้อมูลจากเนื้อไฟล์จริง **ไม่ใช้ LLM** (อ่านซ้ำได้ผลเดิม)
    │   ├── data_dict_lookup.py ← 🆕 คนหยิบพจนานุกรมไปแปะท้าย prompt (`describe_for_prompt()`)
    │   ├── amphoe_zone10.py    ← 🆕 ทะเบียนอำเภอเขต 10 ใช้ตรวจว่าข้อมูลที่ดึงมาอยู่ในเขตจริง
    │   └── vault_placement.py  ← 🆕 คนชี้ว่าไฟล์ใหม่ควรลงโฟลเดอร์ไหนใน vault
    ├── schemas/                ← 📋 พิมพ์เขียว Pydantic models กำหนดโครงร่างข้อมูล (มีหมวด analyze, accident_chat, accident_policy, obsidian, thaijo, pubmed, tools)
    ├── scripts/                ← 🔧 สคริปต์งานมือ 9 ตัว — รันเองเมื่อจำเป็น ไม่ได้อยู่ในเส้นทางปกติ
    │   ├── index_obsidian.py   ← สร้างดัชนี vault ใหม่ (ต้องรันหลังคัดลอกโน้ตมาเครื่องใหม่)
    │   ├── sync_obsidian_pdfs.py ← จับคู่โน้ตกับไฟล์ PDF ใน MinIO
    │   ├── build_data_dict.py  ← 🆕 ปั้นพจนานุกรมให้ไฟล์ที่ยังไม่มี
    │   ├── backfill_definitions.py ← 🆕 ย้อนเติมนิยามจาก HDC ให้ไฟล์ที่นำเข้าไปก่อน `_save_dict` จะถูกแก้
    │   ├── backfill_caveats.py ← 🆕 ย้อนเติมข้อควรระวัง
    │   ├── bulk_import_hdc.py  ← 🆕 นำเข้าทั้งหมวดรวดเดียว
    │   ├── match_hdc_tables.py ← 🆕 จับคู่ชื่อตารางกับรหัสรายงาน (ต้นทางไม่มี endpoint ค้นด้วยชื่อตาราง)
    │   ├── audit_hdc_subcatalog.py ← 🆕 ตรวจว่าดึงครบทั้งหมวดหรือตกหล่น
    │   └── repair_empty_notes.py ← 🆕 ซ่อมโน้ตที่ ingest แล้วได้เนื้อว่าง
    ├── static/                 ← พื้นที่แปะหน้าโชว์ 11 หน้า HTML สำหรับเทสเดโม (จับ mount ไว้เป็น /static) + และมีโฟลเดอร์ js/ ด้วย
    └── obsidian_knowledge/     ← 🌿 บ้านเกิดเก็บ vault .md แบบของจริง (จัดแยกหมวด 5 จังหวัด + กระทรวง MOC) — แหล่งป้อน RAG source ชั้นยอด
```

---

## 2. 🖥️ แดนหน้าบ้าน Frontend — โปรเจกต์ `chatappandpython` (ที่ตั้งชื่อแฝงว่า `musyav2`) พัฒนาบน Next.js 16 เป็นตัวส่งกะพริบหน้าจอ แถมรับจ๊อบเป็นตึกบัญชาการ BFF

```
chatappandpython/
├── package.json                ← ทะเบียนบ้านชื่อแพ็กเกจ musyav2 + รายการช้อปปิ้ง dependencies (ใช้ next 16, react 19, หัวหอกอย่าง pg, minio, jose เป็นต้น)
├── tsconfig.json / next.config.ts / next-env.d.ts ← เซตอัปหลักๆ ของภาษา Type/Next
├── postcss.config.mjs / eslint.config.mjs ← เซตอัปช่วยเกลาโค้ดสวย
├── proxy.ts                    ← คนคอยหันเสาอากาศ proxy
├── Dockerfile                  ← 🐳 สูตรหุงต้ม build ซับซ้อน 3 เด้งชั้นๆ (deps ของพ่วง → builder ปั้นงาน → runner คนเสิร์ฟ)
├── docker-compose.yml          ← ★ แม่งานใหญ่สาย compose เป็นตัวประสานอัญเชิญ 5 ค่ายบริการลงประทับรัน (frontend+python-ai+pg+minio+redis)
├── docker-compose.hub.yml      ← แม่งานใหญ่สาย compose ฉบับเน้นดูดโหลดอิมเมจจากก้อน registry
├── README.md / STRUCTURE.md    ← 📄 เล่มเอกสารทิศทาง
├── database/                   ← ดินแดนยึดครองดาต้าฝั่งแอป
│   ├── schema.sql              ← ตัวพิมพ์สร้างตารางแอปฝั่งตัวเอง (บัญชี accounts, รอบคุย chat_sessions, ชิ้นงาน journal_reports) + คำสั่งหยอดของ seed นำร่อง
│   ├── init-musya.sql          ← สคริปต์จ่อจุดระเบิด init
│   └── init-obsidian.sql       ← สคริปต์จ่อจุดระเบิดตารางตระกูล obsidian
├── docker/                     ← แฟ้มช่วยงาน docker ทั้งหลาย
├── public/                     ← แฟ้มรวมภาพ/ของสาธารณะ public + มีโฟลเดอร์ public/musyaend/js/ ช่วยงานแฝง
├── lib/                        ← 📚 ขุมกำลังคลังแสงลับฝั่งเซิร์ฟเวอร์ (BFF helpers ล้วนๆ)
│   ├── auth.ts                 ← หัวใจยามรักษาความปลอดภัย: บริหารการแฮช bcrypt(12) + จัดสรรตัวเลข JWT (ปั้นผ่าน jose, ตีตรา HS256, ให้บัตรอายุ 7 วัน)
│   ├── internalFetch.ts        ← ★ หน้าด่านตรวจสิทธิ์: อิงตัว requireAuth() + แอบฝังสร้อย internalHeaders(ยัดกุญแจ x-internal-key)
│   ├── db.ts                   ← ตัวสูบจัดการสายน้ำ PostgreSQL pool (ไลบรารี pg)
│   ├── minio.ts                ← ท่อเชื่อมคุยกับโกดัง client MinIO
│   ├── apa.ts                  ← สำนักพิมพ์จัดการสร้าง APA citation ให้
│   ├── fileApaMetadata.ts      ← คนแงะแอบสกัด metadata รื้อหาข้างในไฟล์
│   └── fileInsights.ts         ← เซียนตาเหยี่ยวหา AI insights แอบส่องเนื้อหาไฟล์
└── app/                        ← หัวใจวงจร App Router (สร้างการเกิดหน้าเว็บ + API routes + แก๊ง components)
    ├── layout.tsx / ClientLayout.tsx / page.tsx  ← จัดโครงกระดูกหน้า + หน้าแรกต้อนรับ
    ├── login/ · register/ · forgot-password/     ← เขตต้อนรับคนภายนอก (public)
    ├── account/                ← เขตดูแลหน้าตั้งค่าโปรไฟล์ส่วนตัว
    ├── approved/               ← 🔐 ลานประหารอนุมัติคนผ่านเข้าแอป (ป้ายอาญาสิทธิ์ adminsuper เท่านั้น)
    ├── chat/                   ← 💬 หน้าจอห้องบัญชาการแชทหลัก
    │   ├── page.tsx · LeftPane.tsx · RightPane.tsx ← แบ่งฉากจอ ซ้าย ขวา
    │   ├── *Store.ts           ← สมุดคุมบัญชีสถานะ event stores ไม่พึ่ง redux: มีร้าน chatSession, ร้าน streaming, ฝากของ attachedFiles, จดโน้ต chatDraft, ส่องกบ thaijo, มุดฐาน databaseExplorer
    │   ├── reportSourceStore.ts     ← 🆕 ป้ายสถานะ 5 แหล่ง (pending/running/done/error) ตอนระดมข้อมูลทำรายงาน
    │   ├── preGatherTopicsStore.ts  ← 🆕 สะพานให้เลือก/แก้หัวข้อ "ก่อน" ยิงไประดมข้อมูลจริง (ChatInput ค้าง await รอ)
    │   ├── reportReadyStore.ts      ← 🆕 สัญญาณ "ข้อมูลพร้อมแล้ว รอผู้ใช้กดสร้าง" (เลิก auto-generate)
    │   ├── reportRetry.ts           ← 🆕 ปุ่มลองใหม่รายแหล่ง — ยิง report-gather-retry แล้ว append + เซฟทับลง DB
    │   ├── wizardPersist.ts         ← 🆕 เซฟความคืบหน้า wizard แบบ debounce กันหายตอน reload กลางทาง
    │   ├── reportSavePersist.ts     ← 🆕 เซฟ id+title ของรายงานที่ auto-save แล้ว ให้ปุ่มเปิดรายงานในแชทเก่ายังกดได้
    │   ├── autoToolStore.ts         ← 🆕 ปุ่ม "อัตโนมัติ" — ให้ระบบเลือกเครื่องมือเองแทนที่ผู้ใช้ต้องเดา
    │   ├── obsidianApa.ts           ← 🆕 แปลง ObsidianNoteRef → บรรณานุกรม APA (แกะประเภท/ปี/ชื่อเรื่องจากชื่อไฟล์)
    │   ├── chatTypes.ts        ← โมเดลบอกทรงข้อมูล (🆕 เพิ่มชนิด WizardProgressSaved, SavedReportRef + ฟิลด์ wizardProgress, savedReports)
    │   ├── sessions/[sessionId]/page.tsx  ← เปิดรื้ออ่านเซสชันแชทที่ถูกบันทึกเจาะจง
    │   └── journal-template/   ← แก๊งช่วยปั้นรายงาน (มี buildJournalHtml, journalDocxStyles, journalHtmlStyles เสกหน้า export เป็น DOCX/PDF)
    ├── journal/                ← ห้องเก็บหิ้งคลังรายงานผลงานที่เซฟไว้
    ├── fileapa/                ← ห้องดูแลจัดการขยะไฟล์ + จับทำบรรณานุกรม APA (+ แอบมีห้องย่อย [fileRoute]/, กับ listapa/)
    │   ├── FileMetadata.tsx    ← 🆕 แผงพจนานุกรมข้างตัวอย่างไฟล์ — เรียงข้อควรระวังขึ้นก่อน
    │   │                         เพราะเป็นสิ่งที่ทำให้ตอบผิดได้ แล้วค่อยขอบเขต → ความหมายคอลัมน์
    │   └── HdcImportModal.tsx  ← 🆕 กล่องนำเข้า/รีเฟรชจาก HDC (จับ 409 มาแสดงเป็นตัวเลือกให้คนตัดสินใจ)
    ├── hdc-import/             ← 🆕 หน้าดึงทั้งหมวดรวดเดียว + ลองใหม่เฉพาะลิงก์ที่ล้ม (ต้นทางเด้ง 403 เป็นช่วง ๆ)
    ├── admin/ai-settings/      ← 🆕 หน้าตั้งค่าค่าย AI — เฉพาะสิทธิ์ `adminsuper` เท่านั้น
    ├── pdf-upload/             ← สายพานลำเลียงอัปโหลด PDF อัดยัดเข้าห้องสมุด (🆕 เลือกหลายไฟล์ · ยกเลิกได้ · Ingest Log ละเอียด)
    ├── musyaend/               ← 🚗 ฉากจอลับ Accident Chat กางจอเล่นแบบเต็มตา (+ แทรกทางเข้า db-explorer/, obsidian/)
    ├── actions/upload.ts       ← คำสั่งรับจ้างโยนไฟล์ (server action) อัปโหลดพุ่งตรง
    ├── component/              ← องค์ประกอบจิ๊กซอว์ร่วม (components)
    │   ├── Sidebar.tsx · DatabaseExplorer.tsx
    │   └── chat/               ← จิ๊กซอว์ฝั่งแชท: ตัวแผง ChatInput.tsx (ให้สิทธิ์จิ้มเลือกวิชา tool), จอฉาย MarkdownContent.tsx, ป้ายกระพริบไฟ AgentPipelinePanel.tsx
    │       ├── ReportSourceBadges.tsx ← 🆕 แถบ badge 5 แหล่ง + ปุ่มลองใหม่บนตัวที่ error (ผูก reportSourceStore ผ่าน useSyncExternalStore)
    │       └── ApaReferences.tsx      ← 🆕 บรรณานุกรม APA ท้ายคำตอบคลังความรู้ + ปุ่มคัดลอกทั้งชุด
    └── api/                    ← 🔌 เขตชุมสายด่านตรวจคนเข้าเมือง BFF Route Handlers (เตือน: ทุกท่อนต้องโชว์บัตร JWT ผ่านมือตม. requireAuth เสมอ)
        ├── auth/               ← ระบบออกบัตร: login, logout, register, me, users, forgot-password, reset-password
        ├── chat/               ← ★ ทางแยกสำคัญ route.ts (ตัวชิ่ง proxy ไหลตามโหมด + แยกร่างสตรีม SSE tee/เซฟหนีตาย fallback) + ทะลวงหน้า history/
        ├── files/              ← ย้ายข้าวของ: โยน upload, จัดแจง [fileId] (+แอบล้วง ai-metadata, insights), ฟังก์ชันควบรวม merge/{search,analyze,execute,save}
        ├── generate-apa/       ← เสกคำอ้างอิง APA
        ├── journal-reports/    ← รื้อหิ้งคลังรายงาน (+มีห้องเจาะจง [id]/)
        ├── obsidian/[...path]/ ← รูหนู passthrough → มุดทะลุกำแพงไปโผล่ฝั่ง backend /api/obsidian
        ├── pdf/                ← การจัดการม้วนสาร: ควบรวม files, ทำ ingest (+เช็ค status), ส่ง upload, ลาน view, คลัง vault/*, เปิดส่อง obsidian-view
        ├── python/[prefix]/[...path]/  ← รูหนู passthrough เฝ้าระวังเข้มงวด ปล่อยผ่านเฉพาะ: แก๊ง accident-chat|accident-policy|db|obsidian
        ├── llm/                ← 🆕 เลือกค่าย AI: providers/ (ผู้ใช้ทั่วไป) + admin/providers/{,test} (เฉพาะ adminsuper)
        ├── hdc/                ← 🆕 นำเข้าจากคลังกลาง: preview · subcatalog · import · refresh/[fileId] · imports
        ├── datadict/[fileId]/  ← 🆕 พจนานุกรมข้อมูลให้หน้า fileapa แสดง (404 = ไฟล์นี้ไม่มี ไม่ใช่ error)
        └── thaijo-topics/      ← ควานหาหิ้งหัวข้อใหม่
```

รวม **51 BFF route** · **17 หน้าเว็บ** (นับจริง 1 ส.ค. 2569)

> [!note] กฎเหล็กเรื่องการตรวจสอบสิทธิ์บนพื้นที่นี้ (สำคัญมาก)
> ทราบหรือไม่ว่าในโครงสร้างเวอร์ชันนี้ **โดนถอดไฟล์ทหารยามหน้าประตู `middleware.ts` ทิ้งไปแล้ว** — ระบบหันมาใช้วิธีป้องกันแบบประชิดตัวในแต่ละเส้นทางผ่านคำสั่ง **`requireAuth()`** แทน
> (ซึ่งซ่อนตัวคอยซุ่มอยู่ในแฟ้ม `lib/internalFetch.ts`) โดยตัวมันจะถูกจับไปเรียกเช็คสถานะอยู่บรรทัดแรกในทุกๆ หน้าของ Route Handler หากพบตัวว่าไม่มี/หรือตั๋ว JWT ไม่ผ่านเกณฑ์ปุ๊ป มันจะเตะหน้าหงายคืนค่า 401 ในเสี้ยววิ ส่วนทางออกตอนจะหันไปเรียกข้ามฟากไปหาลูกพี่หลังบ้าน backend ทุกครั้ง มันก็จะถูกบังคับให้กลืนฝังจี้ตามติดตัว `x-internal-key` ไหลผ่านเส้นทาง `internalHeaders()` เป็นของแถมเสมอ

---

## 3. 🗺️ ลายแทงนำทาง "มองหาฟีเจอร์ไหน → ให้ไปส่องไฟล์อะไร" (Feature → File map)

| ความสามารถ (ฟีเจอร์) | เดินหาในไฟล์ฝั่งหน้าบ้าน (Frontend) | เดินหาในไฟล์ฝั่งหลังบ้าน (Backend) |
|---|---|---|
| การล็อกอิน/ลงทะเบียนสมัคร | ล้วงกุญแจที่ `app/api/auth/*`, ตรวจโค้ดที่ `lib/auth.ts` | — (ฝั่งหลังบ้านไม่ต้องสน เพราะทาง BFF ผูกขาดถือตาราง `accounts` เองเบ็ดเสร็จ) |
| ห้องแชทตีฝีปากถาม-ตอบ | คุมกระแสที่ `app/api/chat/route.ts`, ฉากที่ `app/chat/*`, ตัวคุมกรอกที่ `component/chat/ChatInput.tsx` | รุมวิเคราะห์ที่ `routers/analyze.py` → แจกงานแก๊ง `agents/*` |
| งัดแงะสถิติอุบัติเหตุ | (เลือกโหมดส่งผ่านแชท mode `stats`) | ตัวบงการ `agents/accident_chat_orchestrator.py`, ตัวขุดตาราง `tools/accident_chat_sql.py` |
| งมเข็มคลังความรู้องค์กร | ส่องฉากที่ `app/musyaend/obsidian/`, ด่านส่งตรวจ `app/api/obsidian/` | ตัวเปิดทาง `routers/obsidian.py`, ยอดฝีมือรื้อคลัง `agents/obsidian_fullcontext.py` |
| ขุดรากงานวิจัย ThaiJo/PubMed | (เลือกโหมดส่งผ่านแชท mode `research`) | สายดึงไทย `agents/thaijo_agent.py`, สายดึงนอก `agents/pubmed_agent.py` |
| เสกปั้นสร้าง/บันทึกรายงาน | บล็อกหล่อที่ `app/chat/journal-template/*`, หิ้งโชว์ที่ `app/api/journal-reports/` | ด่านจัดหนัก `routers/analyze.py` (ทะลวงไปในโหมด report-gather) |
| การจัดระเบียบไฟล์ & ปั้น APA | ตู้กระจก `app/fileapa/*`, ตัวเสก `lib/apa.ts`, และ `lib/minio.ts` | — (เพราะเก็บของไว้ในเซิร์ฟโดดๆ MinIO) |
| ยืนดูตาปริบๆ สำรวจตาราง DB | ลานสอดแนม `app/musyaend/db-explorer/`, สายส่งภาพ `app/api/db/` | คนชี้เป้าโชว์ `routers/db_explorer.py` |

*แผนผังนี้ทำการถอดโครงสร้างและตรวจเช็คยืนยันกับโค้ดโฟลเดอร์ของจริง ณ วันที่ 2026-07-18 — โปรดทราบว่าโปรเจกต์อาจมีการแตกลูกขยายหลานไฟล์เติบโตขึ้นได้ในอนาคต แนะนำให้ตรวจสอบพาดพิงกับโฟลเดอร์โค้ด Repository จริงอยู่เสมอ*
