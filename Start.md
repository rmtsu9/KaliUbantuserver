# เริ่มเก็บ Dataset: Pre-collection Checklist

ใช้คู่มือนี้ **ทุกครั้งหลังเปิดเครื่องใหม่** ก่อนเริ่มเก็บ Dataset เพื่อให้เครือข่าย เว็บแอป ระบบตรวจจับ และ Data Pipeline พร้อมใช้งาน

> ทำเฉพาะใน Lab/ระบบที่ได้รับอนุญาตเท่านั้น

## ภาพรวมระบบ

| เครื่อง | IP Address | Network Interface | หน้าที่ |
| --- | --- | --- | --- |
| Ubuntu Server | `192.168.100.10/24` | `enp0s8` | Juice Shop, Nginx, Suricata, Logstash, Filebeat และ Packetbeat |
| Kali Linux | `192.168.100.20/24` | `eth1` | เครื่องทดสอบและสร้างทราฟฟิก |

### Data Pipeline

```text
Nginx / Suricata / Juice Shop logs ──► Filebeat ──┐
                                                    ├──► Logstash ──► /var/log/logstash/ai_dataset_YYYY_MM_DD.jsonl
Network traffic on enp0s8 ──────────► Packetbeat ──┘
```

- **Filebeat** อ่านและส่งต่อ Log ที่ตั้งค่าไว้
- **Packetbeat** ดักจับและแปลงข้อมูลทราฟฟิกเครือข่ายเป็น event
- **Logstash** รับ/ประมวลผล event และเขียน Dataset แบบ JSON Lines (`.jsonl`)

`tcpdump` ไม่ใช่ขั้นตอนบังคับอีกต่อไป เพราะ Pipeline นี้ทำงานเป็น background service อยู่แล้ว ใช้ `tcpdump` เฉพาะเมื่อต้องการ PCAP ดิบเป็นหลักฐานสำรอง

---

## 1. ตั้งค่า IP Address

ทำขั้นตอนนี้ทั้งสองเครื่อง

### Ubuntu Server

เปิด Terminal แล้วรัน:

```bash
sudo ip addr flush dev enp0s8
sudo ip addr add 192.168.100.10/24 dev enp0s8
sudo ip link set enp0s8 up
```

### Kali Linux

เปิด Terminal แล้วรัน:

```bash
sudo ip addr flush dev eth1
sudo ip addr add 192.168.100.20/24 dev eth1
sudo ip link set eth1 up
```

### ทดสอบการเชื่อมต่อ

จาก Kali Linux รัน:

```bash
ping 192.168.100.10
```

ผลลัพธ์ที่ต้องการ: `0% packet loss`  
กด `Ctrl+C` เพื่อหยุดการ ping เมื่อทดสอบเสร็จ

---

## 2. เปิดบริการระบบและ Data Pipeline

ทำบน **Ubuntu Server**

### เริ่ม Juice Shop, Nginx และ Suricata

```bash
sudo docker start juiceshop
sudo systemctl restart nginx suricata
```

### เริ่ม Logstash, Filebeat และ Packetbeat

```bash
sudo systemctl restart logstash filebeat packetbeat
```

---

## 3. ตรวจสอบความพร้อมของระบบ (Health Check)

### ตรวจสอบ Juice Shop บน Ubuntu Server

```bash
curl -I http://127.0.0.1:3000
```

ผลลัพธ์ที่ต้องการ:

```text
HTTP/1.1 200 OK
```

### ตรวจสอบหน้าเว็บจาก Kali Linux

เปิด Firefox แล้วเข้า:

<http://192.168.100.10>

ผลลัพธ์ที่ต้องการ: เห็นหน้าเว็บร้านค้าตามปกติ และไม่มีข้อความ `Unable to connect`

### ตรวจสอบสถานะ Data Pipeline บน Ubuntu Server

```bash
sudo systemctl status logstash filebeat packetbeat --no-pager
```

ผลลัพธ์ที่ต้องการ: ทั้งสาม service อยู่ในสถานะ `active (running)`

---

## 4. ตรวจสอบไฟล์ Dataset และเริ่มสร้าง Traffic

Dataset หลักจะถูกเขียนอัตโนมัติโดย Logstash ที่:

```text
/var/log/logstash/ai_dataset_YYYY_MM_DD.jsonl
```

ตรวจสอบไฟล์ของวันปัจจุบันบน Ubuntu Server:

```bash
DATASET_FILE="/var/log/logstash/ai_dataset_$(date +%F | tr '-' '_').jsonl"
sudo ls -lh "$DATASET_FILE"
```

หากไฟล์ยังไม่ปรากฏ ให้สร้างคำขอ HTTP ปกติจาก Kali Linux 1 ครั้ง แล้วตรวจสอบอีกครั้ง:

```bash
curl -I http://192.168.100.10
```

ดู event ล่าสุดแบบต่อเนื่องได้ด้วย:

```bash
sudo tail -f "$DATASET_FILE"
```

กด `Ctrl+C` เพื่อออกจากการดูไฟล์โดยไม่กระทบ service ใด ๆ

### พร้อมเริ่มเก็บ Dataset

เมื่อครบทุกขั้นตอน ระบบพร้อมใช้งาน:

- เครือข่ายระหว่าง Kali และ Ubuntu เชื่อมต่อแล้ว
- Juice Shop, Nginx และ Suricata ทำงานแล้ว
- Logstash, Filebeat และ Packetbeat ทำงานแล้ว
- เว็บเข้าได้จาก Kali Linux
- Logstash กำลังเขียน event อัตโนมัติลงไฟล์ `.jsonl`

จากนั้นจึงสลับไปทำการทดสอบที่ได้รับอนุญาตจาก Kali Linux เพื่อสร้างทราฟฟิกสำหรับ Dataset ได้ทันที

---

## PCAP ดิบ *(Optional)*

หากต้องการเก็บ packet capture ดิบเพิ่มเติม ให้เปิด Terminal แยกบน Ubuntu Server แล้วรัน:

```bash
sudo tcpdump -i enp0s8 -w ~/attack_traffic.pcap
```

ปล่อยคำสั่งนี้ทำงานเฉพาะช่วงที่ต้องการเก็บ PCAP และกด `Ctrl+C` เมื่อเสร็จสิ้น ไฟล์นี้เป็นข้อมูลเสริม ไม่ใช่ Dataset หลักของ pipeline ใหม่
