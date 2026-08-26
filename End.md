# ปิดรอบเก็บ Dataset และส่งออกไฟล์

ใช้คู่มือนี้หลังหยุดสร้างทราฟฟิกจาก Kali Linux แล้ว สำหรับรวบรวม Dataset ที่ประมวลผลผ่าน **Logstash + Filebeat + Packetbeat** และเตรียมระบบสำหรับรอบถัดไป

> ใช้เฉพาะใน Lab หรือระบบที่ได้รับอนุญาตเท่านั้น

## ผลลัพธ์

ไฟล์ Dataset หลัก:

```text
/var/log/logstash/ai_dataset_YYYY_MM_DD.jsonl
```

ไฟล์ Archive ที่ได้:

```text
~/dataset_YYYY-MM-DD_HHMM.tar.gz
```

---

## 1. หยุด Traffic และ Flush ข้อมูลเข้า Logstash

หลังหยุดการสแกนหรือโจมตีจาก Kali Linux ให้รีสตาร์ท Logstash เพื่อให้ข้อมูลที่ค้างอยู่ถูกเขียนลงไฟล์ JSONL:

```bash
sudo systemctl restart logstash
```

ตรวจสอบสถานะ:

```bash
sudo systemctl status logstash --no-pager
```

> หากต้องการป้องกันข้อมูลสูญหาย ควรตรวจสอบ Logstash ให้กลับมาเป็นสถานะ `active (running)` ก่อนดำเนินการขั้นต่อไป

---

## 2. รวบรวม Dataset และปรับสิทธิ์ไฟล์

ไฟล์หลักคือ `ai_dataset_*.jsonl` ซึ่งเป็นข้อมูล ECS JSON Lines ที่รวมจาก Filebeat และ Packetbeat

```bash
# 1. สร้างโฟลเดอร์รวบรวม Dataset
mkdir -p ~/my_dataset

# 2. คัดลอก Dataset หลักจาก Logstash
sudo find /var/log/logstash -maxdepth 1 -type f \
  -name 'ai_dataset_*.jsonl' \
  -exec cp -t "$HOME/my_dataset" {} +

# 3. คัดลอก Raw Logs สำหรับใช้อ้างอิง (Optional)
sudo cp /var/log/nginx/access.log \
  ~/my_dataset/nginx_access.log 2>/dev/null || true

sudo cp /var/log/nginx/error.log \
  ~/my_dataset/nginx_error.log 2>/dev/null || true

sudo cp /var/log/suricata/eve.json \
  ~/my_dataset/suricata_eve.json 2>/dev/null || true

sudo cp /var/log/suricata/fast.log \
  ~/my_dataset/suricata_fast.log 2>/dev/null || true

# 4. บันทึก Log ของ Juice Shop Container
sudo docker logs juiceshop \
  > ~/my_dataset/juiceshop_app.log 2>&1 || true

# 5. ปรับสิทธิ์ให้ผู้ใช้ปัจจุบันดาวน์โหลดผ่าน WinSCP
sudo chown -R "$USER:$USER" ~/my_dataset
```

ตรวจสอบไฟล์ Dataset หลัก:

```bash
ls -lh ~/my_dataset/ai_dataset_*.jsonl
```

---

## 3. บีบอัด Dataset เป็นไฟล์ `.tar.gz`

```bash
tar -czvf ~/dataset_$(date +%F_%H%M).tar.gz \
  -C "$HOME" my_dataset
```

ตรวจสอบไฟล์ Archive:

```bash
ls -lh ~/dataset_*.tar.gz
```

---

## 4. ดาวน์โหลดไฟล์ผ่าน WinSCP

เปิด WinSCP บน Windows แล้วเชื่อมต่อด้วยข้อมูลใดข้อมูลหนึ่ง:

| รายการ | Local/Port Forwarding | Network |
|---|---|---|
| Host name | `127.0.0.1` | `192.168.100.10` |
| Port number | `2222` | `22` |
| User name | `ubantuvb` | `ubantuvb` |

ไปยังโฟลเดอร์:

```text
/home/ubantuvb/
```

จากนั้นดาวน์โหลดไฟล์:

```text
dataset_YYYY-MM-DD_HHMM.tar.gz
```

---

## 5. ล้าง Log และรีเซ็ต Pipeline

ดำเนินการหลังยืนยันว่าดาวน์โหลดไฟล์ Archive ลง Windows เรียบร้อยแล้ว:

```bash
# 1. หยุดบริการ Data Pipeline
sudo systemctl stop filebeat packetbeat logstash

# 2. ลบ Dataset JSONL เดิมของ Logstash
sudo rm -f /var/log/logstash/ai_dataset_*.jsonl

# 3. ล้าง Log ของ Nginx
sudo truncate -s 0 \
  /var/log/nginx/access.log \
  /var/log/nginx/error.log 2>/dev/null || true

# 4. ล้าง Log ของ Suricata
sudo truncate -s 0 \
  /var/log/suricata/eve.json \
  /var/log/suricata/fast.log \
  /var/log/suricata/stats.log 2>/dev/null || true

# 5. ล้าง Log ของ Juice Shop Container
DOCKER_LOG_PATH="$(sudo docker inspect \
  --format='{{.LogPath}}' juiceshop 2>/dev/null || true)"

if [ -n "$DOCKER_LOG_PATH" ] && [ -f "$DOCKER_LOG_PATH" ]; then
  sudo truncate -s 0 "$DOCKER_LOG_PATH"
fi

# 6. ลบโฟลเดอร์รวบรวมชั่วคราว
rm -rf ~/my_dataset

# 7. เริ่มบริการ Pipeline สำหรับรอบใหม่
sudo systemctl start logstash filebeat packetbeat

# 8. ตรวจสอบสถานะบริการ
sudo systemctl status logstash filebeat packetbeat --no-pager
```

> อย่าลบไฟล์ `ai_dataset_*.jsonl` ก่อนสร้าง Archive และตรวจสอบการดาวน์โหลดเสร็จแล้ว

---

## ไฟล์ภายใน Archive

| ไฟล์ | รายละเอียด | ความสำคัญ |
|---|---|---|
| `ai_dataset_*.jsonl` | Log รวม Network Protocol, Host Log และ IDS Alerts ในรูปแบบ ECS JSON Lines | **ไฟล์หลัก** |
| `nginx_access.log` | Raw Access Log จาก Nginx Reverse Proxy | สำรองอ้างอิง |
| `nginx_error.log` | Error Log ของ Nginx | สำรองอ้างอิง |
| `suricata_eve.json` | Raw Security Events จาก Suricata IDS | สำรองอ้างอิง |
| `suricata_fast.log` | Alert Log แบบย่อของ Suricata | สำรองอ้างอิง |
| `juiceshop_app.log` | Application Console Output ของ Juice Shop | สำรองอ้างอิง |

## Checklist

- [ ] หยุด Traffic จาก Kali Linux แล้ว
- [ ] รีสตาร์ทและตรวจสอบ Logstash แล้ว
- [ ] คัดลอก `ai_dataset_*.jsonl` แล้ว
- [ ] สร้างไฟล์ `.tar.gz` แล้ว
- [ ] ดาวน์โหลดไฟล์ผ่าน WinSCP แล้ว
- [ ] ล้าง Dataset และ Raw Logs เดิมแล้ว
- [ ] เริ่ม Filebeat, Packetbeat และ Logstash สำหรับรอบใหม่แล้ว
