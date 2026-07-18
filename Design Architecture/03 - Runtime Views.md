---
title: "Design — Runtime Views (แผนภาพลำดับการทำงานและซีเควนซ์)"
tags: [MUSYA, design, sequence, runtime]
created: 2026-07-18
---

# 🔁 ภาพจำลองลำดับการทำงานเชิงระบบ (Runtime Views — Sequence Diagrams)

← [[02 - Frontend Design]] · กลับไปสู่หน้าภาพรวม → [[00 - Architecture Overview]]

> นี่คือมุมมองเคลื่อนไหว (Dynamic view) ที่จำลองเหตุการณ์จริงของการทำงานข้ามระบบจากบรรดา [[00 - Actors & Use-Case Map|ยูสเคสหลักๆ]] โดยใช้ตัวย่อของผู้แสดง (Actors) ดังนี้:
> **U**=ฝั่งผู้ใช้งานบนเบราว์เซอร์, **BFF**=ระบบหน้าด่าน Next.js, **BE**=ฝั่งสมองกลจัดสรรงาน FastAPI, **Mem**=ความจำอัจฉริยะ (Memory Agent), **Rtr**=สายแยกวง (Router Agent), **Pipe**=ท่อปฏิบัติงานเฉพาะกิจ (Pipeline ที่ถูกเลือก), **PG/Redis/MinIO**=คลังเสบียงฐานข้อมูลต่างๆ

## 1. ลำดับการล็อกอินเข้าสู่ระบบ ([[01 - Account & Access|อ้างอิง UC-02]])

```mermaid
sequenceDiagram
    participant U as หน้าจอ Browser
    participant BFF as ทางเข้า /api/auth/login
    participant PG as ฐานข้อมูล PostgreSQL
    U->>BFF: ยิงคำสั่ง POST {ส่งอีเมล, รหัสผ่าน}
    BFF->>PG: สอบถาม SELECT หาบัญชีจากอีเมล
    PG-->>BFF: ส่งข้อมูลบัญชีกลับมา (สถานะ, แฮชรหัสผ่าน, ระดับบทบาท)
    alt หาก สถานะ status ไม่ใช่ 'approved'
        BFF-->>U: ดีดออก เตือน 403 (รอดำเนินการ/ถูกเด้งออก)
    else ถ้า 'approved' แล้วและรหัส bcrypt ตรงกันเป๊ะ
        BFF->>BFF: ปั๊มตราเซ็นชื่อให้ (ออกเหรียญ JWT เข้ารหัส HS256, อายุ 7 วัน)
        BFF-->>U: คืนคุกกี้ Set-Cookie เป็นชื่อ auth_token (แอบใน HttpOnly) + พร้อมรหัสผ่านทาง 200
    end
```

## 2. ลำดับการแชทถามแบบปกติพร้อมรื้อฟื้นความจำเดิม ([[02 - Chat & Domain Analysis|อ้างอิง UC-06/UC-12]])

```mermaid
sequenceDiagram
    participant U as กล่องพิมพ์แชท ChatInput
    participant BFF as ตัวคุม /api/chat
    participant BE as กองบัญชาการ _orchestrate
    participant Mem as เอเจนต์ด้านความจำ Memory Agent
    participant Rtr as เอเจนต์สับแยกทาง Router Agent
    participant Pipe as ท่อ Pipeline
    participant R as บันทึกแชทข้ามคืน Redis

    U->>BFF: ยิง POST {โหมด:normal, คำถามล่าสุด prompt, รหัสห้อง sessionId, ประวัติ history}
    BFF->>BFF: ตรวจบัตรผ่านเข้างาน (requireAuth เช็ค JWT)
    BFF->>BE: อนุมัติผ่าน POST /api/analyze (แนบรหัสลับ x-internal-key)
    BE->>BE: เสียบกุญแจตรวจสอบช่องว่าง (semaphore.acquire วิ่งไม่เกิน ≤5 สาย)
    BE->>R: ดึงขุดประวัติความหลังเก่ามา
    BE->>Mem: นำคำถามปะติดปะต่อขยายบริบทจากคำถามสั้นๆ ให้รู้เรื่อง
    Mem-->>BE: ส่งกลับมาเป็นโครงสร้างประโยคสมบูรณ์แบบ (สั่ง SSE สถานะ agent_done บอกหน้าจอ)
    BE->>Rtr: โยนให้สับแยกว่ามีเนื้อหาหมวดไหนบ้าง (route_multi_domain)
    Rtr-->>BE: คายผลตอบว่าโดเมนอะไร, เป็นเรื่องซับซ้อนไหม (is_multi)
    BE->>Pipe: สั่งเปิดเครื่องยนต์ท่อทำงาน (เริ่มยิง SSE เปรยว่าเริ่ม/จบของย่อย)
    Pipe-->>U: สตรีมผลลัพธ์มาทาง SSE ให้อ่านทีละบรรทัด (result / final)
    BE->>R: จดบันทึกคำตอบผู้ช่วยเก็บเข้าถัง
    BFF->>BFF: แตกสายสำรองเก็บสถิติถาวร (fallback-persist (tee) → หย่อนลงตาราง chat_sessions)
```

## 3. ลำดับการงัดข้อสถิติข้อมูลอุบัติเหตุแบบรวดเร็ว ([[02 - Chat & Domain Analysis|อ้างอิง UC-07]])

```mermaid
sequenceDiagram
    participant U as กล่องพิมพ์แชท ChatInput
    participant BE as กองบัญชาการ _orchestrate (เข้าสู่ mode:stats)
    participant G as ป้อมสกัดกรองเขตจังหวัด Zone guard
    participant SQL as เอเจนต์ฐานข้อมูล SQL Agent
    participant PG as ตารางดาวกระจาย PostgreSQL

    U->>BE: พิมพ์ถาม "สถิติอุบัติเหตุอุบล ปี 2567"
    BE->>G: มองหาตัวอักษรชื่อจังหวัดในประโยค
    alt ถ้าปรากฏว่ามีแต่ชื่อจังหวัดนอกเขตล้วนๆ
        G-->>U: ยิง SSE ถล่มกลับบอกว่า "ไม่มีข้อมูล" + ร่ายลิสต์ชื่อ 5 จังหวัดที่ควรหา (แล้วแอบจดประจานลง Log)
    else โชคดีอยู่ในเขตที่ดูแล (in-zone)
        BE->>BE: จัดการแปลงอักษร พ.ศ.2567 → ค.ศ.2024 เตรียมพร้อม
        BE->>SQL: เรียกสั่งวิ่งเข้าชน run_accident_chat(ส่งชื่อจังหวัด, ปี)
        SQL->>PG: ดำดิ่งคิวรีข้อมูลใน mart_province_road ประกอบร่างกับ dim_geography
        PG-->>SQL: ผลตอบกลับออกมาเป็นแถวข้อมูล (rows)
        SQL-->>U: ส่ง SSE เรียงเนื้อหาใหม่ตีตราด้วยตาราง Markdown tables + ข้อจำกัดเชิงอรรถให้พองาม (limit footnotes)
    end
```

## 4. ลำดับการสั่งผลิตชิ้นส่วนตั้งต้นเพื่อสร้างรายงานแบบมโหฬาร ([[03 - Report Generation|อ้างอิง UC-13/UC-14]])

```mermaid
sequenceDiagram
    participant U as กล่องพิมพ์แชท ChatInput
    participant BE as กองบัญชาการ _orchestrate (ลุยใน mode:report-gather)
    participant G as ป้อมสกัดกรองเขตจังหวัด Zone guard
    participant POOL as คอกคนงานเธรดพูล ThreadPool (5 คน, วิ่งทิ้งช่วงกันนิดๆ)
    participant SRC as แก๊งจัดหา Obsidian/Stats/ThaiJo/PubMed/Tavily
    participant J as ตู้เซฟเก็บเอกสาร journal_reports

    U->>BE: สั่งงาน "ทำ Policy Brief ... อุบลราชธานี"
    BE->>G: ตรวจเช็คว่าเป็นจังหวัดที่ถูกต้องหรือไม่
    G-->>BE: เรียบร้อย ในเขตพื้นที่ → ให้รันงานต่อได้
    BE->>POOL: ปล่อยม้าทั้ง 5 ออกจากซอง (แต่หน่วงเวลาให้ออกห่างกัน ~1.5 วิ)
    POOL->>SRC: กระจายกำลังไปทุกทิศพร้อมเพรียงกัน (parallel)
    SRC-->>U: ยิงภาพผลสตรีม SSE รายงานตัว agent_start/done + เทเนื้อหา text_chunk (ถมฝั่งขวาของหน้าจอ)
    BE-->>U: สตรีมผลลัพธ์ฉากจบสุดท้าย SSE final {ข้อมูลเนื้อหาก้อนยักษ์รวมกัน + ชี้เป้าตัวอ้างอิงของทุกแหล่งให้ชัดๆ}
    U->>U: ระบบหน้าจอแนะแนวผู้ใช้สร้างรายงาน (report wizard) → ผู้ใช้จิ้มเลือกรูปแบบรายงาน doc_type
    U->>J: กดส่งเซฟ (ส่งข้อมูลงัดหัวเรื่อง, doc_type, พร้อมก้อนร่าง html_content)
    U->>U: นำออกดาวน์โหลดใส่แฟ้ม DOCX / PDF ไปเสิร์ฟผู้ใหญ่
```

## 5. ลำดับการตะลุยค้นความรู้เชิงลึกในองค์กร ([[02 - Chat & Domain Analysis|อ้างอิง UC-09]])

```mermaid
sequenceDiagram
    participant U as กล่องพิมพ์แชท ChatInput
    participant BE as กองบัญชาการ _orchestrate (เปลี่ยนเป็น mode:obsidian)
    participant OB as ปรมาจารย์คลังข้อมูล obsidian_fullcontext
    participant FS as ถังเก็บโน้ต .md ใน Vault
    U->>BE: โยนคำถามหมวดวิชาการนโยบายมา
    BE->>BE: ใช้สมองกรองหาชื่อจังหวัดก่อนเพื่อเป็นแผงกั้นจำกัดบริบทไม่ให้กว้างไป (bound context)
    BE->>OB: ลุยวิ่งเข้าท่า run_obsidian_ask_fullcontext(โยนชื่อจังหวัดใส่)
    OB->>FS: ดึงข้อมูลไฟล์โน้ตเป้าหมายที่ค้นเจอเข้ามาโหลดในหัว (ลิมิตห้ามเกิน ≤500,000 คาแรกเตอร์)
    OB-->>U: ดันส่งคำตอบ SSE โต้กลับไป + ตามมาด้วยรายชื่อโน้ตที่ระบบแอบแว๊บไปอ่าน + เสนอแนะคำถามถัดไปที่น่าจะอยากรู้ต่อ (follow_ups)
```

## 6. กฎรันไทม์บังคับที่คอยสอดส่องครอบคลุมทุกการทำงาน (Cross-cutting runtime rules)

| กฎเหล็กที่ระบบถือสา (Rule) | ด่านตรวจสอบจุดไหน (Where enforced) | อ้างอิงแหล่ง (Trace) |
|---|---|---|
| ต้องโชว์ตั๋ว Auth ทุกครั้งที่จะเข้าทำงาน | ถูกเช็คที่ยาม `middleware.ts` / และ `requireAuth` | FR-AUTH-09 |
| ขุดท่อไปป์ไลน์ให้วิ่งได้ไม่เกิน ≤5 สายต่อเวิร์กเกอร์ | ถูกคุมขังด้วยตัวแปร `_AI_SEMAPHORE` วางไว้ในห้อง `analyze.py` | NFR-PERF-02 |
| กฎการรอจังหวะหน่วงเวลาเมื่อเซิร์ฟเวอร์เต็ม 429 | ถูกสอดไส้ฝังทับอยู่ข้างในวิญญาณ `agent_defaults` | NFR-REL-01 |
| ระบบห้ามตายคากลางอากาศถ้าผู้ใช้ออฟไลน์ | ใช้วิชาแยกสตรีม `tee()` ของ BFF คู่กับวิชาบันทึกลับ `persistFallbackCompletion` | FR-CHAT-14 |
| ถามเรื่องนอกเขต ก็ต้องตีกลับว่าไม่มีข้อมูล (no-data) | ด่านสกัดกั้นหน้าประตูกับคนตรวจจับ `missing_data_logger` | FR-CHAT-08 |
| ภูมิประวัติคำตอบของผู้ช่วย BE ต้องถูกจดจำเสมอ | ฝังประวัติด้วย `history.append_history` (โยนเข้าความจำ Redis) | FR-CHAT-13 |
