# บันทึกผลการทดสอบ Attack Scenario — สัปดาห์ที่ 4

> เนื้อหานี้มาจากการรันเซิร์ฟเวอร์และยิงคำขอทดสอบจริงกับโค้ดในโปรเจกต์นี้ (REST endpoints ใน `index.js`)
> ก่อนส่งงาน ให้แนบภาพหน้าจอ Postman ของตนเองประกอบ และปรับข้อสังเกตให้เป็นสำนวนของตัวเอง

## 1. ทดสอบ Payload ขนาดใหญ่ผิดปกติ (ขั้นตอนที่ 3.4)

ผลที่ได้: 413 Payload Too Large

**วิธีทดสอบ**: ส่ง `POST /api/v1/students` โดยให้ฟิลด์ `name` ยาว 15,000 ตัวอักษร (เกิน limit 10kb ที่ตั้งไว้ใน `express.json({ limit: "10kb" })`)

1. สร้าง Request: POST http://localhost:3000/api/v1/students
2. ไปแท็บ Scripts (หรือ Pre-request Script) ของ request นี้ ใส่:

```json
   pm.variables.set("longName", "x".repeat(15000));
```

3. ไปแท็บ Body → raw → JSON ใส่:

```json
{
  "name": "{{longName}}",
  "major": "วิทยาการคอมพิวเตอร์",
  "email": "big@example.com"
}
```

4. กด Send → ควรได้ 413 Payload Too Large
   **ผลที่ได้**: `413 Payload Too Large`

```json
{
  "error": { "code": "entity.too.large", "message": "request entity too large" }
}
```

**ข้อสังเกต**:

- error message ที่ได้ (entity.too.large) มาจากตัวไลบรารีเองโดยตรง ไม่ได้ผ่านการปรับรูปแบบให้ตรงกับ code แบบอื่นในระบบ (เช่น VALIDATION_ERROR) และ limit นี้ป้องกันได้แค่คำขอเดียวที่ใหญ่เกินไป ไม่ได้ป้องกันการยิงถี่ ๆ จำนวนมาก ต้องใช้ rate limiting เพิ่ม

## 2. ทดสอบ Type Confusion (ขั้นตอนที่ 3.5)

**วิธีทดสอบ**: ส่ง `POST /api/v1/students` โดยให้ `name` เป็นอ็อบเจกต์ `{ "malicious": "payload" }` แทน string

**ผลที่ได้**: `201 Created` — ระบบยอมรับข้อมูลเข้าไปเก็บใน array `students` ทั้งที่ `name` ไม่ใช่ string

```json
{
  "message": "เพิ่มข้อมูลสำเร็จ",
  "data": {
    "id": 3,
    "name": {
      "malicious": "payload"
    },
    "major": "วิทยาการคอมพิวเตอร์",
    "email": "typeconf@example.com"
  }
}
```

### ขั้นตอนการทดสอบ

1. Request เดิม เปลี่ยน Body เป็น:
   ```json
   {
     "name": { "malicious": "payload" },
     "major": "วิทยาการคอมพิวเตอร์",
     "email": "typeconf@example.com"
   }
   ```
2. กด Send → ควรได้ `201 Created` (ระบบยอมรับข้อมูลที่ผิดชนิดเข้าไปเก็บ — นี่คือช่องโหว่ที่ต้องสังเกต)

**ข้อสังเกต**:

- โค้ดเช็คแค่ if (!name) คือดูว่ามีค่าหรือไม่ ไม่ได้เช็คว่าเป็น string จริง object จึงผ่านเข้ามาได้ ถือเป็นช่องโหว่ด้าน input validation ถ้าจะแก้ต้องเพิ่มการเช็ค typeof หรือใช้ schema validation ในอนาคต

## สรุป

ทั้งสองกรณีชี้ให้เห็นข้อจำกัดเดียวกันคือ **middleware ระดับ HTTP (Helmet, CORS, body-size limit) ป้องกันได้เฉพาะภัยคุกคามระดับ transport** แต่ไม่ได้ป้องกัน "ตรรกะ" ของแอปพลิเคชันเอง ซึ่งยังต้องเขียน validation logic เพิ่มเติมในแต่ละ route — สะท้อนหลักการ defense-in-depth ที่ต้องป้องกันหลายชั้นร่วมกัน ไม่ใช่พึ่งพา middleware ชั้นเดียว
