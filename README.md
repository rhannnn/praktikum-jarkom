# Praktikum Jaringan Komputer (GNS3)

Dokumentasi konfigurasi routing, NAT, DHCP (Kea), web server uji, serta simulasi packet loss, latency, dan pembatasan bandwidth untuk mengamati TCP congestion control (Cubic vs Reno).

## Daftar Isi

0. [DNS (Semua Node)](#0-dns-semua-node)
1. [Routing & NAT](#1-routing--nat)
2. [DHCP Server (Kea DHCP4)](#2-dhcp-server-kea-dhcp4)
3. [Web Server & Berkas Uji (Laterano)](#3-web-server--berkas-uji-laterano)
4. [Pengukuran dengan curl & Wireshark](#4-pengukuran-dengan-curl--wireshark)
5. [Skenario 2: Packet Loss 1%](#5-skenario-2-packet-loss-1)
6. [Skenario 3: Loss 1% + Delay 100 ms + Rate 2 Mbps](#6-skenario-3-loss-1--delay-100-ms--rate-2-mbps)
7. [TCP Congestion Control (Cubic vs Reno)](#7-tcp-congestion-control-cubic-vs-reno)
8. [Pengujian dengan iperf3](#8-pengujian-dengan-iperf3)

---

## 0. DNS (Semua Node)

Jalankan di **setiap node** agar bisa me-resolve nama domain (mis. saat `apk update` / `apk add`):

```sh
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Verifikasi:

```sh
cat /etc/resolv.conf
```

---

## 1. Routing & NAT

### RhodesIsland

```sh
ip route add 10.134.3.0/24 via 10.134.1.2 dev eth1
ip route add 10.134.4.0/24 via 10.134.2.2 dev eth2
iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.134.0.0/16 -o eth0
```

### Lungmen

```sh
ip route add 10.134.2.0/24 via 10.134.1.1 dev eth0
ip route add 10.134.4.0/24 via 10.134.1.1 dev eth0
```

### Ursus

```sh
ip route add 10.134.1.0/24 via 10.134.2.1 dev eth0
ip route add 10.134.3.0/24 via 10.134.2.1 dev eth0
```

---

## 2. DHCP Server (Kea DHCP4)

### Victoria

Instalasi:

```sh
apk update
apk add kea-dhcp4
```

Langkah konfigurasi:

```sh
rm /etc/kea/kea-dhcp4.conf
nano /etc/kea/kea-dhcp4.conf      # isi dengan konfigurasi di bawah
mkdir -p /var/lib/kea /run/kea
kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
```

`/etc/kea/kea-dhcp4.conf`:

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.134.3.0/24",
        "pools": [ { "pool": "10.134.3.100 - 10.134.3.120" } ],
        "option-data": [
          { "name": "routers", "data": "10.134.3.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ]
      }
    ]
  }
}
```

### Yan

Instalasi:

```sh
apk update
apk add kea-dhcp4
```

Langkah konfigurasi:

```sh
rm /etc/kea/kea-dhcp4.conf
nano /etc/kea/kea-dhcp4.conf      # isi dengan konfigurasi di bawah
mkdir -p /var/lib/kea /run/kea
kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
```

`/etc/kea/kea-dhcp4.conf`:

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.134.4.0/24",
        "pools": [ { "pool": "10.134.4.50 - 10.134.4.60" } ],
        "option-data": [
          { "name": "routers", "data": "10.134.4.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ],
        "reservations": [
          { "hw-address": "MAC-ETH0-LATERANO", "ip-address": "10.134.4.15" }
        ]
      }
    ]
  }
}
```

> **Catatan:** ganti `MAC-ETH0-LATERANO` dengan MAC address `eth0` milik Laterano agar mendapat IP tetap `10.134.4.15`.

Cara mendapatkan MAC address, jalankan di **Laterano**:

```sh
ip link show eth0
```

Contoh output:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 0c:ab:12:34:56:00 brd ff:ff:ff:ff:ff:ff
```

MAC address ada di baris `link/ether` (di contoh: `0c:ab:12:34:56:00`). Salin ke `hw-address` pada konfigurasi Yan:

```json
{ "hw-address": "0c:ab:12:34:56:00", "ip-address": "10.134.4.15" }
```

---

## 3. Web Server & Berkas Uji (Laterano)

Instal Python terlebih dahulu:

```sh
apk update && apk add python3
```

Skrip `/root/myscripts/start_http.sh`:

```sh
#!/bin/sh

mkdir -p myweb
cd myweb

python3 -m http.server 9999 -d /root/myscripts/myweb
```

Berkas uji dibuat di `/root/myscripts/myweb`:

```sh
dd if=/dev/random of=tes1000.bin  bs=1    count=1000
dd if=/dev/random of=tes10000.bin bs=10   count=1000
dd if=/dev/random of=tes50000.bin bs=50   count=1000
dd if=/dev/random of=tes5M.bin    bs=5000 count=1000
```

| Berkas          | Ukuran  |
| --------------- | ------- |
| `tes1000.bin`   | 1 KB    |
| `tes10000.bin`  | 10 KB   |
| `tes50000.bin`  | 50 KB   |
| `tes5M.bin`     | 5 MB    |

---

## 4. Pengukuran dengan curl & Wireshark

Unduh berkas dan tampilkan ukuran, kecepatan, serta total waktu:

```sh
curl -o /dev/null -s -w "size=%{size_download}B speed=%{speed_download}B/s total=%{time_total}s\n" \
  http://10.134.4.15:9999/tes10000.bin
```

Filter Wireshark untuk melihat satu aliran TCP:

```
tcp.stream == 2
```

---

## 5. Skenario 2: Packet Loss 1%

Dijalankan di **RhodesIsland**.

Hapus qdisc lama:

```sh
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
```

Tambahkan packet loss 1%:

```sh
tc qdisc add dev eth1 root netem loss 1%
tc qdisc add dev eth2 root netem loss 1%
```

Verifikasi:

```sh
tc qdisc show dev eth1
tc qdisc show dev eth2
```

---

## 6. Skenario 3: Loss 1% + Delay 100 ms + Rate 2 Mbps

Dijalankan di **RhodesIsland**. Jaringan diberi packet loss 1%, latency 100 ms, dan pembatasan rate 2 Mbps pada `eth1` dan `eth2`, lalu perilaku TCP congestion control Cubic dan Reno diamati.

```sh
# eth1
tc qdisc add dev eth1 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
tc qdisc add dev eth1 parent 1:1 handle 10: netem delay 100ms loss 1%

# eth2
tc qdisc add dev eth2 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
tc qdisc add dev eth2 parent 1:1 handle 10: netem delay 100ms loss 1%
```

Verifikasi:

```sh
tc qdisc show dev eth1
tc qdisc show dev eth2
```

> Jika qdisc sudah ada sebelumnya, hapus dulu dengan `tc qdisc del dev eth1 root` dan `tc qdisc del dev eth2 root`.

---

## 7. TCP Congestion Control (Cubic vs Reno)

Dijalankan di **Laterano**.

### Cubic

```sh
ip tcp_metrics flush
sysctl -w net.ipv4.tcp_no_metrics_save=1
sysctl -w net.ipv4.tcp_congestion_control=cubic
sysctl net.ipv4.tcp_congestion_control
```

### Reno

```sh
sysctl -w net.ipv4.tcp_congestion_control=reno
ip tcp_metrics flush
sysctl net.ipv4.tcp_congestion_control
```

---

## 8. Pengujian dengan iperf3

Di **Laterano** (server):

```sh
iperf3 -s
```

Di sisi client (ganti `[IP Prefix]` dengan prefix jaringan, mis. `10.134`):

```sh
# TCP
iperf3 -c [IP Prefix].4.15 -t 10

# UDP
iperf3 -c [IP Prefix].4.15 -u -b 5M -t 10
```
