# 🛠️ Building a Portable Network Tool with Raspberry Pi Zero 2W and AI Assistance

> ⚠️ **Disclaimer:** All tests in this project were performed exclusively on my own network. Unauthorized access to third-party networks is illegal. This article is intended for **educational purposes only**.

---

## 📖 Introduction

When I got my hands on a **Raspberry Pi Zero 2W**, I started thinking about what to build with it. The idea that stuck was a **portable network testing tool** — small enough to carry in my backpack, powered by a power bank, and accessible from anywhere via Bluetooth without needing an internet connection.

The interesting twist: I built this project almost entirely with **AI assistance**. I used ChatGPT, Grok, and Claude as a team. I provided the ideas, described the problems, and guided the direction — the AI wrote the solutions.

This article is both a **technical guide** and an honest write-up about developing a project with AI tools.

---

## 🧰 Hardware

| Component | Details |
|---|---|
| **Board** | Raspberry Pi Zero 2W |
| **Operating System** | Raspberry Pi OS Lite (64-bit) |
| **Power** | Power Bank (5V/2A) |
| **Network Adapter** | USB WiFi adapter (monitor mode supported) |
| **Connectivity** | Bluetooth (offline access) + SSH over WiFi |

---

## ⚙️ Setup Process

### 1. First Boot and Bluetooth Connection

The Pi Zero 2W has built-in Bluetooth and WiFi, which made the initial **headless setup** much easier. I established the first connection via **Bluetooth Serial** — no monitor, no keyboard needed.

I got the step-by-step commands from the AI for pairing via `bluetoothctl` and enabling the serial port service:

```bash
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
bluetoothctl
```

Inside `bluetoothctl`:

```
agent on
default-agent
discoverable on
pairable on
```

I connected from my phone using a Bluetooth terminal app and ran all initial commands from there.

---

### 2. Automatic Bluetooth Service — Offline Access

This was the **most critical part** of the project. The goal: after booting, the Pi should automatically set up the Bluetooth RFCOMM connection on its own within ~2 minutes, so I could access it **anywhere without internet** — just Bluetooth range.

This meant:
- 🏠 At home, outdoors, in a network-free environment — **no difference**
- 🔋 If there's a power bank, the Pi runs. If there's Bluetooth, there's access.
- 🚫 No requirement to be on the same WiFi network

**Setting up the RFCOMM service:**

```bash
sudo nano /etc/systemd/system/rfcomm.service
```

Service file content:

```ini
[Unit]
Description=RFCOMM Bluetooth Serial Service
After=bluetooth.target
Requires=bluetooth.target

[Service]
ExecStartPre=/bin/sleep 10
ExecStart=/usr/bin/rfcomm watch hci0 1 getty rfcomm0 115200 vt100 -a pi
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> 💡 `ExecStartPre=/bin/sleep 10` — A 10-second delay to ensure Bluetooth is fully initialized before RFCOMM starts. The AI suggested this; without it, the service would intermittently fail to start.

**Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable rfcomm.service
sudo systemctl start rfcomm.service
```

**Result:** About 2 minutes after boot, the Pi is reachable over Bluetooth RFCOMM. Using a Bluetooth terminal app (e.g. *Serial Bluetooth Terminal*) on my phone, I get **full terminal access** — no internet required, no shared network required.

---

### 3. WiFi Adapter and SSH Setup

After establishing Bluetooth access, I connected the USB WiFi adapter — the one that would handle actual network operations. The key requirement was **monitor mode** support.

Check if the adapter is recognized:

```bash
lsusb
iwconfig
```

Enable SSH permanently:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Connect from laptop via SSH:

```bash
ssh pi@<raspberry-ip>
```

---

### 4. Installing the Tools

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nmap aircrack-ng net-tools wireless-tools
```

| Tool | Purpose |
|---|---|
| `nmap` | Scans devices on the network, detects open ports |
| `aircrack-ng` | WiFi packet analysis, deauth testing |
| `net-tools` / `wireless-tools` | Network interface management |

---

### 5. Switching to Monitor Mode

Monitor mode is required for packet capture and deauth testing:

```bash
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo ip link set wlan1 up
```

Verify:

```bash
iwconfig wlan1
# Output should show: Mode:Monitor
```

---

### 6. Network Scanning

```bash
# Quick scan — find all devices on the network
sudo nmap -sn 192.168.1.0/24

# Detailed port scan on a specific device
sudo nmap -sV 192.168.1.X
```

---

### 7. Deauth Testing *(On Your Own Network Only)*

A deauthentication attack sends **802.11 deauth frames** to disconnect a client from a WiFi network. This is used to test the robustness of your own network setup.

```bash
# List nearby networks
sudo airodump-ng wlan1

# Target a specific network
sudo airodump-ng --bssid <ROUTER_MAC> -c <CHANNEL> wlan1

# Send deauth frames (your own network only!)
sudo aireplay-ng --deauth 10 -a <ROUTER_MAC> wlan1
```

---

## 🐛 Problems Encountered and Solutions

### ❌ Problem 1: Bluetooth service wasn't starting

`bluetoothctl` ran but no devices were visible. When I described the situation to the AI, it identified a **dependency issue** between `bluetooth.service` and `bluetooth.target` and suggested:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart bluetooth
```

### ❌ Problem 2: WiFi adapter didn't support monitor mode

The first adapter gave an error on `iw dev wlan1 set type monitor`. The AI clarified that not all USB WiFi adapters support monitor mode, and recommended chipsets like **Realtek RTL8188** or **Atheros AR9271**.

### ❌ Problem 3: Services weren't persisting across reboots

SSH and RFCOMM had to be started manually after every boot. The fix was simply `systemctl enable` — obvious in hindsight, but easy to overlook mid-setup.

---

## 🤖 Working with AI

I used three different AI tools throughout the project:

| AI | Used For |
|---|---|
| **ChatGPT** | General installation steps and command explanations |
| **Grok** | Debugging error messages, interpreting system logs |
| **Claude** | Understanding service interactions, grasping `systemd` architecture |

I never used any of them as a *"write this for me"* machine. I described my problem, applied the suggested solution, reported the result. This back-and-forth sometimes went **3–4 rounds**.

The most valuable thing: the AI also explained **why** something worked, not just **what** to do. I wasn't just copy-pasting — I was actually learning as I went.

---

## ✅ Conclusion

The **Raspberry Pi Zero 2W + power bank** combination is a near-perfect base for a portable network tool. Small, silent, low power draw.

The **automatic Bluetooth service** was the game-changer. It removed the biggest constraint — needing an internet connection or a shared WiFi network — and made the device truly portable in every sense.

AI assistance sped up the build significantly. But more importantly, every time something broke, I tried to understand the **underlying reason**, not just the fix. That made the difference.

**Next step:** A simple web interface — being able to trigger a network scan from a browser instead of a terminal would make the whole setup even cleaner.

---

## 📚 Resources

- [Raspberry Pi Official Documentation](https://www.raspberrypi.com/documentation/)
- [Aircrack-ng Documentation](https://www.aircrack-ng.org/documentation.html)
- [Nmap Reference Guide](https://nmap.org/book/man.html)

---

*This article is based on my personal experience. All tests were conducted on my own network in a controlled environment.*

# LANG: TUR

# 🛠️ Raspberry Pi Zero 2W ve Yapay Zeka Desteğiyle Taşınabilir Ağ Aracı Yapımı

> ⚠️ **Sorumluluk Reddi:** Bu projede yapılan tüm testler yalnızca kendi ağımda gerçekleştirilmiştir. Başkasının ağına izinsiz erişim yasal suçtur. Bu makale yalnızca **eğitim amaçlıdır**.

---

## 📖 Giriş

**Raspberry Pi Zero 2W**'yi ele geçirdiğimde ne yapacağımı düşünmeye başladım. Aklıma takılan fikir şuydu: **taşınabilir bir ağ test aracı** — çantamda taşıyabildiğim kadar küçük, power bank ile çalışan ve internet bağlantısına ihtiyaç duymadan her yerden Bluetooth üzerinden erişilebilen bir cihaz.

İşin ilginç kısmı şu: Bu projeyi neredeyse tamamen **yapay zeka desteğiyle** tamamladım. ChatGPT, Grok ve Claude'u bir ekip gibi kullandım. Ben fikri verdim, sorunları tarif ettim, yönlendirdim — yapay zekalar çözümleri yazdı.

Bu makale hem bir **teknik rehber** hem de yapay zeka araçlarıyla proje geliştirme üzerine dürüst bir deneyim yazısı.

---

## 🧰 Donanım

| Bileşen | Detay |
|---|---|
| **Ana Kart** | Raspberry Pi Zero 2W |
| **İşletim Sistemi** | Raspberry Pi OS Lite (64-bit) |
| **Güç Kaynağı** | Power Bank (5V/2A) |
| **Ağ Adaptörü** | USB WiFi adaptör (monitor mode destekli) |
| **Bağlantı** | Bluetooth (çevrimdışı erişim) + WiFi üzerinden SSH |

---

## ⚙️ Kurulum Süreci

### 1. İlk Boot ve Bluetooth Bağlantısı

Pi Zero 2W'nin yerleşik Bluetooth ve WiFi'ı, **monitörsüz kurulumu** çok kolaylaştırdı. İlk bağlantıyı **Bluetooth Serial** üzerinden kurdum — monitör ya da klavyeye gerek kalmadı.

`bluetoothctl` ile eşleştirme ve seri port servisini etkinleştirme adımlarını yapay zekadan aldım:

```bash
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
bluetoothctl
```

`bluetoothctl` içinde:

```
agent on
default-agent
discoverable on
pairable on
```

Telefonumdan bir Bluetooth terminal uygulamasıyla bağlandım ve tüm ilk komutları buradan çalıştırdım.

---

### 2. Otomatik Bluetooth Servisi — Çevrimdışı Erişim

Bu, projenin **en kritik parçasıydı**. Hedef şuydu: Pi boot olduktan ~2 dakika sonra, Bluetooth RFCOMM bağlantısını **kendisi otomatik olarak kursun** ve ben internete ihtiyaç duymadan, yalnızca Bluetooth menzilinde her yerden erişebileyim.

Bu şu anlama geliyordu:
- 🏠 Evde, dışarıda, internetsiz ortamda — **hiçbir fark yok**
- 🔋 Power bank varsa Pi çalışıyor. Bluetooth varsa erişim var.
- 🚫 Aynı WiFi ağında olmak zorunluluğu ortadan kalkıyor

**RFCOMM servisini oluşturma:**

```bash
sudo nano /etc/systemd/system/rfcomm.service
```

Servis dosyasının içeriği:

```ini
[Unit]
Description=RFCOMM Bluetooth Serial Service
After=bluetooth.target
Requires=bluetooth.target

[Service]
ExecStartPre=/bin/sleep 10
ExecStart=/usr/bin/rfcomm watch hci0 1 getty rfcomm0 115200 vt100 -a pi
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> 💡 `ExecStartPre=/bin/sleep 10` — RFCOMM başlamadan önce Bluetooth servisinin tam olarak ayağa kalkmasını beklemek için 10 saniyelik gecikme. Bunu yapay zeka önerdi; bu olmadan servis zaman zaman başlamıyordu.

**Servisi etkinleştirme ve başlatma:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable rfcomm.service
sudo systemctl start rfcomm.service
```

**Sonuç:** Pi boot ettikten yaklaşık 2 dakika sonra Bluetooth RFCOMM üzerinden erişilebilir hale geliyor. Telefonumdan bir Bluetooth terminal uygulaması (ör. *Serial Bluetooth Terminal*) ile bağlanıp **tam terminal erişimi** elde edebiliyorum — internet yok, ortak ağ yok, sadece Bluetooth menzili yeterli.

---

### 3. WiFi Adaptörü ve SSH Kurulumu

Bluetooth erişimi sağlandıktan sonra, asıl ağ işlemlerini üstlenecek USB WiFi adaptörünü bağladım. Temel gereksinim **monitor mode** desteğiydi.

Adaptörün tanınıp tanınmadığını kontrol:

```bash
lsusb
iwconfig
```

SSH'ı kalıcı olarak etkinleştirme:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Aynı ağdayken dizüstü bilgisayardan bağlanma:

```bash
ssh pi@<raspberry-ip>
```

---

### 4. Araçların Kurulumu

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nmap aircrack-ng net-tools wireless-tools
```

| Araç | Ne İşe Yarıyor |
|---|---|
| `nmap` | Ağdaki cihazları tarar, açık portları tespit eder |
| `aircrack-ng` | WiFi paket analizi, deauth testi |
| `net-tools` / `wireless-tools` | Ağ arayüzü yönetimi |

---

### 5. Monitor Mode'a Geçiş

Paket yakalama ve deauth testi için WiFi adaptörünün **monitor mode**'da çalışması gerekiyor:

```bash
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo ip link set wlan1 up
```

Doğrulama:

```bash
iwconfig wlan1
# Çıktıda: Mode:Monitor görünmeli
```

---

### 6. Ağ Tarama

```bash
# Hızlı tarama — ağdaki tüm cihazları bul
sudo nmap -sn 192.168.1.0/24

# Belirli bir cihazda detaylı port taraması
sudo nmap -sV 192.168.1.X
```

---

### 7. Deauth Testi *(Yalnızca Kendi Ağınızda)*

Deauthentication saldırısı, bir WiFi istemcisini ağdan düşürmek için **802.11 deauth çerçeveleri** gönderir. Bu, kendi ağ kurulumunuzun sağlamlığını test etmek için kullanılır.

```bash
# Çevredeki ağları listele
sudo airodump-ng wlan1

# Belirli bir ağı hedef al
sudo airodump-ng --bssid <ROUTER_MAC> -c <KANAL> wlan1

# Deauth gönder (yalnızca kendi ağına!)
sudo aireplay-ng --deauth 10 -a <ROUTER_MAC> wlan1
```

---

## 🐛 Karşılaşılan Sorunlar ve Çözümler

### ❌ Sorun 1: Bluetooth servisi başlamıyordu

`bluetoothctl` çalışıyordu ama hiçbir cihaz görünmüyordu. Durumu yapay zekaya anlattığımda, `bluetooth.service` ile `bluetooth.target` arasındaki **bağımlılık sorununu** tespit etti ve şu çözümü önerdi:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart bluetooth
```

### ❌ Sorun 2: WiFi adaptörü monitor mode'u desteklemiyordu

İlk adaptörde `iw dev wlan1 set type monitor` komutu hata verdi. Yapay zeka, tüm USB WiFi adaptörlerin monitor mode'u desteklemediğini açıkladı ve **Realtek RTL8188** veya **Atheros AR9271** çipsetli adaptörleri önerdi.

### ❌ Sorun 3: Servisler her boot'ta sıfırlanıyordu

SSH ve RFCOMM her yeniden başlatmada elle başlatmak gerekiyordu. Çözüm `systemctl enable` ile servisleri boot'a eklemekti — geriye dönünce çok basit görünüyor ama kurulum ortasında gözden kaçıyor.

---

## 🤖 Yapay Zeka ile Çalışma Deneyimi

Proje boyunca üç farklı yapay zeka aracı kullandım:

| Yapay Zeka | Kullanım Alanı |
|---|---|
| **ChatGPT** | Genel kurulum adımları ve komut açıklamaları |
| **Grok** | Hata mesajlarını debug etme, sistem loglarını yorumlama |
| **Claude** | Servislerin birbirleriyle ilişkisini anlama, `systemd` mimarisini kavrama |

Hiçbirini *"şunu yaz"* modunda kullanmadım. Sorunumu tarif ettim, önerilen çözümü uyguladım, sonucu bildirdim. Bu gidip-gelme süreci bazen **3–4 turu** buldu.

En değerli şey şuydu: yapay zeka bir şeyin **neden** çalıştığını da açıklıyordu, yalnızca **ne yapılacağını** değil. Sadece kopyala-yapıştır değil, öğrenerek ilerlediğimi hissettim.

---

## ✅ Sonuç

**Raspberry Pi Zero 2W + power bank** kombinasyonu, taşınabilir bir ağ aracı için neredeyse mükemmel bir temel. Küçük, sessiz, düşük güç tüketimi.

**Otomatik Bluetooth servisi** her şeyi değiştirdi. En büyük kısıtı — internet bağlantısı veya ortak bir WiFi ağı gerekliliğini — ortadan kaldırdı ve cihazı her anlamda gerçekten taşınabilir hale getirdi.

Yapay zeka desteği yapım sürecini önemli ölçüde hızlandırdı. Ama daha önemlisi — her sorun çıktığında yalnızca çözümü değil, **altta yatan nedeni** anlamaya çalıştım. Farkı yaratan buydu.

**Sonraki adım:** Basit bir web arayüzü — terminal yerine tarayıcıdan ağ taraması başlatabilmek her şeyi çok daha temiz hale getirir.

---

## 📚 Kaynaklar

- [Raspberry Pi Resmi Dokümantasyonu](https://www.raspberrypi.com/documentation/)
- [Aircrack-ng Dokümantasyonu](https://www.aircrack-ng.org/documentation.html)
- [Nmap Kullanım Kılavuzu](https://nmap.org/book/man.html)

---

*Bu makale kişisel deneyimlerime dayanmaktadır. Tüm testler kendi ağımda, kontrollü ortamda yapılmıştır.*

