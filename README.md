## Member

| No  | Nama                        | NRP        | Pengerjaan Soal |
| --- | --------------------------- | ---------- | --------------- |
| 1   | Marvelino Davas             | 5027251085 | Soal 1 - 10     |
| 2   | Nathania Tiara Wahyudi      | 5027251089 | Soal 11 - 20    |


### Soal 1: Konfigurasi Topologi & Interface
Untuk mempersiapkan pembangunan The Wired, Lain yang berperan sebagai Router membuat tiga Switch/Gateway: Switch 1 menuju dua Entitas yaitu Alice dan Mika, Switch 2 menuju Chisa, sedangkan Switch 3 menuju Knights dan Eiri. Kelima Entitas tersebut dikonfigurasi sebagai Client di GNS3.

<img width="1107" height="636" alt="dokum1" src="https://github.com/user-attachments/assets/37a6dcf7-6f2d-4f6e-a55c-98ee480038f8" />


Topologi dibuat dengan komponen berikut:

- NAT : sumber internet (DHCP) untuk router.
- Router Lain : terhubung ke NAT (`eth0`) dan ke tiga switch (`eth1`-`eth3`).
- Switch 1, 2, 3 : menghubungkan client ke router.
- Client (Alice, Mika, Chisa, Knights, Eiri) : simulasi client pada tiap subnet.

Prefix kelompok kami adalah `192.236.x.x`. Konfigurasi router pada **/etc/network/interfaces**:

```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
  address 192.236.1.1
  netmask 255.255.255.0

auto eth2
iface eth2 inet static
  address 192.236.2.1
  netmask 255.255.255.0

auto eth3
iface eth3 inet static
  address 192.236.3.1
  netmask 255.255.255.0
```

Konfigurasi tiap client (IP static dengan gateway ke router):

**Alice**

```
auto eth0
iface eth0 inet static
  address 192.236.1.10
  netmask 255.255.255.0
  gateway 192.236.1.1
```

**Mika**

```
auto eth0
iface eth0 inet static
  address 192.236.1.11
  netmask 255.255.255.0
  gateway 192.236.1.1
```

**Chisa**

```
auto eth0
iface eth0 inet static
  address 192.236.2.10
  netmask 255.255.255.0
  gateway 192.236.2.1
```

**Knights**

```
auto eth0
iface eth0 inet static
  address 192.236.3.10
  netmask 255.255.255.0
  gateway 192.236.3.1
```

**Eiri**

```
auto eth0
iface eth0 inet static
  address 192.236.3.11
  netmask 255.255.255.0
  gateway 192.236.3.1
```


### Soal 2: Konfigurasi Internet Router
Karena menurut Lain pada saat itu The Wired masih terisolasi dari dunia luar, konfigurasikan router Lain agar dapat tersambung langsung ke jaringan internet publik melalui NAT/DHCP pada interface eth0.

Interface `eth0` router yang terhubung ke NAT dikonfigurasi untuk mendapatkan IP dari DHCP:

```
auto eth0
iface eth0 inet dhcp
```

```sh
ifup eth0
/etc/init.d/networking restart
ping -c 4 8.8.8.8
```


### Soal 3: Konfigurasi IP Forwarding
Setelah router Lain terhubung ke internet, pastikan seluruh Entitas (Client) di bawah Switch 1, Switch 2, dan Switch 3 dapat saling terhubung dan berkomunikasi satu sama lain melalui konfigurasi routing.

Ketiga subnet sudah terhubung langsung ke router, sehingga yang perlu dilakukan adalah mengaktifkan IP forwarding pada router:

```sh
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Kemudian test ping antar client:

**Alice ke Mika**

<img width="743" height="520" alt="alice ke mika" src="https://github.com/user-attachments/assets/0de3a89e-7467-48fc-abdd-52b2618e6049" />

**Alice ke Chisa**

<img width="745" height="518" alt="alice ke chisa" src="https://github.com/user-attachments/assets/206233bf-769f-4e68-b5c1-24ff766d640d" />


**Alice ke Eiri**

<img width="742" height="518" alt="alice ke eiri" src="https://github.com/user-attachments/assets/d4a098ba-0259-45fa-8477-f9259c378b53" />


### Soal 4: Konfigurasi Firewall & NAT Masquerade
Lain ingin agar setiap Entitas (Client) memiliki kemandirian di The Wired. Konfigurasikan firewall/iptables (NAT Masquerade) dan DNS resolver agar setiap Client dapat terhubung ke internet secara mandiri (dapat melakukan ping ke 8.8.8.8 dan membuka domain web google.com).

Pada router dijalankan NAT Masquerade agar IP privat client diganti dengan IP `eth0` router, serta rule forward agar trafik client dapat keluar dan balasannya masuk kembali:

```sh
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
```

Pada setiap client ditambahkan DNS resolver:

```sh
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Kemudian test dari client:

```sh
ping -c 4 8.8.8.8
ping -c 4 google.com
apk add curl
curl -sI http://google.com | head -n 1
```

**Ping ke 8.8.8.8**

<img width="746" height="517" alt="ping google" src="https://github.com/user-attachments/assets/cc374813-55ca-4a46-b620-9eb854a842ec" />


### Soal 5: Script Verifikasi Status
Eiri tetap berupaya menanamkan kekacauan ke dalam jaringan. Untuk mengantisipasi restart tiba-tiba, pastikan seluruh konfigurasi jaringan tidak hilang saat semua node di-restart. Buat script verifikasi di /root/cek_status.sh pada router Lain yang menampilkan ringkasan interface (ip -br a) dan status tabel NAT (iptables -t nat -L -v -n) setelah reboot.

Agar tidak hilang saat restart, perintah `sysctl` dan `iptables` diletakkan sebagai direktif `up` pada **/etc/network/interfaces** router:

```
auto eth0
iface eth0 inet dhcp
  up sysctl -w net.ipv4.ip_forward=1
  up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
  up iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT

auto eth1
iface eth1 inet static
  address 192.236.1.1
  netmask 255.255.255.0
  up iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

auto eth2
iface eth2 inet static
  address 192.236.2.1
  netmask 255.255.255.0
  up iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT

auto eth3
iface eth3 inet static
  address 192.236.3.1
  netmask 255.255.255.0
  up iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
```

Pada setiap client, ditambahkan satu baris pada blok `eth0` agar DNS tetap ada setelah restart (contoh Alice, client lain sama):

```
auto eth0
iface eth0 inet static
  address 192.236.1.10
  netmask 255.255.255.0
  gateway 192.236.1.1
  up echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Script verifikasi pada router:

```sh
cat << 'EOF' > /root/cek_status.sh
#!/bin/bash
echo "=== Interface Summary (ip -br a) ==="
ip -br a
echo ""
echo "=== NAT Table (iptables -t nat -L -v -n) ==="
iptables -t nat -L -v -n
EOF

chmod +x /root/cek_status.sh
```

Setelah seluruh node di-restart, script dijalankan dan hasilnya menunjukkan IP interface dan rule `MASQUERADE` masih ada:

```sh
/root/cek_status.sh
```

<img width="745" height="518" alt="cek status" src="https://github.com/user-attachments/assets/354ae5fd-fd1d-4129-8e7e-b2ac4ca2bd83" />


### Soal 6: Traffic Generator
Mika mencurigai adanya anomali traffic pada segmen jaringannya. Jalankan generator traffic berikut (link file) pada node Mika, lalu lakukan packet sniffing menggunakan Wireshark pada interface node Mika. Terapkan display filter khusus untuk menyaring paket yang berprotokol DNS atau ICMP. Tunjukkan screenshot hasil filter beserta ringkasan paket yang lolos.

Download script pada Mika, lalu jalankan bersamaan dengan capture pada link Mika ke Switch 1:

```sh
apk add bash bind-tools
wget -O generator.sh "<LINK_FILE>" --no-check-certificate
chmod +x generator.sh
./generator.sh
```


Display filter yang digunakan pada Wireshark:

```
dns || icmp
```

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/1903b8b7-7204-4867-98a7-c9b378b03d88" />


### Soal 7: Konfigurasi FTP Server
Chisa memutuskan mendirikan FTP Server pada node miliknya dengan shared folder di /var/wired/data. Terapkan kebijakan akses: user alice (hak akses read & write), user mika (dibatasi read-only), dan user eiri (dibatasi tanpa izin akses / blacklist). Buktikan konfigurasi dengan membuat file signal_alice.txt dari user alice, dan buktikan penolakan akses saat user eiri mencoba login.

FTP Server menggunakan `vsftpd` pada node Chisa. Pertama install dan buat folder serta user:

```sh
apk update && apk add vsftpd
mkdir -p /var/wired/data

adduser -D -H -h /var/wired/data -s /bin/sh alice
adduser -D -H -h /var/wired/data -s /bin/sh mika
adduser -D -H -h /var/wired/data -s /bin/sh eiri
echo "alice:alice123" | chpasswd
echo "mika:mika123" | chpasswd
echo "eiri:eiri123" | chpasswd

chown alice:alice /var/wired/data
chmod 755 /var/wired/data
```

Kemudian konfigurasi `/etc/vsftpd/vsftpd.conf`:

```sh
cat << 'EOF' > /etc/vsftpd/vsftpd.conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
pam_service_name=vsftpd
seccomp_sandbox=NO
local_root=/var/wired/data
chroot_local_user=YES
allow_writeable_chroot=YES
userlist_enable=YES
userlist_deny=YES
userlist_file=/etc/vsftpd.user_list
user_config_dir=/etc/vsftpd/user_conf
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
EOF
```

Kebijakan per user: `eiri` dimasukkan ke blacklist, dan `mika` dibuat read-only:

```sh
echo "eiri" > /etc/vsftpd.user_list
mkdir -p /etc/vsftpd/user_conf
echo "write_enable=NO" > /etc/vsftpd/user_conf/mika

vsftpd /etc/vsftpd/vsftpd.conf &
```

<img width="822" height="615" alt="image" src="https://github.com/user-attachments/assets/40bf0891-3a5a-447d-aa2e-3a4cb37d035c" />


Test menggunakan user **alice** dari node Alice, dengan membuat file `signal_alice.txt`:

```sh
apk add lftp
echo "signal alice" > signal_alice.txt
lftp -u alice,alice123 192.236.2.10
lftp alice@192.236.2.10:/> put signal_alice.txt
```

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/165dbd53-99ef-48de-8eda-d9cb35eb4b85" />


### Soal 8: Upload Dokumen Laporan
Kelompok rahasia Knights perlu mengirimkan dokumen laporan intelijen ke FTP Server Chisa. Lakukan koneksi FTP client dari node Knights ke FTP Server Chisa menggunakan akun alice. Upload file berikut (link file). Analisis sesi Wireshark dan sebutkan: perintah FTP untuk upload (STOR), kode status sukses server (226), dan port data TCP yang dinegosiasikan pada mode PASV.

Dari node Knights, download file lalu upload ke FTP Server dengan akun alice, sambil capture pada link Knights ke Switch 3:

```sh
apk add lftp
wget -O knights_status_report.txt "<LINK_FILE>" --no-check-certificate
lftp -u alice,alice123 192.236.2.10
lftp alice@192.236.2.10:/> put knights_status_report.txt
```

<img width="1600" height="900" alt="WhatsApp Image 2026-09-16 at 8 05 01 AM" src="https://github.com/user-attachments/assets/20ce4877-3f5c-4cec-a303-e7680e561b58" />


Display filter pada Wireshark:

```
ftp || ftp-data
```

<img width="1600" height="900" alt="WhatsApp Image 2026-09-16 at 8 08 01 AM" src="https://github.com/user-attachments/assets/e45ea2db-d2bc-4b77-88f3-8a86a072a836" />



### Soal 9: Download Dokumen & Uji Read-Only
Mika mengakses dokumen Protokol Tujuh di (link file) dari FTP Server Chisa. Dari node Mika, unduh file tersebut menggunakan akun mika. Setelah itu, buktikan pembatasan read-only dengan mencoba mengunggah file baru dari akun mika, dan tunjukkan pesan error respon server (error 550 Permission denied) saat mika mencoba melakukan upload.

Pertama file diletakkan pada folder shared di node Chisa:

```sh
wget -O /var/wired/data/protocol_7_manifesto.txt "<LINK_FILE>" --no-check-certificate
chmod 644 /var/wired/data/protocol_7_manifesto.txt
```

Kemudian dari node Mika, download file dengan akun mika:

```sh
apk add lftp
lftp -u mika,mika123 192.236.2.10
lftp mika@192.236.2.10:/> get protocol_7_manifesto.txt
```


Lalu coba upload file baru untuk membuktikan read-only:

```sh
echo "Testing write permission from mika" > test_upload.txt
lftp mika@192.236.2.10:/> put test_upload.txt
```

<img width="1600" height="900" alt="WhatsApp Image 2026-09-16 at 8 15 14 AM" src="https://github.com/user-attachments/assets/226014d9-b811-4dce-804f-a16f611be89d" />


Terlihat download berhasil, sedangkan upload ditolak server dengan respons `550 Permission denied`.


### Soal 10: Uji Ketahanan Latensi Jaringan
Knights melancarkan uji ketahanan koneksi ke server Chisa untuk menguji latensi jaringan The Wired. Kirimkan paket ping dari node Knights ke node Chisa dengan payload khusus 128 bytes dan interval 0.3 detik sebanyak 77 paket (ping -c 77 -s 128 -i 0.3 <IP_Chisa>). Buka Wireshark, catat nilai ICMP Type dan Code untuk Echo Request vs Echo Reply, serta analisis packet loss dan RTT (min/avg/max).

Jalankan ping dari Knights sambil capture pada link Knights ke Switch 3:

```sh
ping -c 77 -s 128 -i 0.3 192.236.2.10
```

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/2536cbb4-2cd0-48ff-9026-f4b3701409c7" />


Display filter pada Wireshark:

```
icmp && ip.addr == 192.236.2.10
```

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/accfa155-72e6-4e91-b955-7692d6844348" />


