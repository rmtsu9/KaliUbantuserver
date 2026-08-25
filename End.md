# ปิดรอบเก็บ Dataset และส่งออกไฟล์

ใช้คู่มือนี้หลังสร้างทราฟฟิกสำหรับการทดสอบจาก Kali Linux เสร็จแล้ว เพื่อหยุดการบันทึก รวบรวม Log บีบอัด Dataset และนำไฟล์ออกไปใช้งาน

> ทำเฉพาะใน Lab/ระบบที่ได้รับอนุญาตเท่านั้น

## ภาพรวมผลลัพธ์

เมื่อทำครบแล้ว จะได้ไฟล์ Dataset รูปแบบนี้ใน Home Directory ของ Ubuntu Server:

```text
~/dataset_YYYY-MM-DD_HHMM.tar.gz
```

ตัวอย่าง: `dataset_2026-08-26_1430.tar.gz`

---

## 1. หยุดการดักจับ Packet

บน Ubuntu Server ให้สลับไปยัง Terminal ที่กำลังรัน `tcpdump` อยู่ แล้วกด:

```text
Ctrl + C
```

คำสั่งจะหยุดทำงานและบันทึกไฟล์ `~/attack_traffic.pcap` ให้สมบูรณ์

---

## 2. รวบรวม Log และตั้งสิทธิ์ไฟล์

บน Ubuntu Server รันคำสั่งชุดนี้:

```bash
# สร้างโฟลเดอร์รวบรวม Dataset
mkdir -p ~/my_dataset

# คัดลอก Log จาก Nginx, WAF และ Suricata IDS
sudo cp /var/log/nginx/access.log ~/my_dataset/nginx_access.log
sudo cp /var/log/nginx/error.log ~/my_dataset/nginx_error.log
sudo cp /var/log/suricata/eve.json ~/my_dataset/suricata_eve.json

# ย้าย Packet capture หากมีไฟล์อยู่
[ -f ~/attack_traffic.pcap ] && sudo mv ~/attack_traffic.pcap ~/my_dataset/

# บันทึก Log ของ Juice Shop Container
sudo docker logs juiceshop > ~/my_dataset/juiceshop_app.log 2>&1

# ให้ผู้ใช้ปัจจุบันเป็นเจ้าของไฟล์ เพื่อดาวน์โหลดผ่าน WinSCP ได้
sudo chown -R $USER:$USER ~/my_dataset
```

ไฟล์หลักที่ควรอยู่ใน `~/my_dataset`:

| ไฟล์ | แหล่งข้อมูล |
| --- | --- |
| `nginx_access.log` | HTTP access log ของ Nginx |
| `nginx_error.log` | error log ของ Nginx |
| `suricata_eve.json` | Event log ของ Suricata IDS |
| `attack_traffic.pcap` | Packet capture จาก tcpdump (ถ้ามี) |
| `juiceshop_app.log` | Application log ของ Juice Shop |

---

## 3. บีบอัด Dataset เป็น `.tar.gz`

รันคำสั่งนี้บน Ubuntu Server:

```bash
tar -czvf ~/dataset_$(date +%F_%H%M).tar.gz -C ~ my_dataset
```

ไฟล์ที่ได้จะอยู่ใน Home Directory และชื่อไฟล์จะระบุวันเวลาอัตโนมัติ เช่น:

```text
~/dataset_2026-08-26_1430.tar.gz
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

ใช้เฉพาะเมื่อต้องการเริ่มเก็บข้อมูลรอบใหม่ เช่น เก็บทราฟฟิกปกติเพื่อใช้เป็น Benchmark

```bash
# ล้างเนื้อหา Log เดิม โดยคงไฟล์ไว้
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/nginx/error.log
sudo truncate -s 0 /var/log/suricata/eve.json

# ลบโฟลเดอร์รวบรวมชั่วคราว หลังยืนยันว่า export ไฟล์สำเร็จแล้ว
rm -rf ~/my_dataset
```

> ตรวจสอบว่าได้ดาวน์โหลด `dataset_*.tar.gz` ไปยัง Windows เรียบร้อยก่อนลบ `~/my_dataset`

## Checklist ก่อนจบรอบ

- [ ] หยุด `tcpdump` ด้วย `Ctrl+C`
- [ ] รวบรวม Log ลง `~/my_dataset`
- [ ] สร้างไฟล์ `dataset_*.tar.gz`
- [ ] ดาวน์โหลดไฟล์ไปยัง Windows แล้ว
- [ ] ล้าง Log เดิมแล้ว (หากต้องการเริ่มรอบใหม่)
