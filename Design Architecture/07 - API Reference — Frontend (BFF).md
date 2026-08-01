---
title: "Design — API Reference: Frontend BFF (คู่มืออ้างอิง API ฝั่งหน้าบ้าน BFF)"
tags: [MUSYA, design, api, frontend, bff, reference]
created: 2026-07-18
---

# 📡 คู่มืออ้างอิง API ฝั่ง Frontend BFF (`chatappandpython`) แยกตามรายไฟล์

← [[06 - API Reference — Backend]] · กลับสู่ภาพรวม → [[00 - Architecture Overview]] · อ้างอิงสเปกเดิม: [[02 - Frontend Design]]

> เอกสารฉบับนี้จัดทำขึ้นเพื่อชี้แจง **รายละเอียดตัวจัดการเส้นทาง (Route Handler) ทีละไฟล์** ที่ซ่อนอยู่ในหมวด `app/api/**/route.ts` ฝั่งหน้าบ้าน — จะแจกแจงตั้งแต่ชนิด Method, หน้าที่รับผิดชอบ, 
> ตลอดจนชี้เป้าว่ามันลักลอบส่งต่อ (proxy) ข้อมูลข้ามกำแพงไปให้ใคร
> **กฎเหล็กของเขตนี้:** ทุกๆ เส้นทางเดิน (ยกเว้นเรื่องล็อกอินบางส่วน) จะถูกบังคับให้เดินผ่านด่านตรวจ **`requireAuth()`** เพื่อขอดูบัตร JWT ก่อนเสมอ
> (ถ้าไม่มีบัตร หรือบัตรหมดอายุ จะถูกถีบส่งกลับด้วยรหัส 401 ทันที) และทุกครั้งที่ระบบฝั่งหน้าบ้านจะหันไปตะโกนคุยกับฝั่งหลังบ้าน (backend) มันจะแอบสวมหน้ากากแนบรหัสผ่าน `x-internal-key` ผ่านกลไก `internalHeaders()` ไว้ให้เสมอ
> **ข้อมูลฐาน URL ที่ควรรู้:** `http://localhost:3000` · (หากอยากรู้หน้าตาเดต้าเต็มๆ ให้แวะไปดู [[04 - Data Architecture & Schema]])

---

## 1. กองทะเบียนราษฎร์: ระบบสมาชิก & สิทธิ์ — `app/api/auth/*`

| ชื่อไฟล์ | วิธีเรียก (Method) | หน้าที่ความรับผิดชอบ |
|---|---|---|
| `auth/register/route.ts` | POST | ช่องทางสมัครสมาชิกใหม่ → สั่งสร้างข้อมูลบัญชีลงตาราง `accounts` (โดนป้ายยศเริ่มต้น `role=user`, สถานะรอดำเนินการ `status=pending`), พร้อมจับรหัสผ่านไปสับละเอียดด้วยท่า bcrypt(12) |
| `auth/login/route.ts` | POST | ช่องทางเข้าสู่ระบบ → สแกนหาสถานะว่าผ่านการอนุมัติ `status=approved` หรือยัง + เทียบรหัสผ่านให้ตรง → จากนั้นออกบัตรผ่าน JWT (ตีตรา HS256, ให้ใบอนุญาต 7 วัน) แล้วยัดใส่คุกกี้แบบ HttpOnly (ไม่ให้ฝั่งสคริปต์หน้าเว็บมองเห็น) |
| `auth/logout/route.ts` | POST | ช่องทางออกจากระบบ → สั่งล้างไพ่ เคลียร์คุกกี้ชื่อ `auth_token` ทิ้งลงชักโครก |
| `auth/me/route.ts` | GET / PATCH | ช่องทางดูโปรไฟล์ตัวเอง / หรือส่งฟอร์มขอแก้ไขข้อมูลส่วนตัว |
| `auth/users/route.ts` | GET / PATCH / DELETE | **(เขตหวงห้ามเฉพาะแอดมินขั้นเทพ adminsuper)** ใช้ขอดูลิสต์บัญชีลูกบ้าน / กดปุ่มอนุมัติ-เตะปรับสถานะ / หรือจับลบประหารผู้ใช้ทิ้ง |
| `auth/forgot-password/route.ts` | POST | ช่องทางลืมรหัสผ่าน → ออกป้ายชั่วคราว `reset_token` (ซึ่งมีวันหมดอายุ) |
| `auth/reset-password/route.ts` | POST | ช่องทางตั้งรหัสใหม่ → ตรวจป้าย token ว่ายังไม่บูด → อนุญาตให้ตั้งรหัสผ่านใหม่ (ใช้เสร็จฉีกทิ้งเลย ใช้ซ้ำไม่ได้) |

อ้างอิงยูสเคสการใช้งาน: [[01 - Account & Access]]

---

## 2. กองอำนวยการแชท & ประวัติความจำ — `app/api/chat/*`

| ชื่อไฟล์ | วิธีเรียก (Method) | หน้าที่ความรับผิดชอบ |
|---|---|---|
| `chat/route.ts` | POST | ★ **จุดศูนย์กลางหัวใจของ BFF** — ทำหน้าที่สับรางชี้เป้าแปล `mode` → ส่งต่อไปหาปากทาง endpoint ของฝั่ง backend, แล้วทำตัวเป็นท่อส่งต่อสตรีม **SSE** แบบเรียลไทม์; ที่เด็ดคือมีการใช้ท่านินจาแยกร่าง `stream.tee()` ประกอบกับวิชา `persistFallbackCompletion()` เพื่อกู้ภัยแอบบันทึกคำตอบลงฐานข้อมูลไว้ให้ แม้ผู้ใช้อินเทอร์เน็ตหลุดกระจุยหรือปิดเบราว์เซอร์หนีไปแล้วก็ตาม |
| `chat/history/route.ts` | GET / POST | ช่องทางโหลดประวัติแชทเก่า / และช่องทางบันทึกสถานะเซสชันสนทนารอบใหม่ยัดตู้ (`chat_sessions.messages_json`) |

**คู่มือการสับราง `mode` → วิ่งไปหาฝั่ง backend** (ตามกฎใน `chat/route.ts`):
- โหมด `thaijo` → สับไปเข้าท่อ `/api/thaijo`
- โหมด `thaijo-report` → สับไปเข้าท่อ `/api/thaijo/report`
- แก๊งเครื่องมือ `compare` / `report` / `database` / `pubmed` → สับพุ่งตรงไปเข้าท่อ `/api/{mode}`
- ส่วนพวกหมวดจับฉ่ายอื่นๆ (`normal` / `stats` / `obsidian` / `multi` / `tavily` / `research` / `report-gather` / 🆕 `report-gather-retry`) → จะถูกจับมัดรวมวิ่งไปเข้าประตูใหญ่ `/api/analyze`

**🆕 3 ฟิลด์ที่ BFF ส่งผ่าน (pass-through) ไปให้ backend เพิ่ม** — ตัว `chat/route.ts` ไม่ได้แปลงค่าอะไร แค่หยิบจาก body แล้วยัดต่อไปตรง ๆ:

| ฟิลด์ | ใช้กับโหมด | ความหมายย่อ |
|---|---|---|
| `doc_type` | `report-gather` | ชนิดเอกสารที่เลือกไว้ล่วงหน้า → backend echo กลับเป็น `docType` ใน `final` ให้ wizard ข้ามขั้นเลือกประเภท |
| `retry_source` | `report-gather-retry` | ชื่อแหล่งเดียวที่จะรันซ้ำ (`obsidian`/`stats`/`thaijo`/`pubmed`/`tavily`) |
| `report_title` | `report-gather` | ชื่อเรื่องสั้น ๆ ที่ผู้ใช้พิมพ์จริง แยกจาก `prompt` ที่อาจถูกเสริมหัวข้อจนยาว |

**คำเตือนความอดทน:** ตัวอัปสตรีมตั้งค่ารอไว้สูงสุด **10 นาที** (เผื่อ AI คิดช้า) · อ้างอิงยูสเคส: [[02 - Chat & Domain Analysis]], [[03 - Report Generation]]

---

## 3. กองบริหารจัดการไฟล์ & การเสกอ้างอิง APA — `app/api/files/*`, `app/api/generate-apa`

| ชื่อไฟล์ | วิธีเรียก (Method) | หน้าที่ความรับผิดชอบ |
|---|---|---|
| `files/route.ts` | GET / POST | ขอดูกระดานรายชื่อไฟล์ทั้งหมด / โยนไฟล์ใหม่เข้าไปเก็บ (เป้าหมายคือถัง MinIO หมวด `fileapa`) |
| `files/upload/route.ts` | POST | ช่องทางส่งไฟล์ชิ้นใหญ่ๆ (รับของผ่าน multipart) |
| `files/[fileId]/route.ts` | GET / DELETE / PATCH | ขอเปิดดูไฟล์ตัวนั้น / กระทืบลบไฟล์ทิ้ง / แก้ไขป้ายชื่อเมทาดาทาของไฟล์ |
| `files/[fileId]/ai-metadata/route.ts` | POST | สั่งให้ AI รุมแงะเข้าไปหาเมทาดาทาที่ซ่อนอยู่ |
| `files/[fileId]/insights/route.ts` | GET | ขอดูบทสรุปวิสัยทัศน์ (AI insights) ที่เจาะได้จากตัวไฟล์ |
| `files/merge/search/route.ts` | POST | ช่องทางควานหาไฟล์เพื่อเตรียมการจับแพ็กมัดรวมกัน |
| `files/merge/analyze/route.ts` | POST | สั่งวิเคราะห์ไส้ในให้ชัวร์ก่อนจับรวม |
| `files/merge/execute/route.ts` | POST | กดปุ่มรันคำสั่งจับตะกรุมตะกรามรวมร่างไฟล์ |
| `files/merge/save/route.ts` | POST | จุดเซฟผลงานการรวมร่างเป็นก้อนชุดข้อมูลใหม่ |
| `generate-apa/route.ts` | POST | โรงงานเสกบรรณานุกรมแบบ APA อัตโนมัติ → แล้วโยนผลงานไปเก็บไว้ใน MinIO |

อ้างอิงยูสเคส: [[04 - Knowledge, Files & Admin]]

---

## 4. ห้องสมุดคลังรายงานผลงาน — `app/api/journal-reports/*`

| ชื่อไฟล์ | วิธีเรียก (Method) | หน้าที่ความรับผิดชอบ |
|---|---|---|
| `journal-reports/route.ts` | GET / POST | ขอดูกระดานรายชื่อรายงานของตนเอง / นำส่งรายงานเล่มใหม่เข้าไปเก็บบนหิ้ง (ลงตาราง `journal_reports`) |
| `journal-reports/[id]/route.ts` | GET / DELETE | เปิดอ่านรูปเล่มรายงาน / หรือฉีกรายงานตัวเองทิ้ง — 🆕 ตอน DELETE จะ **กวาดล้างปุ่มรายงานที่ชี้มาที่ id นี้ออกจากทุกเซสชันแชทของเจ้าของคนเดียวกันด้วย** |

> [!warning] 🆕 ทำไม DELETE ต้องไปยุ่งกับ `chat_sessions` ด้วย
> ปุ่ม "เปิดรายงาน" ถูกฝังไว้ใน `messages_json` ของ `chat_sessions` (ผ่าน `reportSavePersist.ts`) ถ้าลบแถวใน `journal_reports` ทิ้งเฉย ๆ ปุ่มเก่าจะยังค้างอยู่ในแชท กดแล้ว fetch คืน 404 **เงียบ ๆ ไม่มีอะไรเกิดขึ้น ผู้ใช้งงว่าพัง** — ฟังก์ชัน `pruneDeletedReportFromSessions()` จึงไล่ scan ทุกเซสชันของ user คนนั้น (`WHERE messages_json::text LIKE '%<id>%'`) แล้วถอดปุ่มออกให้เรียบร้อยในทรานแซกชันเดียวกัน

---

## 5. ประตูวิเศษมุดข้ามแดน Passthrough ไป Backend — กลุ่ม proxy routes

| ชื่อไฟล์ | วิธีเรียก (Method) | แอบโผล่พาทะลุไปส่งที่ปลายทาง |
|---|---|---|
| `db/[...path]/route.ts` | GET / POST | วิ่งตรงดิ่งไปโผล่ที่ backend ท่อ `/api/db/*` (ช่องทาง DB Explorer) |
| `obsidian/[...path]/route.ts` | GET / POST | วิ่งตรงดิ่งไปโผล่ที่ backend ท่อ `/api/obsidian/*` (คลังสมอง) |
| `python/[prefix]/[...path]/route.ts` | * (รับได้หมด) | **จุดนี้มีบัญชีขาว (whitelist) คุมเข้ม**: ปล่อยผ่านให้เฉพาะตระกูล `accident-chat` \| `accident-policy` \| `db` \| `obsidian` เท่านั้น |
| `thaijo-topics/route.ts` | POST | วิ่งตรงดิ่งไปโผล่ที่ backend ท่อ `/api/thaijo/topics` |

> [!note] คำเตือนป้อมปืนด่านความปลอดภัยของโหมดพาสทรู (passthrough)
> สังเกตเส้นทาง `python/[prefix]` ระบบจะจงใจอนุญาตปล่อยผ่านเฉพาะคำนำหน้า (prefix) ที่มีรายชื่ออยู่ในบัญชีขาว (whitelist) เท่านั้น — มาตรการนี้มีไว้สกัดขาไม่ให้ฝั่ง frontend แอบสร้างทางลัดวิ่งเข้าไปล้วงตับ endpoint ลับๆ ภายในทุกตัวของ backend ได้อย่างเสรีเกินงาม โดยไม่ได้ตั้งใจ (มาตรการอุดรอยรั่วตามกติกา [[05 - Non-Functional Requirements|NFR-SEC-04]])

---

## 6. สายพานลำเลียงเอกสาร PDF & เข้ากรุ Vault — `app/api/pdf/*`

| ชื่อไฟล์ | วิธีเรียก (Method) | หน้าที่ความรับผิดชอบ (เป็นตัวส่งผ่าน proxy → วิ่งหาฝั่ง backend `/pdf/*`) |
|---|---|---|
| `pdf/upload/route.ts` | POST | ช่องส่งเอกสาร PDF ข้ามแพะ |
| `pdf/ingest/route.ts` | POST | สับสวิตช์เครื่องบดย่อยกินข้อมูล (ingest) |
| `pdf/ingest/status/[job_id]/route.ts` | GET | แวะมาถามความก้าวหน้าการกินข้อมูล |
| `pdf/files/route.ts` | GET | ขอกางบัญชีรายชื่อไฟล์ PDF ดิบทั้งหมด |
| `pdf/files/[id]/route.ts` | DELETE | สั่งฉีกทำลายไฟล์ทิ้ง |
| `pdf/view/[id]/route.ts` | GET | ขอเบิกตัวมาเปิดดู PDF หน่อย |
| `pdf/obsidian-view/[...assetId]/route.ts` | GET | ช่องทางพิเศษงัดดูไฟล์ PDF ฉบับที่อยู่ในเซฟ vault โดยเฉพาะ |
| `pdf/vault/route.ts` | GET | ขอดูโครงสร้างกระดูกสันหลังของตัว vault |
| `pdf/vault/file/route.ts` | GET / PUT / DELETE | สารพัดบริการ อ่าน / แต่งเติมเขียน / ดึงป้ายลบ ทิ้งไฟล์นามสกุล .md |
| `pdf/vault/rename/route.ts` | POST | ขอเปลี่ยนชื่อใบปะหน้าไฟล์ |
| `pdf/vault/db-stats/route.ts` | GET | ขอดูสถิตินับจำนวน note/chunk ในฐานข้อมูล |
| `pdf/vault/migrate-from-filesystem/route.ts` | POST | คำสั่งปาฏิหาริย์ย้ายมวลสารจาก filesystem เก่าๆ → เข้าไปสถิตใน DB |
| 🆕 `pdf/analyze-placement/route.ts` | POST | ให้ AI เดาปลายทางที่ควรเก็บก่อนกด ingest จริง |
| 🆕 `pdf/ingest-batch/route.ts` | POST | สั่งคิว ingest หลายไฟล์รวดเดียว |
| 🆕 `pdf/ingest-batch/status/[batchId]/route.ts` | GET | ถามความคืบหน้าทั้งคิว + log เต็มรายไฟล์ |
| 🆕 `pdf/ingest-batch/cancel/[batchId]/route.ts` | POST | ยกเลิกคิวที่เหลือ |

---

## 🆕 6.5 ห้องควบคุมค่าย AI — `app/api/llm/*` (2026-07-30)

| ชื่อไฟล์ | วิธีเรียก | หน้าที่ |
|---|---|---|
| `llm/providers/route.ts` | GET | รายชื่อค่ายให้ผู้ใช้ทั่วไปเลือก · **ล้มแล้วคืนรายการว่าง** ไม่ให้หน้าแชทพังตาม |
| `llm/admin/providers/route.ts` | GET / PUT | อ่าน/แก้ค่า — เฉพาะ `adminsuper` |
| `llm/admin/providers/test/route.ts` | POST | ยิงทดสอบจริงว่าค่าที่ตั้งใช้ได้ |

> [!danger] จุดที่ความปลอดภัยทั้งหมดไปกองอยู่
> backend เชื่อ header `X-User-Role` เพราะเชื่อว่า Next.js ตรวจ session มาแล้ว
> **ความรับผิดชอบนั้นอยู่ที่ไฟล์เหล่านี้ล้วน ๆ** — role ต้องมาจาก JWT ฝั่ง server
> (`requireAuth()`) เท่านั้น **ห้ามรับจาก body หรือ header ที่ client ส่งมาเด็ดขาด**
> ไม่งั้นใครก็ปลอมเป็น adminsuper ได้
>
> ถ้าวันหน้าเปิด backend ตรงสู่อินเทอร์เน็ต ต้องย้ายไปตรวจ JWT ที่ฝั่ง Python เอง

---

## 7. บริการรับจ้างเฉพาะกิจของเซิร์ฟเวอร์ (Server Action)

| ชื่อไฟล์ | ชนิดตัวทำงาน | หน้าที่ความรับผิดชอบ |
|---|---|---|
| `app/actions/upload.ts` | Server Action (พุ่งตรง) | เป็นตัวทาสรับใช้อัปโหลดไฟล์จากหน้าจอฟอร์ม (พุ่งชนฟังก์ชันตรงๆ เลย ไม่ต้องอ้อมเสียเวลาผ่านท่อ REST route) |

---

## บทสรุปกติกามารยาทส่วนกลางที่ทุกท่อต้องปฏิบัติตาม

1. **ด่านตรวจคนเข้าเมือง (Auth):** ตัวผู้คุม `requireAuth()` จะสแกนล้วงหาบัตร JWT cookie เสมอ → ถ้าหมดอายุหรือของปลอมจะถีบส่งกลับด้วยหน้าต่าง 401 (สาเหตุเพราะโฟลเดอร์นี้ไม่มีทหารยามหน้ากำแพงใหญ่อย่าง `middleware.ts` คอยเฝ้าให้ — จึงต้องบังคับจับตรวจลายนิ้วมือที่ด่านเก็บเงินรายตัวแทน ไปอ่านต่อได้ที่ [[05 - File Structure Tree]])
2. **การแอบเรียกหา backend ท้ายวัง:** โค้ดจะต้องซ่อนตัวผ่านฟังก์ชัน `internalHeaders()` เพื่อจับยัดสร้อยกุญแจลับ `x-internal-key` เสมอก่อนเดินเข้าห้อง
3. **การพ่นไฟ SSE:** เส้นทางไหนที่รับจ๊อบรีเลย์สตรีมข้อความ จะต้องแปะป้าย header แจ้งล่วงหน้าว่า `Content-Type: text/event-stream`, แถมสั่งปิดบัฟเฟอร์ `X-Accel-Buffering: no` เพื่อกันของไหลมากระตุก
4. **ลัทธิกรรมสิทธิ์ความเป็นเจ้าของ:** พวกหีบสมบัติอย่าง journal / files / sessions จะถูกจับผูกพันธะกับรหัส `userId` ที่ถอดรหัสมาจากบัตร JWT เสมอ — รับประกันได้ว่าไม่มีใครแอบมาขโมยดูผลงานของคนอื่นได้แน่นอน เข้าถึงได้เฉพาะข้าวของตัวเองเท่านั้น
