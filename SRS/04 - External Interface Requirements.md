---
title: "SRS §4 — External Interface Requirements (ความต้องการด้านการเชื่อมต่อภายนอก)"
tags: [MUSYA, SRS, interfaces, api]
created: 2026-07-18
---

# 4. ความต้องการด้านการเชื่อมต่อภายนอก (External Interface Requirements)

← [[03 - Functional Requirements]] · → [[05 - Non-Functional Requirements]]

## 4.1 ส่วนติดต่อผู้ใช้ — เส้นทางหน้าเว็บฝั่ง Frontend (User interface — page routes)

ส่วนนี้อธิบายหน้าจอ (Pages) ทั้งหมดที่ผู้ใช้สามารถเข้าถึงได้และสิทธิ์ที่จำเป็น:

| เส้นทางหน้าเว็บ (Route) | สิทธิ์เข้าถึง (Access) | วัตถุประสงค์ (Purpose) |
|---|---|---|
| `/` | สาธารณะ (public) | หน้าแรก (Landing) — นำเสนอการ์ดความรู้สุขภาพ พร้อมแบนเนอร์ของ สสส. |
| `/login`, `/register`, `/forgot-password` | สาธารณะ (public) | ระบบจัดการสมาชิก เข้าสู่ระบบ, ลงทะเบียน, ลืมรหัสผ่าน |
| `/account` | ผู้ใช้งาน (user) | หน้าสำหรับดูและแก้ไขข้อมูลโปรไฟล์ส่วนตัว |
| `/approved` | **ผู้ดูแลระดับสูงสุด (adminsuper)** | หน้าจัดการรายชื่อผู้ใช้ อนุมัติ / ระงับบัญชี |
| `/chat` | ผู้ใช้งาน (user) | หน้าแชทหลัก — แบ่งเป็นสองฝั่งซ้ายขวาที่ปรับขนาดได้ (ฝั่งซ้าย: ดูสายงานท่อ AI, ฝั่งขวา: แสดงรายงาน) |
| `/chat/sessions/[sessionId]` | ผู้ใช้งาน (user) | เปิดดูประวัติแชทเดิมและสามารถแชทสานต่อได้ (ดึงข้อมูลสถานะเดิมจากฐานข้อมูล) |
| `/journal` | ผู้ใช้งาน (user) | คลังรายงานส่วนตัว — ใช้สำหรับเรียกดู, ลบ, หรือดาวน์โหลดเป็นไฟล์ DOCX / PDF |
| `/fileapa`, `/fileapa/listapa`, `/fileapa/[fileRoute]` | ผู้ใช้งาน (user) | หน้าจัดการไฟล์ที่อัปโหลดทั้งหมด พร้อมระบบสร้างบรรณานุกรม APA อัตโนมัติ |
| `/pdf-upload` | ผู้ใช้งาน (user) | หน้าจออัปโหลดและจัดการไฟล์ PDF เข้าสู่ห้องสมุดองค์กร |
| `/musyaend` | ผู้ใช้งาน (user) | แชทวิเคราะห์ข้อมูลอุบัติเหตุเต็มจอภาพ (มีปุ่มตัวช่วยครบวงจร และแสดงข้อจำกัดของข้อมูล) |
| `/musyaend/db-explorer` | ผู้ใช้งาน (user) | เครื่องมือสำรวจตารางฐานข้อมูลเชิงลึก แบบอ่านอย่างเดียว (Read-only) |
| `/musyaend/obsidian` | ผู้ใช้งาน (user) | หน้าค้นหาและสอบถามข้อมูลจากคลังความรู้ Obsidian |

**ข้อจำกัดและการออกแบบ UI:**
- **ให้ความสำคัญกับภาษาไทย (Thai-first):** ใช้ฟอนต์ IBM Plex Sans Thai เป็นหลัก
- **ระบบสถานะ:** พัฒนาด้วยเทคนิคการจำลองสถานะ Custom event stores ร่วมกับ `useSyncExternalStore` (โดยไม่ได้ใช้งาน Redux)
- **การเรนเดอร์แบบทันที (Incremental rendering):** ข้อความที่ถูกสตรีมมาจะแสดงผลทันทีแบบทีละตัวอักษร

## 4.2 ระบบ Backend-for-Frontend (BFF API) — ผ่าน Next.js Route Handlers

ทุกเส้นทางบังคับให้ผู้ใช้ต้องล็อกอินและส่ง JWT cookie เข้ามาเสมอ (`FR-AUTH-09`) ยกเว้นบางฟังก์ชันที่เป็นสาธารณะ

| ตำแหน่ง (Endpoint) | เมธอด (Method) | วัตถุประสงค์ (Purpose) |
|---|---|---|
| `/api/auth/register\|login\|logout\|me` | POST/GET | สมัครสมาชิก, ล็อกอินเข้า-ออก และตรวจสอบเซสชันปัจจุบัน |
| `/api/auth/forgot-password\|reset-password` | POST | ขั้นตอนการขอรีเซ็ตรหัสผ่าน |
| `/api/auth/users` | GET/PATCH | ฝั่ง Admin: รายชื่อผู้ใช้ทั้งหมด / อัปเดตสถานะบัญชี (สงวนไว้สำหรับ adminsuper) |
| `/api/chat` | POST | **ช่องทางหลัก (Main entry)** — วิเคราะห์ `mode` แล้วเลือกว่าจะส่งไปท่อ Backend จุดไหน และส่งต่อการสตรีม (SSE) |
| `/api/chat/history` | GET/POST | โหลด/เซฟ สถานะการแชทในปัจจุบัน |
| `/api/files`, `/api/files/upload`, `/api/files/[id]` | GET/POST/DELETE | เพิ่ม/ดู/ลบ ไฟล์จาก MinIO |
| `/api/files/[id]/ai-metadata`, `/insights` | GET | ดึงข้อมูลเชิงลึก Metadata ที่ AI แกะสกัดจากไฟล์นั้นๆ |
| `/api/files/merge/{search,analyze,execute,save}` | POST | รวมชุดคำสั่งการวิเคราะห์จากหลายไฟล์เป็นท่อเดียวกัน |
| `/api/generate-apa` | POST | คำสั่งสร้างอ้างอิง APA จากเอกสารที่ระบบได้สกัดไว้ |
| `/api/journal-reports`, `/api/journal-reports/[id]` | GET/POST/DELETE | คลังความรู้ รายงานและเนื้อหาวิจัยที่ผู้ใช้เซฟเก็บไว้ |
| `/api/thaijo-topics` | POST | ระบบแนะนำหัวข้อที่น่าสนใจบน ThaiJo |
| `/api/pdf/*` | หลายแบบ | ระบบไลบรารีรวมไฟล์ PDF สำหรับคลังความรู้ |
| `/api/python/[prefix]/[...path]` | หลายแบบ | ตัวทะลวงแบบ White-list ให้ส่งคำสั่งข้ามไปคุยกับฝั่ง Python Backend ตรงๆ ในเส้นทางจำกัดที่อนุญาต |

### เส้นทางการเลี้ยว (Routing) จากตัวแปร `mode` (ต้นทาง `/api/chat`)

| โหมดที่ส่งมา (`mode`) | ส่งต่อไปที่ Backend Endpoint ใด |
|---|---|
| `thaijo` | `POST /api/thaijo` |
| `thaijo-report` | `POST /api/thaijo/report` |
| `compare` / `report` / `database` / `pubmed` | วิ่งเข้าหา API เฉพาะตัว `/api/{mode}` |
| `normal` / `stats` / `obsidian` / `multi` / `tavily` / `research` / `report-gather` | `POST /api/analyze` |

## 4.3 ระบบเซิร์ฟเวอร์ปัญญาประดิษฐ์ (Backend API) — FastAPI routers (python-ai)

ระบบ Backend ถูกแบ่งการทำงานของ Router ตามหมวดหมู่ชัดเจน:

| ชื่อระบบ (Router) | คำนำหน้า (Prefix) | เส้นทางสำคัญ (Key endpoints) |
|---|---|---|
| analyze | `/api`, `/api/chat` | `POST /api/analyze`, `POST /api/chat`, `GET /health` |
| accident_chat | `/api/accident-chat` | `POST /ask`, `/ask/stream`, `/quick`, `GET /provinces`, `/districts`, `/sample-questions` |
| accident_policy | `/api/accident-policy` | `GET /zone10/data`, `POST /zone10` |
| obsidian | `/api/obsidian` | `POST /search`, `/ask`, `GET /notes`, `/status`, `/vaults`, `POST /index`, `/pdfs/sync` |
| thaijo | `/api/thaijo` | `POST /api/thaijo`, `/report`, `/topics`, `/demo` |
| pubmed | `/api/pubmed` | ฟังก์ชันค้นหาและการวิเคราะห์วารสารทางสุขภาพ PubMed |
| tools_router | `/api` | `POST /compare`, `/report`, `/workplan`, `/database` |
| db_explorer | `/api/db` | `GET /tables`, `/tables/{name}/columns`, `/tables/{name}/rows` |
| pdf_ingest | `/api/pdf-ingest` (proxied) | รับหน้างาน Chunking ไฟล์เอกสาร PDF ทีละส่วน |
| error_log | `/api/errors` | `GET /`, `/stats`, `/summary`, `DELETE /` |

## 4.4 โปรโตคอลการสื่อสารข้อมูล (Communication interfaces)

- **Client ↔ BFF (หน้าจอ ↔ ระบบหน้าด่าน):** ใช้ HTTPS/HTTP แบบปกติในการร้องขอ (JSON requests) แต่ตอนรับคำตอบการแชทจะผ่านกลไก **Server-Sent Events (SSE)** รูปแบบ `text/event-stream` รักษาความปลอดภัยแบบ HttpOnly cookie
- **BFF ↔ Backend (ระบบหน้าด่าน ↔ สมอง AI):** ใช้โปรโตคอล HTTP ฝังพารามิเตอร์รหัสลับ `x-internal-key` ที่ส่วนหัว (Header); เมื่อสตรีม SSE ระบบจะใช้ท่าทาง `tee()` ในการถ่ายโอน; มีข้อจำกัด (Timeout) นานสูงสุด 10 นาทีสำหรับท่อคำสั่งขนาดใหญ่
- **พจนานุกรมกิจกรรมสตรีม (SSE event vocabulary):** `agent_start`, `agent_done`, `crew_plan`, `text_stream_start`, `text_chunk`, `result`, `final`, `error`

## 4.5 การเชื่อมต่อระบบภายนอกตัวแอป (External service interfaces)

| บริการที่ใช้ (Service) | วัตถุประสงค์ (Use) | การเข้าถึง (Auth) |
|---|---|---|
| Google Gemini (ผ่านไลบรารี litellm) | แกนสมอง LLM หลักของแอปและ Agents ทั้งหมด | `GEMINI_API_KEY` |
| OpenAI | เป็นระบบ LLM สำรอง | `OPENAI_API_KEY` |
| Tavily | เครื่องมือสำหรับค้นหาข้อมูลผ่านเครือข่ายอินเทอร์เน็ต (Web search) | `TAVILY_API_KEY` |
| ThaiJo API | ฐานข้อมูลวารสารวิจัยของประเทศไทย | ไม่ใช้รหัส (none) |
| PubMed / NCBI E-utilities | วารสารวรรณกรรมทางการแพทย์ระดับสากล | (ตัวเลือกเสริม) `NCBI_API_KEY` |
| Open-Meteo | เครื่องมือช่วยตรวจสอบสภาพอากาศในแต่ละพื้นที่ | ไม่ใช้รหัส (none) |

## 4.6 อินเทอร์เฟซการจัดการคลังข้อมูล (Data-store interfaces)

- **PostgreSQL** (`musyadata`, ปลั๊กอิน pgvector เวอร์ชัน 16) — เก็บตารางข้อมูลพื้นฐานแอปพลิเคชัน, ตารางวิเคราะห์อุบัติเหตุรูปแบบ Star schema, และดัชนีของ Obsidian
- **MinIO** — Object Storage ภายใน (เปรียบเสมือน AWS S3) เก็บข้อมูลในถัง `fileapa` (ไฟล์ที่อัปโหลดและอ้างอิง) และ `pdf-library` (ไฟล์ PDF)
- **Redis** — หน่วยความจำเก็บประวัติแชทระยะสั้น (เก็บ 6 เทิร์น, อายุ 24 ชั่วโมง) และเป็นที่เก็บพักข้อมูล (Cache) สำหรับ Pipeline กับ ThaiJo

*สามารถอ่านรายละเอียดของตารางข้อมูลทั้งหมดได้ใน [[06 - Data Requirements]]*
