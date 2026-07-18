---
title: MUSYA AI — Software Documentation Vault (ฉบับภาษาไทย)
tags:
  - MUSYA
  - SRS
  - use-cases
  - as-built
created: 2026-07-18
---

# 📘 MUSYA AI — ข้อกำหนดซอฟต์แวร์และกรณีการใช้งาน (Software Requirements & Use Cases)

> **ระบบ:** MUSYA AI — *"เพื่อนรักนักออกแบบนโยบายสุขภาพอัจฉริยะ"* — ระบบ AI ผู้ช่วยสร้างเอกสารนโยบายสุขภาพสำหรับ **เขตสุขภาพที่ 10** (ประกอบด้วยจังหวัด อุบลราชธานี · ศรีสะเกษ · ยโสธร · อำนาจเจริญ · มุกดาหาร)
>
> คลังเอกสาร (Vault) นี้เป็น **ข้อกำหนดสเปคของซอฟต์แวร์ (Software Specification)** ที่ทำการวิเคราะห์และถอดโครงสร้างมาจากโค้ดการทำงานจริงในเวอร์ชันล่าสุด (Reverse-engineered จาก Source Code ในแฟ้ม `new_version_musya/`) ซึ่งประกอบด้วย 
> - ส่วน Backend (API และ AI): [`chatapi.python`](https://github.com/saffyzaza/chatapi.python) (พัฒนาด้วย FastAPI + CrewAI)
> - ส่วน Frontend (แอปพลิเคชันและ BFF): [`chatappandpython`](https://github.com/saffyzaza/chatappandpython) (`musyav2`, พัฒนาด้วย Next.js 16 + BFF)

---

## 🚦 คำแนะนำในการอ่านคลังเอกสารนี้

ผู้ใช้งานควรเริ่มต้นอ่านจาก **SRS (Software Requirements Specification)** เพื่อทำความเข้าใจว่าระบบ "ต้องทำอะไรได้บ้าง" (What the system must do) จากนั้นจึงอ่าน **Use Cases (กรณีการใช้งาน)** เพื่อทำความเข้าใจว่า "ผู้ใช้งานแต่ละกลุ่มใช้งานระบบอย่างไรเพื่อให้บรรลุเป้าหมาย" พร้อมกับตัวอย่างสถานการณ์แบบเจาะลึก 

ข้อกำหนดทุกข้อในเอกสารนี้จะมีรหัสอ้างอิงกำกับ (เช่น `FR-*` สำหรับ Functional, `NFR-*` สำหรับ Non-Functional, `DR-*` สำหรับ Data Requirements) เพื่อให้การออกแบบ Use Cases และการเขียน Test Cases ในอนาคตสามารถตรวจสอบย้อนกลับ (Traceability) ได้อย่างถูกต้อง

## 📄 ข้อกำหนดความต้องการของซอฟต์แวร์ (Software Requirements Specification - SRS)

เอกสารในหมวดนี้จะแจกแจงรายละเอียดและขอบเขตทางเทคนิคและฟังก์ชันของระบบ:
1. [[01 - Introduction & Scope]] — บทนำ, วัตถุประสงค์, ขอบเขตของระบบ, ผู้ใช้งานที่เกี่ยวข้อง และคำจำกัดความ
2. [[02 - Overall Description]] — ภาพรวมของระบบ, บริบทของสถาปัตยกรรม, ข้อจำกัดของระบบ และสมมติฐานการทำงาน
3. [[03 - Functional Requirements]] — รายการฟังก์ชันการทำงานทั้งหมดของระบบ แบ่งตามกลุ่มฟีเจอร์หลัก
4. [[04 - External Interface Requirements]] — ความต้องการด้านส่วนติดต่อ (Interface) เช่น หน้าจอ UI, REST API และการเชื่อมต่อ Backend
5. [[05 - Non-Functional Requirements]] — ข้อกำหนดที่ไม่ใช่ฟังก์ชันหลัก เช่น ประสิทธิภาพการทำงาน (Performance), ความปลอดภัย (Security), ความน่าเชื่อถือ (Reliability) และความสะดวกในการใช้งาน (Usability)
6. [[06 - Data Requirements]] — ข้อกำหนดด้านข้อมูลเชิงลึก เช่น ฐานข้อมูลที่ใช้จัดเก็บ (Persistent stores), เอนทิตี (Entities) และระยะเวลาการเก็บรักษาข้อมูล

## 🎬 กรณีการใช้งานและสถานการณ์จำลอง (Use Cases & Scenarios)

เอกสารหมวดนี้เน้นมุมมองของผู้ใช้งานที่ตอบโต้กับระบบเพื่อทำงานต่างๆ ให้สำเร็จ:
- [[00 - Actors & Use-Case Map]] — แผนผังแสดงผู้ใช้งานทั้งหมด (Actors) และ Use-Case Diagram
- [[01 - Account & Access]] — การจัดการบัญชีผู้ใช้ (การลงทะเบียน, การอนุมัติบัญชี, การเข้าสู่ระบบ และการรีเซ็ตรหัสผ่าน)
- [[02 - Chat & Domain Analysis]] — ระบบการแชทถาม-ตอบ, การวิเคราะห์ข้อมูลเชิงลึก, การดึงสถิติ, และการสืบค้นข้อมูลจากฐานข้อมูลงานวิจัย
- [[03 - Report Generation]] — กระบวนการสร้างรายงานนโยบาย (การรวบรวมข้อมูล → ตัวช่วยสร้าง → รายงานประจำวัน → การนำออกเอกสาร)
- [[04 - Knowledge, Files & Admin]] — การจัดการคลังความรู้ (Vault), ไฟล์เอกสารและการอ้างอิง APA, ระบบสำรวจฐานข้อมูล และการจัดการสำหรับแอดมิน

## 🏗️ การออกแบบสถาปัตยกรรม (Design Architecture — As-Built)

เอกสารหมวดนี้อธิบายว่าระบบถูกวางโครงสร้างอย่างไรเพื่อตอบโจทย์ข้อกำหนดข้างต้น (ถอดจากโค้ดจริง):
- [[00 - Architecture Overview]] — ภาพรวมสถาปัตยกรรม (C4 context/container), ชั้นการทำงาน และคุณสมบัติเชิงคุณภาพ
- [[01 - Backend Design]] — โครงสร้างระบบหลังบ้าน: Routers, เอเจนต์ CrewAI, Pipelines, Tools และการคุมคอนเคอร์เรนซี
- [[02 - Frontend Design]] — โครงสร้างหน้าจอ + BFF: ระบบยืนยันตัวตน, SSE proxy, และการจัดการสถานะแชท
- [[03 - Runtime Views]] — แผนภาพลำดับการทำงาน (Sequence diagrams) ของยูสเคสหลัก
- [[04 - Data Architecture & Schema]] — สถาปัตยกรรมข้อมูลและสคีมาฐานข้อมูล (ERD 3 อาณาเขต, DDL จริง, การไหลข้อมูล)
- [[05 - File Structure Tree]] — ผังโครงสร้างไฟล์ของ Frontend และ Backend + แผนที่ฟีเจอร์→ไฟล์
- [[06 - API Reference — Backend]] — 🆕 รายละเอียด API ฝั่ง Backend รายไฟล์ (10 routers, method/request/response)
- [[07 - API Reference — Frontend (BFF)]] — รายละเอียด API ฝั่ง BFF รายไฟล์ (route handlers + proxy)

## 🔧 เวิร์กโฟลว์ทีละขั้น (Workflow — Frontend → Backend → Database)

เอกสารหมวดนี้ไล่ขั้นตอนการทำงานจริงจากหน้าจอถึงฐานข้อมูล แยกตามเครื่องมือแต่ละตัว:
- [[00 - Workflow Index]] — 🆕 ดัชนี + แกนกลางที่ทุกเวิร์กโฟลว์ใช้ร่วมกัน + แผนที่แหล่งข้อมูล
- [[01 - Chat Normal Workflow]] — ถามทั่วไป (ไม่เลือกปุ่ม, `normal`)
- [[02 - Statistics Workflow]] — ปุ่มสถิติ (อุบัติเหตุ SQL / CSV, `stats`)
- [[03 - Knowledge Vault Workflow]] — ปุ่มคลังความรู้รายงาน (`obsidian`)
- [[04 - Research Workflow]] — ปุ่มวิจัย (ThaiJo/PubMed, `research`)
- [[05 - Web Search Workflow]] — ปุ่มค้นหาทั่วไป (Tavily, `tavily`)
- [[06 - Report Generation Workflow]] — ปุ่มสร้างรายงาน (`report-gather` → journal)
- [[07 - Auth & Session Workflow]] — ล็อกอิน/สมัคร/บันทึกเซสชัน (แตะ DB ตรง)

## 🤖 เอเจนต์รายเครื่องมือ (Agents — prompt + ability)

เอกสารหมวดนี้อธิบายเอเจนต์ AI แต่ละตัวในแต่ละเครื่องมือ ทั้งพรอมต์ (role/goal/backstory) และความสามารถ (tools/LLM):
- [[00 - Agent Catalogue]] — 🆕 สารบัญ + โครงสร้างเอเจนต์ + โมเดล LLM/tier ที่ใช้
- [[01 - Shared Agents (Memory & Router)]] — Memory Agent + Router/Classifier ส่วนกลาง
- [[02 - Stats Tool Agents]] — อุบัติเหตุ SQL (2) · CSV (6) · Multi-CSV (3)
- [[03 - Knowledge Tool Agent]] — Obsidian Full-Context (Gemini ตรง + SYSTEM_PROMPT)
- [[04 - Research Tool Agents]] — ThaiJo (Planner/Insight/Summary/Generator) · PubMed (Keyword/Fetcher)
- [[05 - Web Search Tool Agents]] — Web Search Specialist · Research Answer Writer
- [[06 - Report & Policy Agents]] — Zone10 Policy (3) · Report/Workplan/Compare/Database

## 📦 คู่มือ (Guides)

เอกสารเชิงปฏิบัติ — วิธีติดตั้งและเริ่มใช้งานจริง:
- [[00 - Setup & Onboarding Guide]] — 🆕 ติดตั้งด้วย Docker, ตั้ง `.env` 2 ฝั่ง, seed DB, index vault, troubleshooting

## 📚 เอกสารอ้างอิง (Reference)

เอกสารสำหรับเปิดค้น/ทำความเข้าใจเชิงลึก:
- [[00 - Glossary & Domain Context]] — 🆕 อภิธานศัพท์ (สสจ./Haddon/BSC/HDC/RAG/BFF ฯลฯ) + บริบทสาธารณสุข
- [[01 - SSE Event Protocol]] — 🆕 สัญญาการสตรีม SSE (ทุก event type + payload + ลำดับต่อ pipeline)
- [[02 - Prompt Strategy & Anti-Hallucination]] — 🆕 กลยุทธ์พรอมต์ร่วม + กลไกกันข้อมูลมั่ว + ช่องว่าง determinism
- [[03 - Traceability Matrix]] — 🆕 ตารางสอบย้อนกลับ FR↔UC↔Design↔Agent↔Test + สรุปช่องว่าง

---

## 🧭 สรุปการทำงานของระบบโดยสังเขป (One-paragraph System Summary)

เมื่อเจ้าหน้าที่สาธารณสุขที่เข้าสู่ระบบแล้วทำการพิมพ์คำถามลงในช่องแชท (โดยสามารถเลือก **เครื่องมือ (Tool)** ได้ดังนี้: สถิติ / ค้นหาทั่วไป / วิจัย / คลังความรู้รายงาน / สร้างรายงาน) ระบบ Frontend ที่ทำหน้าที่เป็น **BFF (Backend-for-Frontend)** บน Next.js จะตรวจสอบสิทธิ์ผู้ใช้งาน (Authentication ด้วย JWT) และส่งผ่านคำขอนั้นไปยัง FastAPI Backend ผ่านช่องทางที่มีความปลอดภัยสูงด้วย `INTERNAL_API_KEY` จากนั้น Backend จะเรียกใช้งาน **Memory Agent** และส่งต่อให้ **Router Agent** เพื่อวิเคราะห์และเลือก **ท่อส่งข้อมูลของ CrewAI (CrewAI Pipelines)** ที่เหมาะสมที่สุด (เช่น Accident-SQL, CSV, Multi-CSV, ThaiJo, PubMed, Obsidian, Tavily) ระบบจะไปดึงข้อมูลจากแหล่งต่างๆ (PostgreSQL / MinIO / Redis / External APIs) และประมวลผลส่งคำตอบกลับมาให้ผู้ใช้ในรูปแบบการสตรีมแบบเรียลไทม์ (**Server-Sent Events**) ผู้ใช้สามารถนำผลลัพธ์ที่ได้ไปรวบรวมเป็นเอกสารทางนโยบาย (**Policy Brief / Strategy Plan / Work Plan**) บันทึกลงในคลังเอกสารส่วนตัว และสามารถนำออกแบบ DOCX หรือ PDF ได้ อนึ่ง ขอบเขตของข้อมูลจะถูกจำกัดเฉพาะใน 5 จังหวัดของเขตสุขภาพที่ 10 เท่านั้น

> [!info] คลังเอกสารที่เกี่ยวข้อง (Companion Vault)
> สำหรับเอกสารระดับสถาปัตยกรรม "As-Built" แบบเจาะลึกการวางโครงสร้างระบบ (เช่น รูปแบบการ Deploy, Catalog รายละเอียดของ Agent แต่ละตัว และ Data Dictionary) จะถูกเก็บแยกไว้ในคลังข้อมูลอีกแห่งชื่อ `musya-obsidian-document` คลังเอกสารฉบับนี้จะเน้นไปที่ **ความต้องการและพฤติกรรมของระบบ (Requirements and Behaviour)** เป็นหลัก ไม่รวมการออกแบบเชิงลึกของชิ้นส่วนภายใน
