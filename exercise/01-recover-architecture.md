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
| frontend → cartservice | frontend.yaml (CART_SERVICE_ADDR) | ✅ | เจอใน frontend.yaml บรรทัด 73 และใน rpc.go บรรทัด 58 มีการเรียก GetCart จริง |
| checkoutservice → paymentservice | checkoutservice.yaml (PAYMENT_SERVICE_ADDR) | ✅ | เจอใน checkoutservice.yaml บรรทัด 61 และใน main.go บรรทัด 370 มีการเรียก Charge ตอนตัดเงินจากบัตร |
| checkoutservice → emailservice | checkoutservice.yaml (EMAIL_SERVICE_ADDR) | ✅ | เจอใน checkoutservice.yaml บรรทัด 63 และใน main.go บรรทัด 380 มีการเรียก SendOrderConfirmation ตอนส่งอีเมลยืนยัน |
| recommendationservice → productcatalogservice | recommendationservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR) | ✅ | เจอใน recommendationservice.yaml บรรทัด 60 และใน recommendation_server.py บรรทัด 73 มีการเรียก ListProducts ถ้าไม่ตั้งค่านี้ service จะไม่ยอมเริ่มทำงานเลย (บรรทัด 132) |
| cartservice → redis-cart | cartservice.yaml (REDIS_ADDR) | ✅ | เจอใน cartservice.yaml บรรทัด 50 และใน Startup.cs บรรทัด 34 ใช้ Redis ถ้ามีค่านี้ แต่จริง ๆ โค้ดเลือกได้หลายแบบ ไม่ได้ใช้ Redis อย่างเดียว |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - ใน 5 เส้นที่ตรวจ ไม่เจอเส้นที่ผิด ทุกเส้นมีทั้งใน config และในโค้ดจริง
  - loadgenerator ไม่ควรอยู่ในแผนภาพ เพราะเป็นแค่ตัวทดสอบ และถูก comment ออกใน kustomization.yaml อยู่แล้ว
  - shoppingassistantservice หายไป ทั้งที่มีโค้ดใน src และ frontend ก็ตั้งค่าที่อยู่ไว้ ควรใส่เป็นส่วนเสริม
  - แผนภาพบอกว่า cartservice ใช้ Redis อย่างเดียว แต่ในโค้ดเลือกได้ทั้ง Redis, Spanner, AlloyDB หรือเก็บใน memory

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   ถ้ามีแค่ .proto agent รู้แค่ว่ามี service อะไรบ้างและแต่ละตัวทำอะไรได้ แต่ไม่รู้ว่าใครเรียกใคร ไม่รู้ว่ามี frontend และไม่รู้ว่ามี Redis เพราะ .proto เป็นแค่ข้อตกลงของ interface ไม่ได้บอกว่าตอนใช้งานจริงระบบต่อกันยังไง แผนภาพของ Run A เลยมีแต่กล่อง ไม่มีเส้นเชื่อมเลย

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   ข้อที่น่าจะทำให้เข้าใจผิดที่สุดคือแผนภาพบอกว่า cartservice ใช้ Redis อย่างเดียว คนเข้าทีมใหม่อาจคิดว่าระบบต้องมี Redis เสมอ แล้วไปออกแบบหรือแก้ปัญหาตามนั้น ทั้งที่จริงในโค้ดเปลี่ยนไปใช้ฐานข้อมูลอื่นได้
