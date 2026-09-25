# Panduan Pengerjaan: Topologi Lab Komputer Sekolah (DHCP, Broadcast vs Unicast)

Panduan ini mengikuti **Modul 1 Jarkom**, bagian [2.5 Switch](https://github.com/arsitektur-jaringan-komputer/Modul-Jarkom/tree/master/Modul-1#25-beralih-dari-bridge-ke-switch), [2.6 DHCP (udhcpd/udhcpc)](https://github.com/arsitektur-jaringan-komputer/Modul-Jarkom/tree/master/Modul-1#261-konfigurasi-dhcp-server-udhcpd), [0.6 MTR](https://github.com/arsitektur-jaringan-komputer/Modul-Jarkom/tree/master/Modul-1#06-mtr), dan [5. Termshark](https://github.com/arsitektur-jaringan-komputer/Modul-Jarkom/tree/master/Modul-1#5-termshark).

## Daftar Isi
1. [Rencana Alamat IP](#rencana-alamat-ip)
2. [Soal 1: Membuat Topologi](#soal-1-membuat-topologi)
3. [Soal 2: DHCP Server dan DHCP Client](#soal-2-dhcp-server-dan-dhcp-client)
4. [Soal 3: Uji Konektivitas (ping & mtr)](#soal-3-uji-konektivitas-ping--mtr)
5. [Soal 4: Capture di PC-Pengawas (Termshark)](#soal-4-capture-di-pc-pengawas-termshark)
6. [Soal 5: Penjelasan Hub vs Switch dan Peran DHCP](#soal-5-penjelasan-hub-vs-switch-dan-peran-dhcp)
7. [Checklist Poin Keberhasilan](#checklist-poin-keberhasilan)

---

## Rencana Alamat IP

Prefix IP yang dibagikan: **`10.92`**. Subnet yang dipakai: **`10.92.1.0/24`**.

| Node | Port Switch | Interface | Konfigurasi IP |
|---|---|---|---|
| PC-Siswa-1 | e0 | eth0 | DHCP client |
| PC-Siswa-2 | e1 | eth0 | DHCP client |
| PC-Siswa-3 | e2 | eth0 | DHCP client |
| PC-Siswa-4 | e3 | eth0 | DHCP client |
| PC-Guru | e4 | eth0 | Statis `10.92.1.101/24` (DHCP server) |
| PC-Pengawas | e5 | eth0 | Statis `10.92.1.200/24` (capture point) |

Pool DHCP: **`10.92.1.150` – `10.92.1.155`**, sama seperti rentang pada modul.

---

## Soal 1: Membuat Topologi

1. Buat project baru di GNS3.
2. Tambahkan **1 Ethernet switch** (`Switch1`) dan **6 node `netics-pc`**. Ganti nama node menjadi `PC-Siswa-1`, `PC-Siswa-2`, `PC-Siswa-3`, `PC-Siswa-4`, `PC-Guru`, dan `PC-Pengawas`.
3. Hubungkan `eth0` setiap node ke port switch sesuai tabel di atas (e0 sampai e5).
4. **Jangan start node dulu.** Isi konfigurasi jaringan setiap node terlebih dahulu (Soal 2).

---

## Soal 2: DHCP Server dan DHCP Client

### 2.1 Konfigurasi IP statis PC-Guru dan PC-Pengawas

Klik kanan node → **Configure** → **Edit Network Configuration**.

**PC-Guru**
```
auto eth0
iface eth0 inet static
    address 10.92.1.101
    netmask 255.255.255.0
```

**PC-Pengawas**
```
auto eth0
iface eth0 inet static
    address 10.92.1.200
    netmask 255.255.255.0
```

> IP PC-Pengawas berada di luar pool DHCP. Selama pengujian, PC-Pengawas tidak mengirim ping ke siapa pun, jadi node ini murni menjadi pengamat.

### 2.2 Konfigurasi DHCP server (udhcpd) di PC-Guru

Start **PC-Guru**, lalu buka console-nya.

1. Pastikan IP statis sudah terpasang:
   ```
   ip -br addr
   ```
   Jika belum ada, pasang manual:
   ```
   ip addr add 10.92.1.101/24 dev eth0
   ip link set eth0 up
   ```

2. Buat file lease:
   ```
   touch /etc/dhcpd.leases
   ```

3. Buat file konfigurasi `/etc/dhcpd.conf`:
   ```
   echo "start 10.92.1.150
   end 10.92.1.155
   interface eth0
   max_leases 5
   pidfile /etc/dhcpd.pid
   lease_file /etc/dhcpd.leases
   option subnet 255.255.255.0" > /etc/dhcpd.conf
   ```

4. Jalankan DHCP server:
   ```
   udhcpd -f /etc/dhcpd.conf
   ```
   Opsi `-f` membuat server berjalan di foreground, jadi biarkan console ini tetap terbuka. Untuk perintah lain di PC-Guru, buka console kedua.

> `udhcpd` mengirim OFFER/ACK secara **broadcast** ketika client belum punya IP (`ciaddr = 0`). Karena itu DHCP Offer juga ikut tertangkap di PC-Pengawas (Soal 4).

### 2.3 Konfigurasi DHCP client di PC-Siswa-1 sampai 4

Klik kanan node → **Configure** → **Edit Network Configuration**. Uncomment bagian DHCP dan pastikan konfigurasi statis dalam keadaan ter-comment.

**PC-Siswa-1**
```
auto eth0
iface eth0 inet dhcp
    hostname PC-Siswa-1
```

**PC-Siswa-2**
```
auto eth0
iface eth0 inet dhcp
    hostname PC-Siswa-2
```

**PC-Siswa-3**
```
auto eth0
iface eth0 inet dhcp
    hostname PC-Siswa-3
```

**PC-Siswa-4**
```
auto eth0
iface eth0 inet dhcp
    hostname PC-Siswa-4
```

Save, lalu **start keempat PC-Siswa setelah `udhcpd` di PC-Guru berjalan**. Setiap siswa akan otomatis meminta IP saat booting. Jika siswa sudah terlanjur menyala, restart node tersebut (Stop, lalu Start).

### 2.4 Verifikasi

Di setiap PC-Siswa:
```
ip -br addr
```
`eth0` harus mendapat IP di rentang `10.92.1.150` – `10.92.1.155`.

Di PC-Guru, log `udhcpd` akan menampilkan proses pemberian IP (offer/ack) ke setiap siswa.

Catat IP yang didapat setiap siswa. IP tersebut dipakai di Soal 3 dan Soal 4.

| Node | IP dari DHCP |
|---|---|
| PC-Siswa-1 | `10.92.1.___` |
| PC-Siswa-2 | `10.92.1.___` |
| PC-Siswa-3 | `10.92.1.___` |
| PC-Siswa-4 | `10.92.1.___` |

> **Jika ada siswa yang belum mendapat IP** (misalnya menyala sebelum server siap), minta IP manual sesuai modul bagian 2.6.3:
> ```
> udhcpc -i eth0 -b
> ```

---

## Soal 3: Uji Konektivitas (ping & mtr)

Buktikan semua PC saling terhubung dengan **0% packet loss**.

**Dari PC-Guru ke keempat siswa:**
```
ping -c 4 10.92.1.<IP-Siswa-1>
ping -c 4 10.92.1.<IP-Siswa-2>
ping -c 4 10.92.1.<IP-Siswa-3>
ping -c 4 10.92.1.<IP-Siswa-4>
```

**Antarsiswa dan dari siswa ke guru**, contohnya dari PC-Siswa-1:
```
ping -c 4 10.92.1.101
ping -c 4 10.92.1.<IP-Siswa-2>
```

**MTR** (mode report):
```
mtr -r -c 10 -n 10.92.1.101
mtr -r -c 10 -n 10.92.1.<IP-Siswa-lain>
```
Kolom **Loss%** harus menunjukkan `0.0%`. Karena semua node berada dalam satu switch, hanya ada **1 hop**, yaitu langsung ke tujuan.

---

## Soal 4: Capture di PC-Pengawas (Termshark)

Buka console **PC-Pengawas**, lalu jalankan:
```
termshark -i eth0
```
Biarkan capture berjalan. Semua trafik dipicu dari node **lain**, bukan dari PC-Pengawas.

### 4.1 DHCP Discover/Offer bersifat broadcast

Di salah satu siswa, misalnya **PC-Siswa-3**, lepas IP lalu minta ulang:
```
pkill udhcpc
ip addr flush dev eth0
udhcpc -i eth0 -b
```
Cara lainnya adalah restart node PC-Siswa-3 dari GNS3.

Di Termshark, isi display filter:
```
dhcp
```
(atau `bootp` jika versi tshark-nya lama)

Hasil yang diharapkan:
- **DHCP Discover**: `0.0.0.0` → `255.255.255.255`, MAC tujuan `ff:ff:ff:ff:ff:ff`
- **DHCP Offer**: `10.92.1.101` → `255.255.255.255`, MAC tujuan `ff:ff:ff:ff:ff:ff`

Paket-paket ini tertangkap di PC-Pengawas walaupun PC-Pengawas bukan pengirim maupun tujuan. Artinya, paket tersebut **broadcast**.

### 4.2 ARP bersifat broadcast

Di **PC-Siswa-1**, kosongkan ARP cache lalu ping PC-Siswa-2:
```
ip neigh flush all
ping -c 4 10.92.1.<IP-Siswa-2>
```

Di Termshark, isi display filter:
```
arp
```
Akan terlihat **ARP Request** "Who has 10.92.1.x? Tell 10.92.1.y" dengan tujuan `ff:ff:ff:ff:ff:ff`. Paket ini broadcast, sehingga ikut sampai ke PC-Pengawas.

### 4.3 Ping antar 2 PC bersifat unicast

Masih dari percobaan 4.2 (ping PC-Siswa-1 → PC-Siswa-2), ganti display filter menjadi:
```
icmp
```
Hasilnya **kosong**: tidak ada ICMP Echo Request/Reply yang tertangkap. Setelah ARP selesai, switch sudah mencatat MAC kedua siswa di **MAC Address Table**, sehingga paket ping hanya diteruskan ke port tujuan dan tidak sampai ke PC-Pengawas.

> Screenshot yang disarankan: (1) filter `dhcp` berisi Discover/Offer, (2) filter `arp` berisi ARP Request broadcast, (3) filter `icmp` kosong, dan (4) console PC-Siswa-1 yang menunjukkan ping sukses di waktu yang sama.

---

## Soal 5: Penjelasan Hub vs Switch dan Peran DHCP

### Hub vs Switch

| Aspek | Hub | Switch |
|---|---|---|
| Layer OSI | Layer 1 (Physical) | Layer 2 (Data Link) |
| Cara meneruskan frame | Menyalin setiap frame ke **semua port** | Membaca MAC tujuan dan meneruskan **hanya ke port tujuan** (berdasarkan MAC Address Table) |
| Broadcast | Diteruskan ke semua port | Diteruskan ke semua port |
| Unicast | Tetap diteruskan ke semua port | Hanya ke port tujuan |
| Collision domain | Satu untuk semua port | Satu per port |
| Keamanan | Semua node bisa menyadap trafik node lain | Trafik unicast tidak terlihat oleh node lain |

Hasil capture di Soal 4 membuktikan perbedaan ini. Pada switch, PC-Pengawas hanya melihat trafik **broadcast** (ARP Request, DHCP Discover/Offer), sedangkan ping **unicast** antara PC-Siswa-1 dan PC-Siswa-2 tidak terlihat. Jika Switch1 diganti hub, ping tersebut juga akan tertangkap di PC-Pengawas.

### Peran DHCP dalam pekerjaan Sunny

Tanpa DHCP, Sunny harus mengatur IP setiap PC di laboratorium satu per satu. Cara ini lambat, rawan salah ketik, dan berisiko menimbulkan **IP conflict** (dua PC dengan IP yang sama).

Dengan **PC-Guru sebagai DHCP server (udhcpd)**, setiap PC siswa cukup dikonfigurasi `inet dhcp` sekali. Setiap kali dinyalakan, PC siswa otomatis mendapatkan IP melalui proses **DORA**:

1. **Discover**: siswa mengirim broadcast untuk mencari DHCP server.
2. **Offer**: PC-Guru menawarkan IP dari pool `10.92.1.150–155`.
3. **Request**: siswa meminta IP yang ditawarkan.
4. **Acknowledge**: PC-Guru mengonfirmasi dan mencatat lease di `/etc/dhcpd.leases`.

PC-Guru tetap memakai IP statis karena server DHCP harus memiliki alamat yang tetap agar selalu bisa ditemukan oleh client. Hasilnya, pengelolaan IP menjadi terpusat dan penambahan atau penggantian PC tidak memerlukan konfigurasi manual lagi.

---

## Checklist Poin Keberhasilan

- [ ] **Poin 1**: ping dan mtr antar semua PC (guru ↔ siswa, siswa ↔ siswa) reply, 0% packet loss
- [ ] **Poin 2**: keempat PC-Siswa mendapat IP otomatis dari `udhcpd` di PC-Guru
- [ ] **Poin 3**: ARP dan DHCP Discover/Offer tertangkap di PC-Pengawas (broadcast)
- [ ] **Poin 4**: ICMP ping antara 2 siswa tidak tertangkap di PC-Pengawas (unicast)
- [ ] **Poin 5**: penjelasan hub vs switch dan peran DHCP
