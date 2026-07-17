# LibreNMS Install Guide
## Ubuntu Server 22.04 LTS — VirtualBox Lab Environment

This is the full installation walkthrough for the LibreNMS home lab. Commands are documented with explanations. Replace placeholder values (shown in `ALL_CAPS`) with your own before running.

---

## Prerequisites

| Requirement | Detail |
|-------------|--------|
| Hypervisor | VirtualBox |
| OS | Ubuntu Server 22.04 LTS ISO |
| RAM | 4096 MB |
| CPU | 2 cores |
| Disk | 40 GB (dynamically allocated) |
| Network Adapter 1 | Host-only (VM-to-VM communication) |
| Network Adapter 2 | NAT (internet access for apt) |

> **Why two adapters?** Host-only puts VMs on an isolated subnet (typically `192.168.56.x`) so they can communicate with each other and the host machine. NAT gives the VM outbound internet access for package downloads. Without both, you lose either VM-to-VM visibility or internet connectivity.

---

## Phase 1 — Initial System Setup

After installing Ubuntu Server and logging in, run updates before anything else:

```bash
sudo apt update && sudo apt upgrade -y
```
> `apt update` refreshes the package list. `apt upgrade -y` installs all available updates. `-y` auto-confirms.

Once updated, SSH into the server from your host machine for easier copy/paste:

```bash
ssh YOUR_USERNAME@YOUR_HOST_ONLY_IP
```
> Replace `YOUR_USERNAME` with your Ubuntu user and `YOUR_HOST_ONLY_IP` with the server's Host-only adapter IP (run `ip a` on the server to find it). Working via SSH is significantly easier than the VirtualBox console.

---

## Phase 2 — Install Dependencies

```bash
sudo apt install -y acl curl fping git graphviz imagemagick mariadb-client mariadb-server mtr-tiny nginx-full nmap php8.5-cli php8.5-curl php8.5-fpm php8.5-gd php8.5-gmp php8.5-mbstring php8.5-mysql php8.5-snmp php8.5-xml php8.5-zip python3-dotenv python3-pymysql python3-redis python3-setuptools python3-systemd python3-pip rrdtool snmp snmpd unzip whois
```

> **Note on PHP version:** Ubuntu's default repositories provide PHP 8.5. If you see errors about packages not being found, confirm your available PHP version with `apt-cache search php | grep php8` and adjust the version number in the command accordingly.

**Key packages and their roles:**

| Package | Role |
|---------|------|
| `nginx-full` | Web server — serves the LibreNMS dashboard |
| `php8.5-fpm` | PHP FastCGI Process Manager — executes PHP behind nginx |
| `mariadb-server` | Database engine — stores device data, graphs, alerts |
| `snmpd` | SNMP daemon — makes this server SNMP-monitorable |
| `rrdtool` | Round Robin Database — stores and renders time-series graphs |
| `fping` | Fast ping — how LibreNMS checks device availability |
| `git` | Used to clone LibreNMS from GitHub |

---

## Phase 3 — Create LibreNMS User & Clone Repository

Create a dedicated system user for LibreNMS (least privilege — the app runs as its own isolated user, not root):

```bash
sudo useradd librenms -d /opt/librenms -M -r -s "$(which bash)"
```
> `-d` sets home directory. `-M` skips creating it (the app folder serves as home). `-r` creates a system account. `$(which bash)` dynamically finds the bash path.

```bash
cd /opt
sudo git clone https://github.com/librenms/librenms.git
```

---

## Phase 4 — Set File Permissions

```bash
sudo chown -R librenms:librenms /opt/librenms
sudo chmod 771 /opt/librenms
sudo setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
sudo setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
sudo chmod -R 755 /opt/librenms/html
```

> `chown -R` hands ownership of all files to the librenms user. `chmod 771` sets directory permissions. `setfacl -d` sets default ACL for future files. `setfacl -R` sets ACL on existing files. `chmod 755` on the html directory ensures nginx can read and serve web assets.

> **Real issue hit here:** even after this step, Nginx later returned a 403 Forbidden. The `755` here covers the initial html tree, but this same directory also picked up files afterward (via Composer, migrations, etc.) that needed the same treatment reapplied — and some of those ended up with unnecessarily broad execute permission on regular files. See the Troubleshooting section in the main README for the full diagnosis and the follow-up `find`-based cleanup (directories `755`, files `644`).

---

## Phase 5 — Install PHP Dependencies (Composer)

Switch to the librenms user before running Composer so files are created with correct ownership:

```bash
sudo -u librenms bash
cd /opt/librenms
./scripts/composer_wrapper.php install --no-dev
exit
```
> `--no-dev` skips development-only packages. `exit` returns to your normal user after Composer finishes. Running this as your own user or root instead would create ownership mismatches that surface later as permission errors.

---

## Phase 6 — Configure MariaDB

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add these two lines under `[mysqld]`:

```
innodb_file_per_table=1
lower_case_table_names=0
```

```bash
sudo systemctl restart mariadb
sudo mysql -u root
```

Inside the MariaDB prompt:

```sql
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'YOUR_DB_PASSWORD';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
EXIT;
```
> Replace `YOUR_DB_PASSWORD` with a strong password — never commit a real one to the repo. The librenms user is granted access only to the librenms database (least privilege).

---

## Phase 7 — Configure PHP-FPM

```bash
sudo cp /etc/php/8.5/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/librenms.conf
sudo nano /etc/php/8.5/fpm/pool.d/librenms.conf
```

| Find | Change To |
|------|-----------|
| `[www]` | `[librenms]` |
| `user = www-data` | `user = librenms` |
| `group = www-data` | `group = librenms` |
| `listen = ...` | `listen = /run/php/php-fpm-librenms.sock` |
| `listen.owner = www-data` | leave as `www-data` |
| `listen.group = www-data` | leave as `www-data` |

> **Important — two different things are being configured here, and it's easy to conflate them:**
> - `user` / `group` control what Linux user the **PHP-FPM worker processes** run as. These should be `librenms`, matching the app's file ownership.
> - `listen.owner` / `listen.group` control who can **access the socket file itself**. These must stay `www-data`, because Nginx (the process actually connecting to the socket) runs as `www-data`.
>
> **Real issue hit here:** changed `listen.owner`/`listen.group` to `librenms` to match the app's user, which seemed consistent — but it broke the site with a 502 Bad Gateway, because Nginx no longer had a shared group with the socket. Verify actual on-disk ownership any time with `ls -la /run/php/php-fpm-librenms.sock`. The fix: keep `listen.owner`/`listen.group` on `www-data` regardless of what user the app itself runs as.

---

## Phase 8 — Configure Nginx

```bash
sudo nano /etc/nginx/conf.d/librenms.conf
```

```nginx
server {
 listen      80;
 server_name YOUR_HOST_ONLY_IP;
 root        /opt/librenms/html;
 index       index.php;

 charset utf-8;
 gzip on;
 gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsl text/xml image/x-icon;
 location / {
  try_files $uri $uri/ /index.php?$query_string;
 }
 location ~ [^/]\.php(/|$) {
  fastcgi_pass unix:/run/php/php-fpm-librenms.sock;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  include fastcgi.conf;
 }
 location ~ /\.(?!well-known).* {
  deny all;
 }
}
```

> **Removing the default site matters:** if left enabled, Nginx's default server block can conflict with or intercept requests meant for LibreNMS on port 80.

> **Real issue hit here (reproduced deliberately for documentation):** a typo in the socket path (`.sok` instead of `.sock`) produced a 502 Bad Gateway. `nginx -t` passed clean — it only validates config syntax, not whether the target path actually exists. The real error only surfaces in `/var/log/nginx/error.log`: `connect() to unix:/run/php/php-fpm-librenms.sok failed (2: No such file or directory)`.

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx php8.5-fpm
```

---

## Phase 9 — Configure SNMP

```bash
sudo cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
sudo nano /etc/snmp/snmpd.conf
```

Find `RANDOMSTRINGGOESHERE` and replace with your community string. Confirm this line exists and is uncommented:

```
rocommunity YOUR_COMMUNITY_STRING
```

> The community string is essentially an SNMP password. Use something memorable but not a real password you use elsewhere.

```bash
sudo curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
sudo chmod +x /usr/bin/distro
sudo systemctl enable snmpd
sudo systemctl restart snmpd
```

---

## Phase 10 — Configure .env File

```bash
sudo cp /opt/librenms/.env.example /opt/librenms/.env
sudo nano /opt/librenms/.env
```

Find and uncomment these lines (remove the `#`), then fill in your values:

```
DB_HOST=localhost
DB_DATABASE=librenms
DB_USERNAME=librenms
DB_PASSWORD=YOUR_DB_PASSWORD
```

> **Real issue hit here:** these lines were left commented out on first pass, so LibreNMS never read them and attempted to connect with no credentials at all — surfaced as `Access denied for user 'librenms'@'localhost' (using password: NO)`. The `(using password: NO)` detail is the tell — it confirms no password was even being sent, pointing to a config problem rather than a wrong credential.

```bash
cd /opt/librenms && sudo -u librenms php artisan key:generate
```

> **Real issue hit here:** first attempt threw `MissingAppKeyException` — the app key had never been generated. Must be run as `librenms` specifically (not root, not your personal account) so the newly written key stays correctly owned.

---

## Phase 11 — Cron Jobs & Scheduler

```bash
sudo cp /opt/librenms/dist/librenms.cron /etc/cron.d/librenms
sudo cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
sudo systemctl enable librenms-scheduler.timer
sudo systemctl start librenms-scheduler.timer
sudo cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

```bash
cd /opt/librenms && sudo -u librenms php artisan migrate --force
```
> Must be run from `/opt/librenms`, as the `librenms` user. `--force` skips Laravel's confirmation prompt (a safety check meant for production environments) — appropriate here since this is a controlled lab install.

---

## Phase 12 — Python Dependencies

```bash
sudo pip3 install -r /opt/librenms/requirements.txt --break-system-packages
```
> `--break-system-packages` overrides Ubuntu's PEP 668 protection on the system Python environment. Works fine for a lab; the more correct long-term practice for production is a scoped virtual environment (`python3 -m venv`).

---

## Phase 13 — Validate & Access Dashboard

```bash
sudo -u librenms php /opt/librenms/validate.php
```

Resolve any `[FAIL]` items before proceeding. `[WARN]` items are typically minor.

```
http://YOUR_HOST_ONLY_IP
```

Create your admin account and log in.

---

## Phase 14 — Add Your First Device

**Devices → Add Device**

| Field | Value |
|-------|-------|
| Hostname/IP | `127.0.0.1` (add the server itself first) |
| SNMP Version | `v2c` |
| Community | `YOUR_COMMUNITY_STRING` |
| Port | leave blank (defaults to 161) |

LibreNMS polls every 5 minutes.

---

## Adding Additional Devices (e.g. Kali VM)

```bash
sudo apt install snmpd -y
sudo nano /etc/snmp/snmpd.conf
```

```
rocommunity YOUR_COMMUNITY_STRING
```

```bash
sudo systemctl enable snmpd
sudo systemctl restart snmpd
snmpwalk -v2c -c YOUR_COMMUNITY_STRING 127.0.0.1
```

If data returns, add the device in LibreNMS using its Host-only IP.

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `sudo -u librenms php /opt/librenms/validate.php` | Run health check |
| `sudo tail -f /opt/librenms/logs/librenms.log` | Watch logs live |
| `sudo systemctl status nginx` | Check web server |
| `sudo systemctl status php8.5-fpm` | Check PHP |
| `sudo systemctl status mariadb` | Check database |
| `sudo systemctl status snmpd` | Check SNMP daemon |
| `sudo nginx -t` | Test nginx config syntax (syntax only — won't catch unreachable paths) |
| `ip a` | Show network interfaces and IPs |
| `sudo shutdown now` | Clean shutdown |

---

## Summary of Real Errors Hit During This Build

| Phase | Error | Root Cause |
|---|---|---|
| 4 | 403 Forbidden (later) | Overly broad execute permission on regular files |
| 7 | 502 Bad Gateway | `listen.owner`/`listen.group` changed to `librenms` instead of staying `www-data` |
| 8 | 502 Bad Gateway | Socket path typo in Nginx config |
| 10 | `Access denied ... (using password: NO)` | `.env` credentials commented out |
| 10 | `MissingAppKeyException` | App key never generated |

Full diagnostic breakdown and reasoning for each is in the main [README.md](../README.md).

---
*Part of an ongoing home lab — built while studying for CompTIA Security+ (SY0-701).*
