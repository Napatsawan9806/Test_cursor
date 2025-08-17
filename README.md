# การนำโค้ด marker ไปใช้ในโค้ดหลัก (main)

เอกสารนี้อธิบายว่าฟังก์ชัน/คลาสจากไฟล์ `marker` ถูกนำไปใช้ในไฟล์ `main` อย่างไร และถูกเรียกในขั้นตอนไหนของโฟลว์หลักบ้าง

## สรุปบทบาทของโค้ดใน `marker`
- `MarkerVisionHandler`: จัดการ subscribe/unsubscribe ระบบ vision ของหุ่นยนต์เพื่อรับ callback เมื่อพบ marker และเก็บรายการ marker ที่ตรวจพบชั่วคราว
- `ToFSensorHandler`: จัดการอ่านและกรองค่าจาก ToF sensor เพื่อวัดระยะและตัดสินใจว่าใกล้พอจะตรวจ marker ได้หรือไม่
- `detect_red(...)`: ตรวจจับสีแดงจากภาพกล้อง เพื่อนำไปกรองทิศทางที่จะสแกน marker ต่อ
- ชุดฟังก์ชันสแกน เช่น `scan_red_then_marker_*`: ตัวอย่างการสแกนสีแดงก่อน แล้วจึงตรวจ marker เฉพาะทิศทางที่พบสีแดง

## จุดที่โค้ด marker ถูกใช้ใน `main`
1) การเริ่มต้นระบบตรวจจับ marker (ก่อนเริ่มสำรวจ)
- สร้างออบเจ็กต์ `marker_handler = MarkerVisionHandler()`
- เปิดระบบตรวจจับด้วย `marker_handler.start_continuous_detection(ep_vision)` เพื่อให้ vision เรียก callback เมื่อพบ marker

2) ระหว่างสแกนโหนดปัจจุบัน `scan_current_node_absolute(...)`
- เปิดวิดีโอสตรีมแล้วตรวจจับสีแดงด้วย `detect_red(...)` ในแต่ละทิศทาง (front/left/right และกรณีพิเศษ back)
- บันทึกเฉพาะทิศทางที่พบสีแดงไว้ในรายการ `red_directions`
- สำหรับทุกทิศทางที่พบสีแดง: ก้ม gimbal ลง (-20°) วัดระยะด้วย ToF (กรองสัญญาณด้วย `ToFSensorHandler`)
  - ถ้าระยะเหมาะสม: `marker_handler.reset_detection()` และ `marker_handler.wait_for_markers(...)` เพื่อรอผลตรวจ marker ชั่วคราวจาก vision
  - นำ `marker_ids` ที่พบไปเก็บใน `current_node.markerScanResults[direction]` และถ้าพบจริง กำหนด `current_node.markersFound[direction]` พร้อม `current_node.hasMarkers = True`

3) การใช้งานข้อมูลภายหลังสแกน
- ข้อมูล marker ที่บันทึกต่อโหนดจะถูกแสดงในสรุปกราฟ และถูก export ออกไฟล์ JSON โดยฟังก์ชัน `export_maze_data_to_json(...)`

4) การปิดระบบเมื่อจบโปรแกรม
- ยกเลิก subscribe ด้วย `marker_handler.stop_continuous_detection(ep_vision)` และปิดการเชื่อมต่อหุ่นยนต์

## ฟังก์ชันหลักที่เกี่ยวข้องใน `main`
- `explore_autonomously_with_absolute_directions(...)`: ลูปสำรวจหลักที่เรียก `scan_current_node_absolute(...)` ซึ่งภายในมีขั้นตอน Red-filtered marker scanning
- `scan_current_node_absolute(...)`: ทำ ToF scan + red detection และตรวจ marker เฉพาะในทิศทางที่พบสีแดง
- `export_maze_data_to_json(...)`: รวมผล marker ต่อโหนดและเขียนออกเป็นไฟล์ JSON

หมายเหตุ: โค้ดที่เกี่ยวกับ marker ใน `main` ถูกผนวกรวมไว้ภายในแล้ว (เช่น `MarkerVisionHandler`, `ToFSensorHandler`, `detect_red`) เพื่อให้รันได้ในไฟล์เดียว โดยแนวคิดและโฟลว์การใช้งานสอดคล้องกับไฟล์ `marker`