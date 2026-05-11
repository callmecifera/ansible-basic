# 🛠️ ansible-basic

> Latihan Ansible untuk konfigurasi infrastruktur jaringan server — multi-role setup meliputi DNS, DHCP, Firewall, LDAP, Mail, NFS, VPN, dan Web.

---

## 📋 Deskripsi

Project ini adalah lab latihan Ansible untuk mengotomatisasi konfigurasi **server infrastruktur jaringan** dalam satu environment lokal. Semua role dirancang untuk berjalan di satu server yang berperan sebagai gateway sekaligus server layanan.

**Topologi jaringan:**
```
Internet
    │
[ens34 - NAT]
    │
  Server (172.16.218.1)
    │
[ens33 - Host-Only]
    │
LAN: 172.16.218.0/24
```

---

## 🗂️ Struktur Project

```
ansible-basic/
└── ansible/
    ├── group_vars/
    │   ├── all.yml          # Semua variabel konfigurasi (network, service, firewall)
    │   └── all.template     # Template variabel — copy & sesuaikan untuk env baru
    └── roles/
        ├── dhcp/            # DHCP server (isc-dhcp-server)
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── dns/             # DNS server (bind9)
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── firewall/        # Firewall (iptables + netfilter-persistent) ✅
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── ldap/            # LDAP server (slapd / OpenLDAP)
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── mail/            # Mail server
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── nfs/             # NFS server
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        ├── vpn/             # VPN server (OpenVPN)
        │   ├── tasks/main.yml
        │   └── handlers/main.yml
        └── web/             # Web server (Apache/Nginx)
            ├── tasks/main.yml
            └── handlers/main.yml
```

---

## ⚙️ Prerequisites

**Control node (mesin kamu):**
- Ansible 2.14+
- Python 3.8+
- SSH akses ke target server

**Target server:**
- Ubuntu/Debian
- 2 network interface: satu NAT (internet), satu Host-Only (LAN)
- SSH aktif, user dengan sudo

**Install Ansible:**
```bash
pip install ansible
# atau
sudo apt install ansible -y
```

---

## 🔧 Konfigurasi Awal

### 1. Sesuaikan variabel

Copy template dan edit sesuai environment kamu:
```bash
cp ansible/group_vars/all.template ansible/group_vars/all.yml
```

Edit nilai-nilai penting di `all.yml`:
```yaml
# Wajib disesuaikan
server_ip: "172.16.218.x"     # IP server kamu
domain_name: "latihan.home.arpa"
iface_internal: "ens33"        # Interface LAN (cek dengan: ip a)
iface_external: "ens34"        # Interface WAN/NAT

# LDAP — ganti password default!
ldap_admin_pass: "Admin1234"
```

> ⚠️ Jangan commit `all.yml` ke git kalau sudah berisi password asli. Tambahkan ke `.gitignore`.

### 2. Buat inventory

Buat file `inventory.ini` di root project:
```ini
[server]
server01 ansible_host=172.16.218.1 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[all:vars]
ansible_become=true
ansible_become_method=sudo
```

### 3. Test koneksi
```bash
ansible all -i inventory.ini -m ping
```

---

## 🚀 Menjalankan Playbook

Buat `site.yml` di root project untuk orchestrate semua role:
```yaml
---
- name: Configure infrastructure server
  hosts: server
  roles:
    - firewall
    - dns
    - dhcp
    - web
    - ldap
    - mail
    - nfs
    - vpn
```

**Jalankan semua role:**
```bash
ansible-playbook -i inventory.ini site.yml
```

**Jalankan role tertentu saja:**
```bash
# Hanya firewall
ansible-playbook -i inventory.ini site.yml --tags firewall

# Hanya dns dan dhcp
ansible-playbook -i inventory.ini site.yml --tags "dns,dhcp"
```

**Dry run (simulasi tanpa eksekusi):**
```bash
ansible-playbook -i inventory.ini site.yml --check
```

---

## 🎭 Detail Role

### `firewall` ✅
Konfigurasi iptables sebagai gateway + filter traffic.

Yang dikerjakan:
- Install `iptables`, `iptables-persistent`, `netfilter-persistent`
- Enable IPv4 forwarding (`net.ipv4.ip_forward = 1`)
- Flush semua chain filter & NAT sebelum apply rules
- Policy default: `INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`
- Allow: loopback, established/related, ICMP, SSH (dari trusted subnet)
- Allow service ports: DNS (53), DHCP (67), HTTP/HTTPS (80/443), Mail (25/465/587/993), LDAP (389/636), NFS (111/2049), OpenVPN (1194/udp)
- NAT masquerade traffic LAN ke internet via `iface_external`
- Log dropped packets dengan prefix `iptables-input-drop:` / `iptables-forward-drop:`
- Simpan rules via `netfilter-persistent save`

Port yang dibuka secara default:

| Service | Port | Protocol | Source |
|---------|------|----------|--------|
| SSH | 2222 | TCP | Trusted subnet only |
| DNS | 53 | TCP+UDP | Any |
| DHCP | 67 | UDP | Internal interface |
| HTTP | 80 | TCP | Any |
| HTTPS | 443 | TCP | Any |
| SMTP | 25, 465, 587 | TCP | Any |
| IMAPS | 993 | TCP | Any |
| LDAP | 389, 636 | TCP | Trusted subnet only |
| NFS | 111, 2049 | TCP+UDP | Allowed network |
| OpenVPN | 1194 | UDP | Any |

### `dns`
Setup BIND9 untuk resolusi nama domain `latihan.home.arpa`.
- Forward DNS ke `8.8.8.8`

### `dhcp`
Setup ISC DHCP Server untuk distribute IP ke LAN.
- Range: `172.16.218.100` — `172.16.218.200`
- Lease time: 86400 detik (1 hari)

### `web`
Web server untuk domain `latihan.home.arpa`.
- Web root: `/var/www/html`

### `ldap`
OpenLDAP server.
- Base DN: `dc=latihan,dc=home,dc=arpa`
- Admin DN: `cn=admin,dc=latihan,dc=home,dc=arpa`

### `mail`
Mail server untuk domain `latihan.home.arpa`.

### `nfs`
NFS server untuk file sharing di LAN.
- Export path: `/srv/nfs`
- Allowed network: `172.16.218.0/24`

### `vpn`
OpenVPN server.
- Port: 1194/udp
- VPN network: `10.8.0.0/24`

---

## 📝 Variabel Penting (`group_vars/all.yml`)

| Variabel | Default | Keterangan |
|----------|---------|------------|
| `domain_name` | `latihan.home.arpa` | Domain utama |
| `network` | `172.16.218.0` | Network LAN |
| `server_ip` | `172.16.218.1` | IP server |
| `ssh_port` | `2222` | Port SSH (bukan 22!) |
| `iface_internal` | `ens33` | Interface LAN |
| `iface_external` | `ens34` | Interface WAN |
| `ldap_admin_pass` | `Admin1234` | Password LDAP admin |
| `vpn_port` | `1194` | Port OpenVPN |

---

## 🐛 Troubleshooting

| Error | Kemungkinan Penyebab | Solusi |
|-------|----------------------|--------|
| `UNREACHABLE` | SSH tidak bisa connect | Cek IP di inventory, pastikan SSH aktif |
| `Missing sudo password` | User butuh password sudo | Tambah `--ask-become-pass` |
| `iptables not found` | Package belum terinstall | Jalankan role `firewall` dulu |
| `Interface not found` | Nama interface salah | Cek dengan `ip a` di server, update `iface_internal`/`iface_external` |
| Module `iptables` deprecated | Ansible versi baru | Ganti ke module `ansible.builtin.iptables` |

---

## 📚 Referensi

- [Ansible Docs](https://docs.ansible.com/)
- [Ansible iptables module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/iptables_module.html)
- [Roles best practices](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
