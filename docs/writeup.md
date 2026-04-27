

**โดย:** นายพิชญุตม์ อิ่มใจ นักศึกษาชั้นปีที่ 2 สาขาวิศวกรรมระบบ IoT และสารสนเทศ KMITL 
**ระยะเวลา:** เมษายน 2026  
**Stack:** Cowrie · Grafana · Loki · Telegram Alert · GCP

---

## ทำไมถึงทำโปรเจคนี้

ตอนปิดเทอมปี 2 ขึ้นปี 3 อยากทำโปรเจคที่ได้ลองของจริงสักครั้ง ไม่ใช่แค่อ่านทฤษฎีในห้องเรียน เลยลองทำ Honeypot ดู เพราะเรียนสาย IoT อยู่แล้ว อยากรู้ว่าถ้า deploy อุปกรณ์ขึ้น internet จริงๆ มันจะโดนโจมตียังไงบ้าง

---

## Honeypot คืออะไร

Honeypot คือระบบที่แกล้งทำเป็น server จริง เพื่อหลอกล่อให้ attacker เข้ามา แล้วเก็บข้อมูลพฤติกรรมทุกอย่างไว้ โดยที่ attacker ไม่รู้ว่ากำลังอยู่ในกับดัก

ในโปรเจคนี้ใช้ **Cowrie** ซึ่งเป็น SSH honeypot ที่จำลองเป็น Linux server ปลอม ถ้า attacker SSH เข้ามาได้ก็จะเจอ shell ปลอม ทุกคำสั่งที่พิมพ์จะถูก log ไว้หมดเลย

---

## Architecture

```
Internet (Attackers & Bots)
          ↓
GCP Firewall (เปิด port 2222)
          ↓
Cowrie SSH Honeypot (port 2222)
          ↓
JSON Log File
          ↓
Promtail → Loki → Grafana Dashboard
          ↓
Telegram Alert (real-time)
```

---

## Environment

Cloud : Google Cloud Platform (GCP)
VM e2-micro : 2 vCPU, 1GB RAM + 1GB Swap
OS : Ubuntu 24.04
Honeypot : Cowrie (latest)
Monitoring : Grafana + Loki + Promtail
Alert : Telegram Bot


## ผลลัพธ์ที่ได้

หลังจาก deploy honeypot ขึ้น internet ได้ไม่กี่ชั่วโมง มี bot เริ่มเข้ามา scan แล้ว ภายในไม่กี่วันเก็บข้อมูลได้ดังนี้

### IP ที่เข้ามามากที่สุด

IP : จำนวนครั้ง

170.130.201.38 : 10 ครั้ง
51.158.205.203 : 6 ครั้ง
45.148.10.121 : 5 ครั้ง
47.84.101.65 : 3 ครั้ง
146.190.83.66 : 3 ครั้ง

### Password ที่ bot ใช้บ่อยที่สุด

Password : จำนวนครั้ง

admin : 7 ครั้ง
orangepi : 4 ครั้ง
P : 4 ครั้ง
root : 1 ครั้ง
support : 1 ครั้ง

---

## สิ่งที่น่าสนใจที่เจอ

### 1. Bot เข้ามาเร็วมาก

ผมไม่ได้บอกใครเลยว่ามี server ตัวนี้อยู่ แต่ภายในไม่กี่ชั่วโมงหลัง deploy มี bot เข้ามา scan แล้ว แสดงว่ามีระบบที่ scan ทุก IP บน internet ตลอดเวลา ไม่มีทางซ่อน server ได้เลยถ้าเชื่อมต่อ internet

### 2. Bot ลอง IoT Default Credentials โดยเฉพาะ

password ที่ bot ลองบ่อยที่สุดคือ `orangepi/orangepi` ซึ่งเป็น default credential ของ Orange Pi บอร์ดคอมพิวเตอร์ขนาดเล็กที่นิยมใช้ในโปรเจค IoT แสดงว่า bot พวกนี้มี wordlist ของ IoT device โดยเฉพาะ ไม่ใช่แค่ลอง `admin/admin` ทั่วไป

ในฐานะที่เรียนสาย IoT นี่คือ insight สำคัญมาก เพราะถ้า deploy IoT device แล้วลืมเปลี่ยน default password จะโดนแฮกได้ทันทีที่เชื่อมต่อ internet

### 3. พบ Advanced Bot ที่พยายามใช้เครื่องเป็น Proxy

IP `45.148.10.121` ไม่ได้แค่ login แล้วออก แต่พยายาม TCP forward ไปที่ `turn.cloudflare.com:3478` ซึ่งแสดงว่า bot นี้ต้องการใช้เครื่องที่แฮกได้เป็น **proxy/tunnel** เพื่อซ่อนตัวต่อไป พฤติกรรมแบบนี้ซับซ้อนกว่า bot ทั่วไปมาก

### 4. Bot ทำงานแบบ Automated 100%

ทุก bot ที่เข้ามาใช้ `SSH-2.0-libssh2` แทน OpenSSH ปกติ บอกได้ทันทีว่าเป็น automated script ไม่ใช่คนนั่ง SSH เอง นอกจากนี้แต่ละ session ใช้เวลาแค่ 0.1-1.2 วินาที แล้วก็ disconnect ออกไป แบบนี้เรียกว่า **Credential Stuffing** คือวนลอง password จากลิสต์ไปเรื่อยๆ ถ้าเจออันไหนผ่านก็รายงานกลับไปให้ระบบอื่นมา exploit ต่อ

---

## สิ่งที่ได้เรียนรู้

**1. ไม่มี server ไหนซ่อนได้บน internet**  
แค่เปิด port ไว้ก็มีคนมา scan แล้ว ไม่จำเป็นต้องประกาศให้ใครรู้

**2. Default Password อันตรายมาก**  
IoT device ที่ไม่เปลี่ยน default password จะโดนโจมตีทันทีที่เชื่อมต่อ internet นี่คือปัญหาใหญ่ของ IoT security ในโลกจริง

**3. Log และ Monitoring สำคัญมาก**  
ถ้าไม่มี Grafana dashboard และ Telegram alert เราคงไม่รู้เลยว่าเกิดอะไรขึ้นกับ server แม้แต่น้อย การ monitor ระบบ real-time เป็นสิ่งจำเป็นสำหรับทุก server ที่เชื่อมต่อ internet

**4. Data Persistence ต้องคิดตั้งแต่แรก**  
เจอปัญหาตอนแรกที่เก็บ log ไว้ใน `/tmp` แล้วหายหลัง reboot ต้องย้ายมาเก็บที่ `/var/lib/loki` แทน เป็นบทเรียนเรื่อง production mindset ที่สำคัญมาก

---

## สิ่งที่อยากทำต่อ

- [ ] เพิ่ม GeoIP เพื่อดูว่าการโจมตีมาจากประเทศไหนบ้าง
- [ ] วิเคราะห์ command ที่ bot พิมพ์หลัง login สำเร็จ
- [ ] ทดสอบ deploy MQTT honeypot เพิ่มสำหรับ IoT protocol โดยเฉพาะ
- [ ] เพิ่ม Fail2Ban บน server จริงเพื่อ block IP ที่โจมตีซ้ำ

---

## Tools & References

- [Cowrie Honeypot](https://github.com/cowrie/cowrie)
- [Grafana](https://grafana.com/)
- [Loki](https://grafana.com/oss/loki)
- [Google Cloud Platform](https://cloud.google.com/)

---

_โปรเจคนี้ทำเพื่อการศึกษาเท่านั้น Honeypot ถูก deploy บน isolated VM บน GCP ไม่มีการโจมตีระบบจริงใดๆ_
