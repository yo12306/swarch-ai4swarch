**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** Thanwarat Korcharoenkiat / 61010500
**agent ที่ใช้:** Claude Code (VS Code extension)
**โมเดล:** Claude Opus 5.5
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code (VS Code extension) + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run A, evidence: `demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` (`CLAUDE.md`/`AGENTS.md` โหลดอัตโนมัติเป็นกติกา ไม่ได้ใช้เป็นหลักฐาน) | ใช่ อ่านเฉพาะไฟล์ที่อนุญาต | 5 | ได้ 9 containers (1 ต่อ service ใน proto) แต่พิสูจน์ dependency ไม่ได้เลย แผนภาพจึงไม่มี `Rel`; ไม่มี data store หรือระบบภายนอก; ติด [Inferred] ที่ gRPC และ 1 service = 1 container; ระบุ 7 gaps เช่น checkout → payment/email/shipping และ frontend; agent ไม่ได้ตรวจว่า Mermaid แสดงได้; ตัวอย่าง `frontend.yaml (CART_SERVICE_ADDR)` ใน `AGENTS.md` เป็นคำใบ้ frontend → cart ที่ agent ทุกตัวเห็น; แผนภาพแสดงได้ (9 กล่อง ไม่มีลูกศร Customer ไม่เชื่อมกับอะไร) |
| 2 | 1-B | Claude Code (VS Code extension) + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `prompts/1-recover-architecture.md`, `demo.proto`, ใน `kubernetes-manifests/`: `kustomization.yaml`, `README.md` และ manifest 11 ไฟล์ (รวม `loadgenerator.yaml`); ลองเปิด `shoppingassistantservice.yaml` แต่ไม่มีไฟล์นี้ | ใช่ ทุกไฟล์อยู่ใน evidence | 5 | ได้ 12 containers (รวม `redis-cart` และ `loadgenerator`), 17 dependencies, 8 gaps; ไม่รู้ภาษาของ service เพราะ image เป็นชื่อ placeholder; เจอ `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่มี manifest จึงใส่ไว้ใน gaps; ดูรายชื่อไฟล์ในโฟลเดอร์ไม่ได้ จึงหาไฟล์จาก `kustomization.yaml` แทน; ตรวจใน Explorer แล้วโฟลเดอร์มี 13 ไฟล์ ตรงกับที่ agent อ่านครบ; แผนภาพแสดงได้ แต่ลูกศรและป้ายกำกับซ้อนกันจนอ่านยาก |
| 3 | 2-a | Claude Code (VS Code extension) + Claude Opus 5.5 | `prompts/2-write-adr.md` run a, `context: none`, R-12, `language: Thai` | `prompts/2-write-adr.md`, `docs/adr/template.md` (`CLAUDE.md`/`AGENTS.md` โหลดอัตโนมัติเป็นกติกา) | ใช่ ไม่ได้เปิด `docs/scenario.md` | 5 | พิจารณา 3 ทางเลือก (A sync, B pub/sub, C orchestrator) เลือก B: event `OrderPlaced` ผ่าน message broker; ไม่มีตัวเลขจาก scenario เลย ย้ายไปไว้ใน Open questions 12 ข้อ; ไม่เลือก broker (เสนอ RabbitMQ, Kafka, Pub/Sub ให้ตัดสินภายหลัง) และเสนอ broker แบบ managed เป็นทางแก้ความเสี่ยง; กันแต้มซ้ำด้วย `order_id` + unique constraint; พูดถึง rate limit ของ MailFast แค่ใน Risks ไม่มีตัวเลข; สมมติว่า `checkoutservice` เป็นผู้รับคำสั่งซื้อ (ติดป้าย Assumption); แผนภาพแสดงได้ |
| 4 | 2-b | Claude Code (VS Code extension) + Claude Opus 5.5 | `prompts/2-write-adr.md` run b, context: `scenario.md`, ADR-0001, `main.go`, R-12, `language: Thai` | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` (`CLAUDE.md`/`AGENTS.md` โหลดอัตโนมัติเป็นกติกา) | ใช่ อ่านเฉพาะ template และไฟล์ใน context | 5 | พิจารณา 4 ทางเลือก (sync gRPC, async ใน process, RabbitMQ, Kafka) เลือก RabbitMQ แบบ self-hosted; ใช้ตัวเลขจาก scenario ครบและคำนวณ backlog อีเมลช่วง peak; กันแต้มซ้ำด้วย idempotent consumer + `order_id`; ชี้ความเสี่ยง dual write เพราะ checkout ไม่มี database; อ้างบรรทัดใน `main.go` แทบทุกข้อ; Open questions 11 ข้อ; แผนภาพแสดงได้ |
| 5 | 3 | | `prompts/3-…` | | | | |
| 6 | | | | | | | |
| 7 | | | | | | | |
