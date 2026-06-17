# Central Store — คู่มือติดตั้ง (Google Sheets + Apps Script + GitHub Pages)

ระบบสั่งซื้อ/เบิกสโตร์กลาง แบบ "อนุมัติครั้งเดียวจบ" — เก็บข้อมูลใน Google Sheets, ลอจิกบัญชีอยู่ใน Apps Script, หน้าเว็บโฮสต์บน GitHub Pages ทั้งหมดอยู่บน free tier

## โครงสร้าง

```
GitHub Pages (index.html)  ──fetch──►  Apps Script Web App (Code.gs)  ──►  Google Sheets
   หน้าเว็บ                              API + engine บัญชี                ฐานข้อมูลจริง
```

หลักบัญชี: ซื้อเข้า = สินทรัพย์ (Dr สินค้าคงเหลือ) + VAT ซื้อ | เบิกออก = ค่าใช้จ่าย เข้า cost center | ของสด = ตัดจ่ายตรง | ต้นทุน = ถัวเฉลี่ยถ่วงน้ำหนัก

---

## ส่วนที่ 1 — Backend (Google Sheets + Apps Script)

1. สร้าง Google Sheet ใหม่ (sheets.new) ตั้งชื่ออะไรก็ได้ เช่น `Central Store DB`
2. เมนู **Extensions → Apps Script**
3. ลบโค้ดเดิมใน `Code.gs` ออกทั้งหมด แล้ววางเนื้อหาจากไฟล์ `Code.gs` ลงไป → **Save**
4. ที่แถบฟังก์ชันด้านบน เลือก `setup` แล้วกด **Run** (ครั้งแรกจะให้ Authorize — กดอนุญาตด้วยบัญชี Google ของคุณ)
   - กลับไปดู Google Sheet จะเห็นชีท Items, Journal, AuditLog ฯลฯ พร้อมข้อมูลตัวอย่าง
5. กด **Deploy → New deployment**
   - ไอคอนเฟือง → เลือก **Web app**
   - Execute as: **Me**
   - Who has access: **Anyone**
   - กด **Deploy** → คัดลอก **Web app URL** (ลงท้าย `/exec`) เก็บไว้

> หมายเหตุ: ทุกครั้งที่แก้ `Code.gs` ต้อง **Deploy → Manage deployments → Edit → Version: New version** เพื่อให้ URL เดิมใช้โค้ดใหม่

---

## ส่วนที่ 2 — Frontend (GitHub Pages)

1. สร้าง repo ใหม่บน GitHub เช่น `central-store`
2. อัปโหลดไฟล์ `index.html` เข้า repo (root)
3. ไปที่ **Settings → Pages → Build and deployment → Source: Deploy from a branch**
   - Branch: `main` / folder: `/ (root)` → Save
4. รอสักครู่ เปิด URL ที่ได้ เช่น `https://<user>.github.io/central-store/`
5. หน้าแรกจะให้วาง **Web app URL** จากส่วนที่ 1 → กด **เชื่อมต่อ**

เสร็จแล้ว — กดทดลอง flow ได้เลย (ข้อมูลจะถูกเขียนกลับเข้า Google Sheets จริง)

---

## การทดลอง

- **ภาพรวม**: มี PO ค้างอนุมัติ 1 ใบ (เกินเกณฑ์ ฿5,000) กดอนุมัติครั้งเดียว → ของเข้าคลัง + VAT + ลงบัญชี อัตโนมัติ
- **สั่งซื้อ**: ยอดไม่เกินเกณฑ์ = อนุมัติเอง / เลือกของสด = บังคับเลือกแผนก ตัดจ่ายตรง
- **เบิก**: เลือกแผนก + ของ → ตัดสต็อกทันที ไม่ต้องอนุมัติ ต้นทุนเข้า cost center
- **คลัง**: นับสต็อกสิ้นเดือน → ระบบลงผลต่าง (variance) ให้
- **บัญชี**: สมุดรายวัน (Dr/Cr) · ต้นทุนตามแผนก · audit trail
- ปุ่มเฟือง (มุมขวาบน) → เปลี่ยน URL หรือ **รีเซ็ตข้อมูล** กลับเป็นชุดตัวอย่าง

---

## API (สำหรับต่อยอด เช่น เชื่อม LINE OA)

ทุกคำสั่งเป็น `POST` body เป็น JSON (ใช้ `Content-Type: text/plain` เพื่อเลี่ยง CORS preflight)

| action | payload | ผล |
|---|---|---|
| `state` | — | คืนสถานะทั้งหมด |
| `submitPO` | `{payload:{supplier, department, date, lines:[{itemId,qty,unitPrice}]}}` | ตั้ง PO (auto-post ถ้า ≤ เกณฑ์) |
| `approvePO` | `{no, approver}` | อนุมัติ + รับเข้าคลัง |
| `rejectPO` | `{no}` | ปฏิเสธ |
| `issue` | `{payload:{department, date, lines:[{itemId,qty}]}}` | เบิก + ตัดสต็อก |
| `count` | `{itemId, counted}` | ปรับผลนับ |
| `setThreshold` | `{value}` | ตั้งวงเงินอนุมัติอัตโนมัติ |
| `reset` | — | รีเซ็ตข้อมูลตัวอย่าง |

ทุก action (ยกเว้น state) คืนสถานะใหม่ทั้งก้อนกลับมา

---

## ข้อควรรู้ก่อนใช้จริง

- ต้นแบบยุบขั้น "อนุมัติ = รับของ" เป็นก้าวเดียว ของจริงควรมีขั้นรับของ (GRN) แยก เผื่อรับไม่ครบ
- เลือกวิธีตีราคา (ที่นี่ใช้ถัวเฉลี่ยถ่วงน้ำหนัก) ให้คงที่ตลอดงวด
- VAT ซื้อบันทึกที่ตอนซื้อตามใบกำกับ ไม่ใช่ตอนเบิก
- การใช้ digital approval แทนลายเซ็นกระดาษ ควรเขียน internal control เป็นลายลักษณ์อักษรและเคลียร์กับผู้สอบบัญชีก่อน
