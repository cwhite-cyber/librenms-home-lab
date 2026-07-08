# librenms-home-lab
Network monitoring home lab — LibreNMS on Ubuntu Server 22.04
Home Lab: Network Monitoring with LibreNMS
Overview
Designed and deployed a network monitoring stack in a self-hosted virtual lab, built from the ground up on Ubuntu Server 22.04 LTS. The project simulates a small enterprise monitoring environment: a LibreNMS instance polling network devices via SNMP, backed by MariaDB, PHP-FPM, and Nginx.
Built as part of a structured transition into cybersecurity, alongside CompTIA Security+ (SY0-701) study and Hack The Box practice.
Why I Built This
Monitoring and visibility are foundational to security operations — you can't defend, alert on, or investigate what you can't see. I wanted hands-on experience with the full stack behind a monitoring platform (not just clicking through a GUI), including the Linux service dependencies, database layer, and networking configuration that make it work — and break.
Environment
Component	Details
Host	ThinkPad T490
Hypervisor	VirtualBox
Monitoring server	Ubuntu Server 22.04 LTS
Stack	LibreNMS, MariaDB, PHP-FPM, Nginx, SNMP
Network	Dual-adapter config (NAT + host-only/internal)
Monitored devices	2 network devices (SNMP-polled)
Diagram placeholder — add a simple network topology image here (draw.io / Excalidraw / diagrams.net export works well).
What It Does
Polls monitored devices over SNMP for uptime, interface stats, and health metrics
Stores time-series and device data in MariaDB
Serves the LibreNMS web UI through Nginx + PHP-FPM
Provides a single dashboard for device status across the lab network
Build Process (High-Level)
Provisioned Ubuntu Server 22.04 LTS as a VirtualBox VM with dual network adapters (one NAT for internet/updates, one host-only for lab device communication)
Installed and configured the LEMP-adjacent stack: Nginx, PHP-FPM, MariaDB
Installed LibreNMS and configured its dependencies (cron jobs, `lnms` validation, permissions)
Configured SNMP on target devices and added them to LibreNMS for polling
Validated data collection and dashboard rendering end-to-end
Challenges & Troubleshooting
This is the part that mattered most — the actual learning happened in the failures.
PHP version mismatch
Initial install command referenced `php8.1-*` packages, which were not available in Ubuntu's default repositories, resulting in dependency errors during install.
Fix: Discovered Ubuntu's default repositories already provided PHP 8.5. Updated the install command to reference `php8.5-*` packages instead of `php8.1-*`, which resolved the dependency errors immediately. No third-party PPA was required.
VM networking — internet access lost after adapter change
Switching the server VM from NAT to Host-only adapter (required for VM-to-VM communication) broke internet access entirely, causing `apt` package downloads to fail with connection errors.
Fix: Added a second network adapter to the VM — Adapter 1 set to Host-only for lab network communication (`192.168.56.x` range), Adapter 2 set to NAT for internet access. This dual-adapter approach is a standard pattern for isolated lab environments that still need outbound connectivity. Applied the same fix to the Kali VM once it was added to the monitoring environment.
Database credentials not being read (`Access denied for user 'librenms'@'localhost' (using password: NO)`)
The `(using password: NO)` detail was the key clue — it meant no password was even being sent, pointing to a configuration problem rather than a wrong credential.
Fix: Found the DB credential lines in LibreNMS's `.env` file commented out with `#`, so they were never being read or passed to MariaDB at all. Uncommented them and confirmed the connection succeeded.
Working directory / process identity mismatch
LibreNMS's error output showed its working directory as `/home/cdub` instead of `/opt/librenms`, and file operations were failing with `Permission denied` — a mismatch between the user context running the process and the user (`librenms`) that owns the application's files.
Fix, two parts:
Added myself to the `librenms` group (`sudo usermod -aG librenms cdub`) and logged out/in for the group change to take effect — this restored my own shell's read/write access to the LibreNMS directory.
That alone didn't fix the database migration step, which failed for a related but distinct reason: the migration needed to run as the `librenms` user specifically, not just with group membership. Resolved with `cd /opt/librenms && sudo -u librenms php artisan migrate --force`, explicitly running the command as the correct user. `--force` skips Laravel's confirmation prompt (a safety check meant for production environments) — appropriate here since this was a controlled lab install, not a live system.
Cascading symptom: `MissingAppKeyException`
The same identity/ownership issue surfaced again as a separate-looking error: Laravel's `APP_KEY` had never been generated.
Fix: `sudo -u librenms php artisan key:generate`, again run explicitly as `librenms` so the newly written key was owned correctly rather than reintroducing the same ownership mismatch. Good reminder that one root cause (process identity) can present as multiple, seemingly unrelated errors depending on which part of the application hits it first.
403 Forbidden from Nginx
Nginx could reach the application but was denied at the filesystem permission level — every CSS and JS asset returned 403, leaving the dashboard rendering as unstyled raw HTML.
Fix: Initially resolved broadly with `sudo chmod -R 755 /opt/librenms/html`, which restored access but over-granted execute permission to regular files that never needed it. Followed up by scaling permissions down appropriately — directories kept at `755` (execute needed to traverse them), files tightened to `644` (no execute needed, since PHP-FPM interprets `.php` files rather than executing them directly). Verified the change with a before/after file count: 922 files initially found at `755`, confirmed at `0` after re-running with `644` — narrowing unnecessary permissions across the application. Confirmed the web portal was still fully functional afterward.
PHP-FPM socket ownership mismatch (`listen.owner` / `listen.group`)
While configuring the PHP-FPM pool (`/etc/php/8.5/fpm/pool.d/librenms.conf`), changed `listen.owner` and `listen.group` from the default `www-data` to `librenms` — consistent with the rest of the app running as the dedicated `librenms` user. This immediately broke the site with a 502 Bad Gateway.
Fix: Since only ownership had been changed (not the socket's path), checked the socket file directly with `ls -la /run/php/php-fpm-librenms.sock` to confirm its actual on-disk owner/group/permissions. The socket was now owned by `librenms:librenms`, but Nginx runs as `www-data` — with no shared group, Nginx had no path to reach it. Fixed by setting `listen.group = www-data` in the pool config (keeping `listen.owner = librenms`), so PHP-FPM still runs as the correct application user while granting the Nginx process group-level access to the socket. Restarted PHP-FPM and confirmed the site loaded again.
To fill in specifics: most of the above is documented from terminal history and screenshots; add exact command output/screenshots inline where available for stronger evidence.
What I'd Add Next
pfSense or OPNsense firewall in front of the lab segment
Suricata or Zeek for IDS/network traffic visibility, feeding alerts into LibreNMS or a SIEM
A second monitored VM running a vulnerable service for HTB-style attack/detect exercises
Automate device onboarding with the LibreNMS API instead of manual entry
Documentation
`install-guide.md` — full step-by-step install
`cheat-sheet.md` — quick reference for common commands/config
---
Part of an ongoing home lab built while studying for CompTIA Security+ (SY0-701).
