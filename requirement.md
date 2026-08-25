# Requirements และวิธีติดตั้ง Lab

เอกสารนี้ระบุสิ่งที่ต้องเตรียมและติดตั้งสำหรับ Kali Linux และ Ubuntu Server ก่อนใช้งาน Lab เก็บ Dataset ตามคู่มือ [Start.md](Start.md) และ [End.md](End.md)

> ใช้เฉพาะใน Lab หรือระบบที่ได้รับอนุญาตเท่านั้น

## 1. ข้อกำหนดเบื้องต้น

| รายการ | Kali Linux | Ubuntu Server |
| --- | --- | --- |
| ระบบปฏิบัติการ | Kali Linux 64-bit พร้อม Desktop | Ubuntu Server 22.04 LTS หรือ 24.04 LTS 64-bit |
| CPU/RAM ขั้นต่ำแนะนำ | 2 vCPU / 4 GB RAM | 2 vCPU / 4 GB RAM |
| พื้นที่ว่างแนะนำ | 25 GB | 30 GB ขึ้นไป (PCAP และ Log ใช้พื้นที่เพิ่มตามระยะเวลาบันทึก) |
| Network adapter | 2 ตัว: NAT + Host-only/Internal Network | 2 ตัว: NAT + Host-only/Internal Network |
| สิทธิ์ผู้ใช้ | ผู้ใช้ที่ใช้ `sudo` ได้ | ผู้ใช้ที่ใช้ `sudo` ได้ |

Kali แบบ Desktop ควรมี RAM อย่างน้อย 2 GB และพื้นที่ว่าง 20 GB ตามคำแนะนำของ Kali; สำหรับ Lab นี้เพิ่มทรัพยากรเพื่อเปิด Firefox และเครื่องมือพร้อมกันได้ราบรื่นขึ้น [Kali installation requirements](https://www.kali.org/docs/installation/hard-disk-install/)

## 2. การตั้งค่า Network ของ VM

ตั้งค่าทั้งสอง VM ให้มี adapter 2 ตัว:

| Adapter | Kali | Ubuntu | วัตถุประสงค์ |
| --- | --- | --- | --- |
| Adapter 1 | NAT | NAT | ออกอินเทอร์เน็ตเพื่อติดตั้ง/อัปเดตแพ็กเกจ |
| Adapter 2 | Host-only หรือ Internal Network เดียวกัน | Host-only หรือ Internal Network เดียวกัน | เครือข่าย Lab ระหว่าง Kali และ Ubuntu |

Interface ของ adapter ที่สองต้องตรงกับคู่มือนี้:

| เครื่อง | Interface | IP ที่ใช้ใน Lab |
| --- | --- | --- |
| Ubuntu Server | `enp0s8` | `192.168.100.10/24` |
| Kali Linux | `eth1` | `192.168.100.20/24` |

ตรวจสอบชื่อ interface ก่อนตั้งค่า IP:

```bash
ip addr
```

หากชื่อ interface ไม่ตรง ให้แก้ชื่อใน [Start.md](Start.md) และการตั้งค่า Suricata ให้เป็นชื่อจริงของเครื่องนั้น

### การดาวน์โหลดผ่าน WinSCP (Windows Host)

หากต้องการใช้ WinSCP ผ่าน `127.0.0.1:2222` ให้ตั้งค่า **NAT Port Forwarding ของ Ubuntu VM**:

| ค่า | กำหนดเป็น |
| --- | --- |
| Protocol | TCP |
| Host IP | `127.0.0.1` |
| Host Port | `2222` |
| Guest IP | เว้นว่าง หรือ IP ของ NAT adapter |
| Guest Port | `22` |

และต้องติดตั้ง `openssh-server` บน Ubuntu ตามขั้นตอนด้านล่าง

---

## 3. ติดตั้ง Kali Linux

### ซอฟต์แวร์ที่ต้องมี

| แพ็กเกจ/โปรแกรม | ใช้สำหรับ |
| --- | --- |
| `firefox-esr` | เปิดและตรวจสอบเว็บแอป |
| `curl` | ทดสอบ HTTP จาก command line |
| `iputils-ping` | ตรวจสอบการเชื่อมต่อเครือข่าย |
| `net-tools` | คำสั่งช่วยตรวจสอบ network เช่น `ifconfig` |
| `nikto`, `sqlmap` *(ทางเลือก)* | ใช้เฉพาะการทดสอบที่ได้รับอนุญาตใน Lab |

### คำสั่งติดตั้ง

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y firefox-esr curl iputils-ping net-tools
```

หากต้องการติดตั้งเครื่องมือทดสอบเพิ่มเติมสำหรับ Lab ที่ได้รับอนุญาต:

```bash
sudo apt install -y nikto sqlmap
```

Kali แนะนำให้ใช้ repository ของ Kali เท่านั้น และไม่ผสม repository ของ Ubuntu/Debian เพราะอาจทำให้ระบบเสียหายได้ [Kali package repositories](https://www.kali.org/docs/general-use/kali-apt-sources/)

### ตรวจสอบ Kali

```bash
firefox --version
curl --version
ping -V
ip addr
```

---

## 4. ติดตั้ง Ubuntu Server

### ซอฟต์แวร์ที่ต้องมี

| ซอฟต์แวร์ | ใช้สำหรับ |
| --- | --- |
| Docker Engine | รัน Juice Shop container |
| Nginx | Reverse proxy และบันทึก HTTP log |
| Suricata | IDS และสร้าง event log (`eve.json`) |
| `tcpdump` | บันทึก packet capture เป็น PCAP |
| OpenSSH Server | รับการเชื่อมต่อจาก WinSCP |
| `curl`, `jq` | Health Check และอ่าน JSON log |

### 4.1 อัปเดตระบบและติดตั้งแพ็กเกจพื้นฐาน

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y nginx suricata tcpdump openssh-server curl jq ca-certificates
```

### 4.2 ติดตั้ง Docker Engine จาก Docker Official Repository

ทำตามขั้นตอนใน [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/) โดยใช้คำสั่งต่อไปนี้:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

สร้างไฟล์ `/etc/apt/sources.list.d/docker.sources` ด้วย editor เช่น `sudo nano /etc/apt/sources.list.d/docker.sources` แล้ววางเนื้อหานี้:

```text
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: <UBUNTU_CODENAME>
Components: stable
Architectures: <ARCHITECTURE>
Signed-By: /etc/apt/keyrings/docker.asc
```

แทนค่า:

```bash
. /etc/os-release && echo "$VERSION_CODENAME"
dpkg --print-architecture
```

เช่น Ubuntu 24.04 บนเครื่อง Intel/AMD จะเป็น `noble` และ `amd64` ตามลำดับ จากนั้นติดตั้งและตรวจสอบ Docker:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run hello-world
```

Lab นี้ใช้ `sudo docker` อยู่แล้ว จึงไม่จำเป็นต้องเพิ่มผู้ใช้เข้า Docker group; กลุ่มนี้มีสิทธิ์เทียบเท่า root ตามคำเตือนใน [Docker post-install guide](https://docs.docker.com/engine/install/linux-postinstall/)

### 4.3 ติดตั้งและเริ่ม OWASP Juice Shop

```bash
sudo docker pull bkimminich/juice-shop
sudo docker run -d --name juiceshop --restart unless-stopped -p 127.0.0.1:3000:3000 bkimminich/juice-shop
```

ตรวจสอบ:

```bash
sudo docker ps
curl -I http://127.0.0.1:3000
```

ควรเห็นสถานะ `HTTP/1.1 200 OK` จากคำสั่ง `curl`  
คำสั่ง Docker image และ port `3000` อ้างอิงจาก [OWASP Juice Shop repository](https://github.com/juice-shop/juice-shop)

### 4.4 ตั้งค่า Nginx เป็น Reverse Proxy

Nginx อย่างเดียวเป็น reverse proxy ไม่ใช่ WAF โดยอัตโนมัติ หาก Lab ต้องการ WAF ต้องติดตั้งและตั้งค่ากฎ WAF เพิ่มเติมแยกต่างหาก

สร้างไฟล์ `/etc/nginx/sites-available/juiceshop` ด้วย `sudo nano /etc/nginx/sites-available/juiceshop` แล้ววาง:

```nginx
server {
    listen 80 default_server;
    server_name _;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

เปิดใช้งาน configuration และตรวจสอบ:

```bash
sudo ln -s /etc/nginx/sites-available/juiceshop /etc/nginx/sites-enabled/juiceshop
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable --now nginx
curl -I http://127.0.0.1
```

`proxy_pass` เป็น directive ที่ใช้ส่ง request ต่อไปยัง upstream server ตาม [NGINX HTTP proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

### 4.5 ตั้งค่า Suricata

เปิดไฟล์ตั้งค่า:

```bash
sudo nano /etc/suricata/suricata.yaml
```

ตรวจสอบให้มีหรือปรับค่าให้ตรงกับ Lab:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.100.0/24]"

af-packet:
  - interface: enp0s8
```

ดาวน์โหลด/อัปเดตกฎ ทดสอบ configuration และเปิดบริการ:

```bash
sudo suricata-update
sudo suricata -T -c /etc/suricata/suricata.yaml -i enp0s8
sudo systemctl enable --now suricata
sudo systemctl status suricata --no-pager
```

Suricata ต้องตั้งค่า interface ที่ต้องตรวจสอบให้ตรงกับเครื่องจริง และใช้ signature/rule เพื่อสร้าง alert; ดูรายละเอียดได้ที่ [Suricata Quickstart Guide](https://docs.suricata.io/en/latest/quickstart.html)

### 4.6 เปิด SSH สำหรับ WinSCP

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

## 5. ตรวจสอบความพร้อมก่อนเริ่มงาน

บน Ubuntu Server:

```bash
sudo systemctl is-active docker nginx suricata ssh
sudo docker ps --filter name=juiceshop
curl -I http://127.0.0.1:3000
curl -I http://127.0.0.1
```

จาก Kali Linux:

```bash
ping -c 4 192.168.100.10
curl -I http://192.168.100.10
```

เมื่อทุกคำสั่งทำงานตามปกติ ให้ดำเนินการตาม [Start.md](Start.md) ก่อนเริ่มเก็บ Dataset

## เอกสารอ้างอิง

1. [Kali Linux Installation Requirements](https://www.kali.org/docs/installation/hard-disk-install/)
2. [Kali Linux Package Repositories](https://www.kali.org/docs/general-use/kali-apt-sources/)
3. [Docker Engine — Install on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
4. [Docker Engine — Linux Post-installation](https://docs.docker.com/engine/install/linux-postinstall/)
5. [OWASP Juice Shop — Official Repository](https://github.com/juice-shop/juice-shop)
6. [NGINX HTTP Proxy Module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
7. [Suricata Quickstart Guide](https://docs.suricata.io/en/latest/quickstart.html)
