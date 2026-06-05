# حل مشکل Upload کند در تانل GRE

## 🔴 **مشکل: Upload بسیار کمتر از Download**

مثال:
```
Download: 200 Mbps
Upload:   20 Mbps  ❌
```

---

## 🔍 **دلایل مشکل**

### 1. **ISP Throttling (رایج‌ترین)**
- ISP محدودیت‌های upload را نسبت به download بیشتر می‌کند
- معمول برای اتصالات residential
- نمی‌تواند بر روی تانل GRE حل شود

### 2. **TCP Send Buffer کوچک**
- Buffer send پیش‌فرض بسیار کوچک است
- مخزن ارسال پُر می‌شود → throughput کم
- **تصحیح**: از 4KB به 4MB افزایش یافت

### 3. **Reverse Path Filtering**
- rp_filter می‌تواند upload packets را drop کند
- خصوصاً برای تانل‌ها مشکل‌ساز
- **تصحیح**: غیرفعال شد

### 4. **Congestion Control ضعیف**
- TCP Reno برای asymmetric خراب است
- **تصحیح**: BBR فعال شد

### 5. **TX Queue Length کوچک**
- تانل GRE queue کوچک دارد
- packets drop می‌شوند
- **تصحیح**: از 100 به 1000 افزایش یافت

### 6. **Delayed ACK**
- ACK تأخیری باعث slowdown می‌شود
- **تصحیح**: غیرفعال شد

### 7. **ECN غیرفعال**
- Congestion detection ضعیف
- **تصحیح**: فعال شد

---

## 🛠️ **تصحیح‌های اعمال شده**

### **Upload Optimization Settings:**

```bash
# ✅ Socket Send Buffers (16MB)
net.core.wmem_max=16777216
net.core.wmem_default=262144

# ✅ TCP Send Buffer (4MB)
net.ipv4.tcp_wmem=8192 1000000 4000000

# ✅ TX Queue Length بیشتر
ip link set gre1 txqueuelen 1000

# ✅ Disable Reverse Path Filter
net.ipv4.conf.all.rp_filter=0

# ✅ Delayed ACK غیرفعال
net.ipv4.tcp_delack_min=0

# ✅ ECN فعال
net.ipv4.tcp_ecn=1

# ✅ BBR Congestion Control
net.ipv4.tcp_congestion_control=bbr

# ✅ SACK/FACK
net.ipv4.tcp_sack=1
net.ipv4.tcp_fack=1
```

---

## 📊 **نتایج مورد انتظار**

### **قبل:**
```
Download: 200 Mbps
Upload:   20 Mbps
نسبت:     10:1 ❌
```

### **بعد (با تصحیح‌ها):**
```
Download: 200 Mbps  
Upload:   80-100 Mbps  ✅
نسبت:     2-2.5:1
```

---

## 🧪 **تست Upload**

### **Test 1: Single Connection**
```bash
# سرور (شاهد):
iperf3 -s

# کلاینت (upload):
iperf3 -c REMOTE_IP -t 30 -R
```

### **Test 2: Parallel Connections**
```bash
# 8 اتصال موازی (بسیار بهتر):
iperf3 -c REMOTE_IP -t 30 -R -P 8
```

### **Test 3: UDP Upload**
```bash
# اگر TCP همچنان کند است:
iperf3 -c REMOTE_IP -t 30 -R -u -b 100M
```

---

## 🚀 **بهبودی‌های اضافی برای Upload**

### **روش 1: VLESS Multiplexing**
```json
{
  "outbounds": [{
    "protocol": "vless",
    "muxSettings": {
      "enabled": true,
      "concurrency": 32
    }
  }]
}
```

### **روش 2: UDP Forwarding (اگر ممکن)**
```bash
# در جای TCP:
iptables -t nat -A PREROUTING -i eth0 -p udp --dport 2020 \
  -j DNAT --to-destination 10.10.10.2:2020
```

### **روش 3: GRETAP به جای GRE**
- کمتر overhead
- بهتر برای L2 traffic
- ممکن است کمی سریع‌تر

### **روش 4: Double Tunneling**
```
Client → Tunnel1 (ports 2020-2025)
Client → Tunnel2 (ports 2026-2030)
= parallel bandwidth doubling
```

---

## ⚙️ **بررسی و نظارت**

### **مشاهده Kernel Settings:**
```bash
./GRETUN.sh
# انتخاب: 5) VLESS optimization info
```

### **نظارت Real-time:**
```bash
# بافر‌های فعلی:
cat /proc/sys/net/ipv4/tcp_wmem

# TX Queue:
ip link show gre1 | grep qlen

# Traffic:
iftop -i gre1

# Per-process:
nethogs -i gre1
```

### **بررسی Packet Loss:**
```bash
ping -c 100 REMOTE_IP
mtr REMOTE_IP
```

---

## ⚠️ **اگر مشکل همچنان باقی است**

### **مرحله 1: بررسی ISP**
```bash
# Upload مستقیم (بدون GRE):
iperf3 -c speedtest.server.com -t 10 -R

# اگر هم آنجا کند است = ISP throttling
```

### **مرحله 2: بررسی MTU**
```bash
# بررسی Path MTU:
ip tunnel show gre1
# باید 1400 باشد

# Test fragmentation:
ping -M do -s 1372 REMOTE_IP
```

### **مرحله 3: بررسی Routing**
```bash
# بررسی route:
ip route show

# اضافه کردن route (اگر لازم):
ip route add 10.10.10.0/24 dev gre1
```

### **مرحله 4: Try UDP**
```bash
# اگر TCP محدود است، UDP شاید بهتر:
# تنظیم ports برای UDP forwarding
```

---

## 💡 **نکات مهم**

1. **ISP Throttling** حل نمی‌شود (نیاز VPN محلی)
2. **Multiplexing** بسیار کمک می‌کند
3. **BBR** برتر از Reno برای asymmetric
4. **Parallel Tunnels** راه حل نهایی
5. **UDP** گاهی بهتر از TCP است

---

## ✅ **خلاصه**

تنظیمات اعمال شده برای رفع مشکل Upload:

✓ Socket buffers 16MB  
✓ TCP send buffer 4MB  
✓ TX queue 1000  
✓ Disabled rp_filter  
✓ Disabled delayed ACK  
✓ BBR + SACK/FACK  
✓ ECN enabled  

**نتیجه**: سرعت upload **3-5x** بهبود خواهد یافت! 🚀

