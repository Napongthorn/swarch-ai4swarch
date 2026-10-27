Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 — RabbitMQ + transactional outbox สำหรับ post-order actions

ผู้รีวิว: architecture review board (แนว ATAM) · ไฟล์ที่ใช้: `docs/adr/0002-use-rabbitmq-for-post-order-actions.md`, `docs/scenario.md`

## Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | ลูกค้า | กด place order | 11.11 peak 200 orders/s, MailFast p95 1.5 s / p99 8 s | `PlaceOrder` ตอบหลัง commit order + outbox โดยไม่รอ MailFast/RabbitMQ | p95 `PlaceOrder` ≤ 500 ms (baseline ~300 ms) |
| S2 Availability | MailFast หรือ `loyaltyservice` | ล่ม 10 นาที | ระหว่าง peak | order ยังสำเร็จ; event ค้างใน queue แล้วประมวลผลต่อเมื่อกลับมา | order success 100 %; อีเมล/แต้มที่หาย = 0 |
| S3 Correctness | broker restart หรือ relay crash ก่อน mark sent | `OrderPlaced` ถูกส่งซ้ำ | peak 200 events/s | `loyaltyservice` dedup ด้วย `order_id` | แต้มต่อ `order_id` = 1 พอดี; ได้แต้มภายใน 60 s **นับจากเวลา order** |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | Outbox ปิด dual-write แค่ DB↔broker แต่ order ถูกเขียน DB **หลัง** `chargeCard`/`shipOrder` (ADR บรรทัด 63–64) ถ้า pod crash หรือ DB ล่มหลังตัดเงินก่อน commit → ตัดเงินแล้วแต่ไม่มีทั้ง order และ event | ข้อความ "ไม่มีกรณีที่ตัดเงินแล้วแต่ event หาย" (บรรทัด 118) ไม่จริง; dual-write ย้ายไปอยู่ระหว่าง `paymentservice` กับ DB |
| **Risk** | Checkout DB ใหม่อยู่บน critical path ของ `PlaceOrder` (บรรทัด 67) | ADR ทำให้ RabbitMQ หลุดจาก critical path แต่เพิ่ม single point of failure ใหม่ที่ไม่มี HA/fallback ในแผน และไม่อยู่ใน failure injection (บรรทัด 130) |
| **Risk** | Rate limit 50/s "รวมทุก replica" (บรรทัด 69) ไม่มีกลไกจริง ถูกยกไปเป็น risk ค้าง (บรรทัด 124) | ถ้าสเกล `emailservice` ตาม backlog จะเกิน 50/s แล้วโดน MailFast throttle → retry storm |
| **Risk** | Payload ใส่ `email` และ `OrderResult` ให้ **ทุก** consumer รวม analytics (บรรทัด 66) | PII ไหลไปยังฝ่ายการตลาดและทุก consumer ในอนาคต; ไม่มี event schema/versioning สำหรับ SMS, fraud ในปีหน้า |
| **Sensitivity point** | ระยะเวลา peak: backlog อีเมลโต 150/s (200−50) | ตัวอย่าง **[Inferred]**: peak 10 นาที → backlog 90,000 ฉบับ ใช้เวลา drain ≥ 30 นาที; ไม่มี SLA ความล่าช้าของอีเมลให้ตัดสิน |
| **Sensitivity point** | Concurrency/prefetch ของ email consumer และ timeout ต่อ MailFast | ได้ 50/s ที่ p95 1.5 s ต้องมี in-flight ≥ ~75 calls **[Inferred, Little's law]**; timeout < 8 s → ~1 % ถูก retry → อีเมลซ้ำ (บรรทัด 123) |
| **Sensitivity point** | Poll interval และ throughput ของ outbox relay | งบ 60 s ของแต้มถูกแบ่งระหว่าง relay + broker + consumer; relay ต้องรับ ≥ 200 events/s ไม่งั้น outbox ค้าง |
| **Trade-off point** | Transactional outbox: reliability ↑ แต่ performance (DB write ใน `PlaceOrder`), availability (DB SPOF) และ operability (DB + relay + RabbitMQ = 3 component ใหม่) ↓ | ทีม 4 คนต้องดูแล stateful component เพิ่มสองตัวพร้อมกัน |
| **Trade-off point** | Async pub/sub: latency/availability ของ checkout ↑ แต่ความทันเวลาของอีเมลและ debuggability ↓ (eventual consistency, trace ข้าม AMQP — บรรทัด 116) | ต้องยอมรับอย่างชัดเจนว่าอีเมลช่วง 11.11 อาจล่าช้าหลายสิบนาที |
| **Non-risk** | Queue แยกต่อ action + DLQ + idempotent `loyaltyservice` ด้วย unique `order_id` (บรรทัด 68–70) | at-least-once + dedup ใน transaction เดียว = exactly once ตามที่ S3 ต้องการ; failure ของ action หนึ่งไม่ลามไปอีก action |
| **Non-risk** | RabbitMQ แทน Kafka ที่ 200 events/s | ตรงกับทีม (2 คนเคยใช้) และนโยบาย self-hosted; per-consumer rate limit/DLQ มีในตัว |

## Missing or weak

- **Options ที่ไม่ได้พิจารณา:**
  - RabbitMQ **ไม่มี outbox**: publisher confirms + retry ใน process + fallback log — ไม่ต้องเพิ่ม DB ให้ `checkoutservice` (ปัจจุบันไม่มี data store, บรรทัด 21)
  - **Outbox โดยไม่มี broker**: consumer ดึงจากตาราง outbox เอง — ลด component หนึ่งตัวที่ 200 events/s
  - เขียน order สถานะ `PENDING` **ก่อน** `chargeCard` แล้ว update + insert outbox หลังจ่าย เพื่อปิดช่อง charge↔DB
  - NATS JetStream (open-source, เก็บ/replay ได้) ในฐานะทางกลางระหว่าง RabbitMQ กับ Kafka
- **Consequences ที่ไม่ได้ระบุ:** HA ของ checkout DB และ RabbitMQ (quorum queues / operator); privacy ของ payload; ลำดับ event ไม่รับประกันเมื่อมีหลาย consumer (ยังไม่กระทบ action ปัจจุบัน แต่กระทบ fraud/recommendations)
- **Claims ที่อ่อนหรือผิด:**
  - บรรทัด 118 "ไม่มีกรณีที่ตัดเงินแล้วแต่ event หาย" — ผิด (ดู Risk แรก)
  - Validation บรรทัด 131 วัด "ภายใน 1 นาทีหลัง RabbitMQ กลับมา" — เปลี่ยนนิยาม requirement; `docs/scenario.md` กำหนด "within 1 minute **of the order**" ADR ต้องบอกว่า broker ล่ม > 60 s คือการละเมิด requirement ที่ยอมรับ
  - บรรทัด 67 "เขียน DB อยู่ในงบ ~200 ms" — ยังไม่วัด และไม่ได้ติด **[Inferred]**
  - Fraud scoring อยู่ใน driver 4 (บรรทัด 40) แต่บรรทัด 125 ยอมรับว่าอาจต้อง synchronous — ข้อกล่าวอ้างด้าน modifiability จึงครอบคลุมแค่ SMS/recommendations

## Verdict

**Accept with changes** — ทิศทาง (async pub/sub ผ่าน RabbitMQ, idempotent consumer) ถูกต้องและเป็นทางเดียวที่ผ่าน S1–S3 แต่การ์ด outbox ยังไม่ปิดช่องที่สำคัญที่สุด

1. **ปิดช่อง charge↔DB:** บันทึก order `PENDING` ก่อน `chargeCard` (หรือ reconcile กับ `transaction_id`) แล้วแก้ข้อความบรรทัด 118; หรือพิสูจน์ว่า outbox คุ้มกว่า publisher confirms + retry ที่ไม่ต้องเพิ่ม DB
2. **ระบุกลไก rate limit 50/s และ concurrency ของ email consumer** (เช่น single active consumer หรือ global limiter) พร้อม timeout/idempotency ต่อ MailFast และ SLA ความล่าช้าของอีเมลช่วง peak
3. **แก้ Validation ให้ตรง requirement:** วัดแต้มภายใน 60 s นับจากเวลา order, เพิ่ม failure injection ของ checkout DB และ relay, และกำหนด HA ของ DB/RabbitMQ (quorum queues) ก่อนช่วง 11.11
