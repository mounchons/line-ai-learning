# RAG Gemini + PGVector — คู่มือการตั้งค่า

Workflow ถามตอบจากเอกสารของเราเอง (RAG — Retrieval-Augmented Generation):
อัปโหลดเอกสารผ่าน Form → ตัดเป็นส่วนย่อย → สร้าง embedding ด้วย **Google Gemini** → เก็บใน **PostgreSQL + pgvector**
แล้วให้ AI Agent (Gemini) ค้นเอกสารที่เกี่ยวข้องก่อนตอบทุกครั้ง

ไฟล์ workflow: [`rag-gemini-pgvector.json`](rag-gemini-pgvector.json)

---

## 1. ภาพรวมการทำงาน

```
[นำเข้าเอกสาร]
Upload Document (Form) → Split Files → Insert into PGVector ──► ตาราง rag_documents
                                         ├─ Embeddings Gemini (insert)
                                         └─ Default Data Loader
                                              └─ Recursive Character Text Splitter

[ถามตอบ]
When chat message received → RAG Agent
                               ├─ Google Gemini Chat Model
                               ├─ Postgres Chat Memory ──► ตาราง rag_chat_histories
                               └─ Tool: Knowledge Base (PGVector, retrieve-as-tool) ◄── ตาราง rag_documents
                                          └─ Embeddings Gemini (query)
```

| Node | หน้าที่ |
|---|---|
| **Upload Document** | Form อัปโหลดไฟล์ (หลายไฟล์ได้) `.pdf .docx .txt .md .csv` ที่ `/form/rag-upload` |
| **Split Files** | แยกไฟล์ที่อัปโหลดมาเป็น 1 item ต่อ 1 ไฟล์ และย้าย binary ไปไว้ที่ key `data` เพื่อให้ทุก chunk รู้ว่ามาจากไฟล์ไหน |
| **Default Data Loader** | อ่านไฟล์ตามชนิด (loader `auto`) และติด metadata `source` (ชื่อไฟล์), `mimeType`, `uploadedAt` |
| **Recursive Character Text Splitter** | ตัดข้อความเป็น chunk ละ 1000 ตัวอักษร ซ้อนกัน 200 ตัวอักษร |
| **Embeddings Gemini (insert / query)** | `models/gemini-embedding-001` (3072 มิติ) — **สองตัวต้องใช้ model เดียวกันเสมอ** |
| **Insert into PGVector** | เขียน chunk + vector ลงตาราง `rag_documents` (สร้าง extension `vector` และตารางให้เองในรอบแรก) |
| **RAG Agent** | system prompt บังคับให้ค้น `Knowledge_Base` ก่อนตอบ ตอบจากเอกสารเท่านั้น ไม่พบให้ตอบ "ไม่พบข้อมูลนี้ในเอกสาร" และระบุ `ที่มา:` เป็นชื่อไฟล์ |
| **Knowledge Base** | ค้น 5 chunk ที่ใกล้เคียงที่สุด (cosine distance) จาก `rag_documents` |
| **Postgres Chat Memory** | เก็บประวัติ 10 รอบล่าสุดต่อ `sessionId` ในตาราง `rag_chat_histories` (อยู่ถาวร ไม่หายเมื่อ restart) |

ตาราง `rag_documents` (LangChain PGVectorStore สร้างให้):

| คอลัมน์ | ชนิด | เนื้อหา |
|---|---|---|
| `id` | uuid | primary key |
| `text` | text | เนื้อหา chunk |
| `metadata` | jsonb | `{"source": "...", "mimeType": "...", "uploadedAt": "...", ...}` |
| `embedding` | vector | vector 3072 มิติจาก Gemini |

---

## 2. สิ่งที่ต้องเตรียม

### 2.1 PostgreSQL ต้องมี pgvector

`docker-compose.yml` เปลี่ยน image ของ service `postgres` จาก `postgres:18` เป็น **`pgvector/pgvector:pg18-trixie`**
(Postgres 18 ตัวเดียวกัน + extension `vector`) ข้อมูลเดิมใน volume `db_storage` ใช้ต่อได้เลย

```bash
docker compose up -d postgres
docker exec n8n-postgres psql -U n8n -d n8n -c "select name, default_version from pg_available_extensions where name='vector';"
```

ต้องเห็นแถว `vector | 0.8.x`

> ใช้ tag **`pg18-trixie`** ไม่ใช่ `pg18` — tag `pg18` เป็น Debian bookworm (glibc 2.36) ซึ่งไม่ตรงกับ `postgres:18` เดิม (trixie, glibc 2.41)
> Postgres จะขึ้นเตือน `collation version mismatch` และ index ที่เรียงข้อความอาจเพี้ยน

### 2.2 Credentials ใน n8n

| Credential | ใช้ที่ | ค่าที่ต้องตั้ง |
|---|---|---|
| **Postgres account** (`postgres`) | Insert into PGVector, Knowledge Base, Postgres Chat Memory | Host **`postgres`**, Port **`5432`**, Database/User/Password ตาม `.env` (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`), SSL: disable |
| **Google Gemini(PaLM) Api account** (`googlePalmApi`) | Embeddings ทั้งสองตัว, Chat Model | API key จาก https://aistudio.google.com/apikey |

> **Host ห้ามเป็น `localhost`** — n8n รันใน container, `localhost` ในนั้นคือตัว n8n เอง จะได้ error
> `connect ECONNREFUSED ::1:5432` ต้องใช้ชื่อ service `postgres` (และพอร์ต 5432 ไม่ใช่ 5689 ที่เป็นพอร์ตฝั่ง host)
>
> บน n8n ของโปรเจกต์นี้ (ตรวจเมื่อ 2026-09-23) credential **Postgres account** ยังชี้ไปที่ `localhost` อยู่
> ทดสอบแล้วได้ error ข้างบนที่ node Insert into PGVector — **ต้องแก้ Host เป็น `postgres` ก่อนใช้งานครั้งแรก**

ถ้า import ไปยัง n8n เครื่องอื่นที่ id ของ credential ไม่ตรง ให้เปิดแต่ละ node แล้วเลือก credential ใหม่

---

## 3. Import เข้า n8n

ผ่าน UI: **Workflows → Import from File** เลือก `workflows/rag-gemini-pgvector.json`

หรือผ่าน CLI (Git Bash):

```bash
export MSYS_NO_PATHCONV=1
docker cp workflows/rag-gemini-pgvector.json n8n:/tmp/rag-gemini-pgvector.json
docker exec n8n n8n import:workflow --input=/tmp/rag-gemini-pgvector.json
```

(`id` ของ workflow คือ `RagGeminiPgVec01` import ซ้ำจะทับของเดิม ไม่สร้างซ้ำ — ถ้าเปิด editor ค้างไว้ให้ reload ก่อน ไม่งั้นกด Save จะทับของที่ import)

---

## 4. วิธีใช้

### 4.1 นำเข้าเอกสาร

- **ทดสอบใน editor:** ดับเบิลคลิก node **Upload Document** → *Test step* / *Execute workflow* → เปิดลิงก์ Test URL แล้วอัปโหลดไฟล์
- **ใช้งานจริง:** Publish/Activate workflow แล้วเปิด `http://localhost:5679/form/rag-upload`

เมื่อสำเร็จจะขึ้นข้อความ "นำเข้าเอกสารเรียบร้อยแล้ว" ตรวจใน DB ได้:

```bash
docker exec n8n-postgres psql -U n8n -d n8n -c \
  "select metadata->>'source' as source, count(*) as chunks from rag_documents group by 1 order by 1;"
```

### 4.2 ถามคำถาม

กดปุ่ม **Open chat** ใน editor แล้วถามเรื่องที่อยู่ในเอกสาร เช่น "นโยบายคืนสินค้าเป็นอย่างไร"
AI จะเรียก `Knowledge_Base` แล้วตอบพร้อมบรรทัด `ที่มา: <ชื่อไฟล์>` — ถ้าไม่มีในเอกสารจะตอบว่า "ไม่พบข้อมูลนี้ในเอกสาร"

---

## 5. จัดการข้อมูล

อัปโหลดไฟล์ชื่อเดิมซ้ำ **ไม่ได้ลบของเก่า** จะได้ chunk ซ้ำ ให้ลบก่อนอัปโหลดใหม่:

```sql
-- ลบเอกสารไฟล์เดียว
DELETE FROM rag_documents WHERE metadata->>'source' = 'ชื่อไฟล์.pdf';

-- ล้างคลังความรู้ทั้งหมด
TRUNCATE rag_documents;

-- ล้างประวัติแชท
TRUNCATE rag_chat_histories;
```

รันผ่าน `docker exec -it n8n-postgres psql -U n8n -d n8n` หรือ client ใด ๆ ที่ต่อ `localhost:5689`

---

## 6. ปรับแต่ง

| ต้องการ | แก้ที่ |
|---|---|
| chunk ใหญ่/เล็กลง | **Recursive Character Text Splitter** → `chunkSize`, `chunkOverlap` (เอกสารไทยประโยคยาว 800–1500 กำลังดี) — ต้องนำเข้าใหม่ |
| ดึง context มากขึ้น | **Knowledge Base** → `topK` (มากขึ้น = ตอบครบขึ้น แต่ใช้ token มากขึ้น) |
| เปลี่ยน model ตอบ | **Google Gemini Chat Model** → `modelName` |
| เปลี่ยน model embedding | แก้ **ทั้ง** Embeddings Gemini (insert) และ (query) ให้ตรงกัน แล้ว `DROP TABLE rag_documents;` และนำเข้าเอกสารใหม่ทั้งหมด (vector คนละ model / คนละมิติ เทียบกันไม่ได้) |
| แยกคลังความรู้หลายชุด | เปลี่ยน `tableName` ทั้งใน Insert และ Knowledge Base ให้ตรงกัน หรือใช้ `options → collection` |

### เรื่อง index

`gemini-embedding-001` ได้ vector 3072 มิติ pgvector เก็บได้ แต่ index แบบ HNSW / IVFFlat รองรับ `vector` ไม่เกิน 2000 มิติ
workflow นี้จึงค้นแบบ sequential scan ซึ่งเร็วพอสำหรับเอกสารระดับหลักพัน–หมื่น chunk
ถ้าข้อมูลใหญ่กว่านั้นค่อยพิจารณาใช้ model embedding ที่มิติเล็กลง หรือ `halfvec`

---

## 7. ข้อควรระวัง

- **โควตา Gemini free tier:** จำกัดจำนวน request ต่อวันต่อ model (key นี้เคยโดน 429 ที่ประมาณ 20 ครั้ง/วันสำหรับ `gemini-2.5-flash-lite`)
  คำถาม 1 ครั้ง = embedding 1–2 ครั้ง + chat 2–3 ครั้ง ใช้ร่วมกับ workflow อื่นที่ใช้ key เดียวกัน
- **Form ไม่มีการยืนยันตัวตน:** ใครรู้ URL ก็ใส่เอกสารเข้าคลังความรู้ได้ (และทำให้ AI ตอบผิดได้)
  ถ้าจะเปิดให้คนอื่นใช้ ให้ตั้ง *Authentication → Basic Auth* ที่ node **Upload Document** และอย่า expose พอร์ต 5679 ออกนอกเครื่อง
- **ข้อมูลลับ:** เนื้อหาเอกสารถูกส่งไปที่ Google เพื่อสร้าง embedding และตอบคำถาม อย่าอัปโหลดเอกสารที่ห้ามออกนอกองค์กร
- Form ตั้ง `responseMode: lastNode` เบราว์เซอร์จะรอจนสร้าง embedding เสร็จ ไฟล์ PDF ใหญ่อาจขึ้น timeout ทั้งที่การนำเข้ายังทำงานต่อจนเสร็จ — ตรวจผลด้วย SQL ในข้อ 4.1
- เอกสาร PDF ที่เป็นภาพสแกน (ไม่มี text layer) จะอ่านไม่ได้ ต้อง OCR ก่อน
