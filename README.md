# AI-Assisted Seamless Video Loop Engine

เครื่องมือแยกต่างหาก (ไม่เกี่ยวข้องกับระบบ CAMS) สำหรับแปลงวิดีโอสั้นให้เป็นวิดีโอ Loop ความยาวตามที่กำหนด โดยเน้นให้รอยต่อระหว่างรอบมองแทบไม่เห็น ทำงานทั้งหมดฝั่ง client ในเบราว์เซอร์ ไม่มีการอัปโหลดไฟล์ขึ้นเซิร์ฟเวอร์ และไม่ต้องใช้ API key ใด ๆ

## วิธีใช้งาน

เปิด `index.html` ในเบราว์เซอร์ (Chrome/Edge แนะนำ) แล้ว:

1. อัปโหลด/ลากวางไฟล์วิดีโอต้นฉบับ (แนะนำ 2–30 วินาที ที่มีการเคลื่อนไหวต่อเนื่อง)
2. เลือก Processing Mode (FAST / BALANCED / HIGH QUALITY / AI QUALITY)
3. ตั้งความยาวเป้าหมาย (Target Duration) — มีปุ่มลัด 30 วิ / 1 นาที / 5 นาที / 10 นาที
4. เลือก Loop Strategy (AUTO แนะนำ), FPS, คุณภาพ/บิตเรต, รูปแบบไฟล์ (MP4/WebM) และตัวเลือกเสียง
5. กด "วิเคราะห์และสร้างวิดีโอ Loop" แล้วรอประมวลผล — ระหว่างนี้จะเห็น log แต่ละ stage ของ pipeline
6. ตรวจสอบ/แก้ไขจุด START–END ที่ตรวจพบได้ก่อน generate ใหม่ (Manual Override)
7. ดูคะแนนคุณภาพ (Loop Quality Score) แล้วดาวน์โหลดผลลัพธ์

## Pipeline ที่ implement จริง

เครื่องมือนี้ทำตาม pipeline ที่ระบุในสเปก (Metadata Analysis → Motion Profile → Scene Cut Detection → Loop Candidate Detection ทั้งคลิป → Candidate Scoring ถ่วงน้ำหนัก → Strategy Selection → Adaptive Transition → Optical Flow (ประมาณ) → Color/Exposure Matching → Seam Blending → Quality Check → Automatic Retry → Repeat Until Target Duration → Final Encoding) โดยเป็น **client-side ล้วน ๆ ในเบราว์เซอร์** (ไม่มี server/FFmpeg/OpenCV/job queue จริง) ตามที่เลือก deploy แบบ static site:

1. **Video Metadata Analysis** — ตรวจ resolution, duration, FPS ต้นฉบับ (ผ่าน `requestVideoFrameCallback`), บิตเรตโดยประมาณ
2. **Motion Profile + Scene Cut Detection** — วิเคราะห์ทั้งคลิปที่ analysis FPS ต่ำ (6–12fps ตาม mode) คำนวณ motion score / camera motion / object motion / brightness ต่อเฟรม และตรวจจับ scene cut เพื่อจำกัดขอบเขตค้นหาให้อยู่ในช็อตเดียว
3. **Loop Candidate Detection (ทั้งคลิป)** — ค้นหาคู่เฟรม START/END ที่ดีที่สุด "ทั่วทั้งคลิป" (ไม่ใช่แค่ต้น/ท้าย) ด้วย coarse-to-fine search ที่จำกัดขอบเขตการคำนวณให้ทำงานได้ในเบราว์เซอร์
4. **Candidate Scoring** — ให้คะแนนถ่วงน้ำหนักตาม Frame Similarity, Motion Similarity, Object Similarity, Optical-Flow Consistency, Brightness, Color, Camera Motion (ปรับน้ำหนักได้ในส่วน Advanced)
5. **Loop Strategy A/B/C + AUTO** — Forward (ตัดตรง), Crossfade, Forward+Reverse (ping-pong) — โหมด AUTO จะลองหลายแบบ+หลาย duration แล้ววัดผลก่อนเลือกที่ดีที่สุด (Automatic Retry)
6. **Adaptive Transition Duration** — คำนวณความยาว transition จากระดับ motion เฉลี่ยของคลิป (Motion สูง → transition สั้น, Motion ต่ำ → transition ยาว) ตามตารางในสเปก
7. **Optical Flow (ประมาณ)** — ใช้ global/regional block-based motion estimation (ไม่ใช่ Farneback/TV-L1/RAFT จริง) เพื่อชดเชยการเลื่อนระหว่าง tail/head ก่อนผสาน — มี confidence gate ป้องกันไม่ให้ใช้ shift ที่ประเมินผิดพลาด (เช่น กรณี motion เป็นวัตถุเดี่ยวไม่ใช่กล้องแพน)
8. **Color/Exposure Matching** — วัดค่าความสว่าง/สี RGB เฉลี่ยของช่วงต้น-ท้าย แล้วปรับผ่าน canvas filter ระหว่างผสาน (HIGH QUALITY / AI QUALITY เท่านั้น)
9. **Seam Blending** — motion-compensated crossfade แบบ smoothstep easing โดยเฟรมที่ถูกผสานไปแล้วในรอบก่อนหน้าจะไม่ถูกแสดงซ้ำในรอบถัดไป (แก้ปัญหา duplicated frames ที่ seam)
10. **Loop Quality Score + Automatic Retry** — วัดคุณภาพจริงที่ seam ภายใน (ไม่ใช่แค่ปลายไฟล์) แล้วให้คะแนน Overall / Motion Continuity / Frame Similarity / Color Continuity / Brightness Continuity / Artifact Score พร้อมลองหลาย strategy/duration ก่อนเลือกตัวที่ดีที่สุด
11. **Repeat Until Target Duration** — คำนวณจำนวนรอบจากความยาวเป้าหมายที่ตั้งไว้ พร้อม anti-repetition variation (สลับ candidate สำรองทุกรอบเลขคู่ ถ้ามี candidate ที่คะแนนใกล้เคียงกัน) เพื่อลดความรู้สึก pattern ซ้ำในวิดีโอยาว
12. **Audio Handling** — ตัวเลือก "เก็บเสียงต้นฉบับ" ใช้ Web Audio API วนเสียงพร้อม equal-power crossfade ที่จุดต่อ (ไม่ใช้ FFmpeg)
13. **Final Encoding** — วาดเฟรมผลลัพธ์ลง canvas แล้วบันทึกผ่าน `canvas.captureStream()` + `MediaRecorder` เลือกได้ทั้ง MP4 (H.264 ถ้าเบราว์เซอร์รองรับ) และ WebM (VP9/AV1) พร้อมคุมบิตเรตตามคุณภาพที่เลือก

## Processing Modes

| Mode | Analysis FPS | ขอบเขตค้นหา | Regional Motion Comp | Color Matching | Retry |
|---|---|---|---|---|---|
| FAST | 6 | ใกล้ต้น/ท้ายคลิปเท่านั้น | ✗ | ✗ | 1 |
| BALANCED | 8 | ทั้งคลิป | ✗ (global shift เท่านั้น) | ✗ | 2 |
| HIGH QUALITY | 10 | ทั้งคลิป | ✓ (grid 3×2) | ✓ | 3 |
| AI QUALITY | 12 | ทั้งคลิป (ละเอียดสุด) | ✓ | ✓ | 4 |

## ขอบเขตที่ "แปล" จากสเปกต้นฉบับ (สำคัญ — อ่านก่อนใช้งานจริง)

สเปกต้นฉบับ (23 หัวข้อ/11 phases) ออกแบบมาสำหรับระบบ backend เต็มรูปแบบ (FFmpeg + OpenCV + RAFT/Farneback optical flow + Redis job queue + REST API + AI frame-interpolation model) แต่เครื่องมือนี้เลือก deploy แบบ **static site ไม่มี server** จึงต้อง "แปล" บางส่วนเป็นเทคนิค client-side แทน:

- **Optical Flow**: ใช้ block-based motion estimation ของ Canvas เอง (regional shift ต่อกริด, พร้อม confidence gate) แทน Farneback/TV-L1/RAFT จริง — เป็นการประมาณ ไม่ใช่ dense optical flow
- **Frame Interpolation**: ใช้ motion-compensated blending หลายขั้นระหว่างช่วง transition แทนโมเดล AI interpolation (เช่น RIFE)
- **FFmpeg / OpenCV**: ไม่ได้ใช้จริง — encode ผ่าน Canvas + `MediaRecorder` ของเบราว์เซอร์แทน
- **REST API / Job Queue / Storage**: ไม่ได้ implement เพราะไม่มี server — การประมวลผลทั้งหมดทำงาน synchronous ในแท็บเบราว์เซอร์ของผู้ใช้แทนที่จะเป็น background job
- **"AI QUALITY" mode**: ยังเป็นเทคนิค client-side ล้วน ๆ (ค้นหาละเอียดขึ้น + interpolation หลายขั้นขึ้น) **ไม่ได้เรียกใช้โมเดล AI จริง** เพราะไม่มี GPU/server

## ข้อจำกัดอื่น ๆ

- `videoBitsPerSecond` เป็นเป้าหมาย/เพดานที่ส่งให้ตัวเข้ารหัสของเบราว์เซอร์ ขนาดไฟล์จริงยังขึ้นกับเนื้อหาวิดีโอด้วย
- การเพิ่ม FPS ผลลัพธ์ให้สูงกว่าต้นฉบับ ไม่ได้เพิ่มรายละเอียดการเคลื่อนไหวจริง (สุ่มตัวอย่างถี่ขึ้นเท่านั้น)
- Target Duration ที่ได้จริงอาจคลาดเคลื่อนเล็กน้อยจากที่ตั้งไว้ (ปัดตามจำนวนรอบเต็มของ segment ที่ตรวจพบ)
- Mode/FPS/Duration สูงขึ้น = ใช้เวลาประมวลผลนานขึ้นมาก (การ encode ทำงานแบบ real-time ผ่าน `MediaRecorder` — วิดีโอผลลัพธ์ยาว 5 นาที ใช้เวลาประมวลผลใกล้เคียง 5 นาทีจริง)
- แนะนำให้ใช้ Chrome หรือ Edge เนื่องจาก Safari รองรับ `canvas.captureStream()` ไม่สมบูรณ์
