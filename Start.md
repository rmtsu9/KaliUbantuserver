# เริ่มเก็บ Dataset: Pre-collection Checklist

ใช้คู่มือนี้ **ทุกครั้งหลังเปิดเครื่องใหม่** ก่อนเริ่มเก็บ Dataset เพื่อให้เครือข่าย เว็บแอป และระบบบันทึกทราฟฟิกพร้อมใช้งาน

> ทำเฉพาะใน Lab/ระบบที่ได้รับอนุญาตเท่านั้น

## ภาพรวม

| เครื่อง | IP Address | Network Interface | หน้าที่ |
| --- | --- | --- | --- |
| Ubuntu Server | `192.168.100.10/24` | `enp0s8` | Juice Shop, Nginx (WAF), Suricata (IDS), เก็บ Packet |
| Kali Linux | `192.168.100.20/24` | `eth1` | เครื่องทดสอบและสร้างทราฟฟิก |

## 1. ตั้งค่า IP Address

ทำขั้นตอนนี้ทั้งสองเครื่อง

### Ubuntu Server

เปิด Terminal แล้วรัน:

```bash
sudo ip addr add 192.168.100.10/24 dev enp0s8
sudo ip link set enp0s8 up
```

### Kali Linux

เปิด Terminal แล้วรัน:

```bash
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

## 2. เปิดบริการที่จำเป็นบน Ubuntu Server

### เริ่ม Juice Shop Container

```bash
sudo docker start juiceshop
```

### รีสตาร์ท WAF และ IDS

```bash
sudo systemctl restart nginx suricata
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

---

## 4. เริ่มดักจับทราฟฟิกเครือข่าย

บน Ubuntu Server เปิด Terminal แยกไว้ 1 หน้าต่าง แล้วรันคำสั่งนี้:

```bash
sudo tcpdump -i enp0s8 -w ~/attack_traffic.pcap
```

ปล่อย Terminal นี้ให้ทำงานค้างไว้ตลอดช่วงที่เก็บ Dataset  
เมื่อเก็บข้อมูลเสร็จ ให้กด `Ctrl+C` เพื่อหยุดและปิดไฟล์ PCAP อย่างสมบูรณ์

---

## พร้อมเริ่มเก็บ Dataset

ตรวจครบทั้ง 4 ขั้นตอนแล้ว ระบบพร้อมใช้งาน:

- เครือข่ายระหว่าง Kali และ Ubuntu เชื่อมต่อแล้ว
- Juice Shop, Nginx และ Suricata ทำงานแล้ว
- เว็บเข้าได้จาก Kali Linux
- `tcpdump` กำลังบันทึกทราฟฟิกลง `~/attack_traffic.pcap`

จากนั้นจึงสลับไปทำการทดสอบที่ได้รับอนุญาตจาก Kali Linux เพื่อสร้างทราฟฟิกสำหรับ Dataset ได้ทันที
