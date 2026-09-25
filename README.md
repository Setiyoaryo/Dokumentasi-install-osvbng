# Dokumentasi Install OSVBNG

Dokumentasi pribadi untuk mempelajari, memasang, menguji, dan memahami OSVBNG sebagai Broadband Network Gateway (BNG).

Repository ini mendokumentasikan dua metode instalasi:

1. **Native Linux** — OSVBNG/VPP/FRR dijalankan langsung pada host Linux.
2. **Docker** — OSVBNG dijalankan di container dengan network namespace dan interface yang dipetakan ke host.

Dokumentasi ini berfokus pada **fungsi BNG dan troubleshooting**, bukan benchmarking throughput atau optimasi DPDK.

---

## Status Progress

### Sudah berhasil diuji

- OSVBNG v0.16.0
- VPP 26.06-release
- Docker deployment
- Access/Core interface mapping
- VPP + FRR startup
- Local authentication
- PPPoE subscriber
- IPv4 subscriber pool
- VLAN subscriber
- MikroTik sebagai PPPoE CPE
- Subscriber mendapatkan IPv4 `10.255.0.2`
- OSVBNG API via UDS `/run/osvbng/api.sock`

### Roadmap berikutnya

- RADIUS Authentication
- RADIUS Accounting
- CoA / Disconnect
- QinQ
- Core routing
- IPoE / DHCP
- QoS
- IPv6 / DHCPv6 / Prefix Delegation
- CGNAT
- High Availability
- Production hardening

---

# 1. Arsitektur Lab

Topology lab yang digunakan:

```
                    CORE
                 10.0.0.1/30
                     |
                     |
                eth1 / CORE
                  OSVBNG
                10.0.0.2/30
                     |
                eth0 / ACCESS
                     |
                  VLAN 100
                     |
                 MikroTik
                 PPPoE CPE
```

Host Linux:

| Interface | Fungsi | IP |
|---|---|---|
| `ens3` | CORE | `10.0.0.2/30` |
| `ens4` | CUSTOMER/ACCESS | `10.10.0.1/24` |

Di container OSVBNG:

| Interface | Fungsi |
|---|---|
| `eth0` | ACCESS |
| `eth1` | CORE |

Docker mapping:

```
ens4 <-> br-eth0 <-> veth-eth0 <-> eth0
ens3 <-> br-eth1 <-> veth-eth1 <-> eth1
```

Subscriber:

- Gateway: `10.255.0.1/32`
- Pool: `10.255.0.0/16`
- Contoh subscriber: `10.255.0.2`

---

# 2. Instalasi Native Linux

## Catatan

Metode native berarti komponen OSVBNG dijalankan langsung pada Linux, tanpa container isolation.

Komponen utama:

- OSVBNG
- VPP
- FRR
- iproute2
- systemd/service management
- konfigurasi namespace dan interface

> Dokumentasi native harus mengikuti binary, package, release OSVBNG, VPP, dan FRR yang benar-benar digunakan. Jangan mengasumsikan layout Docker berlaku identik pada native installation.

## Tahapan

### 2.1 Siapkan host

Minimal pastikan:

```bash
uname -a
ip -br link
free -h
df -h
```

Periksa CPU dan memory karena VPP dapat menjadi komponen paling sensitif terhadap resource.

### 2.2 Pasang dependency

Dependency mengikuti release OSVBNG yang digunakan.

Komponen yang perlu tersedia:

```
VPP
FRR
iproute2
hugetlbfs/hugepages bila dibutuhkan oleh build
```

### 2.3 Siapkan interface

Contoh:

```
ens3 = CORE
ens4 = ACCESS
```

Native deployment harus memastikan interface yang dipakai OSVBNG benar-benar tersedia dan tidak sedang digunakan oleh service lain.

### 2.4 Konfigurasi OSVBNG

Konsep konfigurasi yang dipakai:

- access interface
- core interface
- loopback
- subscriber gateway
- IPv4 pool
- subscriber group
- AAA policy
- PPPoE
- API

Contoh konsep:

```yaml
interfaces:
  - name: eth0
    type: physical
    enabled: true

  - name: eth1
    type: physical
    enabled: true
    lcp: true

  - name: loop0
    type: loopback
    enabled: true

  - name: loop100
    type: loopback
    enabled: true
```

Native deployment tetap perlu divalidasi menggunakan generator/config validator OSVBNG.

---

# 3. Instalasi Docker

## 3.1 Image

Image yang digunakan:

```
veesixnetworks/osvbng:latest
```

Version binary yang berhasil diuji:

```
osvbngd --version
v0.16.0
```

VPP:

```
v26.06-release
```

## 3.2 Container

Karena OSVBNG membutuhkan akses networking/VPP:

```bash
docker run -d \
  --name osvbng \
  --privileged \
  --network none \
  -v /opt/osvbng/osvbng.yaml:/etc/osvbng/osvbng.yaml:ro \
  -e OSVBNG_WAIT_FOR_INTERFACES=true \
  -e OSVBNG_ACCESS_INTERFACE=eth0 \
  -e OSVBNG_CORE_INTERFACE=eth1 \
  veesixnetworks/osvbng:latest
```

Interface dibuat setelah container berjalan:

```bash
./setup-interfaces.sh osvbng eth0:ens4 eth1:ens3
```

Verifikasi:

```
ens4 <-> br-eth0 <-> veth-eth0 <-> eth0
ens3 <-> br-eth1 <-> veth-eth1 <-> eth1
```

## 3.3 Kenapa `--privileged`?

OSVBNG container perlu melakukan operasi yang berhubungan dengan:

- network namespace
- VPP
- interface
- namespace dataplane
- routing daemons
- sysctl tertentu
- hugepages

Tanpa capability yang sesuai, startup dapat gagal pada tahap yang berbeda.

Untuk production, privilege harus direview kembali dan dibatasi sesuai kebutuhan deployment.

---

# 4. Konfigurasi PPPoE Local Authentication

Subscriber group yang diuji:

```yaml
subscriber-groups:
  groups:
    residential:
      vlan-tpid: dot1q
      ipv4-profile: residential-v4
      vlans:
        - svlan: "100"
          cvlan: any
          interface: loop100
          parent-interface: eth0
          access-types:
            - pppoe
      aaa-policy: pppoe-policy
```

IPv4 pool:

```yaml
ipv4:
  profiles:
    residential-v4:
      gateway: 10.255.0.1
      dns:
        - 8.8.8.8
        - 8.8.4.4
      pools:
        - name: subscriber-pool
          network: 10.255.0.0/16
          priority: 1
```

Local user dibuat melalui CLI:

```text
exec subscriber auth local users create --username user1 --password test --enabled true
```

Verifikasi:

```text
show subscriber auth local users
```

---

# 5. PPPoE Test dengan MikroTik

Pada sisi MikroTik, PPPoE client harus menggunakan VLAN subscriber yang sesuai.

Contoh:

```routeros
/interface vlan
add name=vlan100 interface=ether1 vlan-id=100

/interface pppoe-client
add name=pppoe-test \
    interface=vlan100 \
    user=user1 \
    password=test \
    add-default-route=no \
    use-peer-dns=no \
    disabled=no
```

Hasil test:

- Session state: `active`
- AccessType: `pppoe`
- Username: `user1`
- Outer VLAN: `100`
- IPv4 address: `10.255.0.2`
- Service Group: `residential`

Verifikasi di OSVBNG:

```text
show subscriber sessions
```

---

# 6. Common Issues & Troubleshooting

## 6.1 VPP mati setelah startup

### Gejala

```text
Dataplane process not running
```

atau:

```text
vppctl: connect: connection refused
```

### Root cause yang ditemukan

Pada lab 2 GiB RAM, Linux OOM killer beberapa kali membunuh `vpp_main`.

Contoh:

```text
Out of memory: Killed process ... (vpp_main)
```

### Kenapa menjadi issue?

VPP dapat membutuhkan memory yang signifikan saat startup. Host lab hanya memiliki 2 GiB RAM dan sebelumnya OSVBNG entrypoint meminta 512 hugepages 2 MiB:

```
512 x 2 MiB = 1 GiB
```

Dengan RAM terbatas dan tanpa swap, memory pressure dapat menyebabkan OOM.

### Solusi lab

Kurangi hugepages:

```bash
echo 64 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

Tambahkan swap jika storage memungkinkan.

---

## 6.2 Host disk penuh

### Gejala

```text
No space left on device
```

Docker bahkan dapat gagal membuat container atau layer baru.

### Root cause yang ditemukan

VM GNS3 awalnya hanya mempunyai sekitar 2.8 GB disk.

### Solusi

Perbesar disk QEMU dari GNS3, lalu extend partition/filesystem.

Contoh hasil akhir:

```text
/dev/sda1   13G   2.4G   9.5G   20% /
```

Untuk partition GPT Debian:

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```

---

## 6.3 `growpart: command not found`

### Penyebab

Package `cloud-guest-utils` belum terpasang.

### Solusi

```bash
sudo apt update
sudo apt install -y cloud-guest-utils
```

---

## 6.4 Spam kernel ICMP

### Gejala

```text
icmp: detected local route for 10.0.0.2 during ICMP sending
```

### Penyebab

Pesan berasal dari kernel networking host, bukan dari osvbngd.

### Menurunkan console log level

```bash
sudo dmesg -n 3
```

Agar persistent:

```bash
sudo tee /etc/sysctl.d/99-console-loglevel.conf >/dev/null <<'EOF'
kernel.printk = 3 4 1 3
EOF

sudo sysctl --system
```

Ini tidak memperbaiki route/ICMP. Ini hanya mengurangi pesan kernel yang ditampilkan ke console.

---

## 6.5 FRR berhenti di "Waiting for children to finish applying config"

### Gejala

```text
Waiting for children to finish applying config...
```

### Penyebab pada lab

OSVBNG image mencoba menyalakan banyak routing daemon sekaligus:

```
bgpd
ospfd
ospf6d
isisd
bfdd
ldpd
```

Untuk tahap awal PPPoE local auth, daemon tersebut belum dibutuhkan.

### Solusi lab

Gunakan minimal:

```
bgpd=no
ospfd=no
ospf6d=no
isisd=no
bfdd=no
ldpd=no

vtysh_enable=yes

zebra_options="  -A 127.0.0.1 -s 67108864 -M dplane_fpm_nl"
staticd_options="-A 127.0.0.1"

log_file={{ .LogFile }}
```

Saat mulai belajar BGP/OSPF/MPLS, daemon harus diaktifkan kembali sesuai kebutuhan.

---

## 6.6 `af_packet_create_v2_reply` terlalu lama

### Gejala

```text
reply is taking too long (>1s): af_packet_create_v2_reply
```

### Interpretasi

Operasi tersebut berkaitan dengan pembuatan AF_PACKET host-interface. Jangan langsung menyimpulkan bug AF_PACKET.

Pada lab ini ternyata VPP kemudian terbukti dibunuh OOM.

### Troubleshooting

Periksa:

```bash
dmesg -T | grep -Ei 'vpp|oom|killed process|af_packet'
```

Jika ada:

```text
Out of memory: Killed process ... vpp_main
```

maka fokus ke resource host, bukan ke VLAN/PPPoE.

---

## 6.7 PPPoE MikroTik "connecting" lalu disconnect

### Gejala

```text
pppoe-test: connecting...
pppoe-test: terminating... - disconnected
```

dan OSVBNG:

```text
show subscriber sessions

No data
```

### Penyebab

Pada test awal MikroTik mengirim PPPoE untagged pada `ether1`, sedangkan subscriber group OSVBNG menggunakan VLAN 100.

### Solusi

Gunakan VLAN 100 di MikroTik:

```routeros
/interface vlan
add name=vlan100 interface=ether1 vlan-id=100
```

Kemudian PPPoE client:

```routeros
/interface pppoe-client
set pppoe-test interface=vlan100 disabled=no
```

Setelah itu session berhasil:

```text
State        active
AccessType   pppoe
Username     user1
OuterVLAN     100
IPv4Address  10.255.0.2
ServiceGroup residential
```

---

## 6.8 API `curl localhost:8080` connection refused

### Penyebab yang mungkin

API belum startup atau listener tidak dikonfigurasi dengan format yang benar.

OSVBNG v0.16.0 menggunakan listener object:

```yaml
plugins:
  northbound.api:
    enabled: true
    listeners:
      - address: ":8080"
```

Selain TCP listener, UDS digunakan:

```text
/run/osvbng/api.sock
```

CLI `osvbngcli` dapat memakai UDS secara otomatis.

---

## 6.9 `listeners` salah format

Jangan menggunakan:

```yaml
listeners:
  - ":8080"
```

Gunakan:

```yaml
listeners:
  - address: ":8080"
```

Pada v0.16.0 listener didefinisikan sebagai object, bukan string.

---

# 7. Resource Requirement Lab

Lab yang diuji memakai VM dengan:

- 2 vCPU
- 2 GiB RAM
- disk awal sekitar 2.8 GB

Untuk penggunaan seperti ini:

- Disk > 10 GB lebih nyaman.
- Swap berguna.
- Hugepages harus diperhatikan.
- Jangan mengaktifkan seluruh routing stack bila belum diperlukan.

Untuk production, sizing tidak boleh menggunakan angka lab ini secara langsung.

---

# 8. Validation Checklist

## OSVBNG

```bash
docker ps
docker logs osvbng
```

## VPP

```bash
docker exec osvbng vppctl -s /run/osvbng/cli.sock show version
docker exec osvbng vppctl -s /run/osvbng/cli.sock show interface
```

## FRR

```bash
docker exec osvbng ip netns exec dataplane /usr/lib/frr/frrinit.sh status
```

## API

```bash
docker exec osvbng ls -lah /run/osvbng/api.sock
```

## Subscriber

```text
show subscriber auth local users
show subscriber sessions
```

---

# 9. Current Milestone

Milestone saat ini:

```
OSVBNG
  |
  +-- VPP                        [OK]
  +-- FRR minimal                [OK]
  +-- API                        [OK]
  +-- Local AAA                  [OK]
  +-- PPPoE                      [OK]
  +-- VLAN subscriber            [OK]
  +-- IPv4 pool                  [OK]
  +-- MikroTik PPPoE CPE         [OK]
  +-- Subscriber IP assignment   [OK]
```

Contoh session yang berhasil:

```
username       = user1
access-type    = pppoe
outer-vlan     = 100
ipv4-address   = 10.255.0.2
service-group  = residential
state          = active
```

---

## Next

Next milestone adalah **RADIUS Authentication**, kemudian **RADIUS Accounting**.

Local authentication tetap dipertahankan sebagai baseline test agar setiap perubahan dapat dibandingkan dengan konfigurasi yang sudah terbukti bekerja.

> Dokumentasi ini adalah catatan lab dan pembelajaran. Konfigurasi production wajib melalui validasi, security review, resource sizing, monitoring, backup, dan failure testing terlebih dahulu.
