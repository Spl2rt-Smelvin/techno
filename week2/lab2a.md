# Lab 2.A — A service and a timer

## 1. Final Unit Files

### `/etc/systemd/system/disk-report.service`

<pre>
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
</pre>

### `/etc/systemd/system/disk-report.timer`

<pre>
[Unit]
Description=Run disk-report every five minutes
Documentation=systemd.time(7)

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
</pre>

---

## 2. Initial Error Diagnosis

### Initial Journal Error Output:

<pre>
Sep 22 05:40:15 myserver systemd[1]: Starting disk-report.service - Append disk usage to log...
Sep 22 05:40:15 myserver disk-report.sh[3685]: /usr/local/bin/disk-report.sh: line 2: /var/log/disk-report.log: Permission denied
Sep 22 05:40:15 myserver disk-report.sh[3690]: /usr/local/bin/disk-report.sh: line 3: /var/log/disk-report.log: Permission denied
Sep 22 05:40:15 myserver systemd[1]: disk-report.service: Main process exited, code=exited, status=1/FAILURE
Sep 22 05:40:15 myserver systemd[1]: disk-report.service: Failed with result 'exit-code'.
Sep 22 05:40:15 myserver systemd[1]: Failed to start disk-report.service - Append disk usage to log.
</pre>

### Annotation:

* **What it said:** The service crashed with status=1/FAILURE because lines 2 and 3 of /usr/local/bin/disk-report.sh encountered /var/log/disk-report.log: Permission denied.
* **What it told me:** The service runs unprivileged as the reports user (User=reports), but /var/log/disk-report.log was created by root during the initial manual test, preventing reports from writing to it.
* **Which part confirmed the cause:** The explicit line `/usr/local/bin/disk-report.sh: line 2: /var/log/disk-report.log: Permission denied` confirmed the failure was caused by filesystem permissions on the log file.

---

## 3. Why Option B is Better than Option A

* **Least Privilege:** systemd opens the log as root before dropping privileges, so `reports` never needs write access to `/var/log`.
* **Not Fragile:** If the log file is rotated or removed, Option B automatically recreates it; Option A breaks until permissions are fixed manually.
* **Separation of Concerns:** Logging redirection stays in systemd configuration rather than hardcoded inside the script.

---

## 4. Timer Verification

Output of `systemctl list-timers disk-report.timer`:

<pre>
NEXT                        LEFT LAST                             PASSED       UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  18h Tue 2026-09-22 05:45:04 UTC 2min 42s ago disk-report.timer disk-report.service
</pre>

---

## 5. Successful Runs

Output from `journalctl -u disk-report.service`:

<pre>
Sep 22 05:43:44 myserver systemd[1]: disk-report.service: Deactivated successfully.
Sep 22 05:45:04 myserver systemd[1]: disk-report.service: Deactivated successfully.
</pre>

---

## 6. Service Account Flags

* **`--no-create-home`:** Service accounts execute background tasks and do not require personal home directories.
* **`--shell /usr/sbin/nologin`:** Disables interactive logins to prevent shell access.
* **Risk if omitted:** If compromised, an attacker gains an interactive shell and home directory to stage tools and escalate privileges.
