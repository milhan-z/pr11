# Panduan Pengerjaan — Topologi Lab Komputer Sekolah (DHCP & Hub vs Switch)

Prefix IP: `10.92` — subnet yang dipakai: `10.92.1.0/24`.

| Node | Interface | IP |
|---|---|---|
| PC-Guru (DHCP server) | eth0 | `10.92.1.1/24` (statis) |
| PC-Siswa-1 s/d 4 | eth0 | DHCP, pool `10.92.1.10 – 10.92.1.50` |
| PC-Pengawas | eth0 | tanpa IP (hanya capture) |

## 1. Topologi

- 1 Ethernet switch + 6 node Linux (PC-Siswa-1..4, PC-Guru, PC-Pengawas).
- Kabel: Siswa-1→e0, Siswa-2→e1, Siswa-3→e2, Siswa-4→e3, Guru→e4, Pengawas→e5 (semua lewat eth0).

Paket yang dibutuhkan: `isc-dhcp-server` (Guru), `termshark` (Pengawas), `mtr` (Siswa/Guru).
Jika belum ada, pasang NAT sementara ke **eth1** node yang perlu install, lalu lepas.
Jangan hubungkan NAT ke Switch1 — DHCP bawaan NAT GNS3 akan bentrok dengan PC-Guru.

## 2. Konfigurasi IP (Edit network configuration)

**PC-Guru**
```
auto eth0
iface eth0 inet static
    address 10.92.1.1
    netmask 255.255.255.0
```

**PC-Siswa-1 s/d 4**
```
auto eth0
iface eth0 inet dhcp
```

**PC-Pengawas**
```
auto eth0
iface eth0 inet manual
    up ip link set eth0 up
```

## 3. DHCP Server di PC-Guru

```bash
echo 'INTERFACESv4="eth0"' > /etc/default/isc-dhcp-server
```

`/etc/dhcp/dhcpd.conf`:
```
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 10.92.1.0 netmask 255.255.255.0 {
    range 10.92.1.10 10.92.1.50;
    always-broadcast on;
}
```

```bash
rm -f /var/run/dhcpd.pid
service isc-dhcp-server restart
service isc-dhcp-server status
```

> `always-broadcast on;` wajib — tanpanya DHCP OFFER dikirim unicast dan tidak akan tertangkap di PC-Pengawas.

Nyalakan PC-Guru lebih dulu, baru PC-Siswa. Jika siswa belum dapat IP: `dhclient -r eth0 && dhclient -v eth0`.

## 4. Verifikasi DHCP

```bash
ip a show eth0                      # di tiap PC-Siswa
cat /var/lib/dhcp/dhcpd.leases      # di PC-Guru
```

## 5. Uji Konektivitas

```bash
ping -c 5 <IP tujuan>
mtr -r -c 10 <IP tujuan>
```
Target: semua reply, 0% packet loss (Guru ↔ semua siswa, dan antarsiswa).

## 6. Capture di PC-Pengawas

```bash
termshark -i eth0
```

**DHCP broadcast** — di salah satu siswa:
```bash
dhclient -r eth0 && dhclient -v eth0
```
Filter `bootp` / `dhcp` → DISCOVER & OFFER ke `ff:ff:ff:ff:ff:ff` / `255.255.255.255`.

**ARP broadcast & ping unicast** — di siswa lain:
```bash
ip neigh flush all
ping -c 3 <IP siswa lain>
```
- Filter `arp` → ARP request broadcast terlihat.
- Filter `icmp` → kosong (ping antarsiswa unicast, tidak sampai ke Pengawas).

## 7. Penjelasan untuk Asisten

**Hub vs Switch.** Hub bekerja di layer 1 dan meneruskan setiap frame ke semua port, sehingga PC-Pengawas akan ikut melihat ping antarsiswa. Switch bekerja di layer 2 dengan MAC table: frame unicast hanya diteruskan ke port tujuan, sedangkan frame broadcast tetap disebar ke semua port. Itu sebabnya ARP dan DHCP discover/offer tertangkap di Pengawas, tetapi ICMP antarsiswa tidak. Switch lebih efisien (collision domain per port) dan lebih aman.

**Peran DHCP.** Tanpa DHCP, Sunny harus mengatur IP tiap PC secara manual — lambat dan rawan salah (IP ganda, subnet keliru). Dengan PC-Guru sebagai DHCP server, setiap PC siswa otomatis mendapat IP melalui proses DORA (Discover, Offer, Request, Ack) saat dinyalakan. PC-Guru tetap memakai IP statis karena server DHCP harus memiliki alamat tetap.
