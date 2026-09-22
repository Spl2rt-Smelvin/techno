# Lab 2.A — A service and a timer

## 1. Final Unit Files

### `/etc/systemd/system/disk-report.service`
[Unit]
Description=Append disk usage to log
Documentation=man:df(1)
After=local-fs.target

[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
StandardOutput=append:/var/log/disk-report.log
StandardError=append:/var/log/disk-report.log


### `/etc/systemd/system/disk-report.timer`
[Unit]
Description=Run disk-report every five minutes
Documentation=systemd.time(7)

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target


---

## 2. Initial Error Diagnosis

### Initial Journal Error Output:
Sep 22 05:40:15 myserver systemd[1]: Starting disk-report.service - Append disk usage to log...
Sep 22 05:40:15 myserver disk-report.sh[3685]: /usr/local/bin/disk-report.sh: line 2: /var/log/disk-report.log: Permission denied
Sep 22 05:40:15 myserver disk-report.sh[3690]: /usr/local/bin/disk-report.sh: line 3: /var/log/disk-report.log: Permission denied
Sep 22 05:40:15 myserver systemd[1]: disk-report.service: Main process exited, code=exited, status=1/FAILURE
Sep 22 05:40:15 myserver systemd[1]: disk-report.service: Failed with result 'exit-code'.
Sep 22 05:40:15 myserver systemd[1]: Failed to start disk-report.service - Append disk usage to log.

### Annotation:
* What it said: The service crashed with an exit code of status=1/FAILURE because lines 2 and 3 of /usr/local/bin/disk-report.sh encountered /var/log/disk-report.log: Permission denied.
* What it told me: The service is running unprivileged as the reports user (User=reports), but /var/log/disk-report.log was created by root during the initial manual test, so the unprivileged user has no write access to it.
* Which part confirmed the cause: The exact line /usr/local/bin/disk-report.sh: line 2: /var/log/disk-report.log: Permission denied confirmed the problem was specifically filesystem permissions blocking writes to that log path, rather than an issue with the df command or the service unit configuration.

---

## 3. Why Option B is Better than Option A

* Least Privilege: systemd opens the log as root, so the reports user never needs write access to /var/log.
* Not Fragile: If the log file is deleted or rotated, Option B automatically recreates it; Option A breaks.
* Separation of Concerns: Output paths stay configured inside the unit file, keeping the script simple and portable.

---

## 4. Timer Verification

Output of `systemctl list-timers disk-report.timer`:
NEXT                        LEFT LAST                             PASSED       UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  18h Tue 2026-09-22 05:45:04 UTC 2min 42s ago disk-report.timer disk-report.service

---

## 5. Successful Runs

Output from `journalctl -u disk-report.service`:
Sep 22 05:43:44 myserver systemd[1]: disk-report.service: Deactivated successfully.
Sep 22 05:45:04 myserver systemd[1]: disk-report.service: Deactivated successfully.

---

## 6. Service Account Flags

* --no-create-home: Service accounts only run automated scripts and do not need a personal /home/ folder.
* --shell /usr/sbin/nologin: Blocks interactive shell logins, closing a direct entry point for attackers.
* Risk if omitted: If compromised, an attacker would gain an interactive terminal and a writable home folder to download tools, stage files, and escalate privileges.
