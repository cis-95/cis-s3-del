# CIS S3 Del (S3 Batch Deleter)

โปรเจกต์นี้เป็นเครื่องมือสำหรับ **ลบไฟล์ใน S3 bucket จำนวนมาก** โดยออกแบบให้ลิสต์ไฟล์จาก bucket ทีละก้อน (batch) และลบทีละก้อนเพื่อลดผลกระทบต่อระบบ สามารถกำหนดขนาด batch ได้ตามต้องการ (เช่น 100, 500, 1000 ไฟล์ต่อครั้ง)


## คุณสมบัติ
- ลบไฟล์แบบ **Batch Delete API** (สูงสุด 1000 ไฟล์/ครั้ง)
- ตั้งค่า batch size ได้เองด้วย flag `-batch`
- มีตัวเลือก `-prefix` สำหรับลบเฉพาะไฟล์ที่ขึ้นต้นด้วย prefix
- สามารถกำหนด `-sleep_ms` เพื่อพักระหว่างการลบแต่ละ batch

## การติดตั้ง CIS-S3-DEL บน Linux (ubuntu, debian, centos, mac os)
### 1. Download File ติดตั้ง
```bash
sudo wget https://github.com/cis-95/cis-s3-del/releases/latest/download/cis-s3-del -O /usr/local/bin/cis-s3-del
```

### 2. ให้สิทธิ์ในการรันไฟล์ cis-s3-del
```bash
sudo chmod +x /usr/local/bin/cis-s3-del
```

### 3. ตรวจสอบ Version เพื่อทดสอบว่า Script ทำงานได้
```bash
cis-s3-del version
```

## วิธีการใช้งาน

### 1) ลบตรง ๆ ไม่แบ่ง shard
ตัวอย่าง รันแบบ 4 workers, batch 500, พิมพ์สถิติทุก 10 วินาที และพัก 200ms ระหว่าง batch
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -prefix "" \
  -batch 500 \
  -workers 4 \
  -list_workers 1 \
  -list_shard_mode none \
  -sleep_ms 200 \
  -stats_sec 10
```

ตัวอย่างการรันใน Windows ต้องอยู่บรรทัดเดียวกัน
```bash
cis-s3-del -endpoint s3.netapp.local:9000 -secure=false -access "$S3_ACCESS_KEY" -secret "$S3_SECRET_KEY" -bucket my-bucket -prefix "" -batch 500 -workers 4 -list_workers 1 -list_shard_mode none -sleep_ms 200 -stats_sec 10
```

### 2) Hex1 (0–f, 16 shards)
ใช้กับ key ที่เป็น UUID, hash หรือไฟล์ที่เริ่มด้วย hex

📌 จะแตกการลิสต์เป็น 16 เส้นทาง (0..f) ทำงานขนานกันได้
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -batch 500 \
  -workers 4 \
  -list_workers 4 \
  -list_shard_mode hex1 \
  -sleep_ms 200 \
  -stats_sec 10
```

###  3) Hex2 (00–ff, 256 shards)
ใช้กับ bucket ที่ไฟล์มหาศาล (พันล้าน+) และชื่อเป็น UUID/Hash

📌 ช่วยกระจายโหลดลิสต์มากขึ้น (256 shards)
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -batch 500 \
  -workers 8 \
  -list_workers 8 \
  -list_shard_mode hex2 \
  -sleep_ms 200 \
  -stats_sec 10
```

###  4) Alpha (a–z, 26 shards)
ใช้กับ bucket ที่ key เป็นชื่อโฟลเดอร์/ไฟล์ภาษาอังกฤษ

📌 ใช้ได้ดีถ้าไฟล์ขึ้นต้นด้วยโฟลเดอร์ เช่น reports/, images/, logs/
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -batch 500 \
  -workers 4 \
  -list_workers 4 \
  -list_shard_mode alpha \
  -sleep_ms 200 \
  -stats_sec 10
```

### 5) Auto (Adaptive split)
เริ่มจาก shard ใหญ่ แล้วแตก shard อัตโนมัติเมื่อเกิน threshold

📌 ถ้าเจอ shard ไหน list ไปแล้วเกิน 50,000 objects จะ auto split เป็น 0..f (หรือ a..z ตามโหมดที่เลือก)
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -batch 500 \
  -workers 4 \
  -list_workers 4 \
  -list_shard_mode auto \
  -auto_split_threshold 50000 \
  -sleep_ms 200 \
  -stats_sec 10
```

### 6) Detect (ตรวจ distribution)
ไม่ลบไฟล์ แค่สุ่ม list ตัวอย่างมานับ prefix ตัวแรก/สองตัวแรก

📌 ใช้สำรวจ key distribution → จะรู้ว่าเหมาะกับ hex1, hex2, alpha หรือ auto
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -list_shard_mode detect \
  -detect_limit 20000
```

ตัวอย่างผลลัพธ์:
```bash
[detect] prefix distribution (sample=20000):
  [1 char]
    '0' : 15234
    'a' : 3102
    'r' : 1664
  [2 chars]
    '00' : 8123
    '01' : 7111
    'ra' : 900
    're' : 764
```
