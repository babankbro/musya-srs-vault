# 📘 MUSYA AI — Software Documentation Vault

คลังเอกสารซอฟต์แวร์ (Obsidian vault) ของ **MUSYA AI** — *"เพื่อนรักนักออกแบบนโยบายสุขภาพอัจฉริยะ"*
ระบบผู้ช่วยสร้างเอกสารนโยบายสุขภาพด้วย AI Agent + RAG สำหรับ **เขตสุขภาพที่ 10**
(อุบลราชธานี · ศรีสะเกษ · ยโสธร · อำนาจเจริญ · มุกดาหาร)

> เอกสารทั้งหมดถอด (reverse-engineer) มาจากซอร์สโค้ดจริง 2 รีโพ:
> **Backend** `chatapi.python` (FastAPI + CrewAI) · **Frontend** `chatappandpython` (Next.js 16 + BFF)

---

## 🚀 เริ่มอ่านจากตรงนี้

- 🏠 **[index.md](index.md)** — หน้าหลัก (Map of Content) รวมลิงก์ทุกหมวด
- 📦 **[Setup & Onboarding Guide](Guides/00%20-%20Setup%20%26%20Onboarding%20Guide.md)** — ติดตั้งและรันระบบด้วย Docker
- 🏗️ **[Architecture Overview](Design%20Architecture/00%20-%20Architecture%20Overview.md)** — ภาพรวมสถาปัตยกรรม (C4)

## 🗂️ โครงสร้างคลังเอกสาร (40 โน้ต, 7 หมวด)

| หมวด | เนื้อหา |
|---|---|
| 📄 [SRS/](SRS) | ข้อกำหนดความต้องการซอฟต์แวร์ (Introduction, Functional/Non-Functional, Data, Interfaces) |
| 🎬 [Use Cases/](Use%20Cases) | กรณีการใช้งาน + สถานการณ์จำลอง (Actors, Account, Chat, Report, Admin) |
| 🏗️ [Design Architecture/](Design%20Architecture) | สถาปัตยกรรม, Backend/Frontend design, Data schema, File tree, API reference, Runtime views |
| 🔧 [Workflow/](Workflow) | เวิร์กโฟลว์ทีละขั้น (Frontend → Backend → Database) แยกตามเครื่องมือ |
| 🤖 [Agents/](Agents) | เอเจนต์ AI ทุกตัว (prompt: role/goal/backstory + ability: tools/LLM) |
| 📦 [Guides/](Guides) | คู่มือปฏิบัติ (Setup & Onboarding) |
| 📚 [Reference/](Reference) | อ้างอิง: Glossary, SSE Protocol, Prompt Strategy, Traceability Matrix |

---

## 📌 หมายเหตุการเปิดดู

- ไฟล์ `.md` เปิดอ่านบน GitHub ได้เลย และ **แผนภาพ Mermaid render อัตโนมัติ**
- วอลต์นี้ออกแบบสำหรับ **Obsidian** — ลิงก์ภายในเป็น `[[wikilink]]` ซึ่ง **บน GitHub จะไม่กดได้**
  (แนะนำเปิดในแอป Obsidian เพื่อใช้ graph view + backlinks + callouts เต็มรูปแบบ)
- เอกสารเป็นภาพ **As-Built** — หากแก้โค้ดภายหลัง ควรตรวจสอบกับซอร์สจริงเสมอ

## ⚠️ Disclaimer

เอกสารกล่าวถึงค่า **dev/seed credentials** (เช่นบัญชีทดสอบใน `schema.sql`) เพื่อประกอบคำอธิบาย
ข้อมูลเหล่านี้เป็นค่าสำหรับพัฒนาเท่านั้น — **ต้องเปลี่ยน/หมุนก่อนใช้งานจริง (production)**
วอลต์นี้ **ไม่มี** API key จริงใด ๆ (Gemini/OpenAI/Tavily)

---

*จัดทำโดยการวิเคราะห์ซอร์สโค้ด · เขตสุขภาพที่ 10 · มหาวิทยาลัยอุบลราชธานี / สสส.*
