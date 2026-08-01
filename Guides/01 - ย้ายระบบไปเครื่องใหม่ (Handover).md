---
title: "Guide — ย้ายระบบไปตั้งที่เครื่องใหม่ให้เหมือนเดิมทุกอย่าง"
tags: [MUSYA, guide, setup, handover, backup, migration]
created: 2026-08-01
---

# 📦 ย้ายระบบไปเครื่องใหม่ (Handover)

← ติดตั้งครั้งแรก [[00 - Setup & Onboarding Guide]] · โครงสร้างข้อมูล [[04 - Data Architecture & Schema]]

> คู่มือ 00 ตอบว่า *"ติดตั้งระบบเปล่า ๆ ยังไง"* — เล่มนี้ตอบว่า **"ย้ายของทั้งหมด
> ไปเครื่องใหม่ให้ได้สภาพเดียวกันเป๊ะยังไง"** รวมข้อมูลที่ไม่ได้อยู่ใน git
>
> **ทุกคำสั่งในเล่มนี้รันจริงบนเครื่องต้นทางแล้วเมื่อ 2026-08-01** ไม่ใช่คำสั่งที่คัดมาลอย ๆ

---

## 0. โครงสร้าง git — **3 repo แยกกัน อยู่คนละบัญชี**

สำรวจจริง 2026-08-01 — ไม่มี repo แม่ครอบทั้งหมด แต่ละส่วนเป็น repo ของตัวเอง:

| ส่วน | GitHub | branch |
|---|---|---|
| Backend | `saffyzaza/chatapi.python` | `main` |
| Frontend | `saffyzaza/chatappandpython` | **`mid`** ⚠️ ไม่ใช่ main |
| เอกสาร (vault นี้) | `babankbro/musya-srs-vault` | — |

> [!warning] เอกสารอยู่คนละบัญชีกับโค้ด
> vault อยู่ใต้ `babankbro` ส่วนโค้ดอยู่ใต้ `saffyzaza` — คนที่ได้สิทธิ์บัญชีเดียว
> จะได้ของไม่ครบ **ต้องให้สิทธิ์ทั้งสองบัญชี** หรือย้ายมารวมบัญชีเดียวก่อนส่งมอบ

> [!danger] ก่อนย้ายเครื่อง ต้อง commit ให้หมดก่อน
> ณ วันเขียนมีงานค้างไม่ได้ commit: **backend 67 ไฟล์ · frontend 25 ไฟล์ · vault 24 ไฟล์**
> ถ้าย้ายเครื่องด้วยการ `git clone` อย่างเดียว **งานทั้งหมดนี้จะหายทันที**
>
> ```bash
> for d in chatapi.python chatappandpython musya-srs-vault; do
>   echo "── $d"; git -C $d status --short | head -20
> done
> ```

### ตั้ง git ให้เป็นของตัวเอง

ถ้าจะย้ายไปบัญชี/องค์กรของตัวเอง — ทำทีละ repo:

```bash
cd chatapi.python
git remote rename origin upstream          # เก็บของเดิมไว้อ้างอิง
git remote add origin https://github.com/<บัญชีคุณ>/chatapi.python.git
git push -u origin main
```

ทำแบบเดียวกันกับอีก 2 repo (frontend ใช้ branch `mid`)

> [!tip] ถ้าอยากรวมเป็น repo เดียว (monorepo)
> ทำได้แต่ต้องยอมเสียประวัติ commit ของแต่ละส่วน หรือใช้ `git subtree add` เพื่อรักษาไว้
> · **ข้อดี** clone ครั้งเดียวจบ เวอร์ชันโค้ดกับเอกสารตรงกันเสมอ
> · **ข้อเสีย** เสียเวลาจัดตอนนี้ และ CI/deploy ที่ผูกกับ repo เดิมต้องแก้ตาม
> — ถ้ายังไม่มีปัญหาอะไร **แยก 3 repo ต่อไปก็ทำงานได้ปกติ**

### สิ่งที่ **ห้าม** ขึ้น git (ตรวจแล้วว่ากันไว้ถูกต้องทั้งหมด ✅)

| ไฟล์ | เหตุผล | สถานะ |
|---|---|---|
| `chatapi.python/.env` | มี API key จริง | ✅ ถูก ignore |
| `chatappandpython/.env.local` | มี JWT secret + key | ✅ ถูก ignore |
| `*.dump` · `*.tar.gz` | ไฟล์ 21–61 MB ที่นอนอยู่ใน working tree | ✅ ถูก ignore |
| `data-minio/` | 2.5 GB — อยู่นอก repo อยู่แล้ว | ✅ |

**ตรวจซ้ำก่อน push ทุกครั้ง** — ครั้งเดียวที่หลุดคือหลุดถาวร (ต่อให้ลบ commit ทีหลัง
key ก็ถูกอ่านไปแล้ว ต้องออกใหม่สถานเดียว):

```bash
git -C chatapi.python check-ignore -v .env
git -C chatappandpython check-ignore -v .env.local
```

ไม่มี output = **ไม่ถูก ignore** = อันตราย

---

## 0.1 เข้าใจก่อนว่า "ของ" อยู่ที่ไหนบ้าง

โค้ดอยู่ใน git แต่ **ข้อมูลจริงไม่ได้อยู่ในนั้นเลย** — ย้ายแค่โค้ดจะได้ระบบเปล่า

| ของ | อยู่ที่ | ขนาดจริง | อยู่ใน git? |
|---|---|---|---|
| โค้ด backend/frontend | `chatapi.python/` · `chatappandpython/` | — | ✅ |
| **ฐานข้อมูล** | Docker named volume `postgres_data` | **148 MB** | ❌ ต้อง dump |
| **ไฟล์ CSV + PDF** | `data-minio/` (bind mount) | **2.5 GB** | ❌ คัดลอกโฟลเดอร์ |
| **โน้ตคลังความรู้ `.md`** | `chatapi.python/src/obsidian_knowledge/` | 4.1 MB | ⚠️ ดูข้อ 3.3 |
| **กุญแจและรหัสผ่าน** | `.env` · `.env.local` | — | ❌ **ห้ามขึ้น git** |
| เอกสารระบบ (vault นี้) | `musya-srs-vault/` | 1.4 MB | ✅ |

**ปริมาณ ณ วันเขียน:** 1,545 โน้ต · 165 CSV · 162 PDF · 135 รายการที่ดึงจาก HDC

> [!danger] MinIO เป็น bind mount ไม่ใช่ named volume — ต่างจาก Postgres
> `docker-compose.yml` ผูก `../data-minio/data:/data` ⇒ ไฟล์อยู่บนดิสก์โฮสต์ตรง ๆ
> **คัดลอกโฟลเดอร์ได้เลย ไม่ต้อง export** และ**ลบ container ไม่ทำให้ไฟล์หาย**
>
> ส่วน Postgres ใช้ named volume ที่อยู่ใน VHDX ของ Docker — **ลบ volume แล้วหายจริง**
> เคยเจอมาแล้วตอน VHDX เสีย ต้องกู้จาก dump ([[09 - HDC Sync — สถานะงาน]])

---

## 1. บนเครื่องเดิม — เก็บของ

### 1.1 Dump ฐานข้อมูล

```bash
docker exec chatapp-postgres pg_dump -U postgres -d musyadata -Fc -f /tmp/musyadata.dump
docker cp chatapp-postgres:/tmp/musyadata.dump ./_handover/musyadata.dump
```

**ตรวจว่าใช้ได้จริงก่อนเชื่อ** — ดูขนาดไฟล์อย่างเดียวไม่พอ:

```bash
docker exec chatapp-postgres pg_restore -l /tmp/musyadata.dump | grep -c "TABLE DATA"
```

ต้องได้ **34 ตาราง** (ค่า ณ วันเขียน) · ไฟล์ประมาณ **28 MB**

> ใช้ `-Fc` (custom format) ไม่ใช่ SQL ล้วน เพราะ restore เลือกได้ว่าจะเอาตารางไหน
> และบีบอัดให้ในตัว — 148 MB เหลือ 28 MB

### 1.2 คัดลอกไฟล์ MinIO

```bash
# หยุด stack ก่อน กันไฟล์ถูกเขียนระหว่างคัดลอก
docker compose -f chatappandpython/docker-compose.yml stop minio
# แล้วคัดลอกทั้งโฟลเดอร์ (2.5 GB)
robocopy data-minio  \\เครื่องใหม่\musya\data-minio  /E /Z /R:3
```

โครงสร้างข้างในต้องเป็น:
```
data-minio/data/fileapa/       165 ไฟล์ CSV + __meta__/ + __pathdata__/
data-minio/data/pdf-library/   162 ไฟล์ PDF
```

### 1.3 คัดลอกไฟล์ตั้งค่า

```
chatapi.python/.env
chatappandpython/.env.local
```

> [!warning] สองไฟล์นี้ไม่อยู่ใน git โดยตั้งใจ — มี API key จริงอยู่ข้างใน
> ส่งผ่านช่องทางที่ปลอดภัย **ห้ามแนบไปกับ zip ที่ส่งต่อกันทั่วไป**

### 1.4 คัดลอกโน้ตคลังความรู้

```bash
robocopy chatapi.python\src\obsidian_knowledge  \\เครื่องใหม่\...\obsidian_knowledge  /E
```

---

## 2. บนเครื่องใหม่ — เตรียมพื้น

### 2.1 สิ่งที่ต้องมี

| | เวอร์ชันที่ใช้จริง |
|---|---|
| Docker Desktop | 4.26.0 (WSL2 backend) |
| พื้นที่ว่าง | **≥ 60 GB** — image ~15 GB + ข้อมูล 2.5 GB + ที่ให้ build cache โต |

> [!danger] ตั้งที่เก็บ Docker ไว้ไดรฟ์ที่มีที่ว่างเยอะ **ก่อน**เริ่ม
> ค่าเริ่มต้นของ Docker ลง `C:` แล้ว build cache โตจนเต็ม — เคยทำให้ระบบล่ม 3 ครั้ง
> ในวันเดียว จน VHDX เสียและต้องกู้ข้อมูลใหม่ทั้งหมด
>
> **Settings → Resources → Advanced → Disk image location** ชี้ไปไดรฟ์ที่ว่างเยอะ
> (หรือคีย์ `customWslDistroDir` ใน `%APPDATA%\Docker\settings.json`)
>
> ⚠️ ถ้าแก้ไฟล์ settings เอง **ต้องเขียนแบบไม่มี BOM** — PowerShell `Set-Content -Encoding utf8`
> ใส่ BOM ให้ แล้ว Docker parse JSON ไม่ผ่านและปิดตัวเองเงียบ ๆ ทุกครั้งที่เปิด
> ใช้ `[System.IO.File]::WriteAllText($p, $json, (New-Object System.Text.UTF8Encoding($false)))`

### 2.2 clone จาก git แล้ววางของที่ไม่ได้อยู่ใน git

```bash
mkdir -p musya/new_version_musya && cd musya/new_version_musya

git clone https://github.com/<บัญชีคุณ>/chatapi.python.git
git clone -b mid https://github.com/<บัญชีคุณ>/chatappandpython.git   # ⚠️ branch mid
git clone https://github.com/<บัญชีคุณ>/musya-srs-vault.git
```

แล้ววางของที่ git ไม่ได้เก็บลงไป:

```
musya/new_version_musya/
├── chatapi.python/          ← git clone
│   ├── .env                 ← คัดลอกมา (ข้อ 1.3) — ไม่อยู่ใน git
│   └── src/obsidian_knowledge/   ← คัดลอกมา (ข้อ 1.4)
├── chatappandpython/        ← git clone (branch mid)
│   └── .env.local           ← คัดลอกมา (ข้อ 1.3) — ไม่อยู่ใน git
├── data-minio/              ← คัดลอกมา (ข้อ 1.2) — 2.5 GB ไม่อยู่ใน git
├── musya-srs-vault/         ← git clone
└── _handover/musyadata.dump ← คัดลอกมา (ข้อ 1.1) — 28 MB ไม่อยู่ใน git
```

> [!important] `data-minio` ต้องอยู่ **ระดับเดียวกับ** `chatappandpython`
> compose ผูกด้วย path สัมพัทธ์ `../data-minio/data` — วางผิดชั้นแล้ว MinIO
> จะสร้างโฟลเดอร์ว่างใหม่ให้ ระบบขึ้นได้ปกติแต่**ไม่มีไฟล์สักไฟล์**

---

## 3. ปลุกระบบ

### 3.1 ขึ้น stack

```bash
cd chatappandpython
docker compose up -d --build
```

ครั้งแรกใช้เวลาราว 10–20 นาที (build image ทั้ง 2 ตัว) · รอจน 5 container ขึ้นครบ:

```bash
docker compose ps
```

### 3.2 คืนฐานข้อมูล

```bash
docker cp _handover/musyadata.dump chatapp-postgres:/tmp/restore.dump
docker exec chatapp-postgres pg_restore -U postgres -d musyadata \
    --clean --if-exists --no-owner /tmp/restore.dump
```

> [!warning] ผู้ใช้ Git Bash บน Windows ต้องนำหน้าด้วย `MSYS_NO_PATHCONV=1`
> ไม่งั้น Git Bash แปลง `/tmp/restore.dump` เป็น path แบบ Windows แล้ว
> pg_restore หาไฟล์ไม่เจอ (`C:/Program Files/Git/tmp/...`)

**ตรวจทันทีว่าได้ครบ:**

```bash
docker exec chatapp-postgres psql -U postgres -d musyadata -Atc "
  SELECT 'notes='||count(*) FROM obsidian_notes;
  SELECT 'csv_dict='||count(*) FROM csv_data_dict;
  SELECT 'hdc='||count(*) FROM hdc_import;"
```

ต้องได้ `notes=1545` · `csv_dict=180` · `hdc=135` (ตัวเลข ณ วันเขียน)

### 3.3 ไม่ต้องรัน migration ถ้า restore จาก dump

dump มีโครงสร้างครบอยู่แล้ว — migration `025`–`035` ใช้เฉพาะตอน**ตั้งระบบใหม่จากศูนย์**
หรือเมื่อมีไฟล์ migration ใหม่ที่ dump ยังไม่มี

ตรวจว่าตารางรุ่นใหม่มาครบไหม:

```bash
docker exec chatapp-postgres psql -U postgres -d musyadata -Atc "
  SELECT table_name FROM information_schema.tables
  WHERE table_name IN ('llm_settings','csv_data_dict','hdc_import') ORDER BY 1;"
```

ขาดตัวไหนค่อยรันเฉพาะไฟล์นั้น:

```bash
docker cp chatapi.python/database/03X_xxx.sql chatapp-postgres:/tmp/m.sql
docker exec chatapp-postgres psql -U postgres -d musyadata -f /tmp/m.sql
```

### 3.4 สร้าง path index ของ MinIO ใหม่

ระบบ cache แผนที่ `file_id → path` ไว้ในหน่วยความจำ — เครื่องใหม่ต้องสร้างรอบแรก:

```bash
docker exec chatapp-python-ai python -c "
from src.tools.minio import _load_path_index
print('ไฟล์ที่มองเห็น:', len(_load_path_index(force=True)))"
```

ต้องได้ **165** — ถ้าได้ 0 แปลว่า `data-minio` วางผิดที่ (ดูข้อ 2.2)

---

## 4. ตรวจรับก่อนส่งมอบ

```bash
# 1. บริการขึ้นครบ
curl -s -o /dev/null -w "backend=%{http_code}\n"  http://localhost:8000/docs
curl -s -o /dev/null -w "frontend=%{http_code}\n" http://localhost:3000/

# 2. ชุดทดสอบผ่าน (257 เทสต์ ณ วันเขียน)
docker run --rm -v "<path>/chatapi.python:/w" -w /w \
  chatappandpython-python-ai sh -c "python -m pytest tests/ -q"

# 3. ข้อมูลกับไฟล์ตรงกัน
docker exec chatapp-minio ls /data/fileapa | wc -l
docker exec chatapp-postgres psql -U postgres -d musyadata -Atc \
  "SELECT count(*) FROM csv_data_dict"
```

### เช็กลิสต์

- [ ] 5 container `Up` และ postgres/minio ขึ้น `healthy`
- [ ] `/docs` = 200 · `/` = 307 (redirect ไปหน้า login — ถูกต้องแล้ว)
- [ ] `obsidian_notes` = 1,545
- [ ] path index มองเห็น 165 ไฟล์
- [ ] เทสต์ผ่านครบ
- [ ] เข้าหน้าเว็บแล้วล็อกอินด้วยบัญชีเดิมได้
- [ ] ถามคำถามในแชทแล้วได้คำตอบ (พิสูจน์ว่า API key ใช้ได้จริง)

---

## 5. กับดักที่เคยเจอจริง

> [!danger] ตาราง `users` ไม่ใช่ตารางที่ใช้ล็อกอิน
> ฐานข้อมูลมีทั้ง `accounts` (53 บัญชี ใช้จริง) และ `users` (1 แถว ไม่มีโค้ดไหนเรียก)
> ชื่อชวนสับสนมาก — เคยไปให้สิทธิ์ผิดตารางจนไม่มีใครเข้าหน้าตั้งค่าได้เลย
> **บทบาทผู้ดูแลสูงสุดคือ `adminsuper`** ไม่ใช่ `superadmin`

**API key** — ถ้าคัดลอก `.env` มาแต่ระบบตอบไม่ได้ ให้ตรวจว่า key ยังมีเครดิตไหม
เคยเจอ key ของ OpenAI ที่ `list models` ผ่านแต่ยิง chat จริงได้ `429 insufficient_quota`
มีปุ่มทดสอบให้แล้วในหน้า **ตั้งค่า AI** (สิทธิ์ `adminsuper`)

**ข้อมูลกำพร้า** — ตอนเขียนคู่มือนี้พบว่า `csv_data_dict` มี **180 แถว** แต่ MinIO มี
**165 ไฟล์** ⇒ มี **17 แถวที่ไม่มีไฟล์จริง** (เศษจากไฟล์ที่ถูกลบ/เขียนทับ)
และ **2 ไฟล์ที่ยังไม่มีพจนานุกรม** — ไม่กระทบการทำงาน แต่ถ้าอยากได้เครื่องใหม่สะอาด
ให้รันสร้างพจนานุกรมใหม่หลัง restore:

```bash
docker exec chatapp-python-ai python -m src.scripts.build_data_dict
```

**เวลาที่ต้องเผื่อ** — build 10–20 นาที · คัดลอก 2.5 GB ผ่านเครือข่ายอีกตามความเร็ว
· restore ฐานข้อมูล 1–2 นาที

---

## 6. ถ้าจะย้ายแบบ "เริ่มใหม่หมด" ไม่เอาข้อมูลเดิม

ข้ามข้อ 1 กับ 3.2 แล้วทำแทน:

1. `docker compose up -d --build` (schema สร้างเองรอบแรกจาก `schema.sql`)
2. รัน migration `025` → `035` ตามลำดับเลข
3. ดึงข้อมูลสถิติใหม่จาก HDC ผ่านหน้า `/fileapa` → ปุ่ม **"ดึงจาก HDC"**
   (ดู [[08 - HDC Open Data Sync (เขต 10)]]) — ได้ข้อมูลสดกว่าเดิมด้วยซ้ำ
4. ส่วนคลังความรู้ PDF ต้องอัปโหลดและ ingest ใหม่ ([[08 - PDF Ingest Workflow]])
   — ขั้นนี้กินเวลาหลายชั่วโมงและใช้โควตา LLM จริง **ถ้าไม่จำเป็นให้ restore แทน**
