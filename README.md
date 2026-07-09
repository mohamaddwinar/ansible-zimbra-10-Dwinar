# Ansible Zimbra 10.1 Installer

Ansible role untuk instalasi otomatis Zimbra Collaboration Network Edition 10.1.x pada beberapa OS Linux.

Role ini dibuat untuk membantu provisioning Zimbra single-server dengan konfigurasi dasar yang konsisten, termasuk hostname, DNS lokal, dependency package, installer phase 1, `zmsetup`, restart service, dan validasi service utama.

## Supported OS

Role ini sudah diuji pada:

| OS | Status | Zimbra Release |
|---|---:|---|
| Rocky Linux 9 | OK | 10.1.19 NETWORK |
| Rocky Linux 8 | OK | 10.1.19 NETWORK |
| Ubuntu 24.04 | OK | 10.1.19 NETWORK |
| Ubuntu 22.04 | OK | 10.1.19 NETWORK |
| Ubuntu 20.04 | OK | 10.1.19 NETWORK |
| Ubuntu 18.04 | OK | 10.1.19 NETWORK |

> Catatan: Ubuntu 18.04 dan Ubuntu 20.04 sudah tergolong OS lama. Gunakan hanya jika memang masih dibutuhkan untuk testing, lab, migration, atau compatibility check.

## Main Features

- Auto-detect OS family dan OS version.
- Load task berbeda untuk Rocky 8, Rocky 9, Ubuntu 18.04, Ubuntu 20.04, Ubuntu 22.04, dan Ubuntu 24.04.
- Konfigurasi hostname dan `/etc/hosts`.
- Konfigurasi static network.
- Konfigurasi local DNS menggunakan BIND.
- Handling resolver untuk Ubuntu agar tidak bentrok dengan `systemd-resolved` / `resolvconf`.
- Handling EPEL dan repository pada Rocky.
- Download installer Zimbra sesuai OS target.
- Menjalankan Zimbra installer phase 1 secara async.
- Menjalankan `zmsetup.pl` phase 2 secara async.
- Konfigurasi trusted IP.
- Restart Zimbra service.
- Validasi service utama Zimbra.
- Handling OnlyOffice runtime pada Ubuntu 18.04 dan Ubuntu 20.04.

## Role Structure

Struktur umum role:

```text
ansible-zimbra/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── rocky8.yml
│   ├── rocky9.yml
│   ├── ubuntu18.yml
│   ├── ubuntu20.yml
│   ├── ubuntu22.yml
│   └── ubuntu24.yml
├── templates/
│   ├── 01-netcfg.yaml.j2
│   ├── db.zimbra.local.j2
│   ├── named.conf.j2
│   ├── zimbra_answers.txt.j2
│   └── zimbra_config.txt.j2
├── inventory.ini
├── site.yml
├── README.md
└── LICENSE
```

## Requirements

Control node:

- Ansible installed
- SSH access to target server as root or user with privilege escalation
- Python available on target server

Target server:

- Clean VM/server is strongly recommended
- Static IP address
- Valid FQDN
- Internet access to Ubuntu/Rocky repositories and Zimbra repositories
- Minimum RAM depends on selected Zimbra components

Recommended target server condition:

```bash
hostname -f
ip addr
ip route
ping -c 2 8.8.8.8
getent hosts files.zimbra.com
```

## Required Variables

Contoh variable minimal di `defaults/main.yml` atau inventory/group vars:

```yaml
zimbra_fqdn: "zimbra10.example.com"
zimbra_domain: "example.com"
zimbra_admin_user: "admin"
zimbra_admin_password: "ChangeMeStrongPassword123!"
zimbra_timezone: "Asia/Jakarta"
zimbra_dns_forwarders:
  - "8.8.8.8"
  - "1.1.1.1"
```

Penjelasan:

| Variable | Description |
|---|---|
| `zimbra_fqdn` | FQDN server Zimbra, contoh `mail.example.com` |
| `zimbra_domain` | Domain email utama, contoh `example.com` |
| `zimbra_admin_user` | Username admin Zimbra |
| `zimbra_admin_password` | Password admin Zimbra |
| `zimbra_timezone` | Timezone server dan Zimbra |
| `zimbra_dns_forwarders` | DNS forwarder untuk BIND lokal |

## Inventory Example

Contoh `inventory.ini`:

```ini
[zimbra]
mail01 ansible_host=192.168.7.23 ansible_user=root
```

## Playbook Example

Contoh `site.yml`:

```yaml
---
- name: Install Zimbra
  hosts: zimbra
  become: true
  roles:
    - ansible-zimbra
```

## Usage

Cek syntax:

```bash
ansible-playbook -i inventory.ini site.yml --syntax-check
```

Jalankan playbook:

```bash
ansible-playbook -i inventory.ini site.yml -vv
```

## Post Install Check

Login ke server target:

```bash
su - zimbra
zmcontrol status
zmcontrol -v
```

Contoh output sukses:

```text
Host zimbra10.example.com
        amavis                  Running
        antispam                Running
        antivirus               Running
        archiving               Running
        convertd                Running
        ldap                    Running
        license-daemon          Running
        logger                  Running
        mailbox                 Running
        memcached               Running
        mta                     Running
        onlyoffice              Running
        opendkim                Running
        proxy                   Running
        service webapp          Running
        snmp                    Running
        spell                   Running
        stats                   Running
        zimbra webapp           Running
        zimbraAdmin webapp      Running
        zimlet webapp           Running
        zmconfigd               Running
```

## Clean Reinstall

Jika instalasi gagal dan ingin mulai ulang dari awal, gunakan VM snapshot jika tersedia.

Jika tidak ada snapshot, bersihkan Zimbra dari target server:

```bash
su - zimbra -c "zmcontrol stop" || true
pkill -u zimbra || true
pkill -f /opt/zimbra || true
apt-get purge -y 'zimbra-*' || true
dnf remove -y 'zimbra-*' || true
rm -rf /opt/zimbra
rm -rf /tmp/install.log /tmp/zmsetup.log
rm -rf /tmp/zimbra_answers.txt /tmp/zimbra_config.txt
rm -rf /root/.ansible_async/*
rm -rf /root/zcs-*
rm -rf /root/zimbra-*.tgz*
```

Verifikasi:

```bash
dpkg -l | grep -i zimbra || true
rpm -qa | grep -i zimbra || true
ls -ld /opt/zimbra 2>/dev/null || true
ps -ef | grep -i zimbra | grep -v grep || true
```

## Notes

- Role ini ditujukan untuk single-server Zimbra deployment.
- Jalankan pada VM/server kosong untuk hasil paling stabil.
- Ambil snapshot VM sebelum menjalankan playbook.
- Zimbra installer dan package repository dapat berubah sewaktu-waktu.
- Untuk environment production, lakukan review security, backup, DNS publik, firewall, SSL certificate, dan monitoring secara terpisah.

## License

MIT License. Lihat file [LICENSE](LICENSE).
