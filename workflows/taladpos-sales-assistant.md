# TaladPOS Sales Assistant — คู่มือการตั้งค่า

Workflow ตัวอย่าง AI ช่วยขายสินค้าให้ลูกค้า: AI Agent (Gemini) คุยกับลูกค้าผ่านหน้าแชทของ n8n
แล้วเรียก TaladPOS API เพื่อดึงข้อมูลสินค้า ราคา โปรโมชั่น และข้อมูลการขายมาตอบ

ไฟล์ workflow: [`taladpos-sales-assistant.json`](taladpos-sales-assistant.json)

---

## 1. ภาพรวมการทำงาน

```
When chat message received → Config → Login TaladPOS → Session + Token → AI Agent
                                                                           ├─ Google Gemini Chat Model
                                                                           ├─ Simple Memory
                                                                           └─ Tools: Get Products, Get Promotions,
                                                                                     Get Bundle Promotions, Get Best Sellers, Price Cart
```

| Node | หน้าที่ |
|---|---|
| **Config** | จุดตั้งค่าเดียว: `apiBaseUrl`, `username`, `password` |
| **Login TaladPOS** | `POST /api/v1/auth/login` ขอ JWT ทุกข้อความที่ลูกค้าพิมพ์ (token อายุ 480 นาที) |
| **Session + Token** | เก็บ `token`, `apiBaseUrl` และคัดลอก `chatInput`, `sessionId` กลับมา (ผลของ Login ทับ `$json` เดิม ถ้าไม่มี node นี้ Simple Memory จะหา session ไม่เจอ) |
| **AI Agent** | system prompt กำหนดบทบาทพนักงานขาย ตอบภาษาไทย ห้ามเดาราคา ห้ามเปิดเผยยอดขาย/สต็อกจริง |

### Tools ที่ AI เรียกได้

| Tool (ชื่อที่ AI เห็น) | API | ข้อมูลที่ได้ |
|---|---|---|
| `Get_Products` | `GET /api/v1/products?search=&pageSize=100` | id, ชื่อ, ราคาป้าย (ก่อนโปร), สต็อก, `isLowStock`, `isOutOfStock` |
| `Get_Promotions` | `GET /api/v1/promotions?activeOnly=true` | โปรลด % ต่อสินค้า (`Item`) หรือทั้งบิล (`Bill`), สำหรับสมาชิกหรือไม่ |
| `Get_Bundle_Promotions` | `GET /api/v1/conditional-promotions?activeOnly=true` | โปรซื้อครบแถม/ลด พร้อมคำอธิบายภาษาไทย |
| `Get_Best_Sellers` | `GET /api/v1/reports/best-selling-products` (30 วันล่าสุด, 10 อันดับ) | สินค้าขายดีเรียงตามจำนวนที่ขาย |
| `Price_Cart` | `POST /api/v1/sales/preview` | ราคาตะกร้าจริงหลังคิดทุกโปร, ของแถมที่ยังไม่ได้หยิบ — **ไม่บันทึกการขาย ไม่ตัดสต็อก** |

> AI ไม่มี tool สร้างการขาย (`POST /api/v1/sales`) หรือแก้ข้อมูลใด ๆ — ทุก tool เป็นการอ่านอย่างเดียว

---

## 2. สิ่งที่ต้องมีก่อน

| รายการ | ตรวจอย่างไร |
|---|---|
| n8n stack ของโปรเจกต์นี้รันอยู่ | `docker compose up -d` แล้วเปิด http://localhost:5679 |
| TaladPOS API รันอยู่ (Docker บนเครื่องเดียวกัน) | `docker ps --format "table {{.Names}}\t{{.Ports}}"` ต้องเห็น container ของ API |
| บัญชี **manager** ของ TaladPOS | `promotions`, `conditional-promotions`, `reports` เปิดให้ Manager เท่านั้น — บัญชี cashier จะได้ 403 |
| Credential **Google Gemini(PaLM) Api account** ใน n8n | ตัวเดียวกับที่ workflow "LINE MCP Agent" ใช้ |

---

## 3. นำ workflow เข้า n8n

### วิธี A — Copy / Paste (แนะนำ)

1. เปิดไฟล์ `taladpos-sales-assistant.json` → เลือกทั้งหมด → Copy
2. ใน n8n กด **Create workflow** (หรือเปิด workflow ว่าง)
3. คลิกบน canvas แล้วกด `Ctrl+V`
4. ตรวจ node **Google Gemini Chat Model** ว่าเลือก credential แล้ว (ถ้าเป็นสีแดง ให้เลือก credential ใหม่)
5. กด **Save**

### วิธี B — Import ด้วย CLI

รันจาก root ของโปรเจกต์ใน Git Bash:

```bash
export MSYS_NO_PATHCONV=1
docker cp workflows/taladpos-sales-assistant.json n8n:/tmp/taladpos-sales-assistant.json
docker exec n8n n8n import:workflow --input=/tmp/taladpos-sales-assistant.json
```

- ต้องมี `MSYS_NO_PATHCONV=1` ไม่งั้น Git Bash แปลง `/tmp/...` เป็น path ของ Windows แล้ว import ไม่เข้า
- import ซ้ำจะทับ workflow เดิม (id `TaladPosSalesAi1`) — ถ้าเปิดหน้า editor ค้างไว้ให้ reload ก่อน ไม่งั้นกด Save จากหน้าเก่าจะทับของที่เพิ่ง import

---

## 4. ตั้งค่า `apiBaseUrl` (node **Config**)

> ⚠️ **ห้ามใช้ `http://localhost:...`** — n8n รันอยู่ใน container, `localhost` ในนั้นคือตัว n8n เอง ไม่ใช่เครื่องของเรา

เลือกวิธีใดวิธีหนึ่ง:

### ทางเลือก 1 — `host.docker.internal` (ค่าเริ่มต้น, แนะนำบน Docker Desktop)

```
http://host.docker.internal:5054
```

ชื่อนี้ Docker Desktop (Windows/macOS) ชี้กลับมาที่เครื่อง host ให้อัตโนมัติ ไม่ต้องสนใจว่า IP เครื่องเปลี่ยนหรือไม่

### ทางเลือก 2 — IP ของเครื่อง

**หา IP เครื่อง**

- **Windows** — เปิด PowerShell หรือ Command Prompt:
  ```powershell
  ipconfig
  ```
  ดูเฉพาะ adapter ที่มีค่า **Default Gateway** (ปกติคือ `Wireless LAN adapter Wi-Fi` หรือ `Ethernet adapter Ethernet`) แล้วใช้ค่า **IPv4 Address**

  ```
  Wireless LAN adapter Wi-Fi:
     IPv4 Address. . . . . . . . . . . : 192.168.50.155   ← ใช้ค่านี้
     Default Gateway . . . . . . . . . : 192.168.50.1
  ```

  ข้าม adapter เหล่านี้: `vEthernet (WSL ...)`, `vEthernet (Default Switch)`, `192.168.56.x` (VirtualBox) — เป็น network เสมือน

  คำสั่งสั้นที่ได้เฉพาะ IP ของ adapter ที่ออก internet:
  ```powershell
  (Get-NetIPConfiguration | Where-Object { $_.IPv4DefaultGateway }).IPv4Address.IPAddress
  ```
- **macOS**: `ipconfig getifaddr en0` (Wi-Fi) หรือ `en1`
- **Linux**: `hostname -I` (ค่าแรก)

จากนั้นใส่ใน Config เช่น:

```
http://192.168.50.155:5054
```

ข้อควรรู้:
- IP จาก DHCP **เปลี่ยนได้** เมื่อย้าย Wi-Fi หรือรีสตาร์ต router — ถ้าวันหนึ่งเรียกไม่ติดให้หา IP ใหม่
- API ต้องฟังบนทุก interface (`0.0.0.0`) — container ที่ publish port ด้วย `ports:` เป็นแบบนี้อยู่แล้ว
  แต่ถ้ารัน `dotnet run` ตรง ๆ บนเครื่อง API จะฟังแค่ `localhost` → ใช้ IP ไม่ได้ (ได้ *Connection refused*) ให้ใช้ทางเลือก 1 แทน
- ถ้า `host.docker.internal` ใช้ได้แต่ IP ใช้ไม่ได้ มักเป็น Windows Firewall บล็อกพอร์ต

### ทางเลือก 3 — ต่อ container ของ API เข้า network เดียวกับ n8n

```bash
docker network connect line-ai-learning_default <ชื่อ-container-ของ-API>
```

แล้วใช้ `http://<ชื่อ-container-ของ-API>:<พอร์ตฝั่ง container>` เช่น `http://taladpos-api:8080`
(ใช้พอร์ต **ฝั่งขวา** ของ `ports:` เพราะคุยกันภายใน network ไม่ผ่าน host)

การ `docker network connect` แบบนี้หายไปเมื่อ container ของ API ถูกสร้างใหม่ (`docker compose up` หลังแก้ config / `down`)
ถ้าจะใช้ถาวรให้ประกาศ network `line-ai-learning_default` เป็น `external` ใน compose ของ TaladPOS แทน

### พอร์ตที่ต้องใช้ (ทางเลือก 1 และ 2)

ใช้พอร์ต **ฝั่งซ้าย** ของ `ports:` ของ container API:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
# taladpos-api   0.0.0.0:5054->8080/tcp   → ใช้ 5054
```

### ทดสอบว่า n8n มองเห็น API

```bash
docker exec n8n wget -S -O- http://host.docker.internal:5054/api/v1/products
```

| ผลลัพธ์ | ความหมาย |
|---|---|
| `HTTP/1.1 401 Unauthorized` | ✅ ต่อถึง API แล้ว (401 เพราะยังไม่ได้ส่ง token — ถูกต้อง) |
| `Connection refused` | IP ถูกแต่ไม่มีอะไรฟังที่พอร์ตนั้น — API ไม่รัน, พอร์ตผิด หรือ API ฟังแค่ localhost |
| `timed out` / `bad address` | IP หรือชื่อ host ผิด หรือ firewall บล็อก |
| `HTTP/1.1 307` | API redirect ไป HTTPS — ใช้ URL `https://...` ตามที่ API ตั้งไว้ หรือปิด HTTPS redirect ฝั่ง API |

---

## 5. ตั้งค่าบัญชีผู้ใช้ TaladPOS

ค่าในไฟล์คือบัญชีตัวอย่างของ TaladPOS: `manager` / `Manager123!`
ถ้าติดตั้ง TaladPOS แล้วเปลี่ยนรหัสผ่าน ให้แก้ที่ node **Config**

### (แนะนำเมื่อใช้รหัสจริง หรือจะ commit ขึ้น repo public) เก็บรหัสผ่านใน Credential แทน

รหัสผ่านใน node Config จะติดไปกับไฟล์ JSON ทุกครั้งที่ export / commit ถ้าไม่ต้องการ:

1. n8n → **Credentials** → **Create** → เลือก **Custom Auth**
2. ช่อง JSON ใส่:
   ```json
   { "body": { "username": "manager", "password": "รหัสจริง" } }
   ```
3. node **Login TaladPOS** → Authentication = **Generic Credential Type** → Generic Auth Type = **Custom Auth** → เลือก credential ที่สร้าง
4. ช่อง **JSON** (Body) เปลี่ยนเป็น `{}` — n8n จะรวม `username`/`password` จาก credential เข้า body ให้
5. ลบ field `username` และ `password` ออกจาก node **Config**

> การอ่านจาก environment variable (`$env.XXX`) ถูกปิดไว้โดยค่าเริ่มต้นใน n8n 2.x จึงไม่ได้ใช้วิธีนั้น

---

## 6. ทดลองใช้งาน

1. เปิด workflow → กดปุ่ม **Open chat** (มุมล่าง)
2. ลองถามตัวอย่าง (ผลที่คาดไว้อิงข้อมูลตัวอย่างของ TaladPOS ช่วงกันยายน–ตุลาคม 2026):

| คำถาม | สิ่งที่ควรเกิด |
|---|---|
| มะม่วง 3 ลูก จ่ายเท่าไหร่ | เรียก `Get_Products` → `Price_Cart` ตอบ **90 บาท** (โปรมะม่วงซื้อ 2 แถม 1) |
| มีโปรอะไรบ้าง | เรียก `Get_Promotions` + `Get_Bundle_Promotions` บอกโปรพร้อมระบุโปรสำหรับสมาชิก |
| แนะนำผลไม้ขายดีหน่อย | เรียก `Get_Best_Sellers` แนะนำเฉพาะของที่มีสต็อก |
| มีทุเรียนไหม | ตอบว่าหมดชั่วคราว และเสนอสินค้าอื่น |

3. ดูว่า AI เรียก tool อะไรบ้าง: แท็บ **Executions** → เปิด execution → คลิก node **AI Agent** / tool แต่ละตัว

---

## 7. ปรับแต่งเพิ่มเติม

| ต้องการ | แก้ที่ |
|---|---|
| เปลี่ยน model | node **Google Gemini Chat Model** → Model (เช่น `models/gemini-2.5-flash`) |
| ช่วงวันของสินค้าขายดี | tool **Get Best Sellers** → query `from` เปลี่ยน `minus({ days: 30 })` |
| จำนวนข้อความที่ AI จำ | node **Simple Memory** → Context Window Length (ค่าเริ่มต้น 10) |
| บุคลิก / กฎการขาย | node **AI Agent** → Options → System Message |
| เปิดแชทให้ลูกค้าภายนอกใช้ | node **When chat message received** → เปิด **Make Chat Publicly Available** แล้ว Publish/Activate workflow จะได้ URL แชท |

ข้อควรระวังเมื่อเปิดแชทสาธารณะ: หน้าแชทไม่มีการยืนยันตัวตน ใครมี URL ก็ใช้ได้ และทุกข้อความกินโควตา Gemini
(tool ทุกตัวอ่านข้อมูลอย่างเดียว แต่ทำงานด้วยสิทธิ์ manager ของ TaladPOS)

---

## 8. ข้อจำกัดที่รู้แล้ว

- **ตะกร้าที่มีสินค้าหมดปน** (เช่น "มะม่วง 3 + ทุเรียน 1"): จากการทดสอบ Gemini รุ่นเล็กมักไม่เรียก `Price_Cart`
  และอาจตอบราคาป้ายที่ยังไม่หักโปร (เช่นตอบ 135 บาท แทน 90 บาท) — system prompt มีกฎกันไว้แล้วแต่ model ยังทำตามไม่สม่ำเสมอ
  ถ้าต้องใช้จริงควรลอง model ที่เก่งกว่า
- **ราคาสำหรับสมาชิก**: `Price_Cart` ส่ง `memberId: null` เสมอ จึงคิดราคาแบบลูกค้าทั่วไป — โปรสำหรับสมาชิกจะถูกบอกเป็นข้อมูลเท่านั้น
- **รูปสินค้า**: `imageUrl` ในข้อมูลตัวอย่างเป็น `http://localhost:3000/...` เปิดได้เฉพาะบนเครื่องที่รันเว็บ TaladPOS
- **ข้อมูลตัวอย่างมีวันหมดอายุ**: ยอดขายตัวอย่างอยู่ช่วง 2026-08-05 ถึง 2026-09-14 และโปรส่วนใหญ่หมดเขต 2026-10-16
  เลยช่วงนั้น `Get_Best_Sellers` / โปรโมชั่นจะว่าง (ไม่ใช่ workflow เสีย)
- Login ใหม่ทุกข้อความ (เพิ่ม 1 request ต่อข้อความ) — เรียบง่ายเหมาะกับตัวอย่าง

---

## 9. แก้ปัญหา

| อาการ | สาเหตุ / วิธีแก้ |
|---|---|
| **Login TaladPOS** error `ECONNREFUSED` / timeout | API ไม่รัน, `apiBaseUrl` ใช้ `localhost`, IP หรือพอร์ตผิด — ทดสอบตามหัวข้อ 4 |
| **Login TaladPOS** ได้ 401 `invalid_credentials` | username/password ใน Config (หรือ Custom Auth credential) ผิด |
| tool ได้ **403 Forbidden** | login ด้วยบัญชี cashier — ต้องใช้บัญชี manager |
| Gemini **429** `quota exceeded` | Free tier จำกัด **20 requests/วัน ต่อ model** (1 ข้อความที่เรียก tool ใช้ ~2–4 requests และใช้โควตาร่วมกับ LINE MCP Agent) — รอรีเซ็ต, เปลี่ยน model หรือเปิด billing |
| Gemini **503** high demand | ฝั่ง Google โหลดสูงชั่วคราว — ส่งข้อความใหม่ หรือเปิด Retry On Fail ที่ node model |
| AI ตอบว่างเปล่า | model รุ่นเล็กบางครั้งคืนข้อความว่างหลังเรียก tool — ส่งซ้ำ หรือเปลี่ยน model |
| แชทใน editor ขึ้น *Failed to receive response* | `WEBHOOK_URL` / `N8N_EDITOR_BASE_URL` ใน `docker-compose.yml` ต้องเป็น `http://localhost:5679/` |
| Simple Memory error *No session ID found* | มีการลบ/เปลี่ยนชื่อ node **Session + Token** หรือ field `sessionId` |
| `Get_Best_Sellers` ได้ `[]` | ไม่มียอดขายใน 30 วันล่าสุด — ขยายช่วง `from` (หัวข้อ 7) |
