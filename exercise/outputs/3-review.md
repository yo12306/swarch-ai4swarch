> รีวิวโดย agent: Claude Code (ส่วนขยาย VS Code) · model: Claude Opus 5.5 (`claude-opus-5-5`)

# ATAM review: ADR-0002 ใช้ RabbitMQ สำหรับงานหลังการสั่งซื้อ

**หลักฐาน:** อ่านเฉพาะ `docs/adr/0002-use-rabbitmq-for-post-order-events.md` (อ้างเป็น `ADR:บรรทัด`) และ `docs/scenario.md` (อ้างเป็น `scenario:บรรทัด`) ตามขอบเขตของ task จึงไม่ได้ตรวจว่าการอ้างถึง `main.go` ใน ADR ตรงกับโค้ดหรือไม่ · ADR นี้ร่างโดย agent และ model เดียวกับผู้รีวิว (ADR:1) จึงอาจมีจุดบอดร่วมกัน

## Quality-attribute scenarios

- **S1 Performance.** Source: ลูกค้า · Stimulus: `PlaceOrder` 200 ครั้ง/s (scenario:25) · Environment: ช่วง 11.11 ขณะ MailFast ตอบ p95 1.5 s, p99 8 s (scenario:26) · Response: ตัดเงิน ส่งของ publish `OrderPlaced` แล้วตอบทันที โดยไม่รอ email, loyalty, analytics · Response measure: p95 ของ `PlaceOrder` ≤ 500 ms (วันนี้ ~300 ms, scenario:24) และไม่มีการเรียก MailFast ใน request path
- **S2 Reliability.** Source: infrastructure · Stimulus: RabbitMQ ใช้ไม่ได้ทั้ง cluster 60 s และ pod `checkoutservice` หนึ่งตัว restart ระหว่างนั้น · Environment: peak 200 orders/s · Response: order ยังสำเร็จ; event ที่ broker ยังไม่ confirm ถูกเก็บแบบถาวรแล้วส่งต่อเมื่อ broker กลับมา · Response measure: order ที่ล้มเพราะ broker = 0; event หาย = 0 จาก 12,000 order (200 × 60); ไม่มี `order_id` ใดได้แต้มเกิน 1 ครั้ง (scenario:27-28)
- **S3 Scalability.** Source: การขาย 11.11 · Stimulus: order 200/s ต่อเนื่อง T นาที (ยังไม่รู้ค่า T, ADR:155) · Environment: MailFast รับได้ 50 emails/s และล้มเหลว ~0.5% (scenario:26) · Response: อีเมลพักใน queue ส่งรวมทุก replica ≤ 50/s พร้อม retry; loyalty ไม่ได้รับผลกระทบ · Response measure: rate-limit error = 0; อายุข้อความใน `loyalty.award-points` < 60 s ตลอด peak; publisher ไม่ถูก block; อีเมลสุดท้ายของ peak ช้า ≈ 3T (ค่าสูงสุดที่ธุรกิจรับได้: Not evidenced in the provided files, ADR:156)

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | **R1** event ที่ broker ไม่ confirm ไปค้างใน retry buffer ในหน่วยความจำของ `checkoutservice` (ADR:76) ซึ่งเป็นจุดอ่อนเดียวกับที่ใช้ตัด Option 2 (ADR:65); ไม่ระบุขนาด buffer และพฤติกรรมเมื่อเต็ม และการ drain ตอน SIGTERM (ADR:134) ช่วยไม่ได้ถ้า broker ยังล่ม | broker ล่ม 1 นาทีที่ peak = 12,000 event ใน memory; ถ้า pod restart หรือ buffer เต็ม อีเมล แต้ม และ analytics จะหาย เหลือแค่ ERROR log ที่ไม่มีขั้นตอน replay จึงขัด exactly once (scenario:27) |
| Risk | **R2** backlog อีเมลที่โตขึ้น 150 ฉบับ/s (ADR:34) อยู่ใน cluster เดียวกับ queue ของ loyalty; ADR:137 กล่าวถึง disk เต็มแต่ไม่ระบุผลที่ตามมา คือ disk/memory alarm ของ RabbitMQ จะ block publisher ทั้ง cluster **[Inferred]** | ปัญหาฝั่งอีเมลกลายเป็น publish timeout ที่ checkout แล้วไหลเข้า R1 backlog อีเมลจึงทำให้แต้มหายได้ ขัดกับที่ ADR:66 บอกว่า backlog อีเมล "ไม่ถ่วงแต้ม" |
| Risk | **R3** ผู้ดูแลและ on-call ของ cluster stateful ใหม่มีแค่ 2 คนที่ "เคยใช้" RabbitMQ (ADR:83, scenario:29); ไม่มีแผน cutover และ rollback; ADR ลงวันที่ 2026-10-31 คือ 11 วันก่อน 11.11 (ADR:6) และยังไม่รู้ว่าต้องใช้งานทันหรือไม่ (ADR:163) | ถ้าต้องทัน ทีม 4 คนต้องตั้ง broker สร้าง `loyaltyservice` กับ analytics consumer ปรับ `emailservice` และ load test ใน 11 วัน; on-call 2 คนเป็นจุดล้มเดียวด้านคนในวัน peak (driver 6, ADR:49) |
| Non-risk | publish หนึ่งครั้งเป็นขั้นสุดท้ายหลังตัดเงินและส่งของ แทนการเรียก `emailservice` แบบ synchronous (ADR:73-75) | latency ของ MailFast ออกจาก path ของเป้า ≤ 500 ms (S1) และไม่มี event ของ order ที่ล้ม จึงไม่ต้องมี compensation ของแต้มหรืออีเมล |
| Non-risk | `loyaltyservice` บันทึก `order_id` (unique constraint) และเพิ่มแต้มใน transaction เดียว commit ก่อน ack และมี queue ของตัวเอง (ADR:81) | at-least-once + idempotent consumer กันแต้มซ้ำได้จริง โดยมีเงื่อนไขว่า datastore ที่ยัง "Not decided" (ADR:102) ต้องรองรับ transaction และ unique constraint |
| Non-risk | เลือก RabbitMQ แทน Kafka (ADR:86) | งานนี้เป็น task queue ที่ต้อง ack, retry, DLQ และคุมอัตรารายข้อความที่แค่ 200 msg/s และทีมมีคนเคยใช้ RabbitMQ (scenario:29); สิ่งที่เสียคือ replay ซึ่ง R-12 ไม่ได้กำหนด (scenario:9-14) |
| Sensitivity point | อัตราส่งรวมของ `emailservice` = จำนวน replica × limit ต่อ replica; ADR:80 ไม่ระบุว่า rate limiter ของแต่ละ replica ประสานกันอย่างไร (prefetch ช่วยไม่ได้เพราะจำกัดจำนวนข้อความค้าง ไม่ใช่อัตรา) ซึ่งเป็นปัญหาเดียวกับที่ใช้ตัด Option 2 (ADR:65) | ถ้า autoscale `emailservice` ตาม queue depth ช่วง 11.11 อัตรารวมจะเกิน 50/s; ถ้า rate-limit error ถูกนับเป็น retry อีเมลจำนวนมากจะไหลเข้า DLQ (ADR:138) |
| Sensitivity point | ความยาว peak T (นาที) และอัตรา order หลัง peak r: backlog เพิ่ม 9,000 ฉบับต่อนาทีของ peak, อีเมลสุดท้ายของ peak ช้า ≈ 3T และ queue ว่างหลังจบ peak 150T ÷ (50 − r) นาที | ถ้า r ≥ 50 orders/s backlog จะไม่ลดลงเลย; T กำหนดทั้งความล่าช้าของอีเมลและ disk ของ broker (R2) |
| Trade-off point | deadline ของ publisher confirm (ADR:75) | สั้นไป: broker ช้าชั่วครู่ก็ทำให้ event ตกไปที่ R1 (reliability ลดลง); ยาวไป: p95 ของ `PlaceOrder` เข้าใกล้ 500 ms (performance ลดลง) |
| Trade-off point | event พกข้อมูลครบ เพราะไม่มี order database ให้ consumer ดึงเพิ่ม (ADR:78) | consumer ไม่ต้องเรียกกลับมาที่ checkout (availability ดีขึ้น) แต่ email และที่อยู่ไปอยู่ทุก queue, DLQ และ ERROR log โดยไม่มี retention หรือสิทธิ์เข้าถึง (ADR:76, ADR:132; security ลดลง) และงานที่ต้องใช้ field ใหม่ต้องแก้ `checkoutservice` (modifiability ลดลง) |

## Missing or weak

- **Options not considered:**
  - order store + transactional outbox ในฐานะ option ของ ADR นี้ ไม่ใช่เลื่อนไป ADR ถัดไป (ADR:134): เป็นทางที่ตอบ S2 ได้ตรงที่สุด แลกกับ datastore ใหม่ใน critical path ของ checkout
  - Redis Streams บน Redis ที่ ADR-0001 ใช้เก็บ cart (รู้จากชื่อไฟล์ที่ ADR:9 อ้าง; ทีมดูแล Redis อยู่แล้ว **[Inferred]**): ไม่เพิ่ม cluster ใหม่ให้ทีม 4 คน แต่ต้องสร้าง retry, DLQ และ rate limit เอง
- **Consequences not stated:** ระหว่าง rolling deploy pod `checkoutservice` รุ่นเก่ายังเรียก `emailservice` ทาง gRPC ถ้า `emailservice` เลิกรับ gRPC ก่อน อีเมลจะหายเงียบเพราะ error ถูกแค่ log (ADR:21); และไม่มีแผน rollback ถ้า broker มีปัญหาช่วง 11.11
- **Unsupported or likely wrong claims:**
  - "Option 3 ผ่าน driver 1–5" (ADR:86) และ "broker ล่มนานกว่า 1 นาที แต้มจะไม่หาย" (ADR:136) ขัดกับ R1; OQ4 (ADR:158) ถามว่ายอมให้ event หายได้ไหม ทั้งที่อย่างน้อยสำหรับแต้ม scenario:27 และ driver 3 (ADR:46) ตอบไว้แล้วว่าไม่ได้
  - "งานใหม่แค่ bind queue โดยไม่แก้ `checkoutservice`" (ADR:84, ADR:127): SMS ต้องใช้เบอร์โทรซึ่งไม่ปรากฏใน field ที่ ADR:78 ระบุ และ fraud scoring อาจต้องบล็อก order (ADR:161) จึงอาจต้องแก้ checkout ใน 2 จาก 3 งานที่ marketing คาดไว้ (scenario:14)
  - ใช้ idempotency key แทน `order_id` (ADR:140) กันแต้มซ้ำได้ แต่ไม่กันตัดเงินซ้ำ เว้นแต่ checkout เก็บ key ไว้ (ซึ่งไม่มี datastore, ADR:22) หรือ `paymentservice` dedup ได้ (Not evidenced in the provided files); ทั้งยังเปลี่ยน contract ของ `PlaceOrder` จึงควรเป็น ADR แยก
- **Validation:** failure injection หยุด RabbitMQ แค่ 1 node (ADR:147) จึงไม่ครอบคลุม S2; loyalty วัดอายุข้อความใน queue (ADR:146) แทนเวลาจากสั่งซื้อถึงได้แต้ม ทั้งที่ event มีเวลาที่สั่งอยู่แล้ว (ADR:78); และไม่ระบุว่า load test ยาวเท่า peak จริง จึงอาจไม่เห็น R2

## Verdict

**Accept with changes.** ทิศทาง asynchronous และการเลือก RabbitMQ แทน Kafka มีเหตุผล แต่หลายปัญหามีรากเดียวกันคือ checkout ไม่มี order store (ADR:22): จึงไม่มี outbox (R1), ต้องใช้ event พกข้อมูลครบ (PII และ modifiability) และ idempotency key ไม่มีที่เก็บ (ADR:140)

1. **ปิดช่อง event หาย หรือให้ธุรกิจยอมรับความเสี่ยงเป็นลายลักษณ์อักษร:** เลือก outbox (หรือ spool บน persistent volume) หรือยอมรับความเสี่ยง; ระบุขนาด buffer และพฤติกรรมเมื่อเต็ม; แก้ข้อความใน ADR:86 และ ADR:136; เพิ่ม test ตาม S2 ที่มีเป้า event หาย = 0
2. **กัน peak ของอีเมลไม่ให้ลามไป loyalty และ checkout:** กำหนดกลไก limit 50/s รวมทุก replica (ไม่ autoscale ตาม queue depth, rate-limit error ไม่นับเป็น retry), คำนวณ disk ของ broker จาก T และขนาด event จริงพร้อม alert ก่อนถึง alarm และตกลงความล่าช้าสูงสุดของอีเมลกับธุรกิจ (ADR:155-156)
3. **วางแผน rollout และ ownership:** ลำดับ cutover (`emailservice` รับ gRPC ต่อจนทุก pod ของ checkout publish แล้ว), แผน rollback ช่วง 11.11, on-call ที่ไม่ฝากไว้กับ 2 คน และวันใช้งานจริงเทียบกับ 11.11 (ADR:163)
