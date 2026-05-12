# ansible-basic

Latihan Ansible untuk konfigurasi satu server infrastruktur jaringan. Project ini menyiapkan server Debian/Ubuntu sebagai gateway dan penyedia layanan LAN: firewall, DNS, DHCP, web, LDAP, NFS, mail, dan VPN.

## Topologi

```text
Internet
    |
[ens34 - NAT]
    |
Server 172.16.218.1
    |
[ens33 - Host-Only]
    |
LAN 172.16.218.0/24
```

## Struktur Project

```text
.
├── ansible.cfg
└── ansible/
    ├── inventory.ini
    ├── inventory.template
    ├── site.yml
    ├── site.yml.template
    ├── group_vars/
    │   ├── all.yml
    │   └── all.template
    └── roles/
        ├── firewall/
        ├── dns/
        ├── dhcp/
        ├── web/
        ├── ldap/
        ├── nfs/
        ├── mail/
        └── vpn/
```

## Prerequisites

Control node:

- Ansible 2.14+
- Python 3.8+
- SSH access ke target server

Target server:

- Debian/Ubuntu
- 2 network interface: satu NAT/WAN dan satu Host-Only/LAN
- SSH aktif, user memiliki akses sudo

Install Ansible:

```bash
sudo apt install ansible -y
```

atau:

```bash
pip install ansible
```

## Konfigurasi Awal

Copy template variable dan inventory:

```bash
cp ansible/group_vars/all.template ansible/group_vars/all.yml
cp ansible/inventory.template ansible/inventory.ini
```

Edit nilai penting di `ansible/group_vars/all.yml`:

```yaml
server_ip: "172.16.218.1"
domain_name: "latihan.home.arpa"
iface_internal: "ens33"
iface_external: "ens34"
ldap_admin_pass: "CHANGE_ME"
```

Edit `ansible/inventory.ini`:

```ini
[lks]
lks-server ansible_host=172.16.218.1

[lks:vars]
ansible_user=endmin
ansible_port=2222
ansible_become=true
ansible_python_interpreter=/usr/bin/python3
```

`ansible/group_vars/all.yml` dan `ansible/inventory.ini` berisi konfigurasi lokal. Keduanya sudah masuk `.gitignore` untuk environment baru; jika file tersebut sudah terlanjur tracked, hapus dari index dengan `git rm --cached` sebelum menyimpan secret nyata.

## Validasi

Syntax check:

```bash
ansible-playbook --syntax-check ansible/site.yml
```

Test koneksi:

```bash
ansible all -m ping
```

Dry run:

```bash
ansible-playbook ansible/site.yml --check
```

## Menjalankan Playbook

Jalankan semua role:

```bash
ansible-playbook ansible/site.yml
```

Jalankan role tertentu:

```bash
ansible-playbook ansible/site.yml --tags firewall
ansible-playbook ansible/site.yml --tags "dns,dhcp"
ansible-playbook ansible/site.yml --tags web
```

## Detail Role

### `firewall`

Konfigurasi `iptables` sebagai gateway dan filter traffic:

- Install `iptables`, `iptables-persistent`, `netfilter-persistent`
- Enable IPv4 forwarding
- Default policy: `INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`
- Allow SSH dari trusted subnet
- Allow DNS, DHCP, HTTP/HTTPS, mail, LDAP, NFS, dan OpenVPN
- NAT masquerade LAN ke internet melalui `iface_external`
- Simpan rules dengan `netfilter-persistent save`

### `dns`

Setup BIND9 untuk domain `latihan.home.arpa`:

- Render `named.conf.options`
- Render forward zone
- Render reverse zone
- Validasi dengan `named-checkconf` dan `named-checkzone`

### `dhcp`

Setup Kea DHCPv4 untuk LAN:

- Package: `kea-dhcp4-server`
- Config: `/etc/kea/kea-dhcp4.conf`
- Default range: `172.16.218.100 - 172.16.218.200`
- Default lease time: 86400 detik
- Validasi dengan `kea-dhcp4 -t`

### `web`

Setup Apache HTTP server:

- Package: `apache2`
- Service: `apache2`
- Web root: `/var/www/html`
- Default index page untuk domain lokal
- Validasi dengan `apache2ctl configtest`

### `ldap`, `nfs`, `mail`, `vpn`

Role ini sudah ada dalam struktur project dan `site.yml`, tetapi masih perlu implementasi task produksi/lab yang lengkap.

## Variabel Penting

| Variabel | Default | Keterangan |
|----------|---------|------------|
| `domain_name` | `latihan.home.arpa` | Domain utama |
| `network` | `172.16.218.0` | Network LAN |
| `server_ip` | `172.16.218.1` | IP server |
| `ssh_port` | `2222` | Port SSH |
| `iface_internal` | `ens33` | Interface LAN |
| `iface_external` | `ens34` | Interface WAN/NAT |
| `dhcp_range_start` | `172.16.218.100` | Awal DHCP pool |
| `dhcp_range_end` | `172.16.218.200` | Akhir DHCP pool |
| `web_root` | `/var/www/html` | Apache document root |
| `ldap_admin_pass` | `CHANGE_ME` | Password LDAP admin |
| `vpn_port` | `1194` | Port OpenVPN |

## Troubleshooting

| Error | Kemungkinan Penyebab | Solusi |
|-------|----------------------|--------|
| `UNREACHABLE` | SSH tidak bisa connect | Cek IP, port SSH, user, dan key |
| `Missing sudo password` | User butuh password sudo | Tambah `--ask-become-pass` |
| `Interface not found` | Nama interface salah | Cek `ip a`, lalu update `iface_internal`/`iface_external` |
| DNS zone invalid | Template zone salah | Jalankan role `dns`, lihat output `named-checkzone` |
| Kea config invalid | JSON/config DHCP salah | Jalankan role `dhcp`, lihat output `kea-dhcp4 -t` |

## Referensi

- [Ansible Docs](https://docs.ansible.com/)
- [Ansible iptables module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/iptables_module.html)
- [Ansible roles best practices](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
