---
title: "Agents — สารบัญเอเจนต์ทุกเครื่องมือ (Agent Catalogue)"
tags: [MUSYA, agents, catalogue, crewai, prompt]
created: 2026-07-18
---

# 🤖 สารบัญรวบรวมทีมเอเจนต์แบ่งตามรายเครื่องมือ (Agent Catalogue)

← [[index|หน้าหลัก]] · เส้นทางเวิร์กโฟลว์: [[00 - Workflow Index]] · โครงสร้างดีไซน์: [[01 - Backend Design]]

> เอกสารในโฟลเดอร์ซีรีส์นี้จะพาคุณไปเจาะลึก **คาแรคเตอร์ของเอเจนต์ AI แต่ละตัวที่สิงอยู่ในแต่ละเครื่องมือ** อย่างละเอียดยิบ — 
> ซึ่งจะกางให้ดูทั้ง **"พรอมต์คำสั่ง" (Prompt)** ที่คอยสะกดจิต AI (ประกอบด้วย บทบาท role / เป้าหมาย goal / และภูมิหลัง backstory ที่เราป้อนตั้งค่าให้ตัว CrewAI Agent) 
> รวมไปถึงแจกแจง **"ความสามารถ"** ว่าแต่ละตัวมีอาวุธ tools อะไรให้เรียกใช้บ้าง, เสพพลังจากโมเดล LLM ตัวไหน, ค่าความเพ้อเจ้อ (temperature) เท่าไหร่, และจำกัดรอบความขยันในการคิดลึก (max_iter) แค่ไหน
> (เนื้อหาทั้งหมดนี้ถูกถอดวิญญาณมาจากโค้ดต้นฉบับจริงในไฟล์ `src/agents/*.py` แบบเป๊ะๆ)

## โครงสร้างกายวิภาคของ "ตัวเอเจนต์" ในระบบ MUSYA

เอเจนต์ส่วนใหญ่ในระบบนี้ถูกปั้นขึ้นมาจากเบ้าหลอมของ **เฟรมเวิร์ก CrewAI** (โดยอาศัยคลาส `Agent(role, goal, backstory, tools, llm, max_iter)`)
เมื่อปั้นเสร็จพวกมันจะถูกจับมัดรวมกันทำงานต่อตูดเป็นทีมที่เรียกว่า **Crew** และส่งต่องานกันผ่าน **ใบงาน (Task)** — 
แต่อย่างไรก็ตาม จะมีเอเจนต์บางตัวที่รับจ๊อบเดี่ยวๆ ด้วยการ **ต่อสายตรงยิงคำถามหา Gemini เลย (ผ่านปลั๊กอิน litellm)** โดยไม่พึ่งพากระบวนการของ CrewAI เพื่อความไวแสง

| ชิ้นส่วนองค์ประกอบ | ความหมายที่ซ่อนอยู่ (ในฐานะ "พรอมต์สั่งการ") |
|---|---|
| `role` (บทบาท) | กำหนดหน้าที่/ตัวตน/หรือหมวกที่เอเจนต์ใบนั้นสวมใส่ |
| `goal` (เป้าหมาย) | จุดประสงค์หลักปลายทางที่มันจะต้องดันให้สำเร็จให้จงได้ |
| `backstory` (ภูมิหลัง) | การสร้างอินเนอร์ ปูประวัติความเชี่ยวชาญ + รวมถึงฝังชิป "กติกาการตอบคำถาม" (ส่วนนี้มักจะยาวสุด = ถือเป็นแก่นวิญญาณพรอมต์ที่คุมนิสัย AI) |
| `Task.description` (เนื้อหางาน) | คำสั่งมอบหมายงานแบบเจาะจงเฉพาะครั้ง (เป็นจุดที่เราแนบ ข้อมูลดิบ/ประวัติแชทเก่า/คำถามใหม่ ลงไปให้มันอ่าน) |
| `tools` (อาวุธความสามารถ) | **ความสามารถพิเศษ** — เป็นฟังก์ชันโค้ด (ติดป้าย `@tool`) ที่เอเจนต์ได้รับอนุญาตให้เสกเรียกใช้งานได้ตอนกำลังคิด |
| `llm` / `temp` / `max_iter` | ระดับสมองโมเดลที่ใช้ / ค่าความสุ่มเสี่ยงความครีเอทีฟ / และเพดานขีดจำกัดว่าจะยอมให้มันคิดวนซ้ำแก้ตัวได้กี่รอบ |

## สายพันธุ์สมองกล LLM ที่ประจำการ (แกะจาก `config.py` + ชุดโรงงาน factory ในโค้ด)

> ⚠️ **สำคัญ (ตรวจสอบกับโค้ดจริงแล้ว):** temperature **ไม่ได้เท่ากันทุกตัว** — pipeline ส่วนใหญ่
> ใช้ factory ร่วม `csv_pipeline._get_llm()` (และ factory ในไฟล์ router/tavily/thaijo/error_monitor)
> ที่เป็น `flash-lite` และ **ไม่ตั้ง temperature** (ใช้ค่า default ของ litellm) — ค่า temp ที่ระบุ
> ด้านล่างเป็นเฉพาะไฟล์ที่ตั้งเอง

| ระดับ | โมเดล | ใช้ที่ไหน (ไฟล์) | temp จริง |
|---|---|---|---|
| **fast (ค่า default)** | `gemini-2.5-flash-lite` | `csv_pipeline`, `multi_csv`, `compare`, `database`(crew), `report`, `workplan`(crew), `router`, `tavily`, `thaijo`(crew), `error_monitor` | *ไม่ตั้ง (default)* |
| fast + temp | `gemini-2.5-flash-lite` | Memory (`question_resolver`) 0.1 · PubMed keyword 0.0 · database doc-analyst 0.3 | ตามระบุ |
| accident (chat) | fast / pro (`GEMINI_MODEL_PRO`) | `accident_chat_orchestrator` | fast **0.1** / pro **0.2** |
| accident (policy) | fast / pro | `accident_policy_orchestrator` | fast **0.2** / pro **0.3** |
| obsidian | fast / pro | `obsidian_agent` | fast 0.1 / pro 0.2 |
| **pro** | `gemini-2.5-pro` | `obsidian_fullcontext` **0.2** · workplan Plan Writer **0.3** · ThaiJo Report Generator **0.3** | ตามระบุ |
| **OpenAI** ⚠️ | `gpt-4.1` / `gpt-4.1-mini` | **ThaiJo**: สรุป PDF (gpt-4.1, 0.3) · แปล abstract (gpt-4.1-mini, 0.2) | ตามระบุ |

> [!important] GEMINI_MODEL_PRO ปัจจุบัน
> `config.py` default `GEMINI_MODEL_PRO=gemini-2.5-pro` แต่ **`.env` ที่ใช้รันตอนนี้ตั้งเป็น
> `gemini-2.5-flash-lite`** → เอเจนต์สาย "pro" จึงรันด้วย flash-lite จริง ๆ (ยกเว้นตัวที่ hardcode `gemini-2.5-pro` เช่น workplan Plan Writer / ThaiJo Report Generator)
> **ThaiJo ต้องมี `OPENAI_API_KEY`** เพราะสรุป PDF/แปล abstract ใช้ OpenAI (ไม่ใช่ Gemini)

> [!note] ระบบป้องกันอาการช็อก (ความทนทาน)
> เอเจนต์ทุกตัวในระบบนี้จะถูกผูกยันต์กันตายที่ชื่อ `agent_retry_kwargs()` + วิชา monkey-patch แอบฝังโค้ด **แก้ปัญหาคอขวด 429 backoff** เอาไว้ (มาจากไฟล์ `agent_defaults.py`)
> (ไปอ่านเพิ่มเติมที่ [[01 - Backend Design]]) — สรุปคือถ้า AI โดน API ฝั่ง Google ปฏิเสธว่า "ส่งถี่เกินไป (rate limit)" มันจะไม่ปล่อยให้แอปพัง (crash) แต่จะถ่วงเวลาพักหายใจแป๊บนึงแล้วค่อยหน้าด้านส่งไปขอร้องใหม่เอง

## สารบัญเปิดตัวตี้เอเจนต์ แบ่งตามสายเครื่องมือ

| ลิงก์ไปอ่านรายละเอียด | หน้าที่ประจำปุ่มเครื่องมือ | แกนนำเอเจนต์หลัก |
|---|---|---|
| [[01 - Shared Agents (Memory & Router)]] | 🧭 กองกลางใช้ร่วมทุกโหมด | หมอความจำ Memory Agent, ผู้คุมคิวสับราง Router/Classifier |
| [[02 - Stats Tool Agents]] | 📊 ปุ่มสถิติ (Stats) | ตี้สายอุบัติเหตุ (2 ตัว) · ตี้สายวิเคราะห์ CSV แผ่นเดี่ยว (6 ตัว) · ตี้สายยำรวม Multi-CSV ข้ามแผ่น (3 ตัว) |
| [[03 - Knowledge Tool Agent]] | 🌿 ปุ่มคลังความรู้ (Vault) | ขุนคลัง Obsidian Full-Context (ยิงตรงผ่าน Gemini) |
| [[04 - Research Tool Agents]] | 📖 ปุ่มงานวิจัย (Research) | ตี้สาย ThaiJo (Planner/Insight/Summary/Generator) · ตี้สายหมอ PubMed (Keyword/Fetcher) |
| [[05 - Web Search Tool Agents]] | 🔍 ปุ่มค้นหาเน็ตทั่วไป (Tavily) | นักส่องเน็ต Web Search Specialist · นักเรียบเรียง Research Answer Writer |
| [[06 - Report & Policy Agents]] | 📋 ปุ่มปั้นรายงาน / เขียนนโยบาย | ตี้นักเขียนสารพัดนึก Report/Workplan/Compare/Database · ตี้นโยบายเขตสุขภาพ 10 (3 ตัว) |

*หากอยากรู้ว่าปุ่มเมนูหน้าจอ หรือโหมดส่งไปไหนต่อ ให้ไปกางดูที่ [[02 - Frontend Design]] · แต่ถ้าอยากดมโค้ดดิบแคตตาล็อกภาษาอังกฤษ แวะไปดูที่แฟ้ม `chatapi.python/AGENTS.md` ในโปรเจกต์ได้เลย*
