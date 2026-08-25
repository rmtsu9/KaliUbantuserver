# Kali–Ubuntu Security Monitoring Lab

Lab นี้ใช้สร้างและเก็บข้อมูลทราฟฟิกเว็บในสภาพแวดล้อมที่แยกจากระบบจริง โดยให้ Kali Linux เป็นเครื่องสร้างทราฟฟิก และ Ubuntu Server เป็นเครื่องให้บริการเว็บ บันทึก Log และตรวจจับเหตุการณ์เครือข่าย

> ขอบเขตการใช้งาน: สำหรับการเรียนรู้ การทดสอบ และการเก็บ Dataset บน Lab ที่ได้รับอนุญาตเท่านั้น ห้ามนำขั้นตอนหรือเครื่องมือไปใช้กับระบบที่ไม่มีสิทธิ์ทดสอบ

## วัตถุประสงค์

- สร้าง Dataset จากทราฟฟิกปกติและทราฟฟิกทดสอบที่ควบคุมได้
- เก็บหลักฐานจากหลายแหล่ง เพื่อใช้วิเคราะห์เหตุการณ์และพัฒนาโมเดลตรวจจับ
- เปรียบเทียบมุมมองระดับแพ็กเก็ต ระดับเว็บเซิร์ฟเวอร์ ระดับ IDS และระดับแอปพลิเคชัน

## สถาปัตยกรรม

```text
Kali Linux                                      Ubuntu Server
192.168.100.20                                  192.168.100.10
      │                                                 │
      │  HTTP traffic ผ่าน eth1 / enp0s8                │
      └────────────────────────────────────────────────▶│
                                                        │
                                              ┌─────────▼─────────┐
                                              │ Nginx             │
                                              │ Reverse proxy /   │
                                              │ ชั้น WAF*         │
                                              └─────────┬─────────┘
                                                        │
                                              ┌─────────▼─────────┐
                                              │ Juice Shop        │
                                              │ Docker Container  │
                                              │ port 3000         │
                                              └───────────────────┘

                         ┌─────────────────────────────────────────┐
                         │ การเก็บข้อมูลบน Ubuntu Server           │
                         │ • tcpdump → attack_traffic.pcap         │
                         │ • Nginx → access.log / error.log         │
                         │ • Suricata IDS → eve.json                │
                         │ • Docker → juiceshop_app.log             │
                         └─────────────────────────────────────────┘
```

\*Nginx ทำหน้าที่เป็น reverse proxy ตามการตั้งค่า `proxy_pass`; จะเป็น WAF ได้เมื่อมีการติดตั้งและตั้งค่ากฎ WAF เพิ่มเติม

## หลักการทำงานของแต่ละส่วน

### 1. Kali Linux — เครื่องสร้างทราฟฟิก

Kali Linux ส่งคำขอ HTTP ไปยัง `192.168.100.10` ผ่านเครือข่าย Lab ที่กำหนดไว้ ทราฟฟิกอาจเป็นการใช้งานปกติ หรือการทดสอบช่องโหว่ที่ได้รับอนุญาต โดยการแยกเครื่องทดสอบออกจากเครื่องบริการช่วยให้ระบุทิศทางและต้นทางของทราฟฟิกใน Dataset ได้ชัดเจน

### 2. Nginx — จุดรับ HTTP และ Reverse Proxy

Nginx รับคำขอจาก Kali แล้วส่งต่อไปยัง Juice Shop ตามค่า `proxy_pass` ของโมดูล HTTP proxy ซึ่งรองรับการส่งคำขอไปยัง upstream server และส่งต่อ header สำคัญ เช่น Host หรือ IP ของผู้ร้องขอได้ [เอกสาร Nginx HTTP proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

Log ของ Nginx จึงเป็นหลักฐานระดับ HTTP ที่มีประโยชน์ เช่น เวลา, IP ต้นทาง, method, path, status code และขนาดการตอบกลับ:

- `access.log` — บันทึกคำขอที่เว็บเซิร์ฟเวอร์รับและตอบกลับ
- `error.log` — บันทึกข้อผิดพลาดหรือความผิดปกติของ Nginx

### 3. OWASP Juice Shop — เว็บแอปสำหรับฝึกและทดสอบ

OWASP Juice Shop เป็นเว็บแอปที่ตั้งใจออกแบบให้มีช่องโหว่ เพื่อใช้ในการฝึกอบรมและทดสอบเครื่องมือด้านความปลอดภัย โดยครอบคลุมหัวข้อความเสี่ยงจาก OWASP Top 10 หลายประเภท [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)

ใน Lab นี้ Juice Shop รันใน Docker container ชื่อ `juiceshop` ที่พอร์ต `3000` ทำให้แยกส่วนแอปพลิเคชันออกจากระบบปฏิบัติการหลัก และดึงบันทึกของแอปด้วย `docker logs juiceshop` ได้ โดย Docker ระบุว่าคำสั่งนี้ดึง log ที่มีอยู่ของ container ณ เวลาที่เรียกใช้ [Docker: container logs](https://docs.docker.com/reference/cli/docker/container/logs/)

### 4. Suricata — Network IDS

Suricata ตรวจสอบแพ็กเก็ตบน interface ที่กำหนด และใช้ signature/rule เพื่อสร้าง alert เมื่อพบรูปแบบตรงกับกฎตรวจจับ เอกสาร Suricata ระบุว่า signature คือ rule ที่ใช้ trigger alert และสามารถจัดการ ruleset ด้วย `suricata-update` ได้ [Suricata Quickstart](https://docs.suricata.io/en/latest/quickstart.html)

ไฟล์ `eve.json` เป็นผลลัพธ์แบบ JSON ซึ่งบันทึกเหตุการณ์ได้หลายชนิด เช่น alert, flow, HTTP, DNS และสถิติ ทั้งนี้ชนิดเหตุการณ์ที่มีจริงขึ้นกับการตั้งค่า `suricata.yaml` และ rules ที่เปิดใช้งาน

### 5. tcpdump — หลักฐานระดับแพ็กเก็ต

`tcpdump` ดักจับข้อมูลที่ผ่าน `enp0s8` และบันทึกลงไฟล์ PCAP (`attack_traffic.pcap`) ซึ่งเก็บรายละเอียดระดับเครือข่าย เช่น IP, port, protocol, เวลา และ payload ตามที่รูปแบบทราฟฟิกอนุญาตให้บันทึกได้

PCAP มีประโยชน์เมื่อจำเป็นต้องตรวจสอบลำดับแพ็กเก็ตหรือวิเคราะห์ทราฟฟิกย้อนหลัง ขณะที่ Log ระดับอื่นจะช่วยอ่านบริบทของเหตุการณ์ได้รวดเร็วกว่า

## เหตุผลที่ต้องเก็บ Log หลายชั้น

แหล่งข้อมูลแต่ละชนิดให้มุมมองคนละระดับ การนำมาประกอบกันทำให้ตรวจสอบเหตุการณ์ได้ครบกว่าเก็บเพียงแหล่งเดียว

| แหล่งข้อมูล | มุมมองหลัก | ตัวอย่างการใช้งาน |
| --- | --- | --- |
| PCAP (`tcpdump`) | แพ็กเก็ตและการเชื่อมต่อ | ตรวจสอบลำดับหรือรูปแบบทราฟฟิก |
| Nginx access/error log | คำขอและผลตอบกลับ HTTP | ดู URL, status code, client IP และข้อผิดพลาด |
| Suricata EVE JSON | เหตุการณ์จากการตรวจจับตาม rule | ค้นหา alert และข้อมูลประกอบการแจ้งเตือน |
| Docker application log | พฤติกรรมของแอป | เชื่อมโยงคำขอกับผลที่แอปรายงาน |

แนวคิดนี้สอดคล้องกับคำแนะนำของ OWASP ที่ให้บันทึกเหตุการณ์ด้านความปลอดภัยอย่างเหมาะสม เพื่อสนับสนุนการตรวจสอบและการตอบสนองต่อเหตุการณ์ [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

## ลำดับการเก็บ Dataset

1. ใช้ [Start.md](Start.md) เพื่อกำหนด IP เปิดบริการ ตรวจสอบ Health Check และเริ่ม `tcpdump`
2. สร้างทราฟฟิกที่อยู่ในขอบเขตการทดสอบและได้รับอนุญาตจาก Kali Linux
3. ใช้ [End.md](End.md) เพื่อหยุดการดักจับ รวบรวม Log และบีบอัดไฟล์ Dataset
4. เก็บ metadata ของรอบทดสอบเพิ่มเสมอ เช่น วันเวลา ประเภททราฟฟิก การตั้งค่า rule และเวอร์ชันเครื่องมือ

## โครงสร้าง Dataset ที่ส่งออก

```text
dataset_YYYY-MM-DD_HHMM.tar.gz
└── my_dataset/
    ├── attack_traffic.pcap       # Packet capture (ถ้ามี)
    ├── nginx_access.log          # HTTP access log
    ├── nginx_error.log           # Nginx error log
    ├── suricata_eve.json         # IDS events / alerts
    └── juiceshop_app.log         # Application log
```

## ข้อควรระวังด้านคุณภาพข้อมูล

- เริ่ม `tcpdump` ก่อนสร้างทราฟฟิก และหยุดด้วย `Ctrl+C` เพื่อปิดไฟล์ PCAP อย่างถูกต้อง
- บันทึกเวลาเริ่มและสิ้นสุดของแต่ละรอบ เพื่อเชื่อมโยงเหตุการณ์ข้าม Log ได้แม่นยำ
- แยกรอบของทราฟฟิกปกติและทราฟฟิกทดสอบ พร้อมกำกับ label ให้ชัดเจน
- ตรวจสอบเวลาและ timezone ของ Kali กับ Ubuntu ให้สอดคล้องกัน
- ก่อนล้าง Log ให้ยืนยันว่าไฟล์ `.tar.gz` ถูกสร้างและดาวน์โหลดสำเร็จแล้ว

## เอกสารอ้างอิง

1. [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
2. [Suricata Documentation — Quickstart Guide](https://docs.suricata.io/en/latest/quickstart.html)
3. [NGINX Documentation — ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
4. [Docker Documentation — docker container logs](https://docs.docker.com/reference/cli/docker/container/logs/)
5. [OWASP Cheat Sheet Series — Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
