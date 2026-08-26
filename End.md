# ปิดรอบเก็บ Dataset และส่งออกไฟล์

ใช้คู่มือนี้หลังสร้างทราฟฟิกสำหรับการทดสอบจาก Kali Linux เสร็จแล้ว เพื่อหยุด Data Pipeline อย่างเป็นระเบียบ รวบรวมไฟล์ JSONL และส่งออก Dataset ไปใช้งานต่อ

> ทำเฉพาะใน Lab/ระบบที่ได้รับอนุญาตเท่านั้น

## ภาพรวมผลลัพธ์

Dataset หลักที่ Logstash สร้างอยู่ที่:

```text
/var/log/logstash/ai_dataset_YYYY_MM_DD.jsonl
```

เมื่อทำครบแล้ว จะได้ไฟล์ archive ใน Home Directory ของ Ubuntu Server:

```text
~/dataset_YYYY-MM-DD_HHMM.tar.gz
```

ตัวอย่าง: `dataset_2026-08-26_1430.tar.gz`

---

## 1. ปิดรอบ Data Pipeline อย่างเป็นระเบียบ

ทำบน Ubuntu Server โดยหยุดตัวเก็บข้อมูลก่อน แล้วจึงหยุดตัวประมวลผล:

```bash
sudo systemctl stop filebeat packetbeat
sudo systemctl stop logstash
```

ตรวจสอบว่าสถานะ service หยุดเรียบร้อย:

```bash
sudo systemctl status logstash filebeat packetbeat --no-pager
```

> การหยุด Filebeat และ Packetbeat ก่อนช่วยป้องกันไม่ให้มี event ใหม่ถูกส่งเข้ามาระหว่างที่กำลังปิด Logstash และเตรียม archive

### PCAP ดิบ *(Optional)*

หากเลือกเปิด `tcpdump` เพิ่มเติมระหว่างรอบนี้ ให้กลับไปยัง Terminal ที่รันอยู่ แล้วกด:

```text
Ctrl + C
```

ขั้นตอนนี้ไม่จำเป็น หากใช้เฉพาะ Logstash + Beats pipeline

---

## 2. รวบรวม Dataset หลักและ Log ประกอบ

รันคำสั่งชุดนี้บน Ubuntu Server:

```bash
# สร้างโฟลเดอร์รวบรวม Dataset
mkdir -p ~/my_dataset

# คัดลอกไฟล์ JSONL ที่ Logstash สร้าง
sudo find /var/log/logstash -maxdepth 1 -type f -name 'ai_dataset_*.jsonl' \
  -exec cp -t "$HOME/my_dataset" {} +

# คัดลอก Log ต้นทางไว้ใช้ตรวจสอบ/เปรียบเทียบเพิ่มเติม
sudo cp /var/log/nginx/access.log ~/my_dataset/nginx_access.log
sudo cp /var/log/nginx/error.log ~/my_dataset/nginx_error.log
sudo cp /var/log/suricata/eve.json ~/my_dataset/suricata_eve.json
sudo docker logs juiceshop > ~/my_dataset/juiceshop_app.log 2>&1

# คัดลอก PCAP เฉพาะกรณีที่เปิด tcpdump แบบ Optional
[ -f ~/attack_traffic.pcap ] && sudo mv ~/attack_traffic.pcap ~/my_dataset/

# ให้ผู้ใช้ปัจจุบันเป็นเจ้าของไฟล์ เพื่อดาวน์โหลดผ่าน WinSCP ได้
sudo chown -R "$USER:$USER" ~/my_dataset
```

ไฟล์สำคัญที่ควรอยู่ใน `~/my_dataset`:

| ไฟล์ | สถานะ | แหล่งข้อมูล |
| --- | --- | --- |
| `ai_dataset_*.jsonl` | **หลัก** | Event ที่ Logstash เขียนจาก Filebeat และ Packetbeat |
| `nginx_access.log` | ประกอบ | HTTP access log ของ Nginx |
| `nginx_error.log` | ประกอบ | error log ของ Nginx |
| `suricata_eve.json` | ประกอบ | Event log ของ Suricata IDS |
| `juiceshop_app.log` | ประกอบ | Application log ของ Juice Shop |
| `attack_traffic.pcap` | ทางเลือก | Packet capture จาก tcpdump หากเปิดเก็บไว้ |

ตรวจสอบว่าไฟล์ Dataset หลักถูกคัดลอกแล้ว:

```bash
ls -lh ~/my_dataset/ai_dataset_*.jsonl
```

---

## 3. บีบอัด Dataset เป็น `.tar.gz`

รันคำสั่งนี้บน Ubuntu Server:

```bash
tar -czvf ~/dataset_$(date +%F_%H%M).tar.gz -C ~ my_dataset
```

ตรวจสอบไฟล์ที่ได้:

```bash
ls -lh ~/dataset_*.tar.gz
```

---

## 4. ดาวน์โหลดไฟล์ไปยัง Windows Host

1. เปิด **WinSCP** บน Windows
2. เชื่อมต่อด้วยข้อมูลต่อไปนี้

| รายการ | ค่า |
| --- | --- |
| Host name | `127.0.0.1` |
| Port number | `2222` |
| User name | `ubantuvb` |
3. เข้า Home Directory ของ Ubuntu Server
4. ลากไฟล์ `dataset_*.tar.gz` จากฝั่ง Ubuntu ไปเก็บในโฟลเดอร์ที่ต้องการบน Windows

---

## 5. เตรียม Lab สำหรับรอบถัดไป *(Optional)*

หลังตรวจสอบว่า archive ถูกสร้างและดาวน์โหลดแล้ว สามารถลบเฉพาะโฟลเดอร์รวบรวมชั่วคราวได้:

```bash
rm -rf ~/my_dataset
```

> อย่าลบหรือ `truncate` ไฟล์ `ai_dataset_*.jsonl` ขณะที่ Logstash ยังทำงานอยู่ เพราะอาจทำให้ Dataset ของรอบถัดไปไม่สมบูรณ์

หากต้องการแยกไฟล์ Dataset ต่อรอบอย่างชัดเจน ควรใช้ Run ID/เวลาใน metadata หรือปรับ Logstash output ให้สร้างชื่อไฟล์ตามรอบการทดลองก่อนเริ่มรอบใหม่

## Checklist ก่อนจบรอบ

- [ ] หยุด Filebeat, Packetbeat และ Logstash แล้ว
- [ ] หยุด `tcpdump` แล้ว (หากเปิดใช้งาน)
- [ ] คัดลอก `ai_dataset_*.jsonl` ลง `~/my_dataset` แล้ว
- [ ] สร้างไฟล์ `dataset_*.tar.gz` แล้ว
- [ ] ดาวน์โหลดไฟล์ไปยัง Windows แล้ว
- [ ] ลบ `~/my_dataset` แล้ว (หากต้องการ)
