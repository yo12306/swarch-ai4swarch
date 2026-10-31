**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `checkoutservice` → `productcatalogservice` | `checkoutservice.yaml` · `PRODUCT_CATALOG_SERVICE_ADDR` (L57–58) | ✅ | yaml L57–58 ตรงตามที่ AI อ้าง (`productcatalogservice:3550`); `main.go` L112 อ่านค่า, L119 เชื่อมต่อ gRPC, L341 สร้าง `ProductCatalogServiceClient`, L344 เรียก `GetProduct` |
| `checkoutservice` → `shippingservice` | `checkoutservice.yaml` · `SHIPPING_SERVICE_ADDR` (L59–60) | ✅ | yaml L59–60 ตรง (`shippingservice:50051`); `main.go` L111 อ่านค่า, L118 เชื่อมต่อ gRPC, L314–315 เรียก `GetQuote`, L387 เรียก `ShipOrder` |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` · `PAYMENT_SERVICE_ADDR` (L61–62) | ✅ | yaml L61–62 ตรง (`paymentservice:50051`); `main.go` L116 อ่านค่า, L123 เชื่อมต่อ gRPC, L370 เรียก `Charge` |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` · `EMAIL_SERVICE_ADDR` (L63–64) | ✅ | yaml L63–64 ตรง (`emailservice:5000`); `main.go` L115 อ่านค่า, L122 เชื่อมต่อ gRPC, L380 เรียก `SendOrderConfirmation` |
| `checkoutservice` → `currencyservice` | `checkoutservice.yaml` · `CURRENCY_SERVICE_ADDR` (L65–66) | ✅ | yaml L65–66 ตรง (`currencyservice:7000`); `main.go` L114 อ่านค่า, L121 เชื่อมต่อ gRPC, L360 เรียก `Convert` |
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` · `CART_SERVICE_ADDR` (L67–68) | ✅ | yaml L67–68 ตรง (`cartservice:7070`); `main.go` L113 อ่านค่า, L120 เชื่อมต่อ gRPC, L325 เรียก `GetCart`, L333 เรียก `EmptyCart` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): 
  - dependency ที่ไม่มีจริง: ไม่พบใน 6 แถวของ checkout ที่ตรวจ (✅ ทั้งหมด)
  - dependency ที่ขาดไป: ไม่มีในค่าตั้งปัจจุบัน แต่แผนภาพไม่แสดง 2 ตัวที่ทำงานเมื่อเปิดใช้เท่านั้น: OpenTelemetry collector (`ENABLE_TRACING=1`) และ Google Cloud Profiler (`ENABLE_PROFILER=1`) AI มองไม่เห็นเพราะไม่มี `main.go`
  - container ที่น่าสงสัย: `loadgenerator` เป็นเครื่องมือทดสอบที่ถูก comment ออกจาก `kustomization.yaml` ไม่ใช่ส่วนของระบบจริง แต่ AI ใส่ไว้ในแผนภาพ ควรตัดออก; `shoppingassistantservice` AI ทำถูกแล้วที่ไม่ใส่ในแผนภาพ (ไม่มี manifest และ component ถูก comment ไว้)


## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
  **ตอบ:** สิ่งที่ A ไม่รู้: ใครเรียกใคร, frontend ที่เป็นทางเข้า, redis-cart, port เพราะไฟล์ `.proto` เก็บแค่ interface ของแต่ละ service
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
  **ตอบ:** loadgenerator อยู่ในแผนภาพ ทำให้คิดว่าเป็นส่วนหนึ่งของระบบจริงที่ส่ง traffic เข้า frontend, สมาชิกใหม่อาจนำมันขึ้น production ไปด้วย ทำให้มี traffic ปลอมเข้าระบบ