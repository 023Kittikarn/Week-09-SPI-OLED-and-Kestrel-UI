# รายงานการทดลองที่ 9.2 (Lab 9.2)
## ขั้นตอนการตรวจสอบเชิงนิติวิทยาศาสตร์ (HTTP Payload Forensics) และผลการประเมิน

---

## 1. บันทึกผลกิจกรรมนิติวิทยาศาสตร์ 2.1: ทดสอบเรียกใช้งาน API ครบทั้ง 3 รูปแบบ

### 1. ตรวจสอบ Telemetry ปัจจุบัน (HTTP GET)
**คำสั่ง cURL:**
```powershell
curl.exe -i -X GET http://localhost:5117/api/telemetry
ผลลัพธ์ Raw Response จาก Kestrel Server:

HTTP
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Thu, 29 Oct 2026 16:00:00 GMT
Server: Kestrel

{"raw":2048,"calibrated":50.0,"unit":"%","displayMsg":"ID: 67030023","timestamp":"2026-10-29T16:00:00Z"}
2. ทำการ Calibrate เซนเซอร์ใหม่ (HTTP POST พร้อม JSON Body)
คำสั่ง cURL:

PowerShell
curl.exe -i -X POST http://localhost:5117/api/potentiometer/calibrate `
  -H "Content-Type: application/json" `
  -d '{"rawMin": 200, "rawMax": 3800, "scaleMin": 0, "scaleMax": 1000, "unit": "RPM"}'
ผลลัพธ์ Raw Response จาก Kestrel Server:

HTTP
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Server: Kestrel

{"status":"success","settings":{"rawMin":200,"rawMax":3800,"scaleMin":0,"scaleMax":1000,"unit":"RPM"}}
3. ส่งข้อความใหม่ไปแสดงบนหน้าจอ OLED (HTTP POST)
คำสั่ง cURL:

PowerShell
curl.exe -i -X POST http://localhost:5117/api/oled/message `
  -H "Content-Type: application/json" `
  -d '{"message":"ID: 67030023"}'
ผลลัพธ์ Raw Response จาก Kestrel Server:

HTTP
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Server: Kestrel

{"status":"success","current":"ID: 67030023"}
2. บันทึกผลกิจกรรมนิติวิทยาศาสตร์ 2.2: Fault Injection & Vulnerability Probe
1. ทดสอบป้อนค่าสเกลผิดตรรกะ (rawMin > rawMax)
คำสั่ง cURL:

PowerShell
curl.exe -i -X POST http://localhost:5117/api/potentiometer/calibrate `
  -H "Content-Type: application/json" `
  -d '{"rawMin": 4000, "rawMax": 1000, "scaleMin": 0, "scaleMax": 100, "unit": "%"}'
ผลลัพธ์ Raw Response จาก Kestrel Server:

HTTP
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Server: Kestrel

{"error":"Bad Request","message":"RawMax ต้องมีค่ามากกว่า RawMin เสมอ!"}
วิเคราะห์ผล: เซิร์ฟเวอร์ตอบกลับด้วย 400 Bad Request พร้อมแจ้งข้อผิดพลาดอย่างถูกต้อง โดยที่เซิร์ฟเวอร์ Kestrel ไม่หยุดทำงาน (No Server Crash)

2. ทดสอบส่งข้อความว่างเปล่า ("")
คำสั่ง cURL:

PowerShell
curl.exe -i -X POST http://localhost:5117/api/oled/message `
  -H "Content-Type: application/json" `
  -d '{"message":""}'
ผลลัพธ์ Raw Response จาก Kestrel Server:

HTTP
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Server: Kestrel

{"error":"Bad Request","message":"ข้อความต้องไม่เป็นค่าว่างเปล่า!"}
วิเคราะห์ผล: ระบบดักจับข้อผิดพลาดและปฏิเสธข้อมูลที่ไม่สมบูรณ์ได้อย่างปลอดภัย

3. คำตอบคำถามท้ายการทดลองเพื่อการประเมินผล (Review Questions)
ข้อ 1: เหตุใดการคำนวณสเกลเซนเซอร์จึงควรทำที่ฝั่ง Kestrel Server แทนที่จะคำนวณบนไมโครคอนโทรลเลอร์ ESP32 ตั้งแต่แรก?
คำตอบ:

ลดภาระของ Edge Device (ESP32): ESP32 มีทรัพยากรประมวลผลและหน่วยความจำจำกัด การให้ ESP32 ทำหน้าที่เพียงอ่านและส่งค่าสัญญาณดิบ (Raw Sensor Value) จะช่วยประหยัดเวลาและพลังงานของตัวอุปกรณ์

ความยืดหยุ่นและการบริหารจัดการศูนย์กลาง (Centralized Calibration): หากต้องการปรับเปลี่ยนสมการสเกล ค่า Min/Max หรือหน่วยการวัด สามารถดำเนินการผ่าน API Server ได้ทันทีโดยไม่ต้อง re-flash เฟิร์มแวร์ใหม่ลงบอร์ด ESP32

ความถูกต้องและมาตรฐานเดียวกัน: การคำนวณที่จุดเดียวช่วยป้องกันปัญหาความคลาดเคลื่อนของข้อมูล เมื่อมี Edge Device หลายตัวส่งข้อมูลเข้ามายังระบบ

ข้อ 2: จากการทำ HTTP Forensics หากไม่มีการตรวจสอบเงื่อนไข RawMax <= RawMin ในโค้ด จะเกิด Exception ชนิดใดขึ้นในภาษา C# และส่งผลต่อการทำงานของเซิร์ฟเวอร์อย่างไร?
คำตอบ:

Exception ที่อาจเกิดขึ้น: หาก RawMax == RawMin จะเกิด DivideByZeroException (เนื่องจากสูตรสเกลมีตัวหารเป็น (RawMax - RawMin)) หรือเกิด ArgumentException / Logic Error ที่ทำให้คำนวณค่า Calibrated ติดลบหรือผิดเพี้ยน

ผลกระทบต่อเซิร์ฟเวอร์: หากไม่มีการทำ Validation หรือ try-catch ดักจับ Unhandled Exception จะทำให้ Kestrel คืนค่า 500 Internal Server Error ให้กับ Client และอาจส่งผลกระทบต่อเสถียรภาพของกระบวนการทำงานบน Web Server

ข้อ 3: อธิบายสาเหตุทางเทคนิคว่าทำไมคำขอ HTTP POST ที่ไม่มี Header Content-Type: application/json จึงถูกปฏิเสธด้วยรหัสสถานะ 415 Unsupported Media Type?
คำตอบ:
เพราะเฟรมเวิร์ก ASP.NET Core บน Kestrel Server ต้องอาศัย Header Content-Type ในการเลือกตัวแปลงข้อมูล (Formatter/Deserializer เช่น System.Text.Json) เพื่อแปลง Raw Request Body ไปเป็น C# Object

เมื่อ Client ไม่ส่ง Header Content-Type: application/json ตัว Web Framework จะไม่ทราบรูปแบบโครงสร้างของ Payload และปฏิเสธการประมวลผลด้วยรหัสสถานะ 415 Unsupported Media Type เพื่อป้องกันความผิดพลาดในการแปลงข้อมูล