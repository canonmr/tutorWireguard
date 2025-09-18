# Panduan Lengkap: Membangun Site-to-Site VPN dengan Wireguard dan Mikrotik

## Daftar Isi
1. [Pengenalan Wireguard dan Manfaatnya](#pengenalan-wireguard-dan-manfaatnya)
2. [Persiapan dan Inventarisasi Infrastruktur](#persiapan-dan-inventarisasi-infrastruktur)
3. [Perencanaan Jaringan](#perencanaan-jaringan)
4. [Setup VPS Wireguard (wg-easy)](#setup-vps-wireguard-wg-easy)
5. [Konfigurasi Client di wg-easy](#konfigurasi-client-di-wg-easy)
6. [Konfigurasi Mikrotik Lokasi Pertama](#konfigurasi-mikrotik-lokasi-pertama)
7. [Konfigurasi Mikrotik Lokasi Kedua](#konfigurasi-mikrotik-lokasi-kedua)
8. [Setup Routing di VPS](#setup-routing-di-vps)
9. [Testing dan Verifikasi](#testing-dan-verifikasi)
10. [Troubleshooting](#troubleshooting)
11. [Monitoring dan Maintenance](#monitoring-dan-maintenance)
12. [Optimisasi dan Best Practices](#optimisasi-dan-best-practices)

---

## Pengenalan Wireguard dan Manfaatnya

### Apa Itu Wireguard?

Wireguard adalah protokol VPN modern yang dirancang untuk menjadi sederhana, cepat, dan aman. Dikembangkan sebagai alternatif yang lebih efisien dibanding protokol VPN tradisional seperti IPSec atau OpenVPN, Wireguard menggunakan kriptografi modern dengan overhead yang minimal.

### Mengapa Diperlukan?

Dalam era digital saat ini, banyak organisasi memiliki multiple lokasi yang memerlukan konektivitas aman antar-jaringan. Tantangan umum yang dihadapi:

1. **Keterbatasan IP Publik**: Provider internet residential (seperti Indihome) tidak menyediakan IP publik static
2. **Biaya Dedicated Line**: Koneksi MPLS atau dedicated line memiliki biaya tinggi
3. **Keamanan Internet**: Traffic antar-lokasi melalui internet publik memerlukan enkripsi
4. **Kompleksitas Konfigurasi**: Protokol VPN tradisional memerlukan setup yang rumit
5. **Maintenance Overhead**: Solusi proprietary memerlukan maintenance khusus

### Manfaat Wireguard untuk Site-to-Site VPN

#### Keunggulan Teknis:
- **Performance Superior**: Throughput tinggi dengan CPU usage minimal
- **Battery Efficient**: Cocok untuk mobile dan IoT devices
- **Simple Configuration**: File konfigurasi hanya beberapa baris
- **Modern Cryptography**: ChaCha20 untuk enkripsi, Poly1305 untuk autentikasi
- **Roaming Support**: Seamless handoff saat IP address berubah

#### Keunggulan Operasional:
- **Cost Effective**: Menggunakan internet existing + VPS murah
- **Scalable**: Mudah menambah lokasi baru
- **Cross-Platform**: Support Windows, Linux, macOS, iOS, Android
- **Open Source**: Audit code tersedia, no vendor lock-in
- **Low Latency**: Overhead protokol minimal

### Potensi Manfaat Implementasi

#### Untuk Organisasi:
1. **Unified Network**: Semua lokasi terkoneksi seperti satu jaringan besar
2. **Centralized Services**: File server, database, aplikasi internal dapat diakses dari semua lokasi
3. **Remote Work Support**: Karyawan dapat akses resource kantor dari rumah
4. **Disaster Recovery**: Backup dan replikasi data antar-lokasi
5. **Cost Reduction**: Mengurangi biaya dedicated line hingga 70-90%

#### Untuk IT Operations:
1. **Centralized Management**: Admin dapat manage semua lokasi dari pusat
2. **Monitoring Terpusat**: CCTV, sensor IoT, sistem monitoring dari semua lokasi
3. **Software Distribution**: Update dan patch dapat didistribusi terpusat
4. **Troubleshooting Remote**: Akses perangkat di lokasi remote untuk maintenance

#### Untuk End Users:
1. **Seamless Access**: Akses file dan aplikasi seperti di jaringan lokal
2. **Better Performance**: Latency lebih rendah dibanding VPN client tradisional
3. **Transparent Operation**: User tidak perlu tahu adanya VPN
4. **Always Connected**: Koneksi otomatis dan persistent

### Kendala yang Dijembatani Wireguard

#### 1. **NAT Traversal Problem**
**Masalah**: Router ISP menggunakan NAT, membuat koneksi direct antar-lokasi tidak mungkin
**Solusi Wireguard**: Menggunakan VPS sebagai relay point dengan UDP hole punching

#### 2. **Dynamic IP Issues**  
**Masalah**: IP address berubah-ubah, koneksi terputus
**Solusi Wireguard**: Automatic endpoint discovery dan roaming support

#### 3. **Complex Firewall Rules**
**Masalah**: Setup IPSec memerlukan port forwarding dan firewall rules rumit
**Solusi Wireguard**: Hanya memerlukan 1 UDP port (51820)

#### 4. **Performance Bottleneck**
**Masalah**: VPN tradisional memiliki overhead tinggi
**Solusi Wireguard**: Kernel-space implementation dengan overhead minimal

#### 5. **Maintenance Complexity**
**Masalah**: Certificate management, key rotation, compatibility issues
**Solusi Wireguard**: Simple key exchange, no certificate authority needed

#### 6. **Vendor Lock-in**
**Masalah**: Proprietary solution sulit di-migrate
**Solusi Wireguard**: Open standard, portable configuration

---

## Persiapan dan Inventarisasi Infrastruktur

### Langkah 1: Inventarisasi Hardware Existing

#### A. Survey Mikrotik di Setiap Lokasi

**Informasi yang Harus Dikumpulkan:**

1. **Model dan Spesifikasi Router**
   ```
   Model: [Contoh: RB4011iGS+]
   RouterOS Version: [Harus v7.x untuk native Wireguard]
   CPU: [Architecture dan clock speed]
   RAM: [Minimal 256MB untuk Wireguard]
   Storage: [Available space untuk config]
   ```

2. **Interface dan Konektivitas**
   ```
   WAN Interface: [ether1, ether-WAN, dll]
   LAN Interface: [bridge-LAN, ether2-10, dll]
   WiFi Capability: [wlan1, external AP, CAPsMAN]
   Port Count: [Jumlah ethernet port tersedia]
   ```

3. **Konfigurasi IP Current**
   ```
   WAN IP: [Dynamic dari DHCP ISP]
   LAN Subnet: [192.168.x.0/24]
   Gateway IP: [192.168.x.1]
   DHCP Range: [192.168.x.10-254]
   DNS Servers: [Primary, Secondary]
   ```

#### Command untuk Inventarisasi di Mikrotik:

> **⚡ QUICK CHECK**: Jika Mikrotik Anda sudah dikonfigurasi dan berjalan normal, cukup jalankan command berikut untuk mendapatkan informasi yang diperlukan tanpa mengubah konfigurasi existing.

```bash
# Cek model dan versi
/system resource print
/system package print

# Cek interface
/interface print

# Cek IP configuration
/ip address print
/ip route print

# Cek DHCP setup
/ip dhcp-server print
/ip dhcp-server network print

# Cek firewall rules existing
/ip firewall filter print
/ip firewall nat print
```

#### B. Survey Router ISP di Setiap Lokasi

**Informasi yang Diperlukan:**

1. **Spesifikasi Router ISP**
   ```
   Brand/Model: [Huawei, TP-Link, ZTE, dll]
   Firmware Version: [Version number]
   Admin Access: [Username/password tersedia?]
   Configuration Access: [Web interface, telnet, SSH]
   ```

2. **Network Configuration**
   ```
   WAN IP: [IP yang diberikan ISP]
   Gateway ISP: [Gateway dari provider]
   DNS ISP: [Primary/Secondary DNS]
   DHCP Range: [Range IP untuk LAN]
   WiFi Settings: [SSID, security type]
   ```

3. **Advanced Settings**
   ```
   UPnP Status: [Enabled/Disabled]
   Port Forwarding: [Rules existing]
   DMZ Setting: [Available/Used]
   Firewall Level: [Low/Medium/High]
   ```

**Cara Mengakses Router ISP:**
```bash
# Cek gateway default (biasanya router ISP)
ipconfig /all    # Windows
ip route show    # Linux

# Common default IPs router ISP:
192.168.1.1      # Indihome, First Media
192.168.0.1      # Biznet, MyRepublic  
10.0.0.1         # Some fiber providers
```

#### C. Survey Koneksi Internet

**Speed Test dari Setiap Lokasi:**
```bash
# Test bandwidth
speedtest-cli

# Test latency ke beberapa target
ping 8.8.8.8 -c 10
ping 1.1.1.1 -c 10

# Test MTU size (penting untuk Wireguard)
ping -f -l 1472 8.8.8.8    # Windows
ping -M do -s 1472 8.8.8.8 # Linux
```

**Informasi Provider:**
```
ISP Provider: [Telkom/Indihome, First Media, dll]
Package Speed: [Download/Upload Mbps]
IP Type: [Dynamic/Static]
IPv6 Support: [Available/Not available]
Port Restrictions: [Blocked ports if any]
```

### Langkah 2: Inventarisasi VPS/Cloud Server

> **💡 SKIP POINT**: Jika VPS Anda sudah ada dengan Wireguard pre-installed atau managed service, cukup catat informasi berikut: IP VPS, domain (jika ada), port Wireguard (default 51820), dan access ke web interface management.

#### A. Spesifikasi VPS yang Dibutuhkan

**Minimum Requirements:**
```
CPU: 1 vCPU (2 vCPU recommended)
RAM: 1GB (2GB recommended untuk >10 clients)
Storage: 20GB SSD
Bandwidth: Unlimited atau >1TB/month
OS: Ubuntu 20.04 LTS atau CentOS 8
```

**Provider Recommendations:**
```
International: DigitalOcean, Vultr, Linode, AWS
Domestic: Biznet Gio, Idcloudhost, Qwords
Budget: Contabo, Hetzner, OVH
```

#### B. Network Requirements VPS

**Port dan Protokol:**
```
Wireguard: 51820/UDP (customizable)
Management: 51821/TCP (wg-easy web interface)
SSH: 22/TCP (untuk admin)
Optional: 80/TCP, 443/TCP (jika pakai domain)
```

**Firewall Configuration:**
```bash
# Buka port yang diperlukan
ufw allow 22/tcp
ufw allow 51820/udp
ufw allow 51821/tcp
ufw enable
```

#### C. Domain dan DNS (Optional tapi Recommended)

**Manfaat Menggunakan Domain:**
- Tidak perlu update config jika IP VPS berubah
- Easier to remember dan professional
- SSL certificate untuk web interface

**Setup DNS:**
```
Type: A Record
Name: vpn (atau subdomain lain)
Value: [IP_ADDRESS_VPS]
TTL: 3600 (1 hour)
```

### Langkah 3: Planning IP Address Scheme

#### A. Subnet Planning

**Prinsip IP Planning:**
1. Setiap lokasi harus memiliki subnet berbeda
2. Wireguard network harus tidak conflict dengan subnet existing
3. Reserved range untuk future expansion
4. Dokumentasi lengkap untuk troubleshooting

**Contoh Schema:**
```
Wireguard Network: 10.8.0.0/24
├── VPS Server: 10.8.0.1
├── Lokasi A: 10.8.0.2
├── Lokasi B: 10.8.0.4  
├── Lokasi C: 10.8.0.6 (future)
└── Mobile Users: 10.8.0.100-200

Local Networks:
├── Lokasi A: 192.168.110.0/24
├── Lokasi B: 192.168.113.0/24
└── Lokasi C: 192.168.120.0/24 (future)
```

#### B. Conflict Detection

**Check untuk IP Conflicts:**
```bash
# Di setiap Mikrotik, check existing subnets
/ip address print
/ip route print

# Pastikan tidak ada overlap dengan:
- 10.8.0.0/24 (Wireguard range)
- Subnet lokasi lain
- VPN client ranges existing
```

### Langkah 4: Dokumentasi Pre-Implementation

#### A. Network Diagram

Buat diagram sederhana yang menunjukkan:
- Topology existing setiap lokasi
- IP address ranges
- Planned Wireguard connections
- Critical services yang akan diakses

#### B. Implementation Checklist

**Pre-Implementation:**
- [ ] RouterOS version check (minimal v7.x)
- [ ] VPS provisioning dan basic security
- [ ] Domain registration dan DNS setup
- [ ] Backup existing Mikrotik configuration
- [ ] Maintenance window planning
- [ ] Contact persons di setiap lokasi

**Durante Implementation:**
- [ ] VPS setup dan Docker installation
- [ ] wg-easy deployment dan testing
- [ ] Client creation dan configuration download
- [ ] Mikrotik configuration per location
- [ ] Incremental testing per location
- [ ] Full site-to-site testing

**Post Implementation:**
- [ ] Performance testing dan optimization
- [ ] Monitoring setup
- [ ] Documentation update
- [ ] User training
- [ ] Backup procedures

#### C. Rollback Plan

**Jika Implementation Gagal:**
1. Restore Mikrotik configuration dari backup
2. Remove Wireguard interface dan routes
3. Verify original connectivity restored
4. Document issues untuk future reference

**Emergency Contacts:**
- Local IT person per lokasi
- Internet provider support
- VPS provider support
- Mikrotik technical support

---

## Perencanaan Jaringan

### Analisis Topologi Current vs Target

#### Current State (Sebelum Wireguard):
```
Lokasi A: [Internet ISP A] → [Router ISP A] → [Mikrotik A] → [LAN A]
Lokasi B: [Internet ISP B] → [Router ISP B] → [Mikrotik B] → [LAN B]

Masalah: Tidak ada konektivitas antar-lokasi
```

#### Target State (Setelah Wireguard):
```
Lokasi A: [Internet] → [Mikrotik A] ←→ [VPS Wireguard] ←→ [Mikrotik B] ← [Internet] :Lokasi B
                         ↓                   ↑                     ↓
                    [LAN A] ←---- Encrypted Tunnel ----→ [LAN B]
```

### Routing Strategy

**Routing Table Planning:**

*Di Mikrotik Lokasi A:*
```
0.0.0.0/0 → [ISP Gateway]           # Default internet
192.168.113.0/24 → wg-site2site     # Ke Lokasi B via Wireguard
10.8.0.0/24 → wg-site2site          # Wireguard management network
```

*Di VPS:*
```
192.168.110.0/24 → 10.8.0.2         # Subnet Lokasi A via peer A
192.168.113.0/24 → 10.8.0.4         # Subnet Lokasi B via peer B
```

*Di Mikrotik Lokasi B:*
```
0.0.0.0/0 → [ISP Gateway]           # Default internet
192.168.110.0/24 → wg-site2site     # Ke Lokasi A via Wireguard
10.8.0.0/24 → wg-site2site          # Wireguard management network
```

---

## Setup VPS Wireguard (wg-easy)

> **💡 SKIP POINT**: Jika Anda sudah memiliki VPS dengan Wireguard terinstall dari provider hosting (seperti panel Plesk, cPanel, atau layanan managed), Anda dapat melompati ke [Konfigurasi Client di wg-easy](#konfigurasi-client-di-wg-easy) dan menggunakan interface existing yang sudah tersedia.

### Langkah 1: Persiapan VPS

#### A. Initial Server Setup
```bash
# Update sistem
apt update && apt upgrade -y

# Install basic tools
apt install -y curl wget htop nano ufw fail2ban

# Setup basic security
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 51820/udp
ufw allow 51821/tcp
ufw --force enable

# Configure fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

#### B. Docker Installation
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version
```

#### C. Directory Structure Setup
```bash
# Buat directory untuk wg-easy
mkdir -p /opt/wireguard
cd /opt/wireguard

# Setup permissions
chown -R root:root /opt/wireguard
chmod 755 /opt/wireguard
```

### Langkah 2: Konfigurasi wg-easy

#### A. Docker Compose Configuration
```yaml
# File: /opt/wireguard/docker-compose.yml
version: '3.8'

services:
  wg-easy:
    environment:
      # Basic Configuration
      - LANG=en
      - WG_HOST=your-domain.com              # Ganti dengan domain/IP VPS
      - PASSWORD=YourSecurePassword123!      # Ganti dengan password kuat
      - WG_PORT=51820                        # Port Wireguard
      
      # Network Configuration  
      - WG_DEFAULT_ADDRESS=10.8.0.x          # Auto-assign IP range
      - WG_DEFAULT_DNS=1.1.1.1,1.0.0.1      # DNS servers
      - WG_ALLOWED_IPS=10.8.0.0/24           # Default allowed IPs
      
      # Advanced Options
      - WG_PERSISTENT_KEEPALIVE=25           # Keepalive untuk NAT
      - WG_DEFAULT_MTU=1420                  # MTU setting
      
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    volumes:
      - ~/.wg-easy:/etc/wireguard
    ports:
      - "51820:51820/udp"                    # Wireguard port
      - "51821:51821/tcp"                    # Web interface
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1                # Enable IP forwarding
      - net.ipv4.conf.all.src_valid_mark=1   # Enable source validation
```

#### B. Environment Customization

**Untuk Multiple Provider DNS:**
```yaml
- WG_DEFAULT_DNS=1.1.1.1,1.0.0.1,8.8.8.8,8.8.4.4,202.147.193.110
```

**Untuk Custom Network Range:**
```yaml
- WG_DEFAULT_ADDRESS=10.8.0.x
- WG_ALLOWED_IPS=10.8.0.0/24,192.168.0.0/16
```

### Langkah 3: Deployment dan Testing

#### A. Start Services
```bash
# Navigate ke directory
cd /opt/wireguard

# Start container
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f
```

#### B. Initial Testing
```bash
# Test container health
docker exec wg-easy wg show

# Test web interface (dari browser)
# http://[VPS_IP]:51821

# Check port listening
netstat -tulpn | grep :51820
netstat -tulpn | grep :51821
```

#### C. Basic Security Check
```bash
# Check firewall status
ufw status verbose

# Check Docker network
docker network ls
docker network inspect wireguard_default

# Check system resources
htop
df -h
```

---

## Konfigurasi Client di wg-easy

> **🎯 START HERE**: Jika Anda memiliki Wireguard managed service dari hosting provider atau sudah memiliki wg-easy running, mulai dari langkah ini. Anda hanya perlu akses ke web interface management Wireguard.

### Langkah 1: Akses Web Interface

#### A. Login ke wg-easy
1. Buka browser ke `http://[VPS_IP]:51821`
2. Masukkan password yang sudah ditetapkan di docker-compose.yml
3. Verify dashboard menampilkan interface Wireguard dengan 0 clients

#### B. Interface Overview
- **Connected Clients**: Menampilkan jumlah client aktif
- **Total Transfer**: Data yang telah ditransfer
- **Add Client**: Button untuk menambah client baru
- **Client List**: List client existing dengan status

### Langkah 2: Membuat Client untuk Setiap Lokasi

#### A. Client untuk Lokasi Pertama
1. **Klik "Add Client"**
2. **Isi informasi client:**
   ```
   Name: Lokasi-Utama
   (IP akan auto-assign, biasanya 10.8.0.2)
   ```
3. **Klik "Create"**

#### B. Client untuk Lokasi Kedua  
1. **Klik "Add Client"** 
2. **Isi informasi client:**
   ```
   Name: Lokasi-Cabang
   (IP akan auto-assign, biasanya 10.8.0.4)
   ```
3. **Klik "Create"**

### Langkah 3: Konfigurasi Site-to-Site Routing

#### A. Edit Client Lokasi Pertama
1. **Click icon "Edit" (pensil) pada client "Lokasi-Utama"**
2. **Pada bagian "Allowed IPs"**: 
   ```
   10.8.0.0/24
   ```
3. **Pada bagian "Server Allowed IPs"**: 
   ```
   192.168.110.0/24
   ```
   (Masukkan subnet dari lokasi pertama)
4. **Klik "Save"**

#### B. Edit Client Lokasi Kedua
1. **Click icon "Edit" (pensil) pada client "Lokasi-Cabang"**
2. **Pada bagian "Allowed IPs"**: 
   ```
   10.8.0.0/24
   ```
3. **Pada bagian "Server Allowed IPs"**: 
   ```
   192.168.113.0/24
   ```
   (Masukkan subnet dari lokasi kedua)
4. **Klik "Save"**

### Langkah 4: Download Konfigurasi Client

#### A. Download Config untuk Lokasi Pertama
1. **Pada client "Lokasi-Utama", klik icon "Download" atau QR code**
2. **Simpan file dengan nama: `lokasi-utama.conf`**
3. **Buka file dan catat informasi penting:**
   ```
   [Interface]
   PrivateKey = [PRIVATE_KEY_LOKASI_UTAMA]
   Address = 10.8.0.2/24
   
   [Peer]
   PublicKey = [SERVER_PUBLIC_KEY]
   PresharedKey = [PRESHARED_KEY_LOKASI_UTAMA]
   Endpoint = your-domain.com:51820
   AllowedIPs = 10.8.0.0/24, 192.168.113.0/24
   ```

#### B. Download Config untuk Lokasi Kedua
1. **Pada client "Lokasi-Cabang", klik icon "Download" atau QR code**
2. **Simpan file dengan nama: `lokasi-cabang.conf`**
3. **Buka file dan catat informasi penting:**
   ```
   [Interface]
   PrivateKey = [PRIVATE_KEY_LOKASI_CABANG]
   Address = 10.8.0.4/24
   
   [Peer]
   PublicKey = [SERVER_PUBLIC_KEY]
   PresharedKey = [PRESHARED_KEY_LOKASI_CABANG]
   Endpoint = your-domain.com:51820
   AllowedIPs = 10.8.0.0/24, 192.168.110.0/24
   ```

### Langkah 5: Verifikasi Konfigurasi di wg-easy

#### A. Cek Status di Terminal VPS
```bash
# Cek konfigurasi yang sudah dibuat
docker exec wg-easy wg show

# Expected output:
interface: wg0
  public key: [SERVER_PUBLIC_KEY]
  private key: (hidden)
  listening port: 51820

peer: [PUBLIC_KEY_LOKASI_UTAMA]
  preshared key: (hidden)
  allowed ips: 10.8.0.2/32, 192.168.110.0/24

peer: [PUBLIC_KEY_LOKASI_CABANG]
  preshared key: (hidden)
  allowed ips: 10.8.0.4/32, 192.168.113.0/24
```

#### B. Troubleshooting Config Issues
**Jika Allowed IPs salah:**
1. Edit ulang client di web interface
2. Pastikan Server Allowed IPs sesuai dengan subnet masing-masing lokasi
3. Re-download configuration file

**Jika Client tidak muncul di wg show:**
1. Refresh web interface
2. Check Docker logs: `docker logs wg-easy`
3. Restart container jika diperlukan: `docker-compose restart`

---

## Konfigurasi Mikrotik Lokasi Pertama

### Langkah 1: Persiapan dan Akses

#### A. Backup Konfigurasi Existing
```bash
# Login ke Mikrotik via Winbox atau SSH
# Backup konfigurasi current
/export file=backup-pre-wireguard-lokasi1

# Backup semua certificates (jika ada)
/certificate export-certificate numbers=0,1,2,3
```

#### B. Verifikasi Sistem Requirements
```bash
# Cek RouterOS version (harus v7.x)
/system resource print

# Cek available interfaces
/interface print

# Cek routing table current
/ip route print
```

### Langkah 2: Konfigurasi Wireguard Interface

#### A. Buat Wireguard Interface
```bash
/interface wireguard add listen-port=51820 mtu=1420 name=wg-site2site private-key="[PRIVATE_KEY_DARI_CONFIG_LOKASI_UTAMA]"
```

**Parameter Explanation:**
- `listen-port=51820`: Port untuk Wireguard communication
- `mtu=1420`: Maximum Transmission Unit (disesuaikan untuk avoid fragmentation)
- `name=wg-site2site`: Nama interface yang mudah diidentifikasi
- `private-key`: Private key dari file konfigurasi yang sudah didownload

#### B. Assign IP Address ke Interface
```bash
/ip address add address=10.8.0.2/24 interface=wg-site2site comment="Wireguard-Site2Site-IP"
```

#### C. Verifikasi Interface Creation
```bash
# Cek apakah interface sudah dibuat
/interface wireguard print

# Cek IP address assignment
/ip address print where interface=wg-site2site
```

### Langkah 3: Konfigurasi Peer (ke VPS)

#### A. Add Peer ke VPS Server
```bash
/interface wireguard peers add allowed-address=0.0.0.0/0,::/0 endpoint-address=your-domain.com endpoint-port=51820 interface=wg-site2site public-key="[SERVER_PUBLIC_KEY]" preshared-key="[PRESHARED_KEY_LOKASI_UTAMA]" persistent-keepalive=25s comment="VPS-Server"
```

**Parameter Explanation:**
- `allowed-address=0.0.0.0/0,::/0`: Traffic yang diizinkan melalui peer ini
- `endpoint-address`: Domain atau IP VPS
- `endpoint-port=51820`: Port Wireguard di VPS
- `public-key`: Public key VPS server
- `preshared-key`: Additional encryption key
- `persistent-keepalive=25s`: Keep connection alive untuk NAT traversal

#### B. Verifikasi Peer Configuration
```bash
# Cek peer configuration
/interface wireguard peers print

# Expected output menunjukkan peer dengan endpoint ke VPS
```

### Langkah 4: Routing Configuration

#### A. Add Route ke Lokasi Remote
```bash
/ip route add dst-address=192.168.113.0/24 gateway=wg-site2site distance=1 comment="to-Lokasi-Cabang-via-Wireguard"
```

#### B. Add Route untuk Wireguard Management
```bash
/ip route add dst-address=10.8.0.0/24 gateway=wg-site2site distance=1 comment="Wireguard-Management-Network"
```

#### C. Verifikasi Routing Table
```bash
# Lihat routing table lengkap
/ip route print

# Lihat hanya routes via Wireguard
/ip route print where gateway=wg-site2site
```

### Langkah 5: Firewall Configuration

#### A. Allow Established/Related Connections
```bash
/ip firewall filter add action=accept chain=forward connection-state=established,related comment="allow-established-related" place-before=0
```

#### B. Allow Wireguard Traffic
```bash
# Allow traffic from Wireguard interface
/ip firewall filter add action=accept chain=forward in-interface=wg-site2site comment="allow-from-wireguard" place-before=0

# Allow traffic to Wireguard interface  
/ip firewall filter add action=accept chain=forward out-interface=wg-site2site comment="allow-to-wireguard" place-before=0

# Allow Wireguard traffic to router itself
/ip firewall filter add action=accept chain=input in-interface=wg-site2site comment="allow-wg-to-router" place-before=0
```

#### C. NAT Configuration for Wireguard
```bash
# Bypass NAT untuk traffic Wireguard (optional, untuk avoid double NAT)
/ip firewall nat add action=accept chain=srcnat src-address=10.8.0.0/24 comment="bypass-nat-for-wireguard" place-before=0
```

### Langkah 6: Testing Konektivitas

#### A. Test Ping ke VPS
```bash
/ping 10.8.0.1 count=5

# Expected: 0% packet loss dengan reasonable latency
```

#### B. Initial Interface Status Check
```bash
# Cek status interface
/interface wireguard print

# Should show "R" (running) flag
```

---

## Konfigurasi Mikrotik Lokasi Kedua

### Langkah 1: Backup dan Persiapan
```bash
# Backup konfigurasi existing
/export file=backup-pre-wireguard-lokasi2

# Verifikasi sistem
/system resource print
/interface print
```

### Langkah 2: Buat Wireguard Interface
```bash
/interface wireguard add listen-port=51820 mtu=1420 name=wg-site2site private-key="[PRIVATE_KEY_DARI_CONFIG_LOKASI_CABANG]"
```

### Langkah 3: IP Address Assignment
```bash
/ip address add address=10.8.0.4/24 interface=wg-site2site comment="Wireguard-Site2Site-IP"
```

### Langkah 4: Konfigurasi Peer
```bash
/interface wireguard peers add allowed-address=0.0.0.0/0,::/0 endpoint-address=your-domain.com endpoint-port=51820 interface=wg-site2site public-key="[SERVER_PUBLIC_KEY]" preshared-key="[PRESHARED_KEY_LOKASI_CABANG]" persistent-keepalive=25s comment="VPS-Server"
```

### Langkah 5: Routing ke Lokasi Remote
```bash
# Route ke lokasi pertama
/ip route add dst-address=192.168.110.0/24 gateway=wg-site2site distance=1 comment="to-Lokasi-Utama-via-Wireguard"

# Route untuk management network
/ip route add dst-address=10.8.0.0/24 gateway=wg-site2site distance=1 comment="Wireguard-Management-Network"
```

### Langkah 6: Firewall Rules (identik dengan lokasi pertama)
```bash
/ip firewall filter add action=accept chain=forward connection-state=established,related comment="allow-established-related" place-before=0
/ip firewall filter add action=accept chain=forward in-interface=wg-site2site comment="allow-from-wireguard" place-before=0
/ip firewall filter add action=accept chain=forward out-interface=wg-site2site comment="allow-to-wireguard" place-before=0
/ip firewall filter add action=accept chain=input in-interface=wg-site2site comment="allow-wg-to-router" place-before=0
/ip firewall nat add action=accept chain=srcnat src-address=10.8.0.0/24 comment="bypass-nat-for-wireguard" place-before=0
```

### Langkah 7: Testing Konektivitas Awal
```bash
# Test ke VPS
/ping 10.8.0.1 count=5

# Test ke Mikrotik lokasi pertama (akan gagal sampai routing VPS dikonfigurasi)
/ping 10.8.0.2 count=5
```

---

## Setup Routing di VPS

> ⚠️ PENTING: Langkah ini WAJIB dilakukan untuk semua implementasi, termasuk yang menggunakan managed Wireguard service. Tanpa routing manual ini, komunikasi antar-subnet tidak akan berfungsi.

> 🔧 ALTERNATIVE: Jika VPS Anda tidak memberikan SSH access atau menggunakan managed service yang restricted, hubungi support provider untuk menambahkan static routes ini, atau pertimbangkan menggunakan VPS lain dengan full control.

### Langkah 1: Akses VPS Container

#### A. Untuk Self-Hosted wg-easy:
```bash
# SSH ke VPS
ssh root@your-vps-ip

# Masuk ke container wg-easy
docker exec -it wg-easy /bin/sh
```

#### B. Untuk Managed Service:
Jika menggunakan managed Wireguard service, Anda mungkin perlu:
1. Akses control panel hosting
2. Cari menu "Advanced Wireguard Configuration" atau sejenisnya  
3. Look for "Static Routes" atau "Custom Routes"
4. Jika tidak tersedia, hubungi technical support

### Langkah 2: Tambahkan Static Routes

#### A. Routing Manual di Container
```bash
# Route traffic ke subnet lokasi pertama via peer lokasi pertama
ip route add 192.168.110.0/24 via 10.8.0.2

# Route traffic ke subnet lokasi kedua via peer lokasi kedua
ip route add 192.168.113.0/24 via 10.8.0.4
```

#### B. Verifikasi Routes
```bash
# Lihat routing table lengkap
ip route show

# Expected output harus menunjukkan:
# 192.168.110.0/24 via 10.8.0.2 dev wg0 
# 192.168.113.0/24 via 10.8.0.4 dev wg0
```

#### C. Test Routing dari VPS
```bash
# Test ping ke gateway setiap lokasi (akan berhasil jika Mikrotik sudah dikonfigurasi)
ping -c 3 192.168.110.1
ping -c 3 192.168.113.1

# Exit dari container
exit
```

### Langkah 3: Membuat Routes Persistent

#### A. Create Startup Script (untuk Docker wg-easy)
```bash
# Di VPS host, buat script untuk auto-add routes
cat > /opt/wireguard/add-routes.sh << 'EOF'
#!/bin/bash
# Wait for wg-easy container to be fully up
sleep 30

# Add static routes
docker exec wg-easy ip route add 192.168.110.0/24 via 10.8.0.2 2>/dev/null || true
docker exec wg-easy ip route add 192.168.113.0/24 via 10.8.0.4 2>/dev/null || true

echo "Wireguard static routes added successfully"
EOF

# Make script executable
chmod +x /opt/wireguard/add-routes.sh
```

#### B. Add to Systemd (optional untuk auto-start)
```bash
# Create systemd service
cat > /etc/systemd/system/wg-routes.service << 'EOF'
[Unit]
Description=Wireguard Static Routes
After=docker.service
Wants=docker.service

[Service]
Type=oneshot
ExecStart=/opt/wireguard/add-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Enable service
systemctl enable wg-routes.service
```

### Langkah 4: Alternative - PostUp Script (Advanced)

#### A. Modify Wireguard Config (jika ada akses)
```bash
# Edit wg0.conf jika memungkinkan
[Interface]
# ... existing config ...
PostUp = ip route add 192.168.110.0/24 via 10.8.0.2
PostUp = ip route add 192.168.113.0/24 via 10.8.0.4
PostDown = ip route del 192.168.110.0/24 via 10.8.0.2
PostDown = ip route del 192.168.113.0/24 via 10.8.0.4
```

---

## Testing dan Verifikasi

### Langkah 1: Testing Connectivity Bertahap

#### A. Test VPS to Mikrotik
```bash
# Dari VPS container
docker exec wg-easy ping -c 5 10.8.0.2  # ke Mikrotik lokasi pertama
docker exec wg-easy ping -c 5 10.8.0.4  # ke Mikrotik lokasi kedua

# Expected: 0% packet loss untuk kedua lokasi
```

#### B. Test Mikrotik to VPS
```bash
# Dari Mikrotik lokasi pertama
/ping 10.8.0.1 count=5

# Dari Mikrotik lokasi kedua  
/ping 10.8.0.1 count=5

# Expected: 0% packet loss dengan latency reasonable (50-300ms tergantung geografis)
```

#### C. Test Mikrotik to Mikrotik
```bash
# Dari lokasi pertama ke kedua
/ping 10.8.0.4 count=5

# Dari lokasi kedua ke pertama
/ping 10.8.0.2 count=5

# Expected: 0% packet loss, latency lebih tinggi karena double-hop via VPS
```

### Langkah 2: Testing Site-to-Site Communication

#### A. Test Gateway Communication
```bash
# Dari Mikrotik lokasi pertama ke gateway lokasi kedua
/ping 192.168.113.1 count=5

# Dari Mikrotik lokasi kedua ke gateway lokasi pertama
/ping 192.168.110.1 count=5

# Expected: 0% packet loss - ini adalah test krusial untuk site-to-site
```

#### B. Test Device-to-Device (jika ada device di LAN)
```bash
# Dari device di lokasi pertama (192.168.110.x) ping ke device di lokasi kedua
ping 192.168.113.10  # contoh IP device di lokasi kedua

# Dan sebaliknya
ping 192.168.110.10  # dari lokasi kedua ke lokasi pertama
```

### Langkah 3: Comprehensive Network Testing

#### A. Traceroute Analysis
```bash
# Dari Mikrotik lokasi pertama, trace ke lokasi kedua
/tool traceroute address=192.168.113.1

# Expected path: 
# 1. 10.8.0.1 (VPS)
# 2. 192.168.113.1 (Gateway lokasi kedua)
```

#### B. Bandwidth Testing
```bash
# Test bandwidth using iperf3 (jika tersedia)
# Di lokasi pertama sebagai server
/tool bandwidth-test protocol=tcp

# Di lokasi kedua sebagai client
/tool bandwidth-test direction=both protocol=tcp address=192.168.110.1
```

#### C. DNS Resolution Testing
```bash
# Test DNS resolution through tunnel
/ping google.com interface=wg-site2site count=3

# Test internal hostname resolution (jika dikonfigurasi)
/ping hostname-lokasi-kedua count=3
```

### Langkah 4: Status Monitoring dan Health Check

#### A. Wireguard Status di VPS
```bash
# Check handshake status
docker exec wg-easy wg show

# Expected output harus menunjukkan:
# - Latest handshake < 2 menit untuk setiap peer
# - Transfer data > 0 untuk received dan sent
# - Endpoint IP yang benar untuk setiap peer
```

#### B. Interface Status di Mikrotik
```bash
# Cek status interface Wireguard
/interface wireguard print

# Cek peer status detail
/interface wireguard peers print detail

# Cek routing table
/ip route print where gateway=wg-site2site
```

#### C. Traffic Monitoring
```bash
# Monitor real-time traffic di Mikrotik
/tool traffic-monitor interface=wg-site2site

# Monitor dari VPS
docker exec wg-easy watch -n 2 'wg show'
```

### Langkah 5: Performance Benchmarking

#### A. Latency Baseline
```bash
# Dari setiap lokasi ke VPS
/ping 10.8.0.1 count=20

# Record baseline latency untuk monitoring future
```

#### B. Throughput Testing
```bash
# Large file transfer test
# Copy file antar lokasi dan measure throughput
# Expected throughput: 10-50% dari bandwidth internet tersedia (normal untuk VPN)
```

#### C. Connection Stability Test
```bash
# Long-running ping test
/ping 192.168.113.1 count=100 interval=5

# Check untuk packet loss atau latency spikes
```

---

## Troubleshooting

> 🚨 EMERGENCY GUIDE: Jika mengalami masalah urgent dan perlu solusi cepat, lihat [Diagnostic Commands](#diagnostic-commands) untuk command-command testing yang dapat dijalankan segera untuk identifikasi masalah.

### Masalah Umum dan Solusi

#### 1. Peer Tidak Terhubung ke VPS
**Symptoms:** 
- No handshake di `wg show`
- Ping ke 10.8.0.1 gagal dari Mikrotik
- Status "not connected" di wg-easy interface

**Diagnosis:**
```bash
# Di VPS - cek apakah peer terlihat
docker exec wg-easy wg show

# Di Mikrotik - cek peer configuration  
/interface wireguard peers print detail

# Cek firewall VPS
ufw status
```

**Solusi:**
```bash
# 1. Cek firewall VPS
sudo ufw allow 51820/udp

# 2. Restart Wireguard interface di Mikrotik
/interface wireguard disable wg-site2site
/interface wireguard enable wg-site2site

# 3. Verifikasi endpoint address dan port
/interface wireguard peers print
# Pastikan endpoint-address benar

# 4. Test koneksi UDP ke VPS
# Dari lokasi Mikrotik test port 51820
```

#### 2. Handshake Berhasil tapi Ping ke VPS Gagal
**Symptoms:**
- Latest handshake recent di wg show
- Transfer data = 0 atau sangat kecil
- Ping timeout ke 10.8.0.1

**Diagnosis:**
```bash
# Cek routing di Mikrotik
/ip route print where gateway=wg-site2site

# Cek IP address Wireguard interface
/ip address print where interface=wg-site2site

# Cek firewall rules
/ip firewall filter print where chain=forward
```

**Solusi:**
```bash
# 1. Pastikan IP address correct
/ip address print where interface=wg-site2site
# Should show 10.8.0.2 atau 10.8.0.4

# 2. Add explicit route jika perlu
/ip route add dst-address=10.8.0.1/32 gateway=wg-site2site

# 3. Check firewall rules
/ip firewall filter print where in-interface=wg-site2site
# Pastikan ada rule allow

# 4. Temporary disable firewall untuk test
/ip firewall filter disable numbers=[rule-numbers]
```

#### 3. Mikrotik ke Mikrotik Berhasil, tapi Site-to-Site Gagal
**Symptoms:**
- Ping 10.8.0.2 ↔ 10.8.0.4 berhasil
- Ping 192.168.110.1 ↔ 192.168.113.1 gagal
- VPS routing mungkin belum dikonfigurasi

**Diagnosis:**
```bash
# Cek routing di VPS container
docker exec wg-easy ip route show

# Cek apakah ada route ke subnet lokal
# Should have: 192.168.110.0/24 via 10.8.0.2
#             192.168.113.0/24 via 10.8.0.4
```

**Solusi:**
```bash
# 1. Add manual routing di VPS (langkah wajib)
docker exec wg-easy ip route add 192.168.110.0/24 via 10.8.0.2
docker exec wg-easy ip route add 192.168.113.0/24 via 10.8.0.4

# 2. Verifikasi routing
docker exec wg-easy ip route show

# 3. Test ping dari VPS ke gateway lokal
docker exec wg-easy ping -c 3 192.168.110.1
docker exec wg-easy ping -c 3 192.168.113.1
```

#### 4. Koneksi Tidak Stabil atau Sering Putus
**Symptoms:**
- Handshake kadang hilang
- Ping success rate < 90%
- Connection drops secara berkala

**Diagnosis:**
```bash
# Monitor handshake stability
watch -n 5 'docker exec wg-easy wg show'

# Check persistent keepalive setting
/interface wireguard peers print detail

# Monitor latency variation
/ping 10.8.0.1 count=50
```

**Solusi:**
```bash
# 1. Adjust keepalive interval
/interface wireguard peers set 0 persistent-keepalive=15s

# 2. Check MTU setting
/interface wireguard set wg-site2site mtu=1280

# 3. Test different MTU sizes
/ping 10.8.0.1 size=1400 count=10
/ping 10.8.0.1 size=1200 count=10

# 4. Monitor ISP connection stability
/ping 8.8.8.8 count=100 interval=1
```

#### 5. Performance Issues (High Latency/Low Throughput)
**Symptoms:**
- Latency > 500ms consistently
- Throughput < 10% of available bandwidth
- High packet loss

**Diagnosis:**
```bash
# Baseline internet performance
/ping 8.8.8.8 count=20

# VPS performance
/ping 10.8.0.1 count=20

# Compare direct vs tunneled performance
/tool bandwidth-test address=192.168.113.1 protocol=tcp
```

**Solusi:**
```bash
# 1. Optimize MTU
/interface wireguard set wg-site2site mtu=1420

# 2. Check VPS resource usage
docker stats wg-easy

# 3. Consider VPS location/provider change
# Test latency to different VPS locations

# 4. QoS implementation (advanced)
/queue simple add target=wg-site2site max-limit=50M/50M
```

### Advanced Troubleshooting

#### 6. Firewall Conflicts dengan Hotspot atau Existing Rules
**Symptoms:**
- Mikrotik dengan hotspot tidak bisa akses Wireguard
- Existing firewall rules interfere
- Dynamic rules conflict

**Diagnosis:**
```bash
# List semua firewall rules dengan order
/ip firewall filter print

# Cek hotspot rules
/ip firewall filter print where hotspot!=

# Identify conflicting rules
/ip firewall filter print where action=drop
```

**Solusi:**
```bash
# 1. Reorder firewall rules (place-before=X)
/ip firewall filter move [rule-number] destination=[new-position]

# 2. Add specific allow rules untuk Wireguard
/ip firewall filter add action=accept chain=forward src-address=10.8.0.0/24 place-before=0
/ip firewall filter add action=accept chain=forward dst-address=10.8.0.0/24 place-before=0

# 3. Bypass hotspot untuk Wireguard traffic
/ip firewall nat add action=accept chain=hotspot src-address=10.8.0.0/24 place-before=0
```

#### 7. DNS Issues
**Symptoms:**
- IP address ping works, hostname doesn't
- Slow DNS resolution
- DNS leaks

**Diagnosis:**
```bash
# Test DNS resolution
/ping google.com
/ping google.com interface=wg-site2site

# Check DNS settings
/ip dns print
```

**Solusi:**
```bash
# 1. Set DNS servers via Wireguard
# Update client config with proper DNS

# 2. DNS forwarding setup
/ip dns set allow-remote-requests=yes servers=1.1.1.1,8.8.8.8

# 3. Local DNS for internal hostnames (advanced)
/ip dns static add name=lokasi-cabang address=192.168.113.1
```

### Diagnostic Commands

#### Quick Health Check Commands

**Di VPS:**
```bash
# All-in-one status check
docker exec wg-easy sh -c "
echo '=== Wireguard Status ==='
wg show
echo -e '\n=== Routing Table ==='
ip route show
echo -e '\n=== Container Stats ==='
ps aux | head -5
"
```

**Di Mikrotik:**
```bash
# Comprehensive status check
/interface wireguard print; /interface wireguard peers print; /ip route print where gateway=wg-site2site; /ping 10.8.0.1 count=3
```

**Network Connectivity Matrix:**
```bash
# Test all connections dalam satu command
echo "Testing VPS connectivity..."
docker exec wg-easy ping -c 2 10.8.0.2
docker exec wg-easy ping -c 2 10.8.0.4

echo "Testing site-to-site..."  
docker exec wg-easy ping -c 2 192.168.110.1
docker exec wg-easy ping -c 2 192.168.113.1
```

---

## Monitoring dan Maintenance

> 📊 MONITORING PRIORITY: Untuk implementasi production, fokus utama monitoring ada pada koneksi stability dan bandwidth usage. Jika Anda menggunakan managed service, beberapa monitoring mungkin sudah disediakan oleh provider.

### 1. Monitoring Rutin

#### A. Daily Health Check
```bash
# Automated health check script
#!/bin/bash
# File: /opt/scripts/wg-healthcheck.sh

echo "=== Wireguard Health Check - $(date) ==="

# Check container status
if docker ps | grep -q wg-easy; then
    echo "✓ wg-easy container: RUNNING"
else  
    echo "✗ wg-easy container: DOWN"
    exit 1
fi

# Check peers connectivity
echo -e "\n=== Peer Status ==="
docker exec wg-easy wg show | grep -A 3 "peer:"

# Check recent handshakes (should be < 3 minutes)
HANDSHAKE_THRESHOLD=180  # 3 minutes in seconds
docker exec wg-easy wg show | grep "latest handshake" | while read line; do
    if echo "$line" | grep -q "minute"; then
        echo "✓ Recent handshake: $line"
    else
        echo "⚠ Old handshake: $line" 
    fi
done

# Test basic connectivity
echo -e "\n=== Connectivity Test ==="
if docker exec wg-easy ping -c 2 10.8.0.2 >/dev/null 2>&1; then
    echo "✓ Lokasi pertama: REACHABLE"
else
    echo "✗ Lokasi pertama: UNREACHABLE"
fi

if docker exec wg-easy ping -c 2 10.8.0.4 >/dev/null 2>&1; then
    echo "✓ Lokasi kedua: REACHABLE"  
else
    echo "✗ Lokasi kedua: UNREACHABLE"
fi

# Test site-to-site connectivity
echo -e "\n=== Site-to-Site Test ==="
if docker exec wg-easy ping -c 2 192.168.110.1 >/dev/null 2>&1; then
    echo "✓ Site-to-Site Lokasi 1: OK"
else
    echo "✗ Site-to-Site Lokasi 1: FAILED"
fi

if docker exec wg-easy ping -c 2 192.168.113.1 >/dev/null 2>&1; then
    echo "✓ Site-to-Site Lokasi 2: OK"
else  
    echo "✗ Site-to-Site Lokasi 2: FAILED"
fi

echo "=== Health Check Completed ==="
```

#### B. Weekly Monitoring Tasks
```bash
# Bandwidth usage analysis
#!/bin/bash
# File: /opt/scripts/wg-bandwidth.sh

echo "=== Weekly Bandwidth Report - $(date) ==="

# Get transfer stats from wg show
docker exec wg-easy wg show | grep "transfer:" | while read line; do
    echo "Transfer data: $line"
done

# Log to file for historical tracking
echo "$(date): $(docker exec wg-easy wg show)" >> /var/log/wireguard-stats.log

# Check disk usage
df -h /opt/wireguard
```

#### C. Performance Monitoring
```bash
# Latency monitoring script
#!/bin/bash
# File: /opt/scripts/wg-latency.sh

echo "=== Latency Monitoring - $(date) ==="

# Test latency ke setiap peer
for ip in 10.8.0.2 10.8.0.4; do
    echo "Testing latency to $ip:"
    docker exec wg-easy ping -c 5 $ip | tail -1
    echo ""
done

# Test site-to-site latency  
for ip in 192.168.110.1 192.168.113.1; do
    echo "Testing site-to-site latency to $ip:"
    docker exec wg-easy ping -c 5 $ip | tail -1
    echo ""
done
```

### 2. Automated Monitoring Setup

#### A. Crontab Configuration
```bash
# Install monitoring scripts
sudo mkdir -p /opt/scripts
sudo chmod +x /opt/scripts/*.sh

# Setup crontab untuk automated monitoring
crontab -e

# Add these lines:
# Daily health check at 9 AM
0 9 * * * /opt/scripts/wg-healthcheck.sh >> /var/log/wg-health.log 2>&1

# Hourly connectivity test
0 * * * * /opt/scripts/wg-latency.sh >> /var/log/wg-latency.log 2>&1

# Weekly bandwidth report
0 9 * * 1 /opt/scripts/wg-bandwidth.sh >> /var/log/wg-bandwidth.log 2>&1
```

#### B. Log Rotation Setup
```bash
# Create logrotate config
sudo tee /etc/logrotate.d/wireguard << EOF
/var/log/wg-*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    create 644 root root
}
EOF
```

#### C. Alerting System (Advanced)
```bash
# Email alert untuk connection failures
#!/bin/bash
# File: /opt/scripts/wg-alert.sh

ALERT_EMAIL="admin@yourdomain.com"

# Check connectivity
if ! docker exec wg-easy ping -c 3 10.8.0.2 >/dev/null 2>&1; then
    echo "Wireguard connection to Lokasi 1 FAILED at $(date)" | \
    mail -s "Wireguard Alert: Connection Failed" $ALERT_EMAIL
fi

if ! docker exec wg-easy ping -c 3 10.8.0.4 >/dev/null 2>&1; then
    echo "Wireguard connection to Lokasi 2 FAILED at $(date)" | \
    mail -s "Wireguard Alert: Connection Failed" $ALERT_EMAIL  
fi
```

### 3. Backup dan Recovery

#### A. Configuration Backup
```bash
# Weekly backup script
#!/bin/bash
# File: /opt/scripts/wg-backup.sh

BACKUP_DIR="/opt/backups/wireguard"
DATE=$(date +%Y%m%d)

mkdir -p $BACKUP_DIR

echo "=== Wireguard Backup - $(date) ==="

# Backup wg-easy configuration
docker exec wg-easy tar -czf - /etc/wireguard > $BACKUP_DIR/wg-config-$DATE.tar.gz

# Backup docker-compose.yml
cp /opt/wireguard/docker-compose.yml $BACKUP_DIR/docker-compose-$DATE.yml

# Backup custom scripts
tar -czf $BACKUP_DIR/scripts-$DATE.tar.gz /opt/scripts/

# Keep only last 10 backups
ls -t $BACKUP_DIR/*.tar.gz | tail -n +11 | xargs rm -f

echo "Backup completed: $BACKUP_DIR/wg-config-$DATE.tar.gz"
```

#### B. Mikrotik Configuration Backup
```bash
# Backup semua Mikrotik configurations
# Di setiap Mikrotik, setup automatic backup:

# Create backup script di Mikrotik
/system script add name=weekly-backup source={
    /export file=("backup-" . [/system clock get date])
    /system backup save name=("binary-backup-" . [/system clock get date])
}

# Schedule backup mingguan
/system scheduler add name=auto-backup on-event=weekly-backup start-time=02:00:00 interval=7d
```

#### C. Recovery Procedures
```bash
# Disaster recovery script
#!/bin/bash  
# File: /opt/scripts/wg-recovery.sh

echo "=== Wireguard Disaster Recovery ==="

# Stop current container
docker-compose -f /opt/wireguard/docker-compose.yml down

# Restore dari backup terbaru
LATEST_BACKUP=$(ls -t /opt/backups/wireguard/wg-config-*.tar.gz | head -1)
echo "Restoring from: $LATEST_BACKUP"

# Extract backup
tar -xzf $LATEST_BACKUP -C /

# Restart services
docker-compose -f /opt/wireguard/docker-compose.yml up -d

# Wait and test
sleep 30
/opt/scripts/wg-healthcheck.sh

echo "Recovery completed"
```

### 4. Performance Optimization

#### A. Resource Monitoring
```bash
# Monitor VPS resource usage
#!/bin/bash
# File: /opt/scripts/wg-resources.sh

echo "=== Resource Usage - $(date) ==="

# CPU and Memory usage
echo "System resources:"
top -bn1 | grep "Cpu\|Mem"

# Docker container stats
echo -e "\nContainer resources:"
docker stats wg-easy --no-stream

# Disk usage
echo -e "\nDisk usage:"
df -h /opt/wireguard /var/lib/docker

# Network interface stats
echo -e "\nNetwork statistics:"
cat /proc/net/dev | grep -E "(wg0|eth0)"
```

#### B. Performance Tuning
```bash
# VPS optimization untuk Wireguard
#!/bin/bash
# File: /opt/scripts/wg-optimize.sh

echo "Applying Wireguard optimizations..."

# Network kernel parameters
echo 'net.core.default_qdisc=fq' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_congestion_control=bbr' >> /etc/sysctl.conf
echo 'net.core.rmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf

# Apply settings
sysctl -p

# Docker container limits adjustment
docker update --memory=1g --cpus="1.5" wg-easy

echo "Optimization completed"
```

#### C. Bandwidth Management
```bash
# Bandwidth limiting per client (di Mikrotik)
# Add bandwidth limits untuk prevent VPS overload

# Limit Wireguard interface total bandwidth
/queue simple add name=wg-limit target=wg-site2site max-limit=50M/50M

# Per-destination bandwidth control
/queue simple add name=to-remote-site target=192.168.113.0/24 parent=wg-limit max-limit=25M/25M
```

---

## Optimisasi dan Best Practices

### 1. Security Hardening

#### A. VPS Security Enhancement
```bash
# Enhanced VPS security untuk production
#!/bin/bash

# Change default SSH port
sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# Disable password authentication (use keys only)
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Setup fail2ban untuk Wireguard
cat > /etc/fail2ban/jail.d/wireguard.conf << EOF
[wireguard]
enabled = true
filter = wireguard
port = 51820
protocol = udp
logpath = /var/log/wireguard.log
maxretry = 3
bantime = 3600
EOF

# Restart services
systemctl restart sshd fail2ban
```

#### B. Certificate dan Key Management
```bash
# Automated key rotation (advanced)
#!/bin/bash
# File: /opt/scripts/wg-rotate-keys.sh

# WARNING: Key rotation requires client reconfiguration
# Use only dalam maintenance window

BACKUP_DIR="/opt/backups/keys"
mkdir -p $BACKUP_DIR

echo "=== Key Rotation Process ==="
echo "1. Backup current configuration"
docker exec wg-easy tar -czf - /etc/wireguard > $BACKUP_DIR/pre-rotation-$(date +%Y%m%d).tar.gz

echo "2. Generate new server keys"
# This process requires manual intervention untuk update clients
# Implement only if automatic client update mechanism exists
```

#### C. Access Control Lists
```bash
# Granular firewall rules untuk specific services
# Di Mikrotik, implement service-specific rules:

# Allow only specific services through VPN
/ip firewall filter add action=accept chain=forward in-interface=wg-site2site protocol=tcp dst-port=80,443,22,3389 comment="allowed-services-via-vpn"

# Block unnecessary protocols
/ip firewall filter add action=drop chain=forward in-interface=wg-site2site protocol=udp dst-port=135-139 comment="block-netbios"

# Time-based access control (office hours only)
/ip firewall filter add action=accept chain=forward in-interface=wg-site2site time=8h-18h,mon,tue,wed,thu,fri comment="office-hours-only"
```

### 2. Scalability Planning

#### A. Multiple Site Support
```bash
# Planning untuk additional sites
# IP allocation schema:

# Wireguard Network: 10.8.0.0/24
# VPS Server: 10.8.0.1
# Site A: 10.8.0.2 (192.168.110.0/24)
# Site B: 10.8.0.4 (192.168.113.0/24)  
# Site C: 10.8.0.6 (192.168.120.0/24) - future
# Site D: 10.8.0.8 (192.168.130.0/24) - future
# Mobile Users: 10.8.0.100-200

# Automated site addition script
#!/bin/bash
# File: /opt/scripts/add-new-site.sh

SITE_NAME=$1
SITE_IP=$2
SITE_SUBNET=$3

if [ $# -ne 3 ]; then
    echo "Usage: $0 <site-name> <wireguard-ip> <local-subnet>"
    echo "Example: $0 Site-C 10.8.0.6 192.168.120.0/24"
    exit 1
fi

echo "Adding new site: $SITE_NAME"
echo "Wireguard IP: $SITE_IP"  
echo "Local subnet: $SITE_SUBNET"

# Add routing untuk new site
docker exec wg-easy ip route add $SITE_SUBNET via $SITE_IP

echo "Site $SITE_NAME added. Configure Mikrotik dengan IP $SITE_IP"
```

#### B. Load Balancing (Advanced)
```bash
# Multiple VPS setup untuk high availability
# Setup VRRP atau DNS round-robin untuk multiple VPS endpoints

# Example DNS configuration:
# vpn1.yourdomain.com -> VPS_IP_1
# vpn2.yourdomain.com -> VPS_IP_2
# vpn.yourdomain.com  -> Round-robin ke vpn1/vpn2

# Client configuration dengan multiple endpoints:
# [Peer]
# PublicKey = SERVER_KEY_1
# Endpoint = vpn1.yourdomain.com:51820
# 
# [Peer]  
# PublicKey = SERVER_KEY_2
# Endpoint = vpn2.yourdomain.com:51820
```

#### C. Bandwidth Optimization
```bash
# Advanced QoS setup di Mikrotik
# Priority traffic berdasarkan aplikasi

# High priority: VoIP, video conferencing
/ip firewall mangle add chain=forward action=mark-connection new-connection-mark=voip protocol=udp dst-port=5060,10000-20000
/queue simple add name=voip-priority target=voip max-limit=10M/10M priority=1

# Medium priority: HTTP, HTTPS
/ip firewall mangle add chain=forward action=mark-connection new-connection-mark=web protocol=tcp dst-port=80,443
/queue simple add name=web-traffic target=web max-limit=30M/30M priority=3

# Low priority: File transfers, backup
/ip firewall mangle add chain=forward action=mark-connection new-connection-mark=bulk protocol=tcp dst-port=21,22,873
/queue simple add name=bulk-traffic target=bulk max-limit=20M/20M priority=8
```

### 3. Integration dengan Existing Infrastructure

#### A. Active Directory Integration
```bash
# Setup RADIUS authentication untuk Wireguard (advanced)
# Requires enterprise Wireguard solution atau custom implementation

# Alternative: Certificate-based authentication
# Generate certificates per user/location
openssl genrsa -out site-a.key 2048
openssl req -new -key site-a.key -out site-a.csr
openssl x509 -req -in site-a.csr -signkey site-a.key -out site-a.crt
```

#### B. Centralized Logging
```bash
# Setup syslog forwarding dari semua Mikrotik ke central server
# Di setiap Mikrotik:

/system logging action add name=remote-syslog target=remote remote=SYSLOG_SERVER_IP
/system logging add topics=wireguard action=remote-syslog

# ELK Stack setup untuk log analysis (advanced)
# Docker compose untuk Elasticsearch, Logstash, Kibana
# Analysis Wireguard connection patterns, bandwidth usage, errors
```

#### C. Network Management Integration
```bash
# SNMP monitoring setup
# Di Mikrotik enable SNMP untuk monitoring tools

/snmp set enabled=yes
/snmp community set public address=MONITORING_SERVER_IP/32 security=none

# Monitor Wireguard interface statistics
# OIDs untuk interface monitoring:
# Interface status: 1.3.6.1.2.1.2.2.1.8
# Interface traffic: 1.3.6.1.2.1.2.2.1.10/1.3.6.1.2.1.2.2.1.16
```

### 4. Disaster Recovery Planning

#### A. Complete Site Failure Recovery
```bash
# Site failure recovery procedures
#!/bin/bash
# File: /opt/scripts/site-failover.sh

FAILED_SITE=$1
BACKUP_ROUTE=$2

echo "=== Site Failover Procedure ==="
echo "Failed site: $FAILED_SITE"

# Reroute traffic via alternative path
if [ "$FAILED_SITE" == "site-a" ]; then
    # Reroute Site A traffic via Site B
    echo "Rerouting Site A traffic via Site B"
    # Implementation specific untuk network topology
fi

# Update DNS untuk site-specific services
# Update monitoring untuk adjusted expectations
# Notify users tentang alternative access methods

echo "Failover procedures completed"
```

#### B. VPS Provider Migration
```bash
# VPS migration checklist dan procedures
#!/bin/bash
# File: /opt/scripts/vps-migration.sh

echo "=== VPS Migration Procedure ==="

# 1. Setup new VPS dengan identical configuration
# 2. Export/import wg-easy configuration  
# 3. Update DNS records
# 4. Update Mikrotik endpoint configurations
# 5. Test connectivity
# 6. Cutover traffic
# 7. Decommission old VPS

echo "Migration completed"
```

### 5. Cost Optimization

#### A. Bandwidth Usage Optimization
```bash
# Analyze dan optimize bandwidth usage
#!/bin/bash
# File: /opt/scripts/bandwidth-analysis.sh

echo "=== Bandwidth Usage Analysis ==="

# Top bandwidth consuming connections
docker exec wg-easy wg show | grep transfer | sort -k3 -nr

# Recommendation untuk traffic optimization:
echo "1. Implement compression untuk file transfers"
echo "2. Schedule large transfers during off-peak hours"
echo "3. Use local caching untuk frequently accessed resources"
echo "4. Consider dedicated backup channels untuk bulk data"
```

#### B. Multi-Provider Strategy
```bash
# Cost comparison dan optimization
#!/bin/bash

echo "=== Cost Optimization Analysis ==="

# Current costs:
# VPS: $X/month
# Bandwidth: $Y/GB
# Management time: Z hours/month

# Alternatives:
# Multiple smaller VPS vs single large VPS
# Regional VPS providers untuk better pricing
# Bandwidth commitments untuk volume discounts

echo "Analyze monthly usage patterns untuk optimal pricing"
```

---

## Kesimpulan dan Panduan Lanjutan

### Implementasi yang Telah Dicapai

Dengan mengikuti panduan ini, Anda telah berhasil membangun infrastruktur site-to-site VPN yang:

#### **Connectivity Achievement:**
- Menghubungkan dua lokasi dengan subnet berbeda secara aman
- Menggunakan koneksi internet existing tanpa memerlukan IP publik static  
- Menyediakan encrypted tunnel dengan protokol Wireguard modern
- Memungkinkan komunikasi seamless antar-jaringan lokal

#### **Technical Implementation:**
- VPS sebagai central relay point dengan wg-easy management
- Mikrotik RouterOS sebagai gateway lokal di setiap site
- Automated routing untuk site-to-site communication
- Comprehensive firewall dan security configuration

#### **Operational Capabilities:**
- Real-time monitoring dan health checking
- Automated backup dan recovery procedures
- Performance monitoring dan optimization
- Scalable architecture untuk additional sites

### Expected Performance Metrics

#### **Normal Operation Parameters:**
```
Latency: 100-300ms (site-to-site)
Throughput: 20-70% of available internet bandwidth
Uptime: >99% dengan proper monitoring
Packet Loss: <1% under normal conditions
Connection Establishment: <30 seconds after network change
```

#### **Resource Usage (VPS):**
```
CPU: 10-30% untuk 2-5 sites
RAM: 256MB-1GB depending on client count
Bandwidth: 100GB-1TB/month depending on usage
Storage: 1-5GB untuk configuration dan logs
```

### Cost-Benefit Analysis

#### **Implementation Costs:**
- VPS hosting: $5-20/month
- Domain registration: $10-15/year
- Setup time: 4-8 hours initial implementation
- Maintenance: 2-4 hours/month

#### **Cost Savings vs Alternatives:**
- vs Dedicated Line: 70-90% cost reduction
- vs MPLS: 60-80% cost reduction  
- vs Multiple Static IP: 50-70% cost reduction
- vs Proprietary VPN solutions: 40-60% cost reduction

#### **Operational Benefits:**
- Unified network management
- Centralized resource access
- Remote troubleshooting capability
- Scalable untuk business growth
- Modern, maintainable technology stack

### Scalability Roadmap

#### **Phase 1: Basic Implementation** (Completed)
- 2 sites connected
- Basic monitoring
- Manual routing configuration

#### **Phase 2: Enhanced Operations** (Next 3-6 months)
- Automated monitoring dan alerting
- Performance optimization
- Enhanced security hardening
- Documentation dan training

#### **Phase 3: Scale Out** (6-12 months)
- Additional sites (3-5 total)
- Load balancing dan redundancy
- Advanced QoS implementation
- Integration dengan existing IT infrastructure

#### **Phase 4: Enterprise Features** (12+ months)
- Centralized authentication integration
- Advanced analytics dan reporting
- Multi-region deployment
- Disaster recovery automation

### Maintenance Schedule

#### **Daily Tasks (Automated):**
- Connectivity health checks
- Performance monitoring
- Log analysis untuk anomalies
- Bandwidth usage tracking

#### **Weekly Tasks:**
- Comprehensive system backup
- Security updates review
- Performance trend analysis
- Capacity planning review

#### **Monthly Tasks:**
- Security audit dan hardening
- Configuration review dan optimization
- Cost analysis dan optimization
- Documentation updates

#### **Quarterly Tasks:**
- Disaster recovery testing
- Security penetration testing
- Technology updates evaluation
- Business requirements review

### Troubleshooting Quick Reference

#### **Emergency Contact Information:**
```
VPS Provider Support: [Provider contact]
Internet Provider A: [ISP A contact]  
Internet Provider B: [ISP B contact]
Mikrotik Support: [Technical support]
Internal IT Contacts: [Local contacts per site]
```

#### **Critical Recovery Commands:**
```bash
# VPS Emergency Recovery:
docker-compose -f /opt/wireguard/docker-compose.yml restart
/opt/scripts/wg-recovery.sh

# Mikrotik Emergency Recovery:
/system backup load name=[backup-file]
/interface wireguard disable wg-site2site; /interface wireguard enable wg-site2site

# Network Connectivity Test:
/ping 8.8.8.8 count=5
/ping 10.8.0.1 count=5  
/ping [remote-gateway] count=5
```

#### **Emergency Bypass Procedures:**
Jika Wireguard completely down:
1. Document current issue dan impact
2. Implement temporary solutions (port forwarding, etc.)
3. Communicate dengan affected users
4. Focus pada recovery before enhancement
5. Post-incident review dan improvements

### Long-term Strategic Considerations

#### **Technology Evolution:**
- Monitor Wireguard protocol updates
- Evaluate RouterOS version upgrades  
- Consider IPv6 implementation
- Plan untuk quantum-safe cryptography transition

#### **Business Alignment:**
- Regular review dengan business requirements
- Scale planning berdasarkan growth projections
- Integration planning dengan new acquisitions
- Budget planning untuk infrastructure evolution

#### **Security Posture:**
- Regular security assessments
- Compliance requirements monitoring
- Threat landscape evolution tracking
- Incident response plan maintenance

### Final Implementation Checklist

Before considering implementation complete, verify:

- [ ] All connectivity tests passing consistently
- [ ] Monitoring dan alerting functional
- [ ] Backup dan recovery procedures tested
- [ ] Documentation complete dan accessible
- [ ] User training conducted
- [ ] Support procedures established
- [ ] Security hardening implemented
- [ ] Performance baseline established
- [ ] Cost tracking implemented
- [ ] Change management procedures defined

### Support dan Community Resources

#### **Official Documentation:**
- Wireguard Official Docs: https://www.wireguard.com/
- Mikrotik RouterOS Manual: https://help.mikrotik.com/
- Docker Documentation: https://docs.docker.com/

#### **Community Forums:**
- Mikrotik Community Forum
- Reddit r/mikrotik dan r/WireGuard  
- Stack Overflow untuk specific technical issues

#### **Professional Services:**
Consider professional consultation untuk:
- Large scale deployments (>10 sites)
- Complex integration requirements
- High availability implementations
- Compliance-critical environments

---

**Implementation Guide Version:** 1.0  
**Last Updated:** September 2025  
**Compatibility:** RouterOS 7.x, Wireguard Protocol, wg-easy 15+  
**Target Audience:** Network administrators, IT professionals, System integrators

> **Disclaimer:** Implementation dalam production environment memerlukan testing menyeluruh dalam lab environment terlebih dahulu. Author tidak bertanggung jawab atas downtime atau issues yang muncul dari implementasi ini. Selalu lakukan backup lengkap sebelum implementasi dan siapkan rollback procedures.