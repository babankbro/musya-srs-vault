---
title: "Guide — Setup & Onboarding (คู่มือติดตั้งและเริ่มต้นใช้งาน)"
tags: [MUSYA, guide, setup, onboarding, docker]
created: 2026-07-18
---

# 📦 คู่มือติดตั้งและเริ่มต้นใช้งาน (Setup & Onboarding)

← กลับไป [[index|หน้าหลัก]] · ดูโครงสร้างสถาปัตยกรรม: [[00 - Architecture Overview]] · ดูการตั้งค่าโดเมน: [[00 - Glossary & Domain Context]]

> คู่มือนี้จะสอนวิธีชุบชีวิตระบบ MUSYA AI ขึ้นมาทำงานตั้งแต่ศูนย์ (From scratch) — โดยเราจะเน้นท่ามาตรฐานคือการใช้ **Docker Compose** (แนะนำที่สุด)
> ซึ่งเราจะมีโฟลเดอร์รวบยอดชื่อ `chatappandpython` ทำหน้าที่เป็นผู้กำกับวงดนตรี (Orchestrator) สั่งเปิดทั้ง 5 เซิร์ฟเวอร์ย่อยขึ้นมาพร้อมกัน

---

> [!tip] จะ **ย้ายระบบพร้อมข้อมูลเดิม** ไปเครื่องใหม่ใช่ไหม
> เล่มนี้สอนติดตั้ง**ระบบเปล่า** — ถ้าต้องการยกข้อมูลเดิมไปด้วย (ฐานข้อมูล 148 MB · ไฟล์ 2.5 GB · โน้ต 1,545 ใบ)
> ให้ไปที่ [[01 - ย้ายระบบไปเครื่องใหม่ (Handover)]] แทน

## 0. เตรียมเสบียงก่อนออกรบ (Prerequisites)

| สิ่งที่ต้องมี | คำอธิบาย/ข้อควรระวัง |
|---|---|
| **Docker Desktop** (engine ต้องติดเครื่องอยู่) | เพราะเราต้องปลุกปล้ำรันถึง 5 container; ถ้าลืมเปิดโปรแกรม Docker ก่อน พิมพ์คำสั่งไปก็พังแน่นอน |
| **โครงสร้างโฟลเดอร์** | โฟลเดอร์ `chatapi.python/` (โค้ด AI) และโฟลเดอร์ `chatappandpython/` (หน้ากากเว็บ+ตัวรวม) จะต้องถูกวางไว้ **ระดับเดียวกันเสมอ** (เพราะไฟล์ compose มันแอบอ้างอิง path แบบย้อนกลับ `../chatapi.python`) |
| **GEMINI_API_KEY** | ของมันต้องมี! (ขาดสิ่งนี้ AI ก็ใบ้รับประทาน) |
| **TAVILY_API_KEY** / **NCBI_API_KEY** | เป็นออปชันเสริม (Tavily เอาไว้ใช้ตอนกดปุ่มค้นหาเว็บทั่วไป / NCBI เอาไว้เจาะเปเปอร์แพทย์ PubMed) |
| Git | เอาไว้โคลน (Clone) ดูดโค้ดลงเครื่อง |

---

## 1. การฝังขุมทรัพย์ Environment (ตั้งค่า 2 ฝั่ง)

ระบบเราเป็นระบบคู่ขนาน จึงต้องใช้ **2 ไฟล์ `.env`** — แบ่งกันอ่านคนละฝั่ง ห้ามสับสนเด็ดขาด:

### ก. ฝั่งปัญญาประดิษฐ์: `chatapi.python/.env` (ให้ Python-AI อ่าน)
กลุ่มตัวแปรที่ต้องใส่: กุญแจ AI (`GEMINI_API_KEY`, `OPENAI_API_KEY`, `TAVILY_API_KEY`), ชื่อโมเดล (`GEMINI_MODEL=gemini-2.5-flash-lite`, `GEMINI_MODEL_PRO`), ช่องทางต่อ PostgreSQL (`DB_*`), Redis (`REDIS_URL`), MinIO (`MINIO_*`, `MINIO_BUCKET=fileapa`), Obsidian (`OBSIDIAN_VAULT_PATH`, `OBSIDIAN_DEFAULT_VAULT=health_region_10`), CORS, `INTERNAL_API_KEY` และ 🆕 **`PUBLIC_APP_URL`**

> [!tip] 🆕 `PUBLIC_APP_URL` — ตัวแปรใหม่ที่มักถูกลืม (ค่าเริ่มต้น `http://localhost:3000`)
> **ต้องตั้งให้ตรงกับ `NEXT_PUBLIC_APP_URL` ฝั่งหน้าบ้านเสมอ** ใช้แปลง path สัมพัทธ์ (เช่น `/api/pdf/view/123`) ให้เป็น URL เต็มก่อนฝังลงข้อความที่ป้อนให้ LLM อ่าน — เพราะ LLM ไม่ยอมทำ path สัมพัทธ์ให้เป็นลิงก์คลิกได้ในเอกสาร HTML ที่มันเขียนเอง (ต่างจากฝั่ง React ที่ `<a href>` แบบสัมพัทธ์ทำงานได้ปกติ) **ถ้าตั้งผิด ลิงก์เอกสารคลังความรู้ในรายงานที่ generate ออกมาจะพาไปหน้าที่ไม่มีอยู่จริง**
> สามารถไปลอกการบ้านได้จากไฟล์ `.env.example` · หมายเหตุกาดอกจัน: ค่าตัวแปรหลายตัวจะถูก **สวมทับ (override) โดย Docker compose** ตอนสั่งรัน (เช่น มันจะแอบเปลี่ยนให้ไปเรียก `DB_HOST=postgres`, `MINIO_ENDPOINT=minio`, `REDIS_URL=redis://redis:6379/0` เองอัตโนมัติ)

### ข. ฝั่งหน้าบ้าน: `chatappandpython/.env.local` (ให้ Frontend/BFF อ่าน)
คีย์สำคัญระดับชาติ: `PYTHON_API_URL`, `NEXT_PUBLIC_API_URL`, `DATABASE_URL`, `MINIO_*`, `MINIO_BUCKET`, `MINIO_APA_BUCKET`, **`JWT_SECRET`**, **`INTERNAL_API_KEY`**, `NEXT_PUBLIC_APP_URL`

> [!danger] ระเบิดเวลา: 2 ฝั่งต้องรหัสตรงกัน
> ตัวแปร **`INTERNAL_API_KEY`** ใน `.env` (ฝั่ง AI) กับใน `.env.local` (ฝั่งเว็บ) **ต้องพิมพ์ให้เหมือนกันเป๊ะทุกตัวอักษร!**
> ถ้าไม่ตรงกัน ตัว BFF หน้าบ้านจะโดนยามหน้าด่านหลังบ้านถีบกระเด็น (401 Unauthorized) · และตัวแปร **`JWT_SECRET`** ต้องมีความยาวไม่ต่ำกว่า 8 ตัวอักษร (ห้ามมักง่ายใส่ 1234)

---

## 2. ปลุกชีพเซิร์ฟเวอร์ (Build & Run — แนะนำด้วยท่า Docker)

```bash
cd chatappandpython
docker compose up -d --build
```

Docker Compose จะทำหน้าที่เนรมิตและจุดระเบิดสตาร์ท 5 บริการขึ้นมาตามลำดับความพร้อม (health):

| ชื่อบริการ (Container) | พอร์ต (Port) | วิธีเข้าไปเช็คว่ารอดไหม |
|---|---|---|
| chatapp-frontend | 3000 | เข้าเว็บ http://localhost:3000 |
| chatapp-python-ai | 8000 | แวะดูคู่มือ API http://localhost:8000/docs (หน้าเว็บ Swagger) |
| chatapp-postgres | 5432 | รันคำสั่ง healthcheck `pg_isready` |
| chatapp-minio | 9000 / 9001 | เข้าเว็บจัดการไฟล์ http://localhost:9001 (ล็อกอิน `minioadmin` / `minioadmin`) |
| chatapp-redis | 6379 | — (มันทำงานเงียบๆ ของมัน) |

> คาถาฉุกเฉิน: พิมพ์ `docker compose ps` (เช็คว่าใครตายไหม) · พิมพ์ `docker compose logs -f python-ai` (แอบดูจอมอนิเตอร์ของ AI) · พิมพ์ `docker compose down` (ปิดโรงงานเก็บกวาดให้เรียบ)

---

## 3. การปลูกผักลงแปลง (Seeding ฐานข้อมูล — ทำเองอัตโนมัติรอบแรก)

ตอนที่คุณสั่ง `up` ครั้งแรกสุด ฐานข้อมูล PostgreSQL จะลุกขึ้นมารันสคริปต์จุดเริ่มต้น (init) จาก volume เองแบบอัตโนมัติ:

| ลำดับรัน | ชื่อไฟล์สคริปต์ | มันแอบสร้างอะไรบ้าง |
|---|---|---|
| 01 | `chatapi.python/database/schema.sql` | สร้างตารางสถิติอุบัติเหตุ (star schema) + ตารางระบบเว็บ (`accounts`/`chat_sessions`/`journal_reports`) + **แอบสร้างบัญชีทดสอบแจกฟรี** |
| 10 | `chatapi.python/database/accident.sql` | ปั๊มข้อมูลสถิติอุบัติเหตุจำลองใส่ลงไป |
| 25 | `chatappandpython/database/init-obsidian.sql` | สร้างตารางความรู้ `obsidian_*` (เปิดใช้ extension pg_trgm) + ผูก vault ชื่อ `health_region_10` |

> สำหรับสคริปต์ Migration แก้งานเพิ่มเติมงวดหลังๆ (`026` ไปจนถึง `030`) ถ้าอยากได้ระบบฟูลออปชันครบเซ็ต (มีตาราง chunk/pdf_assets/mart) คุณต้องเอื้อมมือไปกดรันด้วยตัวเองผ่านคำสั่ง `python database/migrate.py`

---

## 4. จัดตู้หนังสือ (Index คลังความรู้ Obsidian — ขาดไม่ได้ถ้าจะใช้โหมดคลังความรู้)

ตาราง `obsidian_notes` มันจะโล่งโจ้งกลวงโบ๋ จนกว่าคุณจะสั่งให้มันไปดูดกวาดข้อมูลจากไฟล์ `.md` ของจริงเข้าไปเก็บ:

```bash
# วิธี A — สั่งผ่าน API (รอจนกว่าเว็บและ AI จะติดไฟเขียวทั้งคู่ก่อนนะ)
curl -X POST http://localhost:8000/api/obsidian/index \
  -H "x-internal-key: <INTERNAL_API_KEY ที่คุณตั้งไว้>"

# วิธี B — ทะลวงมิติเข้าไปสั่งรันสคริปต์ในคอนเทนเนอร์ตรงๆ
docker exec chatapp-python-ai python -m src.scripts.index_obsidian
docker exec chatapp-python-ai python -m src.scripts.sync_obsidian_pdfs   # สั่งอัปโหลดไฟล์ PDF ยัดเข้าตู้ MinIO
```

วิธีเช็คผลงาน: ลองยิง `GET /api/obsidian/status` ถ้าสำเร็จ ต้องเห็นตัวเลข `note_count > 0` (มีสมุดโน้ตอยู่ในตู้แล้ว)

---

## 5. ฤกษ์งามยามดี เข้าใช้งานครั้งแรก

- เปิดเบราว์เซอร์ไปที่ http://localhost:3000 → เข้าสู่ระบบ
- ลายแทงบัญชีทดสอบ (ที่แอบฝังไว้แต่แรกใน `schema.sql`): `musya@gmail.com` / `123456musya` (สิทธิ admin),
  `supermusya@gmail.com` / `123456musya` (สิทธิระดับพระเจ้า adminsuper), ส่วนบัญชีลูกเจี๊ยบ `musya01` ถึง `musya50` / รหัสผ่านจะเป็น `1234musya` ตามด้วยเลข `NN`
> [!warning] กฎเหล็กคนทำงาน: ก่อนจะยกขึ้นเซิร์ฟเวอร์เปิดให้ประชาชนใช้จริง (Production) **คุณต้องประหารลบบัญชีพวกนี้ทิ้งให้เกลี้ยง / หรือเปลี่ยนรหัสผ่านใหม่ทั้งหมด** + อย่าลืมสุ่มรหัส `JWT_SECRET` และ `INTERNAL_API_KEY` ใหม่ด้วย (ไปอ่านคำเตือนได้ที่ [[07 - Auth & Session Workflow]], หรือ [[05 - Non-Functional Requirements|NFR-SEC-08]])

---

## 6. สายอินดี้ โหมด Local Dev (ไม่ง้อ Docker — ทางเลือกสำหรับคนชอบประกอบเอง)

```bash
# ฝั่งระบบ AI (Backend)
cd chatapi.python
pip install -r requirements.txt
# (คุณต้องไปแก้ .env ให้กลายเป็น DB_HOST=localhost, MINIO_ENDPOINT=localhost, REDIS_URL=redis://localhost:6379/0 ก่อนรันนะ)
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# ฝั่งหน้าบ้าน (Frontend)
cd chatappandpython
npm install
npm run dev   # เปิดเว็บ http://localhost:3000
```
> คำเตือน: ท่านี้คุณจะต้องมีเซิร์ฟเวอร์ PostgreSQL / MinIO / Redis ติดตั้งและเปิดรันรออยู่ในคอมพิวเตอร์ของคุณเองอยู่ก่อนแล้วนะ (หรือใช้วิธีลักไก่ สั่ง compose ให้เปิดแค่ 3 ตัวนี้ขึ้นมาก็ได้)

> [!warning] เวอร์ชัน Python สำคัญมาก
> `Dockerfile` ตรึงไว้ที่ **python:3.12-slim** — ถ้าเครื่องคุณเป็น Python เวอร์ชันใหม่กว่านั้นมาก (เช่น 3.14) แพ็กเกจอย่าง `crewai` / `litellm` / `psycopg2-binary` จะติดตั้งไม่ผ่าน ให้ใช้ท่า Docker แทน หรือสร้าง venv ด้วย Python 3.12 โดยเฉพาะ

---

## 6.5 🆕 รันชุดทดสอบ (pytest 43 เทสต์)

> [!important] เทสต์ **ไม่ได้อยู่ใน Docker image**
> `Dockerfile` คัดลอกเข้าไปแค่ `main.py` กับ `src/` เท่านั้น ดังนั้น `docker compose exec chatapp-python-ai pytest` จะขึ้นว่า **"no tests ran"** เสมอ — ไม่ใช่ว่าเทสต์พัง แต่มันไม่มีอยู่ในคอนเทนเนอร์ตั้งแต่แรก

วิธีที่ใช้ได้จริงคือ mount โฟลเดอร์รีโพเข้าไปในคอนเทนเนอร์ชั่วคราว (ใช้อิมเมจที่ build ไว้แล้วเพื่อไม่ต้องลง dependency ซ้ำ):

```bash
docker run --rm -v "<path เต็มของ chatapi.python>:/work" -w //work --env-file .env -e DB_HOST=postgres --network chatappandpython_default chatappandpython-python-ai python -m pytest -q
```

ผลที่ควรได้: `43 passed`

> เกร็ด: `-w //work` ที่มี slash 2 ตัวเป็นท่าเฉพาะของ Git Bash บน Windows (กันไม่ให้ MSYS แปลง path ให้อัตโนมัติ) ถ้าใช้ PowerShell หรือ Linux ใส่ `-w /work` ตามปกติได้เลย

---

## 7. คลินิกแก้กรรม (Troubleshooting ปัญหาที่พบบ่อย)

| อาการผีหลอก | สาเหตุเบื้องลึก / วิธีปัดเป่า |
|---|---|
| `docker` พ่นด่ากลับมาว่า Internal Server Error | คุณลืมเปิดโปรแกรม Docker Desktop หรือ engine มันยังวอร์มเครื่องไม่เสร็จ — เปิดแล้วนั่งรอให้มันขึ้นไฟเขียว ready ก่อน |
| พิมพ์แชทไปแล้วมันเถียงกลับว่า "ระบบยุ่ง" | คิวชนกำแพง semaphore ≤5/worker — ไม่ต้องตกใจ รอแป๊บนึงให้คนอื่นคุยเสร็จแล้วค่อยพิมพ์ใหม่ ([[05 - Non-Functional Requirements|NFR-PERF-02]]) |
| กดปุ่มคลังความรู้แล้วมันหาอะไรไม่เจอเลย | คุณลืมทำขั้นตอนที่ 4 (จัดตู้หนังสือ) ไปรันคำสั่ง index ซะ |
| สั่งค้นหา ThaiJo แล้วมันล่ม (403/timeout) | นี่คือชะตากรรมของโลกภายนอก (เว็บเขาอาจจะล่ม หรือแบนเราชั่วคราว) — ลองกดใหม่ / หรือหวังพึ่งแหล่งที่ 2 เอา |
| กดปุ่มค้นหาทั่วไป (ลูกโลก) แล้วระบบนิ่ง | คุณลืมใส่คีย์ `TAVILY_API_KEY` ในไฟล์ `.env` |
| LLM โวยวายพ่น error 429 ออกมา | ยิงถล่มจนโควต้าทะลุ (rate limit) — แต่ไม่ต้องกังวล ระบบเรามีกลไก backoff ถ่วงเวลาให้แล้ว แค่นั่งรอเดี๋ยวมันก็รันผ่านเอง ([[05 - Non-Functional Requirements|NFR-REL-01]]) |
| ตัว BFF หน้าบ้าน ทะลวงคุยกับ AI หลังบ้านไม่ผ่าน (ติด 401/403) | กุญแจ `INTERNAL_API_KEY` 2 ฝั่งดันพิมพ์ไม่เหมือนกัน ไปแก้ซะ |
| พิมพ์ถามข้อมูลของจังหวัดนอกเขต 10 → แล้ว AI บอกไม่มีข้อมูล | ปกติครับ นี่คือผลงานของยามหน้าประตู (out-of-zone guard) ที่ตั้งใจเขียนไว้กีดกัน ([[03 - Functional Requirements|FR-CHAT-08]]) |
| 🆕 สั่ง `pytest` ในคอนเทนเนอร์แล้วขึ้น "no tests ran" | ไม่ใช่เทสต์พัง — `Dockerfile` ไม่ได้ COPY โฟลเดอร์ `tests/` เข้าไปใน image ต้อง mount รีโพเข้าไปตอนรันแทน (ดูส่วนที่ 6.5) |
| 🆕 ลิงก์เอกสารคลังความรู้ในรายงานที่ generate ออกมา กดแล้วพาไปหน้าที่ไม่มีจริง | ตั้ง `PUBLIC_APP_URL` (ฝั่ง AI) ไม่ตรงกับ `NEXT_PUBLIC_APP_URL` (ฝั่งเว็บ) — ไปแก้ให้ตรงกันซะ (ดูส่วนที่ 1.ก) |
| 🆕 กดปุ่มคลังความรู้แล้วมีข้อความแปลก ๆ อย่าง `## FILE: ...` หรือ `[[...]]` โผล่ในคำตอบ | ตามสเปคต้องไม่เกิดขึ้นเลย (มีด่านกัน 3 ชั้น) — ถ้าเจอจริงคือ **บั๊ก** ให้ดู log ที่ขึ้นว่า `[fullctx] เนื้อหาดิบยังหลุดหลัง retry` แล้วแจ้งทีมพัฒนา ([[02 - Prompt Strategy & Anti-Hallucination]] ส่วนที่ 3.1) |
| 🆕 ระดมข้อมูลทำรายงานแล้วมีแหล่งนึงพัง | ไม่ต้องเริ่มใหม่ทั้งชุด — กดปุ่ม **"ลองใหม่"** บน badge สีแดงของแหล่งนั้นได้เลย ระบบจะรันเฉพาะแหล่งเดียวแล้วต่อท้ายเนื้อหาเดิมให้ |

---

## 8. เช็กลิสต์ก่อนออกบิน (Quick checklist)
- [ ] สตาร์ท Docker engine ติดไฟเขียวแล้ว
- [ ] สร้างไฟล์ `.env` (ฝั่ง AI) + `.env.local` (ฝั่งเว็บ) ครบถ้วน, ตรวจสอบ `INTERNAL_API_KEY` ว่าตรงกันแบบฝาแฝด
- [ ] 🆕 ตรวจ `PUBLIC_APP_URL` (ฝั่ง AI) ว่าตรงกับ `NEXT_PUBLIC_APP_URL` (ฝั่งเว็บ)
- [ ] 🆕 แตกไฟล์ `chatapi.python/src/obsidian_knowledge.7z` ลงเป็นโฟลเดอร์ `src/obsidian_knowledge/` แล้ว (compose bind-mount path นี้ — ถ้าไม่แตกไว้ คอนเทนเนอร์จะเห็นคลังความรู้ว่างเปล่า)
- [ ] 🆕 รัน `pytest` ผ่านครบ 43 เทสต์ (ดูส่วนที่ 6.5)
- [ ] สั่ง `docker compose up -d --build` รันผ่านฉลุย → ทั้ง 5 container ยืนยิ้ม healthy
- [ ] ลองเปิด http://localhost:8000/docs แล้วเห็นหน้า Swagger (สถานะ 200) · ลองเปิด http://localhost:3000 หน้าเว็บขึ้น (สถานะ 200)
- [ ] รันคำสั่ง index ตู้หนังสือ Obsidian vault เรียบร้อยแล้ว (เช็คแล้ว `note_count > 0`)
- [ ] ล็อกอินเข้าแอปได้ · ลองถามคำถามสุขภาพในโซนเขต 10 แล้วได้คำตอบยาวๆ กลับมา

*✅ การันตีจากประสบการณ์รันจริง: เมื่อพิมพ์ `docker compose up -d --build` → สมาชิกทั้ง 5 บริการลุกขึ้นมายืน healthy ได้อย่างสวยงาม, แถมเทสต์หน้าเอกสาร API /docs ก็โชว์ 200, ฝั่งเว็บ frontend ก็ 200 ผ่านฉลุย!*
