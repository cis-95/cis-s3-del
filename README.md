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

### 2. ให้สิทธิ์ในการรันไฟล์ OneBox
```bash
sudo chmod +x /usr/local/bin/cis-s3-del
```

## วิธีการใช้งาน
```bash
cis-s3-del \
  -endpoint s3.netapp.local:9000 \
  -secure=false \
  -access "$S3_ACCESS_KEY" \
  -secret "$S3_SECRET_KEY" \
  -bucket my-bucket \
  -prefix "" \
  -batch 100 \
  -sleep_ms 200
```