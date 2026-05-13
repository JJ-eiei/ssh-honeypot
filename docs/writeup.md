**โดย:** นายพิชญุตม์ อิ่มใจ นักศึกษาสาขาวิศวกรรมระบบไอโอทีและสารสนเทศ สจล.
**ระยะเวลา:** เมษายน 2026  
**Stack:** Cowrie · Grafana · Loki · Telegram Alert · GCP

---

## ทำไมถึงทำโปรเจคนี้

ช่วงปิดเทอมปี 2 ผมอยากลองทำโปรเจคของจริงสักครั้ง พอดีผมเรียนสาย IoT อยู่แล้ว เลยอยากรู้ว่าถ้า deploy device หรือ server ขึ้น internet จริงๆ มันจะโดนโจมตียังไงบ้าง

---

## Honeypot คืออะไร

Honeypot คือระบบที่จำลองตัวเองเป็น server จริง เพื่อหลอกล่อ attacker ให้เข้ามา แล้วเก็บข้อมูลพฤติกรรมทุกอย่างไว้โดยที่ attacker ไม่รู้ตัว

ในโปรเจคนี้ใช้ **Cowrie** ที่เป็น SSH honeypot ที่จำลองเป็น Linux server ปลอม ถ้า attacker SSH เข้ามาได้ก็จะเจอ shell ปลอม ทุกคำสั่งที่พิมพ์จะถูก log ไว้หมดเลย

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

| Component  | รายละเอียด                        |
|------------|-----------------------------------|
| Cloud      | Google Cloud Platform (GCP)       |
| VM         | e2-micro — 2 vCPU, 1GB RAM + 1GB Swap |
| OS         | Ubuntu 24.04                      |
| Honeypot   | Cowrie (latest)                   |
| Monitoring | Grafana + Loki + Promtail         |
| Alert      | Telegram Bot                      |

เลือก e2-micro เพราะอยู่ใน free tier ของ GCP พอดี

---

## ผลลัพธ์ที่ได้

หลังจาก deploy ขึ้น internet ได้ไม่กี่ชั่วโมง มี bot เริ่มเข้ามา scan แล้ว ข้อมูลที่เก็บได้เบื้องต้นรวมมีดังนี้

### IP ที่เข้ามามากที่สุด

| อันดับ | IP               | จำนวนครั้ง |
|--------|------------------|------------|
| 1      | 170.130.201.38   | 10 ครั้ง   |
| 2      | 51.158.205.203   | 6 ครั้ง    |
| 3      | 45.148.10.121    | 5 ครั้ง    |
| 4      | 47.84.101.65     | 3 ครั้ง    |
| 5      | 146.190.83.66    | 3 ครั้ง    |

### Password ที่ bot ใช้บ่อยที่สุด

| อันดับ | Password | จำนวนครั้ง |
|--------|----------|------------|
| 1      | admin    | 7 ครั้ง    |
| 2      | orangepi | 4 ครั้ง    |
| 3      | P        | 4 ครั้ง    |
| 4      | root     | 1 ครั้ง    |
| 5      | support  | 1 ครั้ง    |

---

## สิ่งที่น่าสนใจที่เจอ

### 1. Bot เข้ามาเร็วมาก

ผมไม่ได้บอกใครว่ามี server ตัวนี้อยู่ ไม่มี domain ไม่ได้โพสต์ที่ไหน แต่ภายในไม่กี่ชั่วโมงหลัง deploy มี bot เข้ามา scan แล้ว แสดงว่าน่าจะมีระบบที่ scan ทุก IP บน internet ตลอดเวลา ถ้าเชื่อมต่อ internet ก็ไม่มีทางซ่อน server ได้อยู่ดี

### 2. Bot ลอง IoT Default Credentials โดยเฉพาะ

password ที่ bot ลองบ่อยคือ `orangepi/orangepi` ซึ่งเป็น default credential ของ Orange Pi บอร์ดที่นิยมใช้ในโปรเจค IoT แสดงว่า bot พวกนี้มี wordlist ของ IoT device โดยเฉพาะ ไม่ใช่แค่ลอง `admin/admin`

ในฐานะที่ผมเรียนสาย IoT นี่คือ insight สำคัญมาก เพราะ IoT device ที่ไม่เปลี่ยน default password จะโดนโจมตีทันทีที่เชื่อมต่อ internet แน่นอนครับ

### 3. เจอ Bot ที่พยายามใช้เครื่องเป็น Proxy

IP `45.148.10.121` ไม่ได้แค่ login แล้วออก แต่พยายาม TCP forward ไปที่ `turn.cloudflare.com:3478` ด้วย ซึ่งแสดงว่า bot นี้ต้องการใช้เครื่องที่แฮกได้เป็น **proxy/tunnel** เพื่อซ่อนตัวต่อไป พฤติกรรมแบบนี้ซับซ้อนกว่า bot ทั่วไปมาก

### 4. Bot ทำงานแบบ Automated 100%

ทุก bot ที่เข้ามาใช้ `SSH-2.0-libssh2` แทน OpenSSH ปกติ บอกได้ทันทีว่าเป็น automated script ไม่ใช่คนนั่ง SSH เอง แต่ละ session ใช้เวลาแค่ 0.1–1.2 วินาทีแล้วก็ disconnect ออก พฤติกรรมแบบนี้เรียกว่า **Credential Stuffing** คือวนลอง password จากลิสต์ไปเรื่อยๆ ถ้าเจออันไหนผ่านก็รายงานกลับให้ระบบอื่นมา exploit ต่อ

---

## สิ่งที่ได้เรียนรู้

**1. ไม่มี server ไหนซ่อนได้บน internet**  
แค่เปิด port ไว้ก็มีคนมา scan แล้ว ไม่จำเป็นต้องประกาศให้ใครรู้

**2. Default Password อันตรายมาก**  
IoT device ที่ไม่เปลี่ยน default password จะโดนโจมตีทันทีที่เชื่อมต่อ internet เป็นปัญหาใหญ่ของ IoT security ในโลกจริง

**3. Log และ Monitoring สำคัญมาก**  
ถ้าไม่มี Grafana dashboard และ Telegram alert คงไม่รู้เลยว่าเกิดอะไรขึ้นกับ server การ monitor real-time เป็นสิ่งจำเป็นสำหรับทุก server ที่เชื่อมต่อ internet

**4. Data Persistence ต้องคิดตั้งแต่แรก**  
ตอนแรกผมเก็บ log ไว้ใน `/tmp` พอ reboot ข้อมูลหายหมด ต้องย้ายมาเก็บที่ `/var/lib/loki` แทน เป็นบทเรียนเรื่อง production mindset ที่ได้เจอจริงครับ

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

*โปรเจคนี้ทำเพื่อการศึกษาเท่านั้น Honeypot ถูก deploy บน isolated VM บน GCP ไม่มีการโจมตีระบบจริงใดๆ*
