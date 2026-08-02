# Software Specification: Global Daily News Aggregation Platform

## 1. ภาพรวมโครงการ

ระบบนี้มีวัตถุประสงค์เพื่อรวบรวมข่าวสำคัญจากแหล่งข่าวหลักทั่วโลก จัดเก็บข่าวลงในฐานข้อมูล PostgreSQL แสดงผลผ่านเว็บไซต์ที่พัฒนาด้วย Node.js และ Next.js และส่งสรุปข่าวรายวันไปยัง LINE Official Account (LINE OA) โดยมี n8n เป็น workflow automation หลักของระบบ

Repository เป้าหมายสำหรับพัฒนาระบบ: <https://github.com/Ittipolint/dailynews>

## 2. เป้าหมายหลัก

- รวบรวมข่าวจากแหล่งข่าวสำคัญทั่วโลกตามหมวดหมู่
- เก็บข่าวทั้งภาษาดั้งเดิมของแหล่งข่าวและฉบับแปลภาษาไทย
- ใช้ n8n เป็นตัวควบคุม flow หลัก เช่น ดึงข่าว แปลข่าว บันทึกฐานข้อมูล ส่ง LINE OA และบันทึก log
- ใช้ PostgreSQL เป็นฐานข้อมูลสำหรับข่าว metadata translation และ log การทำงาน
- ใช้ Node.js และ Next.js สร้างเว็บไซต์สำหรับค้นหา อ่าน กรอง และจัดหมวดหมู่ข่าว
- ส่ง digest ข่าวรายวันไปยัง LINE OA แบบอัตโนมัติ
- รองรับการตรวจสอบซ้ำของข่าว deduplication และการบันทึกแหล่งที่มา

## 3. ขอบเขตระบบ

### 3.1 In Scope

- News ingestion จาก RSS feeds, official APIs, หรือ scraping เฉพาะกรณีที่ได้รับอนุญาตตามข้อกำหนดของเว็บไซต์
- Translation pipeline จากภาษาต้นฉบับเป็นภาษาไทย
- Data normalization และ categorization
- News storage ใน PostgreSQL
- Workflow orchestration ด้วย n8n
- Website frontend และ backend ด้วย Next.js
- Daily LINE OA push message
- Admin dashboard เบื้องต้นสำหรับดูสถานะ workflow และ log
- Search และ filtering ตามวันที่ แหล่งข่าว ประเทศ ภาษา และหมวดหมู่

### 3.2 Out of Scope ระยะแรก

- ระบบ user-generated content
- ระบบ comment หรือ community moderation
- Mobile native application
- การเผยแพร่ full article ที่ละเมิดลิขสิทธิ์ หากแหล่งข่าวอนุญาตเฉพาะ excerpt ให้เก็บและแสดงเฉพาะ excerpt พร้อม link กลับต้นทาง
- Real-time breaking news alert แบบ push ทันที ยกเว้น LINE digest รายวัน

## 4. สถาปัตยกรรมระบบ

```text
News Sources
  | RSS / API / Allowed Scraping
  v
n8n Workflows
  | fetch -> normalize -> deduplicate -> translate -> classify -> persist -> notify
  v
PostgreSQL
  | news, translations, sources, categories, workflow_logs, delivery_logs
  v
Next.js Website
  | public news pages, search, filters, admin views
  v
LINE Official Account
  | daily digest push / broadcast
```

## 5. องค์ประกอบหลัก

### 5.1 n8n

n8n เป็น workflow engine หลัก ทำหน้าที่ดังนี้

- ตั้ง schedule ดึงข่าวตามรอบเวลา
- ดึงข่าวจาก RSS/API ของแต่ละ source
- ตรวจสอบ schema และ normalize field
- ตรวจจับข่าวซ้ำจาก canonical URL, title similarity, source ID, published timestamp และ content hash
- เรียก translation service เพื่อแปลข่าวเป็นภาษาไทย
- เรียก classification service หรือ rule-based classifier เพื่อจัดหมวดหมู่
- บันทึกข้อมูลลง PostgreSQL
- บันทึก workflow logs และ error logs
- สร้าง daily digest และส่งไป LINE OA
- retry งานที่ล้มเหลวตาม policy

### 5.2 PostgreSQL

PostgreSQL เป็นฐานข้อมูลหลักสำหรับ

- แหล่งข่าว
- ข่าวต้นฉบับ
- ข่าวแปลภาษาไทย
- หมวดหมู่
- log การดึงข่าว
- log การแปล
- log การส่ง LINE OA
- สถานะ workflow และ error tracking

### 5.3 Website: Node.js + Next.js

เว็บไซต์ใช้ Next.js เป็น full-stack framework โดยมีความสามารถดังนี้

- หน้าแรกแสดงข่าวล่าสุดและข่าวเด่น
- หน้า category เช่น World, Business, Technology, Health
- หน้า source สำหรับดูข่าวจากสำนักข่าวใดสำนักข่าวหนึ่ง
- หน้า article แสดงหัวข้อ สรุป เนื้อหาต้นฉบับเท่าที่ได้รับอนุญาต แปลไทย และ link กลับต้นทาง
- Search และ filters
- Admin pages สำหรับดู workflow status, failed jobs และ delivery logs
- SEO metadata, sitemap และ Open Graph tags

### 5.4 LINE Official Account

LINE OA ใช้สำหรับส่งข่าวรายวัน โดยระบบจะ

- สร้าง daily digest จากข่าวสำคัญของวัน
- แยกข่าวตามหมวดหมู่หลัก
- ส่งข้อความภาษาไทยพร้อม link ไปยังเว็บไซต์
- เก็บ delivery status และ error response จาก LINE Messaging API
- จำกัดจำนวนข่าวต่อ message ตามข้อจำกัดของ LINE

## 6. แหล่งข่าวสำคัญ แยกตาม Category

> หมายเหตุ: การใช้งานจริงต้องตรวจสอบเงื่อนไข RSS/API, robots.txt, terms of service และข้อจำกัดด้านลิขสิทธิ์ของแต่ละแหล่งข่าวก่อนใช้งาน

### 6.1 Global / Breaking News

- Reuters
- Associated Press (AP)
- BBC News
- CNN
- Al Jazeera English
- The Guardian
- France 24
- Deutsche Welle (DW)
- NPR
- PBS NewsHour

### 6.2 Asia-Pacific

- Nikkei Asia
- The Japan Times
- Kyodo News
- NHK World-Japan
- South China Morning Post (SCMP)
- The Straits Times
- Channel NewsAsia (CNA)
- Bangkok Post
- The Nation Thailand
- Tempo Indonesia
- The Hindu
- Times of India

### 6.3 Europe

- Euronews
- Politico Europe
- Financial Times Europe
- The Local Europe editions
- El País
- Le Monde
- Der Spiegel
- Corriere della Sera
- Svenska Dagbladet

### 6.4 United States / Americas

- The New York Times
- The Washington Post
- Wall Street Journal
- Los Angeles Times
- USA Today
- Politico
- Axios
- Bloomberg
- CBC News
- The Globe and Mail
- Folha de S.Paulo
- Clarín

### 6.5 Middle East / Africa

- Al Arabiya
- The National News
- Middle East Eye
- Haaretz
- Times of Israel
- Arab News
- Daily Nation Kenya
- Mail & Guardian South Africa
- News24 South Africa
- The Africa Report

### 6.6 Business / Finance / Economy

- Bloomberg
- Financial Times
- The Wall Street Journal
- CNBC
- MarketWatch
- The Economist
- Nikkei Asia
- Forbes
- Fortune
- Business Insider
- Yahoo Finance

### 6.7 Technology / AI / Startups

- TechCrunch
- The Verge
- Wired
- Ars Technica
- MIT Technology Review
- VentureBeat
- The Information
- Hacker News
- The Register
- IEEE Spectrum
- ZDNET

### 6.8 Science / Space / Environment

- Nature News
- Science Magazine
- Scientific American
- NASA News
- Space.com
- National Geographic
- Mongabay
- Carbon Brief
- Inside Climate News
- Yale Environment 360

### 6.9 Health / Medicine

- WHO News
- CDC Newsroom
- NIH News Releases
- STAT News
- MedPage Today
- Medical Xpress
- Healthline News
- BMJ News
- The Lancet News

### 6.10 Sports

- ESPN
- BBC Sport
- Sky Sports
- The Athletic
- Reuters Sports
- AP Sports
- Fox Sports
- Goal.com
- Motorsport.com

### 6.11 Thailand / Southeast Asia Focus

- Thai PBS
- Bangkok Post
- The Nation Thailand
- Khaosod English
- Prachatai English
- Matichon
- Thairath
- Daily News Thailand
- Channel NewsAsia
- The Straits Times

## 7. ภาษาข่าวและการแปล

ระบบต้องเก็บข้อมูลข่าวอย่างน้อย 2 รูปแบบ

1. `original`: ข่าวตามภาษาดั้งเดิมของแหล่งข่าว
2. `thai_translation`: ข่าวฉบับแปลภาษาไทย

ข้อกำหนดเพิ่มเติม

- เก็บ `original_language` ตามมาตรฐาน ISO 639-1 เช่น `en`, `fr`, `de`, `ja`, `zh`, `th`
- เก็บ translation provider และ model/version ที่ใช้
- เก็บ translation status เช่น `pending`, `translated`, `failed`, `skipped`
- หากแหล่งข่าวเป็นภาษาไทยอยู่แล้ว ให้ตั้ง `thai_translation` เท่ากับ original หรือสร้าง normalized Thai summary แยกต่างหาก
- ควรแปล headline, summary และ excerpt ก่อน full content เพื่อลดค่าใช้จ่าย
- ต้องไม่แปลหรือเก็บ full article หากเงื่อนไขลิขสิทธิ์ไม่อนุญาต

## 8. Data Model เบื้องต้น

### 8.1 `sources`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Source ID |
| name | TEXT | ชื่อแหล่งข่าว |
| homepage_url | TEXT | URL หลัก |
| feed_url | TEXT | RSS/API endpoint |
| country | TEXT | ประเทศ |
| language | TEXT | ภาษาหลัก |
| category | TEXT | หมวดหมู่หลัก |
| is_active | BOOLEAN | เปิดใช้งานหรือไม่ |
| fetch_interval_minutes | INTEGER | รอบการดึงข่าว |
| created_at | TIMESTAMPTZ | วันที่สร้าง |
| updated_at | TIMESTAMPTZ | วันที่อัปเดต |

### 8.2 `articles`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Article ID |
| source_id | UUID FK | อ้างอิง source |
| canonical_url | TEXT UNIQUE | URL ข่าวต้นทาง |
| original_title | TEXT | หัวข้อภาษาต้นฉบับ |
| original_summary | TEXT | สรุปภาษาต้นฉบับ |
| original_content | TEXT | เนื้อหาที่ได้รับอนุญาตให้เก็บ |
| original_language | TEXT | ภาษา ISO 639-1 |
| author | TEXT | ผู้เขียน |
| image_url | TEXT | รูปประกอบ |
| published_at | TIMESTAMPTZ | วันที่เผยแพร่ |
| fetched_at | TIMESTAMPTZ | วันที่ดึงข้อมูล |
| content_hash | TEXT | hash สำหรับ deduplication |
| status | TEXT | `draft`, `published`, `archived`, `blocked` |
| created_at | TIMESTAMPTZ | วันที่สร้าง |
| updated_at | TIMESTAMPTZ | วันที่อัปเดต |

### 8.3 `article_translations`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Translation ID |
| article_id | UUID FK | อ้างอิง article |
| target_language | TEXT | เช่น `th` |
| translated_title | TEXT | หัวข้อภาษาไทย |
| translated_summary | TEXT | สรุปภาษาไทย |
| translated_content | TEXT | เนื้อหาแปลเท่าที่อนุญาต |
| provider | TEXT | translation provider |
| model | TEXT | model/version |
| status | TEXT | `pending`, `translated`, `failed` |
| error_message | TEXT | error ถ้ามี |
| created_at | TIMESTAMPTZ | วันที่สร้าง |
| updated_at | TIMESTAMPTZ | วันที่อัปเดต |

### 8.4 `categories`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Category ID |
| slug | TEXT UNIQUE | slug เช่น `world`, `business` |
| name_en | TEXT | ชื่ออังกฤษ |
| name_th | TEXT | ชื่อไทย |
| parent_id | UUID NULL | category parent |

### 8.5 `article_categories`

| Field | Type | Description |
| --- | --- | --- |
| article_id | UUID FK | Article ID |
| category_id | UUID FK | Category ID |
| confidence | NUMERIC | confidence score |

### 8.6 `workflow_logs`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Log ID |
| workflow_name | TEXT | ชื่อ workflow |
| execution_id | TEXT | n8n execution ID |
| status | TEXT | `started`, `success`, `failed`, `partial` |
| source_id | UUID NULL | source ที่เกี่ยวข้อง |
| message | TEXT | รายละเอียด |
| error_detail | JSONB | error payload |
| started_at | TIMESTAMPTZ | เวลาเริ่ม |
| finished_at | TIMESTAMPTZ | เวลาจบ |

### 8.7 `line_delivery_logs`

| Field | Type | Description |
| --- | --- | --- |
| id | UUID PK | Delivery ID |
| digest_date | DATE | วันที่ของ digest |
| line_target_type | TEXT | `broadcast`, `push`, `multicast` |
| status | TEXT | `queued`, `sent`, `failed` |
| message_payload | JSONB | payload ที่ส่ง |
| response_payload | JSONB | response จาก LINE |
| sent_at | TIMESTAMPTZ | เวลาส่ง |
| created_at | TIMESTAMPTZ | วันที่สร้าง |

## 9. n8n Workflow Specification

### 9.1 Workflow: Source Fetcher

Trigger: Cron ทุก 15-60 นาที แล้วแต่ source

Steps:

1. Load active sources จาก PostgreSQL
2. Filter source ที่ถึงเวลา fetch
3. Fetch RSS/API
4. Parse feed items
5. Normalize fields
6. Validate required fields: title, URL, published date
7. Generate content hash
8. Upsert article หรือ skip หากซ้ำ
9. Write workflow log

### 9.2 Workflow: Article Translator

Trigger: ทุก 10 นาที หรือหลัง fetch สำเร็จ

Steps:

1. Query articles ที่ยังไม่มี translation ภาษาไทย
2. Detect original language
3. If original language is `th`, copy/normalize Thai fields
4. Else call translation provider
5. Save translated headline, summary, excerpt/content
6. Mark translation status
7. Log success/error

### 9.3 Workflow: Category Classifier

Trigger: หลัง article ถูก insert หรือ translation สำเร็จ

Steps:

1. Load article title, summary, content excerpt
2. Apply rule-based keyword classifier หรือ AI classifier
3. Assign one or more categories
4. Save confidence score
5. Log result

### 9.4 Workflow: Daily LINE Digest

Trigger: Cron วันละครั้ง เช่น 07:00 Asia/Bangkok

Steps:

1. Query ข่าวใน 24 ชั่วโมงล่าสุด
2. Rank ข่าวด้วย category priority, source priority, recency และ uniqueness
3. เลือกข่าวจำนวนเหมาะสม เช่น 10-20 ข่าว
4. Group ตาม category
5. Generate Thai digest message
6. Include website links
7. Send to LINE OA ผ่าน LINE Messaging API
8. Save delivery log

### 9.5 Workflow: Error Retry

Trigger: Cron ทุก 30 นาที

Steps:

1. Query failed fetch/translation/delivery logs
2. Retry ตาม retry policy
3. หากเกิน max retries ให้ mark เป็น permanently failed
4. แจ้ง admin ผ่าน email หรือ LINE admin group หากจำเป็น

## 10. Website Feature Specification

### 10.1 Public Pages

- `/` หน้าแรก ข่าวล่าสุด ข่าวเด่น และ category highlights
- `/articles/[id-or-slug]` หน้าอ่านข่าว
- `/categories/[slug]` ข่าวตามหมวดหมู่
- `/sources/[sourceSlug]` ข่าวตาม source
- `/search` ค้นหาข่าว
- `/daily/[date]` archive digest รายวัน

### 10.2 Admin Pages

- `/admin` dashboard ภาพรวม
- `/admin/sources` จัดการ source
- `/admin/workflows` ดู workflow status
- `/admin/logs` ดู logs
- `/admin/articles` ตรวจสอบข่าวและสถานะ translation

### 10.3 Article Page Requirements

แต่ละข่าวควรแสดง

- ชื่อข่าวภาษาต้นฉบับ
- ชื่อข่าวภาษาไทย
- แหล่งข่าว
- วันที่เผยแพร่
- ผู้เขียน ถ้ามี
- รูปภาพ ถ้ามีและได้รับอนุญาต
- สรุปภาษาต้นฉบับ
- สรุปภาษาไทย
- หมวดหมู่
- link ไปอ่านต้นฉบับ
- disclaimer เรื่องการแปลอัตโนมัติ

## 11. API Specification เบื้องต้น

### 11.1 Public API

- `GET /api/articles`
  - Query: `category`, `source`, `language`, `from`, `to`, `q`, `page`, `limit`
- `GET /api/articles/:id`
- `GET /api/categories`
- `GET /api/sources`
- `GET /api/daily/:date`

### 11.2 Admin API

- `POST /api/admin/sources`
- `PATCH /api/admin/sources/:id`
- `POST /api/admin/articles/:id/retranslate`
- `POST /api/admin/line/send-digest`
- `GET /api/admin/logs`

## 12. Environment Variables

```env
DATABASE_URL=postgresql://user:password@localhost:5432/dailynews
N8N_WEBHOOK_SECRET=replace_me
TRANSLATION_PROVIDER=openai_or_other_provider
TRANSLATION_API_KEY=replace_me
LINE_CHANNEL_ACCESS_TOKEN=replace_me
LINE_CHANNEL_SECRET=replace_me
NEXT_PUBLIC_SITE_URL=https://example.com
ADMIN_AUTH_SECRET=replace_me
```

## 13. Security Requirements

- เก็บ secrets ใน environment variables หรือ secret manager เท่านั้น
- Admin routes ต้องมี authentication และ authorization
- n8n webhook ต้องตรวจ signature หรือ shared secret
- Rate limit public API และ admin API
- Validate URL และ sanitize HTML content
- ป้องกัน SSRF ในขั้นตอน fetch URL
- ใช้ prepared statements หรือ ORM ที่ป้องกัน SQL injection
- เปิด audit log สำหรับ admin actions

## 14. Compliance และ Copyright

- ตรวจสอบ terms of service ของทุก source ก่อน ingest
- หาก source อนุญาตเฉพาะ headline/excerpt ให้เก็บเฉพาะข้อมูลนั้น
- ต้องแสดง attribution และ canonical link กลับต้นทางเสมอ
- หลีกเลี่ยงการ publish full article หากไม่ได้รับอนุญาต
- เคารพ robots.txt สำหรับ scraping
- เก็บ image เฉพาะ URL reference หรือภาพที่อนุญาตให้ใช้เท่านั้น

## 15. Observability

- เก็บ workflow execution logs ใน PostgreSQL
- ใช้ n8n execution logs สำหรับ debugging
- เพิ่ม health check endpoint เช่น `/api/health`
- Track metrics เช่น
  - จำนวนข่าวที่ fetch ต่อ source
  - จำนวนข่าวซ้ำที่ถูก skip
  - translation success rate
  - LINE delivery success rate
  - average fetch latency
  - failed workflows per day

## 16. Deployment Specification

### 16.1 Recommended Services

- Web app: Dockerized Next.js application
- Workflow: Dockerized n8n
- Database: Managed PostgreSQL หรือ self-hosted PostgreSQL
- Reverse proxy: Nginx หรือ cloud load balancer
- Scheduler: n8n cron nodes
- Hosting options: VPS, Render, Railway, Fly.io, DigitalOcean, AWS, GCP, Azure

### 16.2 Docker Compose Services

- `web`: Next.js app
- `n8n`: workflow engine
- `postgres`: database
- `redis` optional: queue/cache
- `nginx` optional: reverse proxy

## 17. Development Milestones

### Phase 1: Foundation

- สร้าง Next.js project
- ตั้งค่า PostgreSQL schema และ migrations
- ตั้งค่า Docker Compose
- สร้าง source/category seed data
- สร้าง article list และ article detail pages

### Phase 2: n8n Ingestion

- สร้าง workflow ดึง RSS/API
- บันทึก articles ลง PostgreSQL
- ทำ deduplication
- เพิ่ม workflow logs

### Phase 3: Translation และ Classification

- เพิ่ม workflow แปลไทย
- เพิ่ม category classifier
- แสดง original และ Thai translation บนเว็บไซต์

### Phase 4: LINE OA Digest

- สร้าง daily digest generator
- เชื่อมต่อ LINE Messaging API
- บันทึก delivery logs
- เพิ่ม manual resend ใน admin

### Phase 5: Hardening

- เพิ่ม auth สำหรับ admin
- เพิ่ม monitoring และ alerting
- ปรับ performance และ indexing
- ตรวจ compliance ของแต่ละ source

## 18. Acceptance Criteria

- ระบบสามารถดึงข่าวจากอย่างน้อย 20 แหล่งข่าวในหลายหมวดหมู่ได้สำเร็จ
- ข่าวทุกชิ้นมี original title และ canonical URL
- ข่าวที่ไม่ใช่ภาษาไทยมี translation ภาษาไทยอย่างน้อย title และ summary
- ระบบสามารถตรวจข่าวซ้ำและไม่บันทึก duplicate canonical URL
- เว็บไซต์สามารถแสดงข่าวล่าสุด ข่าวตามหมวดหมู่ และหน้ารายละเอียดข่าวได้
- LINE OA ส่ง digest รายวันได้สำเร็จและมี delivery log
- Admin สามารถดู failed workflows และ retry ได้
- ทุก article page มี attribution และ link กลับต้นทาง

## 19. Open Questions

- จะใช้ translation provider ใดเป็นหลัก และมีงบประมาณต่อวันเท่าไร
- ต้องการเก็บ full content หรือเฉพาะ title/summary/excerpt ตามสิทธิ์ของแต่ละ source
- จะใช้ LINE broadcast ถึงผู้ติดตามทั้งหมด หรือ push/multicast เฉพาะกลุ่ม
- ต้องการระบบ ranking ข่าวแบบ rule-based หรือ AI-based
- ต้องการรองรับ multi-tenant หรือเฉพาะเว็บไซต์เดียว
