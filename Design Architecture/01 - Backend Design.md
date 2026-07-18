---
title: "Design — Backend (FastAPI + CrewAI) (การออกแบบระบบหลังบ้าน)"
tags: [MUSYA, design, backend, crewai, agents]
created: 2026-07-18
---

# ⚙️ การออกแบบระบบ Backend — `chatapi.python`

← [[00 - Architecture Overview]] · หน้าต่อไป → [[02 - Frontend Design]]

> ระบบ Backend ถูกพัฒนาด้วยโครงสร้างที่ชัดเจน เป็นระบบ **ตัวนำทาง (Router) → ตัวอำนวยการ (Orchestrator) → ท่อส่งผ่านงาน (Pipeline) → เครื่องมือ (Tools)** 
> โดยไฟล์ `main.py` ทำหน้าที่ปั้นแอปพลิเคชัน FastAPI ขึ้นมา ขึ้นทะเบียน Routers ทั้งหมด 10 สาย และตั้งค่าเปิดช่องรับไฟล์แนบ `/static` รวมถึงแสดงหน้าทดสอบระบบเล็กๆ ที่ `/ui` 
> สมองกลหลักที่รับผิดชอบเรื่องปัญญาประดิษฐ์ (LLM) คือระบบ Google Gemini (`gemini-2.5-flash-lite` / หรือ `-pro` ตัวเก่ง) ซึ่งบริหารคิวผ่านระบบไลบรารี `litellm`

## 1. ลำดับชั้นของโมดูลซอฟต์แวร์ (Module layers)

| ชื่อชั้น (Layer) | โฟลเดอร์/โมดูลหลัก (Modules) | หน้าที่รับผิดชอบ (Responsibility) |
|---|---|---|
| **ด่านรับเรื่อง (Entry)** | `main.py` | ประกาศแอปพลิเคชัน, จัดการ CORS, ต้อนรับการลงทะเบียน Router, และเสิร์ฟหน้า UI สถิติ |
| **ตัวจำแนกทิศทาง (Routers)** | `src/routers/*` (ทั้งหมด 10 ตัว) | เฝ้าเส้นทาง HTTP API (Endpoints); คอยตรวจสอบพารามิเตอร์; และโยนงานให้ฝ่ายปฏิบัติการ (Agents) |
| **ศูนย์บัญชาการ (Orchestration)** | `analyze.py` (ฟังก์ชัน `_orchestrate()`), `router.py`, `question_resolver.py` | วงจรประสานงานความจำ Memory → ควบคุมทิศทาง Router → เลือกสายพาน Pipeline โดยอิงจาก โดเมนและโหมดการทำงาน |
| **สายพานปฏิบัติการ (Pipelines)** | `src/agents/*` ชุด Crews | กระบวนการทำงานสืบค้นและสังเคราะห์ข้อมูลเฉพาะทาง |
| **คลังเครื่องมือ (Tools)** | `src/tools/*` (เครื่องหมาย `@tool`) | เครื่องมือช่วยเจาะระบบ SQL, ทำงานร่วมกับ MinIO+การรันโค้ดสดๆ, ค้นข้อมูลจาก Obsidian, ค้นเว็บบน Tavily, เก็บแคช, และจดบันทึก Log ปัญหา |
| **โครงสร้างพื้นฐาน (Infra)** | `config.py`, `db/pool.py`, `history.py`, `agent_defaults.py`, `prompt_profile.py` | เก็บค่าตั้งค่า, บริหารจัดการ Connection Pool ให้ดาต้าเบส, ระบบประวัติบน Redis, กฎเกณฑ์หน่วงเวลาหลบขีดจำกัด (Rate-limit), และแม่แบบคำสั่ง Prompt |

## 2. หน้าที่และการตอบสนองของกลุ่ม Routers

| ชื่อ Router | คำนำหน้า (Prefix) | บทบาทหน้าที่ (ดูได้ที่ [[04 - External Interface Requirements]]) |
|---|---|---|
| `analyze` | `/api`, `/api/chat` | **เป็นประตูด่านหลัก**; รันกระบวนการ Memory→Router→Pipeline; ทำงานผ่าน SSE; มีมาตรการล็อก Semaphore ให้วิ่งได้ไม่เกิน ≤5 งาน |
| `accident_chat` | `/api/accident-chat` | เป็นท่อสำหรับพูดคุยโต้ตอบข้อมูลอุบัติเหตุแบบเจาะลึก (เป็นคู่ของ 2-agent); และมีเส้นทาง `/quick` เพื่อเอาข้อมูลดิบๆ โดยไม่พึ่ง LLM |
| `accident_policy` | `/api/accident-policy` | วางนโยบายและส่งออก Policy Brief ของเขต 10; เส้นทาง `/zone10/data` ให้สถิติดิบ, ส่วนเส้นทาง `/zone10` จัดชุดใหญ่แบบเต็มทีม (crew) |
| `obsidian` | `/api/obsidian` | บริหารคลัง Vault ทั่วไป เสิร์ช/ตั้งคำถาม/จัดการดัชนี/และการทำซิงโครไนซ์เอกสาร PDF |
| `thaijo` | `/api/thaijo` | ไปป์ไลน์ดึงข้อมูลวิจัยมาสังเคราะห์และรวมร่างเป็นรายงานวิชาการ |
| `pubmed` | `/api/pubmed` | โหมดดึงบทความเชิงวรรณกรรมการแพทย์ระดับสากล |
| `tools_router` | `/api` | ควบเส้นทางระบบเครื่องมือ `compare`/`report`/`workplan`/`database` |
| `db_explorer` | `/api/db` | เปิดให้คนภายนอกเข้ามาเดินสำรวจในฐานข้อมูลได้ (แบบอ่านอย่างเดียว ไม่เปิดช่องฉีดโค้ด) |
| `pdf_ingest` | — | ระบบทยอยอ่านทีละก้อน (Chunked) สำหรับการย่อยไฟล์ PDF |
| `error_log` | `/api/errors` | ดูบันทึกอาการป่วยของ Agent + ให้ LLM มาช่วยสรุปปัญหาให้ฟังสั้นๆ |

## 3. ขั้นตอนสับหลีกศูนย์บัญชาการ (Orchestration control flow: `analyze.py`)

ทุกคำสั่งจะต้องวิ่งเข้ามาสู่ศูนย์ควบคุม `_orchestrate()` ผ่านช่องทางจำเพาะ (Daemon thread) และถูกแยกสับหลีกตาม **`โหมด (mode)`**, โดย Router จะตัดสินใจหาโดเมนภายในโหมด `normal`/`stats`:

```mermaid
flowchart TD
    A[รับ POST /api/analyze] --> S{สล็อตว่างไหม?<br/>(semaphore.acquire)}
    S -- เต็ม --> Busy[ตอบกลับแบบ SSE: ขออภัยระบบยุ่ง]
    S -- ว่าง --> M[Memory Agent<br/>ขยายบริบทคำถามที่ติดตามมา]
    M --> V[Vault RAG hint<br/>สแกนว่าพูดถึงจังหวัดอะไร]
    V --> MODE{เช็คค่า mode}
    MODE -- stats (สถิติ) --> ACC{เกี่ยวกับอุบัติเหตุไหม?}
    ACC -- ใช่ --> D1[โยนเข้าสายพาน Accident SQL]
    ACC -- ไม่ใช่ --> RS[ให้ route_stats_domains คัดแยก]
    RS --> CSV[โยนเข้าสายพาน CSV / หรือ Multi-CSV]
    MODE -- obsidian (คลังองค์กร) --> OB[โยนเข้า Obsidian full-context (โหลดข้อมูลเต็มสูบ)]
    MODE -- tavily (ค้นหาเว็บ) --> TV[โยนเข้าสายพาน Tavily web]
    MODE -- research (งานวิจัย) --> RP[สั่งรัน ThaiJo กับ PubMed คู่ขนานกัน]
    MODE -- report-gather (สร้างรายงาน) --> RG[รันพร้อมกัน 5 สายรวด + ผ่านด่านคัดกรองพื้นที่]
    MODE -- normal (โหมดคุยทั่วไป) --> RT[ให้ route_multi_domain วิจารณ์หาโดเมน]
    RT --> PIPE[จ่ายงานให้โดเมนย่อย: เดี่ยว / รวม / obsidian / thaijo]
```

**สิ่งที่ระบบถูกออกแบบตั้งใจให้แฝงอยู่ในสับหลีกนี้:**
- **กระบวนการคัดกรองพื้นที่ (Out-of-zone guard)** จะถูกรันคั่นกลางก่อนที่จะเริ่มกระบวนการสืบค้นข้อมูลอุบัติเหตุหรือสร้างรายงาน โดยหากพบว่าเป็นพื้นที่จังหวัดนอกเหนือขอบเขต มันจะปฏิเสธคำขอทันที พร้อมจดบันทึกประจานลงใน `missing_data_logger` ([[03 - Functional Requirements|FR-CHAT-08]])
- **ฟังก์ชันกลายพันธุ์ปี (พ.ศ.→ค.ศ.)** ทุกตัวแปร พ.ศ. จะถูกฟอร์แมตแปลงร่างเพื่อเตรียมการลุยกวาดดาต้าในอุบัติเหตุ (FR-CHAT-09)
- **สลับรางคำถามอุบัติเหตุในหมวดโหมดปกติ (Accident in normal mode)** ถูกกำหนดหักเหไปเข้าทางของเครื่องมือวิเคราะห์ระดับนโยบาย (Obsidian pipeline) แทน เพื่อเปิดทางและสงวนท่อ SQL agent เผื่อไว้ให้โหมด "สถิติ" ที่เป็นเรื่องตัวเลขแบบเจาะจงใช้งานเท่านั้น

## 4. แฟ้มรวมสายพานความสามารถของเอเจนต์ (Agent catalogue - CrewAI crews)

จัดกลุ่มตามที่ระบุไว้ใน `AGENTS.md`:

| กลุ่มการทำงาน (Group) | รายชื่อเอเจนต์ (Agents) | ชื่อช่องทางเข้า Pipeline (Pipeline entry) |
|---|---|---|
| **A · อุบัติเหตุ (SQL, หมวด d1)** | `accident_chat_orchestrator` (เป็นคู่หู: สายหาข้อมูล SQL Agent → สายแต่งคำตอบ Answer Agent), `accident_policy_agent`/`_orchestrator` (3 ประสาน: ค้นหาFetcher → วิเคราะห์Analyst → เขียนWriter), `analyst_accident` | ท่าเข้า: `run_accident_chat()`, `run_zone10_analysis()` |
| **B · ข้อมูลตารางสถิติ (CSV, หมวด d0,d2–d4)** | `csv_pipeline` (6 เซียนช่วยกัน), `multi_csv_pipeline` (นักผจญภัยค้นโฟลเดอร์), `compare_agent`, `report_agent`, `database_agent`, `workplan_agent` | ท่าเข้า: `run_pipeline()`, `run_multi_pipeline()` |
| **C · คลังวิชาการ (Knowledge & research)** | `thaijo_agent` (ดึงFetcher → วางโครงPlanner → สร้างเนื้อหาGenerator), `obsidian_fullcontext`, `obsidian_agent`, `pubmed_agent` | ท่าเข้า: `run_thaijo_pipeline()`, `run_obsidian_ask_fullcontext()`, `run_pubmed_pipeline()` |
| **D · ตัวประสานงาน (Routing & shared)** | `router` (คนชี้ทาง), `question_resolver` (ดูแลความจำ Memory), `progress` (ป้ายบอกทาง), `tavily_pipeline` | — |
| **E · โครงสร้างรากฐาน (Infra)** | `agent_defaults`, `prompt_profile`, `error_monitor_agent` | — |

### กระบวนการดีไซน์ของท่อทำงาน CSV เดียว (CSV single-domain pipeline design pattern)
มีการจัดวางกระบวนทัพดังนี้: `คนหาไฟล์ (File Finder) → คนดูโครงสร้าง (Schema Analyst) → คนวางคำสั่ง (Prompt Profiler) → คนเขียนโค้ดภาษาไพทอน (Python Code Generator) → คนรันโค้ด (Python Executor) → นักวิเคราะห์สรุป (Insight Analyst)`. 
**หลักการที่น่าสนใจ:** LLM จะรับบทเป็นคน **เขียน** โค้ดแพนด้า (Pandas) แต่ทว่าโค้ดนั้นจะถูกนำไปส่งรันใน **กระบะทรายหรือสภาพแวดล้อมจำลอง (Sandboxed subprocess)** อย่างปลอดภัยผ่านไลบรารี `minio.execute_python_code` — เป็นการแยกส่วนสมอง (Reasoning) ออกจากการประมวลผลคำนวณ (Computation) ให้ชัดเจน

### การจับคู่แฟ้ม CSV รวมฮิต (Multi-CSV design principle) — *AI มีหน้าที่หาความหมาย, โค้ดมีหน้าที่ระบุตัวตนจริง*
ตัวแทนในการสำรวจแฟ้มข้อมูล (Folder-Navigator agent) จะมองเห็นข้อมูลเป็น **ต้นไม้ของชื่อตัวชี้วัด (Folder tree of full indicator names)** และเลือกสุ่มโฟลเดอร์จากความหมาย (Semantic match) ไม่ใช่จำแนกจากชื่อไฟล์แบบทื่อๆ 
หลังจากนั้นระบบจะใช้ **โค้ดกำหนดสิทธิ์แบบแน่นอน (Deterministic code)** ในการแก้โจทย์ (`_resolve_folders_to_files` + หา `path_index`) เพื่อบิดเปลี่ยนชื่อโฟลเดอร์พวกนั้น → เป็นชื่อ ID ไฟล์ของจริง 
นอกจากนี้ หากหาสมาชิกเข้ากลุ่มไม่ได้ถึง 2 ไฟล์ ระบบจะหันไปใช้กฎนับคะแนนคำเสริม (Keyword scoring) คัดแยกให้ พร้อมสวมตัวตรวจจับคีย์เวิร์ดภูมิศาสตร์ (Geo-Key Detector) และตัวเช็คความครอบคลุมโดเมน (Domain Coverage Validator) กรองอีกชั้น

## 5. อุปกรณ์เครื่องมือเฉพาะทาง (Tools: `@tool`)

| ชื่อเครื่องมือ (Tool) | ขีดความสามารถ (Capability) | ไปป์ไลน์ที่เบิกใช้งาน (Used by) |
|---|---|---|
| `zone10_accident` | ยิงสอบถามฐานตารางอุบัติเหตุนโยบาย 7 ท่า (`mart_province_road`, `dim_geography`) | ท่อ accident_policy |
| `accident_chat_sql` | มีสกิล SQL ท่าพิสดาร 15+ ชุด (หาจุดวิกฤต/หาพฤติกรรม/หาฤดูกาล/หาเป้าประสงค์ KPI) → ย่อยมาเป็นหน้า Markdown พร้อมห้อยท้ายคำอธิบายข้อจำกัด (footnotes) | ท่อ accident_chat |
| `minio` | สั่งเปิด/อ่านรายชื่อ CSV และท่าไม้ตาย **`execute_python_code`** (สั่งรันใน Subprocess พร้อมไลบรารีวิทยาศาสตร์ pandas/numpy/scipy) ท่าสุดท้ายแก้ปัญหาความคลุมเครือไฟล์ Fuzzy | ท่อตระกูล CSV |
| `obsidian` | โหมด Tier-1: สับเนื้อหาโหมดไฮบริด (pg_trgm) บวกกับขยายเส้นทางเชื่อม (Wikilink); โหมด Tier-2: ท่าไม้ตายสุดท้าย LIKE ค้นคำ | ท่อระบบ obsidian |
| `tavily_search` | กวาดค้นบนอินเทอร์เน็ต | ท่อ tavily_pipeline |
| `weather_tool` | ดึงระบบสภาพอากาศ (Open-Meteo ของไทย) | (เป็นเครื่องมือเสริม) |
| `error_logger` | ดึงสถิติ จัดกลุ่ม จัดประเภท สรุปรายการล้มเหลว | ท่อ csv_pipeline และ error_monitor |
| `thaijo_cache` | ใช้ Redis ช่วยเก็บความทรงจำผลสรุป PDF งานวิจัยที่เคยวิเคราะห์ไปแล้ว จะได้ไม่ทำงานซ้ำ | ท่อ thaijo_agent |
| `missing_data_logger` | บันทึกประจานรายการที่ระบบต้องตีกลับ ว่าหาพื้นที่นั้นไม่เจอ หรือหลุดไปถามที่อื่น | อยู่ที่ด่านประตู Analyze guard |

## 6. แกนโครงสร้างวิศวกรรม (Cross-cutting infrastructure)

- **`config.py`** — ทำงานผ่านออบเจกต์ Pydantic `Settings` ที่โหลดค่ามาจากไฟล์ลับ `.env`, สั่งเก็บแคช `@lru_cache`; จัดการเรื่องรหัส DB DSN, ID โมเดล, การเชื่อม MinIO, ตั้งค่าลิมิต Obsidian, รวมถึงเซตอัป CORS
- **`db/pool.py`** — จัดระบบบ่อเชื่อมต่อดาต้าเบส `psycopg2 ThreadedConnectionPool` (อนุญาตขั้นต่ำ 2 / ห้ามเกิน 20 เส้นทาง); นำเสนอวิธีคิวรีข้อมูลด่วน `query_db` และระบบป้อนข้อมูล `execute_db`
- **`history.py`** — ระบบจดบันทึกประวัติการพูดคุยแบบต่อเนื่องลง Redis (จดจำย้อนหลังได้ 6 ก้าว, หมดอายุสลายไปเองใน 24 ชั่วโมง); ดึงทบทวนอดีตผ่านฟังก์ชัน `build_history_context`
- **`agent_defaults.py`** — ทำการสอดไส้ทับ (Monkey-patches) ลงไปยังแกนในของตัว CrewAI `Agent` โดยมุ่งหวังเพื่อสร้างกระบวนการ **429 exponential backoff (หน่วงเวลาแล้วลองใหม่)** ผ่านท่า `kickoff_with_retry` — นับว่าเป็นปราการเสริมความถึกทนที่สำคัญของทั้งระบบ (ตามข้อตกลง NFR-REL-01)
- **`progress.py`** — ควบคุมการกระจายเสียงแจ้งโปรเกสการทำงานของโมเดล `AgentProgress` ส่งออกทางสตรีม SSE กระเด็นไปโชว์กระพริบบนป้ายพาเนลของหน้าจอ

## 7. แบบจำลองความสัมพันธ์ของเวิร์กเกอร์กับคิวจราจร (Concurrency & threading model)

- การสืบสวนทุกรายการของระบบจะอาศัย `_orchestrate()` ส่งตรงเข้าด้ายมืด **(Daemon thread)** เพื่อหลบหลีกการบล็อกการทำงาน และข้อมูลการสตรีมมิ่ง SSE จะถูกดันส่งต่อกันตามช่องจราจร `asyncio.Queue` โดยมีฟังก์ชัน `run_coroutine_threadsafe` เป็นสะพานข้ามสายธาร
- ทั้งโปรเซสจะถูกสวมหมวกนิรภัยที่เรียกว่า `BoundedSemaphore(5)` ที่ลิมิตอัตราการประมวลผลสูงสุดที่ 5 วงจรรอบของไปป์ไลน์ หากมีคำสั่งหลุดกรอบระบบจะโยนข้อผิดพลาดเป็นจังหวะยุ่งวุ่นวายโชว์ขึ้นเตือนแบบไม่ให้ระบบรอจนค้าง (ตามข้อตกลง NFR-PERF-02, FR-CHAT-15)
- กระบวนการควบรวมขนานคู่ (Parallel multi-source work) อย่างการทำ `research` (วิจัย) หรือการดึงเรื่อง `report-gather` (รวบรวมรายงาน) จะไปอาศัยขุมกำลังแยกจากสระคนงาน `ThreadPoolExecutor` เป็นพิเศษ และในจังหวะสร้างรายงาน (report-gather) ระบบจะมีระบบหน่วงจุดพลุเตือนเริ่มงานสลับช่วงฟันปลา **(staggering)** ทีละประมาน 1.5 วินาที เพื่อไม่ให้เซิร์ฟเวอร์สวรรค์ LLM ร้องหาโควต้าจนล่มไปเสียก่อน (ตามข้อตกลง NFR-PERF-05)
