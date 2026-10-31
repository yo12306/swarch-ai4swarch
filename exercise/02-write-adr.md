**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย เพราะไม่ได้อ่าน scenario ตัวเลขที่ต้องใช้ไปอยู่ใน Open questions | ใช้ครบ: p95 ≤ 500 ms (ปัจจุบัน ~300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / ล้มเหลว 0.5% / 50 อีเมล/s, แต้มภายใน 1 นาที และคำนวณต่อ เช่น backlog อีเมล 9,000 ฉบับต่อนาทีของ peak |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: A sync RPC, B pub/sub ผ่าน broker (เลือก), C orchestrator | 4 ทาง: sync gRPC, async ใน process, RabbitMQ pub/sub (เลือก), Kafka; ตัด managed cloud ออกตั้งแต่ต้นตาม scenario |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: idempotent ด้วย `order_id` + unique constraint; rate limit: พูดแค่ใน Risks (retry, backoff, circuit breaker) ไม่มีตัวเลข | แต้ม: at-least-once + idempotent consumer (บันทึก `order_id` และให้แต้มใน transaction เดียว commit ก่อน ack) และมี queue แยกจึงไม่ติด backlog อีเมล; rate limit: consumer จำกัดรวมทุก replica ≤ 50/s และให้ queue พัก backlog ช่วง peak |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่แต่งตัวเลข แต่มีข้อเสนอที่ขัดกับ scenario ซึ่ง run a ไม่ได้เห็น: ยก Google Cloud Pub/Sub เป็นตัวเลือก และแก้ความเสี่ยงเรื่องทีมไม่มีประสบการณ์ด้วย broker แบบ managed (scenario ให้ใช้ self-hosted และบอกว่า 2 คนเคยใช้ RabbitMQ) | ไม่พบตัวเลขที่แต่งขึ้น ตัวเลขมาจาก scenario หรือคำนวณต่อ; มีค่าที่ agent เลือกเอง เช่น RabbitMQ 3 nodes, ชื่อ queue/exchange และ ~75 คำขอพร้อมกัน (ติด [Inferred]); Locust `loadgenerator` มาจาก ADR-0001 (L49) ไม่ได้แต่ง |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | บรรทัดแรก: เปลี่ยน "เขียนโดย agent" เป็น "ร่างโดย agent ... แก้ไขและรับรองโดย <ชื่อ>" | ให้ชัดว่า agent เป็นผู้ร่าง ส่วนผู้แก้ไขและรับผิดชอบ ADR ฉบับสุดท้ายคือ Thanwarat |
| 2 | ตอบ Open question ข้อ 10 เรื่องผู้ดูแล: ให้ 2 คนที่เคยใช้ RabbitMQ เป็นผู้ดูแลและ on-call ของ broker (เพิ่มในหัวข้อ Decision) | ทีมมี 4 คน และมีแค่ 2 คนที่เคยใช้ RabbitMQ จึงเหมาะเป็นเจ้าของระบบใหม่ |
| 3 | ย้าย Open question ข้อ 8 ไปไว้ใน Risks and mitigations | การ retry `PlaceOrder` แล้วได้ `order_id` ใหม่ ทำให้ตัดเงินและให้แต้มซ้ำ |

