---
title: "Reference — SSE Event Protocol (โปรโตคอลการสตรีมข้อมูล SSE)"
tags: [MUSYA, reference, sse, protocol, streaming]
created: 2026-07-18
---

# 🔌 ข้อมูลอ้างอิง: โปรโตคอลการสตรีมข้อมูล SSE (Server-Sent Events)

← กลับไป [[00 - Glossary & Domain Context]] · ตอนถัดไป → [[02 - Prompt Strategy & Anti-Hallucination]] · กลับไปสารบัญเวิร์กโฟลว์: [[00 - Workflow Index]]

> **SSE (Server-Sent Events)** คือสายเลือดหลัก (กระดูกสันหลัง) ของการสื่อสารทางเดียวจากระบบหลังบ้านไหลไปสู่หน้าจอผู้ใช้ (Backend → BFF → Browser)
> คำตอบเชิงวิเคราะห์ทั้งหมดจะไม่ส่งมาแบบก้อนตู้มเดียว แต่จะไหลพ่นออกมาเป็นสตรีม `text/event-stream` โดยแต่ละบรรทัดที่เด้งมาจะอยู่ในรูปแบบ `data: {json}\n\n`
> เอกสารฉบับนี้รวบรวม **ชนิด event ทั้งหมด + หน้าตาโครงสร้างข้อมูล (payload) ของจริง** (ถอดรหัสชำแหละมาจากไฟล์ `analyze.py` และ `ChatInput.tsx`)

---

## 1. ภาพรวมการไหลของข้อมูล (Flow Overview)

```
[หลังบ้าน Python] ฟังก์ชัน _orchestrate (ทำงานแบบ thread)
   │  แอบจับข้อมูลยัดใส่กล่อง put(event) → ส่งเข้าท่อ asyncio.Queue
   ▼
ไฟล์ analyze.py ฟังก์ชัน stream() จะทำหน้าที่คายของออกมาทีละบรรทัด  →  รูปแบบ "data: {json}\n\n"  (ประกาศตัวเป็น text/event-stream)
   │
   ▼  [ด่านหน้า BFF] ไฟล์ chat/route.ts  (ใช้ท่าไม้ตาย tee: แยกร่างสตรีม เส้นนึงส่ง browser, อีกเส้นลอบส่งให้ fallback-persist เซฟลงฐานข้อมูล)
   ▼
[เบราว์เซอร์] ฝั่ง React/Next.js ไฟล์ ChatInput.tsx  →  ทำการ parse งับท่อน "data: " ทิ้ง → แล้วสั่งอัปเดต state stores สดๆ → โชว์ข้อความดุ๊กดิ๊กบน LeftPane/RightPane
```

- **Header ที่ฝังไว้:** ระบบประกาศตั้งหัว `Content-Type: text/event-stream`, แถมสั่งปิดแคช `Cache-Control: no-cache`, และห้ามเว็บเซิร์ฟเวอร์ Nginx อมข้อมูลด้วย `X-Accel-Buffering: no`
- **วิธีตัดจบสตรีม (EOF):** เมื่อ backend ทำงานเสร็จ จะหย่อนค่า `None` ลงคิว → ตัว `stream()` จะรู้ทันทีว่าหมดรอบ แล้วจัดการปิดสวิตช์ยุติการเชื่อมต่อ (connection close)

---

## 2. ชนิด Event ทั้งหมดในระบบ (Event Types)

| ชนิด `type` | ความหมายและจังหวะที่เกิด | ก้อนข้อมูลหลัก (Payload) |
|---|---|---|
| `agent_start` | สัญญาณว่า เอเจนต์เพิ่งเริ่มตื่นขึ้นมาทำงานในขั้นนั้นๆ (โชว์ไฟกะพริบ) | `step`, `agentName` |
| `agent_done` | สัญญาณว่า เอเจนต์ตัวนั้นๆ ทำงานในขั้นของตนเสร็จแล้ว | `step`, `agentName`, `result` (ผลย่อย), (และอาจมี `reasoning?`, `domain?`, `province?`, `file_count?`, `articleCount?`) |
| `crew_plan` | บอกแผนการรบของ crew (มักเกิดในบาง pipeline ที่ต้องวางแผนก่อน) | รายละเอียดแผน (plan details) |
| `text_stream_start` | สัญญาณปี่กลอง เริ่มเทสตรีมเนื้อหายาวๆ (ไปโชว์ที่จอฝั่งขวา RightPane) | `articleCount` (จำนวนบทความอ้างอิง) |
| `text_chunk` | สะเก็ดชิ้นข้อความที่ทยอยยิงมาทีละคำ (Typewriter effect) | `text` (ก้อนอักษรเล็กๆ) |
| `obsidian_stream_start` | 🆕 สัญญาณว่า Gemini เริ่มเขียนคำตอบคลังความรู้แล้ว — ยิงก่อน `obsidian_chunk` ชุดแรกเสมอ | `step` (= `obsidian_search`) |
| `obsidian_chunk` | 🆕 สะเก็ดคำตอบสด ๆ ของสาย Obsidian ทีละ token (ลด perceived latency ของคำถามที่กิน ~50-60 วิ) | `step`, `text` |
| `report_source_status` | 🆕 สถานะรายแหล่งข้อมูล (badge 5 อัน) ในโหมดสร้างรายงาน — ยิงซ้ำหลายรอบต่อแหล่งตามสถานะที่เปลี่ยน | `source`, `label`, `status` (`pending`\|`running`\|`done`\|`error`), `message?` (เฉพาะตอน error) |
| `result` | สัญญาณคายคำตอบสุดท้ายจบงาน (ใช้เฉพาะโหมดแชทจบในตัว เช่น obsidian/accident) | `content` (เนื้อหา), `domain`, (อาจมี `notesReferenced?`, `followUps?`) |
| `final` | สัญญาณอลังการ ผลลัพธ์สุดท้าย (ใช้เฉพาะโหมดงานช้าง research/report) | `message`, `textResult` (เนื้อหายาว), `articlesText` (ซากอ้างอิง), `articleCount`, `reportTitle`, `agentSteps`, 🆕 `docType`, 🆕 `retrySource` |
| `error` | สัญญาณล่มปากอ่าว เกิดข้อผิดพลาดร้ายแรงขึ้นแล้ว | `message` (ข้อความฟ้อง) |

> **คู่มือถอดรหัส `step`:** โค้ดขั้นตอน เช่น `memory` (เกลาความจำ), `router` (คนสับราง), `accident_sql` (วิ่งหา SQL อุบัติเหตุ), `schema` (ดูโครงสร้าง CSV), `code_gen` (เขียนโค้ด), `insight` (คนนั่งเทียนวิเคราะห์), `vault_rag` (ล้วงคลังความรู้), `obsidian_search` (คนหากระดาษโน้ต), `search`, `fetcher`, `relevance` (🆕 ยามคัดบทความคนละเรื่อง), `planner`, `generator`

> 🆕 **`step: "relevance"`** — ยิงจากทั้ง `run_thaijo_pipeline` และ `run_pubmed_pipeline` ทันทีหลัง `fetcher`
> `agent_done` ของขั้นนี้มีคีย์เพิ่ม: `articleCount` (จำนวนที่เหลือ), `droppedCount` (จำนวนที่คัดออก),
> `reasoning` (รายชื่อบทความที่ถูกคัดพร้อมเหตุผล) · ดู [[04 - Research Tool Agents]] หัวข้อ A0.5
> ⚠️ `articleCount` ของ `fetcher` = จำนวน**ก่อน**กรอง ส่วนของ `relevance` และ `final` = **หลัง**กรอง

> **คู่มือถอดรหัส `source` (ของ `report_source_status`):** มี 5 ค่าตายตัวเท่านั้น — `obsidian` (คลังความรู้), `stats` (สถิติ), `thaijo` (งานวิจัยไทย), `pubmed` (งานวิจัยสากล), `tavily` (ค้นหาเว็บ) ส่วน `label` คือชื่อไทยสำเร็จรูปที่ backend ส่งมาให้แปะบน badge ได้เลย ไม่ต้อง map เอง

---

## 3. หน้าตา Payload ของจริง (JSON Structure)

### 🟢 `agent_done` (กรณีแจ้งผลลัพธ์พร้อมบอกโดเมนด้วย)
```json
{ 
  "type": "agent_done", 
  "step": "router", 
  "agentName": "Router Agent",
  "result": "สถิติ: โรคไม่ติดต่อ", 
  "reasoning": "พบคำใบ้พูดถึงเรื่องเบาหวานและความดัน...",
  "domain": { "code": "d3", "nameTh": "โรคไม่ติดต่อ", "nameEn": "NCDs" } 
}
```

### 🔵 `result` (โหมดที่ถามแล้วจบในตัว — สาย obsidian / accident SQL)
```json
{ 
  "type": "result", 
  "content": "## สรุปภาพรวมยุทธศาสตร์เขต...\nเนื้อหายาวๆ",
  "notesReferenced": [ { "title": "วาระประชุมศปถ. 67", "path": "ประชุม/ศปถ_2567.md" } ],
  "followUps": ["อยากดูสถิติแยกรายอำเภอต่อไหม?", "สนใจดูนโยบายย้อนหลังปี 66 ไหม?"],
  "domain": { "code": "obsidian", "nameTh": "คลังความรู้สุขภาพ เขต 10" } 
}
```

### 🟣 `final` (โหมดลากยาว — สาย research / report-gather)
```json
{ 
  "type": "final", 
  "message": "พบงานวิจัยที่เกี่ยวข้อง 8 รายการ... (โชว์จอซ้าย)",
  "textResult": "## บทสรุปงานวิจัย...\n(เนื้อหาเต็มโชว์จอขวา)",
  "articlesText": "วัตถุดิบอ้างอิงแบบแยกทีละแหล่ง (ซ่อนเก็บไว้ให้ report generator ดึงไปใช้)",
  "articleCount": 8, 
  "reportTitle": "รายงานทบทวนวรรณกรรมอุบัติเหตุปี 2567", 
  "agentSteps": [ /* ประวัติการรันเอเจนต์ทั้งหมด */ ],
  "docType": "policy",
  "retrySource": null
}
```

> [!note] เจาะ 2 ฟิลด์ใหม่ท้าย `final`
> - **`docType`** — echo ชนิดเอกสารที่ผู้ใช้เลือกไว้ตั้งแต่ตอนกดปุ่ม "สร้างรายงาน" (`policy` / `plan` / `workplan`) กลับมาให้หน้าจอ เพื่อให้ wizard **ข้ามขั้นตอนถามซ้ำว่าจะทำเอกสารประเภทไหน** แล้วกระโดดไปขั้นสร้างหัวข้อได้ทันที (ค่าว่างถ้าไม่ได้เลือกมาก่อน)
> - **`retrySource`** — ถ้าไม่ใช่ `null` แปลว่าก้อนนี้มาจากปุ่ม "ลองใหม่" ของแหล่งเดียว หน้าจอต้อง **ต่อท้าย (append) section ใหม่เข้ากับเนื้อหาเดิม** ไม่ใช่ทับทั้งก้อน มิฉะนั้นผลของอีก 4 แหล่งที่สำเร็จไปแล้วจะหายวับ

### 🟣 `final` (โหมด `d0` — AI ทั่วไป · 🆕 2026-07-30)

```json
{
  "type": "final",
  "message": "การควบคุมโรคพยาธิใบไม้ตับทำได้หลายทาง...",
  "domain": { "code": "d0", "nameTh": "ทั่วไป", "nameEn": "General" },
  "chatProvider": { "key": "gemini", "nameTh": "Gemini" },
  "suggestedTools": [
    {
      "tool": "obsidian",
      "labelTh": "ค้นคลังความรู้ (338 เอกสาร)",
      "hintTh": "ตอบใหม่โดยอ้างอิงเอกสารของเขตสุขภาพที่ 10 พร้อมบรรณานุกรม"
    }
  ],
  "agentSteps": [ /* router / reasoning / insight */ ]
}
```

> [!note] เจาะ 2 ฟิลด์ใหม่ของสาย `d0`
> - **`chatProvider`** — บอกว่าคำตอบก้อนนี้มาจากค่ายไหนจริง ๆ (`gemini` / `chatgpt` / `claude`)
>   จำเป็นเพราะผู้ใช้เลือกค่ายได้เอง ถ้าไม่ echo กลับมาก็ไม่มีทางรู้ว่าใครตอบ —
>   ชื่อค่ายยังถูกใส่ใน `agentSteps[].agentName` ด้วย (`Insight Analyst (Gemini)`)
> - **`suggestedTools`** — เครื่องมือที่ระบบเสนอให้ **ถามซ้ำอัตโนมัติ** หน้าจอวาดเป็นปุ่ม ⚡
>   กดแล้วเลือกเครื่องมือ + ใส่คำถามเดิม + ส่งเองครบในคลิกเดียว
>   **ส่งมาเฉพาะเมื่อมีของจริงให้ดึง** — backend นับเอกสารในคลังก่อนเสมอ ได้ 0 = ไม่ส่งฟิลด์นี้
>   (ปุ่มที่กดแล้วได้ "ไม่พบข้อมูล" แย่กว่าไม่มีปุ่ม)
>   · ค่า `tool` ที่เป็นไปได้ตอนนี้: `obsidian` (คลังความรู้) · `stats` (ข้อมูลสถิติ)

### 🟡 `obsidian_stream_start` / `obsidian_chunk` (🆕 สาย Obsidian สตรีมสด)
```json
{ "type": "obsidian_stream_start", "step": "obsidian_search" }
{ "type": "obsidian_chunk", "step": "obsidian_search", "text": "จากรายงานประจำปี 2566 พบว่า" }
```

> [!warning] มี "เบรกมือ" ซ่อนอยู่ในสตรีมนี้
> ระหว่างสตรีม ระบบเฝ้าดู **หาง** ของบัฟเฟอร์ (400 ตัวอักษรท้ายสุด) ตลอดเวลา ถ้าจับได้ว่ามีเนื้อหาดิบของเอกสารต้นฉบับหลุดออกมา (บรรทัด `FILE:`, บล็อก YAML frontmatter, wikilink `[[...]]`) จะ **หยุดยิง `obsidian_chunk` ทันทีกลางคัน** แล้วปล่อยให้ตัว guard/retry ระดับบนจัดการเขียนคำตอบใหม่ — ดังนั้นฝั่งหน้าจอ **ห้ามเอาผลรวมของ `obsidian_chunk` ไปใช้เป็นคำตอบสุดท้ายเด็ดขาด** ต้องรอ `result` เสมอ เพราะสตรีมอาจถูกตัดจบกลางประโยคโดยตั้งใจ (ดู [[02 - Prompt Strategy & Anti-Hallucination]])

### 🟠 `report_source_status` (🆕 ป้ายสถานะ 5 แหล่งในโหมดสร้างรายงาน)
```json
{ "type": "report_source_status", "source": "obsidian", "label": "คลังความรู้ (Obsidian)", "status": "pending" }
{ "type": "report_source_status", "source": "obsidian", "label": "คลังความรู้ (Obsidian)", "status": "running" }
{ "type": "report_source_status", "source": "thaijo", "label": "งานวิจัยไทย (ThaiJo)", "status": "error", "message": "429 quota exceeded" }
```

> ลำดับที่การันตี: ทุกแหล่งจะได้ `pending` **ครบทั้งชุดก่อน** แล้วจึงทยอยเป็น `running` ทีละตัว (ห่างกัน ~1.5 วิ กัน 429) และปิดท้ายด้วย `done` หรือ `error` ตัวใดตัวหนึ่งเสมอ — แหล่งที่ `error` จะมีปุ่ม "ลองใหม่" ขึ้นบน badge ให้กดยิง `report-gather-retry` เฉพาะแหล่งนั้น

### 🔴 `error` (กรณีระบบแครช หรือคนแน่นเซิร์ฟเวอร์เต็ม)
```json
{ 
  "type": "error", 
  "message": "ระบบกำลังประมวลผลเต็มความสามารถ (คิวเต็ม) กรุณารอสักครู่แล้วลองใหม่" 
}
```

---

## 4. ฝั่งหน้าจอ (Frontend) เอา Event ไปปู้ยี่ปู้ยำยังไงต่อ (`ChatInput.tsx`)

| Event ที่เด้งมา | ตัวแปร State ที่โดนอัปเดต | ผลสะเทือนที่แสดงบนหน้าจอ |
|---|---|---|
| `agent_start` / `agent_done` | ร้านค้า `streamingStore` | แถบซ้าย (LeftPane) จะมีแท็บกล่อง AgentPipelinePanel เด้งขึ้นมา โชว์หลอดสเตตัสการวิ่งงานทีละสเต็ป |
| `text_stream_start` / `text_chunk` | ร้านค้า `thaijoStore` / หรือบัฟเฟอร์ของ RightPane | แถบขวา (RightPane) จะโชว์ตัวหนังสือค่อยๆ วิ่งพิมพ์แบบ Typewriter / HTML สดๆ |
| 🆕 `obsidian_stream_start` / `obsidian_chunk` | บัฟเฟอร์สตรีมของ LeftPane | ข้อความคำตอบคลังความรู้ค่อยๆ ไหลขึ้นในฟองแชทระหว่างรอ (แต่ยัง **ไม่ใช่** คำตอบจริง — ตัวจริงมากับ `result`) |
| 🆕 `report_source_status` | ร้านค้า `reportSourceStore` | แถบ badge 5 อัน (`ReportSourceBadges.tsx`) โชว์สถานะรายแหล่งแบบเรียลไทม์ + ปุ่ม "ลองใหม่" บนตัวที่ `error` |
| `result` | ร้านค้า `chatSessionStore` | แสดงข้อความคำตอบของแชท + โผล่ป้ายอ้างอิง (notesReferenced) + โผล่ปุ่มเสนอคำถาม (followUps) |
| `final` | `chatSessionStore` + อัปเดต RightPane | จอซ้ายโชว์ข้อความสั้น + จอขวาโชว์เนื้อหาเต็ม + อาจแถมปุ่มเด้งเปิดหน้าต่าง Report Wizard (ถ้ามี `docType` ติดมาก็ข้ามขั้นเลือกประเภทเอกสารไปเลย) |
| `error` | ร้านค้า `chatSessionStore` | แปะป้ายตัวแดงแสดงข้อความ error ประจานกลางจอแชท |

---

## 5. แอบดูลำดับ Event ที่วิ่งไหลตามแต่ละท่อ (Pipeline Example Flow)

| ท่อการทำงาน (Pipeline) | ลำดับการไหลของ Event เรียงจากซ้ายไปขวา (ทั่วไป) |
|---|---|
| สาย Normal (หลุดเข้า Obsidian) | `agent_start:memory` → `agent_done:memory` → `agent_start:router` → `agent_done:router` → `agent_start:obsidian_search` → 🆕 `obsidian_stream_start` → 🆕 `obsidian_chunk` รัวๆ → `agent_done` → จบที่ `result` |
| สาย Stats (หลุดเข้า อุบัติเหตุ SQL) | `agent_start:router` → `agent_done:router` → `agent_start:accident_sql` → `agent_done` → จบที่ `result` |
| สาย Stats (หลุดเข้าแก๊งทุบ CSV) | `router` → `file_finder` → `schema` → `code_gen` → `executor` → `insight` (ทุกตัวมี start/done ประกบหัวท้าย) → จบที่ `result` |
| สายวิจัย Research | `text_stream_start` → (`agent_start/done` ของทีม ThaiJo + PubMed) → พ่น `text_chunk` รัวๆ → จบที่ `final` |
| สายสร้างรายงาน (Report-gather) | `text_stream_start` → 🆕 `report_source_status:pending` ครบทั้ง 5 แหล่ง → 🆕 `report_source_status:running` ทยอยทีละตัว → (`agent_start/done` ระดมพลจาก 5 แหล่ง) + พ่น `text_chunk` รัวๆ → 🆕 `report_source_status:done`/`error` รายตัว → จบที่ `final` (มี `docType`) |
| 🆕 สายลองใหม่รายแหล่ง (Report-gather-retry) | เหมือนสายบนเป๊ะ **แต่มีแหล่งเดียว** — `text_stream_start` → `report_source_status:pending`/`running` ของแหล่งที่ขอ → `agent_start/done` + `text_chunk` → `done`/`error` → จบที่ `final` (มี `retrySource` = ชื่อแหล่งนั้น ให้หน้าจอ append ไม่ใช่ทับ) |

---

## 6. ความเหนียวแน่นทนทานตายยาก (Durability & Reliability)

- **นกต่อ BFF ใช้ท่าแยกร่าง `tee()`** — มันจะทำการฉีกแยกสายสตรีมเป็น 2 เส้น: สายหนึ่งสาดขึ้นไปให้ browser บนหน้าจอ, ส่วนอีกสายแอบส่งให้ภารโรง `persistFallbackCompletion()` คอยจด
  ภารโรงจะนั่งงับ event `final` / `result` / หรือ `error` ก้อนสุดท้าย แล้วจารึกอัดลงฐานข้อมูล `chat_sessions` ทันที (ป้องกันอุบัติเหตุเน็ตฝั่งผู้ใช้หลุดกลางคัน แล้วสถานะใน DB จะค้างเติ่งเป็น `running` ตลอดกาล)
  เจาะลึกได้ที่ [[07 - Auth & Session Workflow]], หรืออ่านสเปคบังคับทำที่ [[03 - Functional Requirements|FR-CHAT-14]]
- **เซิร์ฟเวอร์เต็ม (Busy State):** ถ้ายามหน้าประตูพบว่าคิว semaphore เต็มแล้ว (คนถามพร้อมกันเยอะจัด) ระบบหลังบ้านจะพ่น event `error` ทิ้งใส่หน้าผู้ใช้ทันที โดยไม่เสียเวลาเปิดท่อ pipeline ใดๆ (ป้องกันเซิร์ฟพัง) (ตามคำสั่งสเปค [[03 - Functional Requirements|FR-CHAT-15]])

*แหล่งอ้างอิงรอยเท้าในโค้ด: ดูท่อส่งที่ `src/routers/analyze.py` (ฟังก์ชัน put/stream), ดูการแยกร่างที่ `app/api/chat/route.ts` (tee), และดูคนแปลภาษาที่ `app/component/chat/ChatInput.tsx` (parser)*
