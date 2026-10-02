# Defender-Wazuh: Full Troubleshooting Guide

Every real failure mode encountered while building this lab, with root causes and exact fixes. Organized by category. If you hit something not listed here, check the **Diagnostic Toolkit** section at the bottom for the commands used to investigate every issue below — the same approach will usually get you to the answer for anything new.

---

## Table of Contents

1. [Wazuh Indexer Startup Issues](#1-wazuh-indexer-startup-issues)
2. [Wazuh Manager Startup Issues](#2-wazuh-manager-startup-issues)
3. [Wazuh Dashboard Issues](#3-wazuh-dashboard-issues)
4. [Password & Credential Issues](#4-password--credential-issues)
5. [Networking Issues (VMware / Host-Level)](#5-networking-issues-vmware--host-level)
6. [Agent Enrollment Issues](#6-agent-enrollment-issues)
7. [Package Manager / Installation Issues](#7-package-manager--installation-issues)
8. [Dashboard UI / Discover View Issues](#8-dashboard-ui--discover-view-issues)
9. [VM-Level / Hypervisor Issues](#9-vm-level--hypervisor-issues)
10. [Active Response & Detection Issues](#10-active-response--detection-issues)
11. [Diagnostic Toolkit](#11-diagnostic-toolkit)

---

## 1. Wazuh Indexer Startup Issues

The indexer was, by a wide margin, the single most failure-prone component in this entire build. Nearly every indexer problem falls into one of five categories below — check them roughly in this order.

### 1.1 — `AccessDeniedException: /etc/wazuh-indexer/backup`

**Symptom:** Indexer fails immediately on start (not a timeout — an actual exit). `journalctl -u wazuh-indexer` shows:
```
Exception in thread "main" org.opensearch.bootstrap.BootstrapException: java.nio.file.AccessDeniedException: /etc/wazuh-indexer/backup
```

**Root cause:** The `wazuh-passwords-tool.sh` script creates `/etc/wazuh-indexer/backup` while running as root, which can leave it with ownership that doesn't match the indexer's own service account (`wazuh-indexer:wazuh-indexer`). On next startup, the indexer tries to read/write that folder and gets denied.

**Fix:**
```bash
sudo rm -rf /etc/wazuh-indexer/backup
sudo systemctl restart wazuh-indexer
```
The indexer recreates the folder correctly on its own.

**This can recur** — any time you run the passwords tool again, the same folder can end up broken again, and it won't surface until the *next* indexer restart (not immediately, since the indexer doesn't re-check this folder while already running). If the indexer fails right after you'd used the passwords tool at some point earlier in the session, check this first.

---

### 1.2 — Timeout on Startup (`Result: timeout`)

**Symptom:** `systemctl status wazuh-indexer` shows `Active: failed (Result: timeout)`. The process was actually running and making progress (visible in `journalctl -u wazuh-indexer -f`) but didn't finish within systemd's window.

**Root causes (check in this order):**

**(a) DNS/hostname lookups hanging.** OpenSearch performs internal hostname resolution during bootstrap. On an isolated host-only network with no real DNS server, each lookup can hang ~10 seconds before timing out. Confirm:
```bash
time getent hosts $(hostname)
```
If this takes more than a second, that's it. Fix:
```bash
echo "127.0.0.1 $(hostname)" | sudo tee -a /etc/hosts
echo "::1 $(hostname)" | sudo tee -a /etc/hosts   # IPv6 half matters too — getent checks both
sudo sed -i 's/^#\?MulticastDNS=.*/MulticastDNS=no/' /etc/systemd/resolved.conf
sudo sed -i 's/^#\?LLMNR=.*/LLMNR=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```
Re-check with `time getent hosts $(hostname)` — should return in well under a second. Note: the IPv4-only `/etc/hosts` entry alone was *not* sufficient in testing — `getent hosts` does a dual-stack lookup, and without an IPv6 entry too, it still falls through to the network (and thus to mDNS/LLMNR) for the IPv6 half. Add both lines.

**(b) systemd's default timeout (90s) is genuinely too short**, especially on first boot with no warm disk cache. Fix:
```bash
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d
sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf > /dev/null << 'EOF'
[Service]
TimeoutStartSec=600
EOF
sudo systemctl daemon-reload
```
Verify the override actually took effect:
```bash
sudo systemctl cat wazuh-indexer
```
Look for the override file listed at the bottom with your `TimeoutStartSec` value. (Note: `systemctl show wazuh-indexer -p TimeoutStartSec` was observed to sometimes return blank/empty output even when the override was correctly applied and working — don't rely on that command alone to confirm; `systemctl cat` is the reliable check.)

**(c) Disk I/O bottleneck from an IDE virtual disk controller (VMware-specific).** This was the underlying cause behind the *worst* and most persistent timeout issues in testing — see [Section 9.1](#91-ide-disk-controller-io-bottleneck) for full details and the fix (switch to SCSI).

**(d) Host RAM contention.** If VMware's host is under memory pressure, it can silently swap a VM's memory to host disk in the background, causing wildly inconsistent guest performance that looks like random timeouts. Check host Task Manager memory usage while the indexer is starting — if it's above ~80%, close other applications or suspend other running VMs before retrying.

---

### 1.3 — Indexer Boots, Then Hangs with Zero New Log Output

**Symptom:** `journalctl -u wazuh-indexer -f` shows normal early boot messages (JVM warnings, module loading) and then goes completely silent for many minutes, eventually timing out.

**Diagnosis:** Check the *actual* dedicated cluster log, not just the systemd journal — it can have more detail:
```bash
sudo tail -100 /var/log/wazuh-indexer/wazuh-cluster.log
```
If this also shows clean, error-free progression with unusually large time gaps between routine steps (e.g., over a minute between trivial log lines), this is **not a crash or bug** — it's the JVM being CPU/IO-starved by the host. See [Section 9.1](#91-ide-disk-controller-io-bottleneck) (disk controller) and check host RAM/CPU contention.

**If you need to confirm the process is genuinely alive and working (not frozen) while it's running:**
```bash
ps aux | grep opensearch
top -bn1 -p <PID>
```
Look at the `%CPU` and process state (`S`=sleeping/idle is fine if brief; `D`=uninterruptible disk wait for a sustained period indicates a real I/O bottleneck, not a hang).

**⚠️ False-positive warning:** `ps aux | grep opensearch` will always find at least one match — the `grep` command itself, since the string "opensearch" appears in its own command line. If the real java process has actually died, you'll see a *new* PID every time you run this command (a fresh `grep` invocation each time), which looks exactly like "a process crash-looping with a new PID every few seconds." To get an accurate answer, exclude grep from its own results:
```bash
pgrep -f opensearch
# or
ps aux | grep -v grep | grep opensearch
```
If this comes back genuinely empty, the process is dead — not looping, just gone. Check `systemctl cat wazuh-indexer | grep -i restart` — if there's no `Restart=` directive, systemd will *not* auto-respawn it after a timeout, and it just sits failed until you restart it manually.

---

### 1.4 — Corrupted Security State (Interrupted Password Tool)

**Symptom:** After force-killing the `wazuh-passwords-tool.sh` script mid-run (Ctrl+C or Ctrl+Z), the indexer starts hanging on every subsequent attempt — even with a generous timeout, even with nothing else competing for resources. Dashboard login stops working with *both* the old and new password.

**Root cause:** `Ctrl+Z` **suspends** a process rather than killing it — it can leave the security index in a half-written, inconsistent state. `Ctrl+C` mid-run has the same risk if it lands during the actual write step. Since the password tool directly modifies OpenSearch's internal security configuration, an interrupted run can corrupt that state in a way that prevents the indexer's security plugin from initializing cleanly on the next boot — producing a hang that looks identical to a resource/timeout issue but isn't.

**Fix — restore from a known-good backup:**

If you have a backup from a previous *successful* run (see the mandatory backup step below), restore it:
```bash
# List available backups (auto-generated by the password tool on each successful run)
ls -la /etc/wazuh-indexer/internalusers-backup/

# Preserve the current (corrupted) file just in case
sudo cp /etc/wazuh-indexer/opensearch-security/internal_users.yml \
        /etc/wazuh-indexer/opensearch-security/internal_users.yml.corrupted-backup

# Restore the most recent known-good backup (adjust filename)
sudo cp /etc/wazuh-indexer/internalusers-backup/internal_users_YYYYMMDD_HHMMSS.yml.bkp \
        /etc/wazuh-indexer/opensearch-security/internal_users.yml

# cp as root does NOT preserve original ownership — fix this or you'll recreate a permissions bug
sudo chown wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/opensearch-security/internal_users.yml
sudo chmod 640 /etc/wazuh-indexer/opensearch-security/internal_users.yml

sudo systemctl reset-failed wazuh-indexer
sudo systemctl start wazuh-indexer
```

**Prevention (do this every time you run the password tool):**
1. **Never interrupt it once started.** It can sit quiet for 60-90+ seconds mid-run — that's normal, not a hang.
2. **The instant it completes successfully, back it up externally**, not just relying on its own auto-backup folder (which can itself have permissions issues):
   ```bash
   sudo cp /etc/wazuh-indexer/opensearch-security/internal_users.yml \
           ~/internal_users-known-good.yml.bkp
   ```

**If restoring the backup doesn't fix it** (the corruption goes deeper than this one file), the more decisive fix is to wipe the indexer's entire data directory and let it rebuild fresh (you'll lose indexed alert history, but not your configuration):
```bash
sudo systemctl stop wazuh-indexer
sudo pkill -9 -f opensearch   # ensure nothing lingering
grep "^path.data" /etc/wazuh-indexer/opensearch.yml   # confirm the actual data path
sudo mv /var/lib/wazuh-indexer /var/lib/wazuh-indexer.broken
sudo mkdir /var/lib/wazuh-indexer
sudo chown wazuh-indexer:wazuh-indexer /var/lib/wazuh-indexer
sudo systemctl start wazuh-indexer
```
You will need to re-run the password tool after this, since the security index is also gone.

---

### 1.5 — Password Tool Fails Instantly with "Backup could not be created"

**Symptom:** Running `wazuh-passwords-tool.sh` fails immediately (no 60-90 second wait first) with `ERROR: The backup could not be created`.

**Root cause:** This is a **different bug from 1.1 above** despite the similar wording. The password tool's "backup" step is not a simple file copy — it opens a live TLS connection to the indexer on port 9200 (via `securityadmin.sh -backup`) to pull the running security configuration. An **instant** failure (vs. a slow one) almost always means the indexer isn't running or isn't listening yet.

**Fix:** Confirm the indexer is actually up first:
```bash
sudo systemctl status wazuh-indexer
sudo ss -tlnp | grep 9200
```
If nothing is listening, start/wait for the indexer before running the password tool at all. **This is a strict ordering requirement** — the indexer must be `active (running)` before this tool will work, not just installed.

If the indexer *is* confirmed running and listening, and this still fails, it likely is the ownership issue from 1.1 instead:
```bash
sudo mkdir -p /etc/wazuh-indexer/backup
sudo chown wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/backup
sudo chmod 750 /etc/wazuh-indexer/backup
```

---

## 2. Wazuh Manager Startup Issues

### 2.1 — Manager Times Out but `wazuh-control start` Works Fine Manually

**Symptom:** `systemctl start wazuh-manager` times out and fails, but manually running `sudo /var/ossec/bin/wazuh-control start` completes cleanly with every daemon reporting "Started."

**Root cause:** The manager has to start ~15 separate internal daemons sequentially. On a resource-constrained or I/O-bottlenecked VM, this legitimately takes longer than systemd's default patience allows — it's not actually failing, just running out of time to report success.

**Fix:**
```bash
# First, stop the manual instance you started to test — don't leave two copies running
sudo /var/ossec/bin/wazuh-control stop

sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf > /dev/null << 'EOF'
[Service]
TimeoutStartSec=300
EOF
sudo systemctl daemon-reload
sudo systemctl restart wazuh-manager
```

### 2.2 — "wazuh-db already running" / "wazuh-syscheckd already running" on Restart

**Symptom:** After a messy stop/start cycle, `wazuh-control status` or the manager's startup log shows confusing "already running" messages for daemons, even right after a restart.

**Root cause:** Stale PID files left over from a previous unclean stop — `wazuh-control` trusts the PID file rather than verifying the process is actually alive.

**Fix:** Do one fully clean stop/start cycle, confirming nothing is actually running in between:
```bash
sudo /var/ossec/bin/wazuh-control stop
ps aux | grep wazuh-      # should show nothing but the grep itself
sudo systemctl restart wazuh-manager
```

### 2.3 — `wazuh-modulesd` Segfaults Repeatedly (`libvulnerability_scanner.so`)

**Symptom:** Kernel log (`dmesg`) shows repeated segfaults:
```
wazuh-modulesd[PID]: segfault at 50 ... in libvulnerability_scanner.so
```
`wazuh-control status` shows `wazuh-modulesd` cycling between running/crashed.

**Root cause:** A packaging bug in this specific OVA build's vulnerability-scanner component — not something you caused, and reproducible on a freshly imported, untouched VM. `wazuh-monitord` auto-respawns the process, so this is self-healing and mostly cosmetic (noisy logs), but it's worth disabling since this lab doesn't use CVE scanning at all.

**Fix:**
```bash
sudo sed -i '/<vulnerability-detector>/,/<\/vulnerability-detector>/ s/<enabled>yes<\/enabled>/<enabled>no<\/enabled>/' /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-manager
```

**Note:** disabling this in `ossec.conf` was observed to *not always* fully stop the crash in testing — the newer subsystem (`content_manager`/`router`/`inventory-harvester`) may be a partial replacement not fully gated by the same config flag in this Wazuh version. If it recurs after disabling, it's safe to ignore — it has no functional impact on the rest of the lab.

---

## 3. Wazuh Dashboard Issues

### 3.1 — Dashboard Shows "active (running)" But Won't Load in Browser

**Symptom:** `systemctl status wazuh-dashboard` shows healthy, but `https://<ip>` in the browser fails to connect (site can't be reached / connection refused).

**Root cause:** Systemd's "active (running)" only confirms the Node.js process hasn't crashed — it says nothing about whether it's actually finished initializing and bound to its port. The dashboard does real first-time bundle/cache-building work that's genuinely disk-heavy.

**Diagnosis:**
```bash
sudo ss -tlnp | grep 443
```
If nothing is listed, it's not actually serving yet.

**Fix:** Usually just patience — check `top -bn1 -p <dashboard-PID>`; if it shows high `wa` (I/O wait) in the overall `top` output and the process itself is burning some CPU, it's genuinely working, just slow (this was observed to take 30+ minutes in one severely I/O-bottlenecked case — see [9.1](#91-ide-disk-controller-io-bottleneck)). Keep checking `ss -tlnp | grep 443` every few minutes rather than restarting the service, which would just restart the slow initialization from scratch.

### 3.2 — "No API available to connect" Right After a Manager Restart

**Symptom:** Dashboard loads, but shows a red "[API connection] No API available to connect" banner immediately after starting/restarting the manager.

**Root cause:** Usually just timing — the Wazuh API (part of the manager) takes a short while to fully initialize all its modules after a restart, and the dashboard's health check can run before that finishes.

**Fix:** Wait ~1-2 minutes, click the refresh icon next to "Check API connection" in the dashboard. If still red after several minutes:
```bash
sudo ss -tlnp | grep 55000        # confirm the manager's API is actually listening
sudo grep -iP 'error|critical' /var/ossec/logs/ossec.log
```

---

## 4. Password & Credential Issues

### 4.1 — Password Rejected Despite Looking "Strong"

**Symptom:** The dashboard's password strength meter shows the password as strong, but `wazuh-passwords-tool.sh` rejects it with:
```
ERROR: The password must have a length between 8 and 64 characters and contain at least one upper and lower case letter, a number and a symbol( .* + ?- )
```

**Root cause:** The tool's actual accepted symbol set is much narrower than general "strong password" conventions — only `. * + ? -` are valid. Any other symbol (`!`, `@`, `#`, etc.) fails, regardless of what a browser-side strength meter says.

**Fix:** Use only letters, numbers, and `. * + ? -`. Example: `Defender-Wazuh2026.`

### 4.2 — "Resource 'admin' is reserved" When Using the Dashboard's Reset Password Button

**Symptom:** Clicking "Reset password" in the dashboard UI for the `admin` account fails with `{"status":"FORBIDDEN","message":"Resource 'admin' is reserved."}`.

**Root cause:** This is by design in OpenSearch's security model — `admin` is a reserved internal account and is deliberately excluded from the self-service password-change API. Only the CLI tool (`wazuh-passwords-tool.sh`) can change it.

**Fix:** Don't use the dashboard button for this account. Use the CLI tool as described in [Section 1.5](#15--password-tool-fails-instantly-with-backup-could-not-be-created) and the main guide.

### 4.3 — Neither Old Nor New Password Works After a Password Change

See [Section 1.4](#14--corrupted-security-state-interrupted-password-tool) — this is the signature of an interrupted password tool run corrupting the security state, not a memory/typo issue.

---

## 5. Networking Issues (VMware / Host-Level)

### 5.1 — VMs Lose Network Connectivity After Host Sleep/Resume

**Symptom:** Everything was working; laptop went to sleep (or sat idle long enough for network-saving power settings to kick in); on resume, VMs can't ping each other or the host, `dhclient`/`ipconfig /renew` hangs or fails.

**Root cause:** Two possible causes, check both:
1. VMware's background Windows services (`VMnetDHCP`, `VMware NAT Service`) can fail to resume cleanly after a host sleep.
2. Windows' power management can put the *virtual* network adapters themselves to sleep along with the physical hardware, and they don't always wake cleanly.

**Fix for (1)** — on the host, elevated PowerShell:
```powershell
Get-Service VMnetDHCP, "VMware NAT Service"
# Restart whichever isn't Running:
Start-Service VMnetDHCP
Start-Service "VMware NAT Service"
```
(`Restart-Service` may fail with "cannot open service" if not run as Administrator — this is a permissions error, not a real fault; re-open PowerShell as Admin.)

**Permanent fix for (2)** — one-time setup, do this before it ever bites you:
1. Device Manager → Network adapters
2. Find every **"VMware Network Adapter VMnetX"** entry
3. Right-click → Properties → Power Management tab → **uncheck** "Allow the computer to turn off this device to save power"

**After either fix, on each VM:**
```bash
# Linux:
sudo dhclient -r && sudo dhclient
```
```powershell
# Windows:
ipconfig /release
ipconfig /renew
```
Then re-test connectivity with `ping` before assuming anything Wazuh-specific is broken.

### 5.2 — Network Adapter Shows `state DOWN` at the Link Layer

**Symptom:** `ip a` shows `eth0: <BROADCAST,MULTICAST> ... state DOWN` — not just "no IP," the link itself is down.

**Root cause:** Most often the virtual NIC got disconnected at the hypervisor level (VMware's per-VM "connect at power on" / connect toggle), not a guest OS network config problem.

**Fix — check VMware side first:**
1. VM Settings → Network Adapter → confirm **"Connect at power on"** is checked
2. Check for a small connect/disconnect toggle in the VM window's status bar — easy to click accidentally

If that's not it, try guest-side:
```bash
sudo ip link set eth0 up
sudo dhclient eth0
nmcli device status   # check if NetworkManager thinks the device is "unmanaged"
sudo systemctl restart NetworkManager
```

### 5.3 — DNS Resolution Timeout Inside a VM ("Temporary failure in name resolution")

**Symptom:** `wget`/`curl` to an external URL fails with `Temporary failure in name resolution`, even though `ping <ip-address>` works fine.

**Root cause:** Usually one of: (a) the VM's adapter is on an isolated host-only network with no path to the internet at all — expected and correct if that's intentional; or (b) a transient DHCP/DNS server hiccup right after a cold boot.

**Fix:** If internet access is actually needed for that VM (e.g., downloading the Wazuh agent installer), temporarily switch its adapter to **NAT** in VMware settings, do the download, then switch back to the host-only network (VMnet1) for the rest of the lab. If it's meant to be isolated and this is just a boot-time hiccup:
```bash
sudo dhclient -r && sudo dhclient
ping -c 3 8.8.8.8
```

---

## 6. Agent Enrollment Issues

### 6.1 — Agent Stuck Retrying: "Unable to connect to enrollment service"

**Symptom:** `agent-auth` or the agent's own log repeats:
```
ERROR: (1208): Unable to connect to enrollment service at '[<ip>]:1515'
```
...even though `ping` and `Test-NetConnection`/basic port checks to that IP succeed.

**Root cause:** A raw TCP connection succeeding does **not** mean the actual enrollment handshake (which is TLS-based) will succeed. The most common cause of this exact pattern is **clock skew** between the agent and the manager — TLS handshakes fail if the systems' clocks are too far apart.

**Diagnosis:**
```powershell
# Windows agent:
[DateTime]::UtcNow
```
```bash
# Wazuh manager:
date -u
```
Compare the two **UTC** values directly (ignore local time/timezone display — only the underlying UTC moment matters for TLS).

**Fix:**
```powershell
# Set the Windows VM's timezone to UTC to remove timezone-conversion confusion entirely
Set-TimeZone -Id "UTC"
# Then set the exact time to match the manager's `date -u` output
Set-Date -Date "YYYY-MM-DD HH:MM:SS"
```
Standardizing every VM to UTC is also just good SOC practice — it avoids timestamp confusion across multiple log sources during real investigations, not just a workaround for this bug.

**Note:** VMware's "Synchronize guest time with host" option in VM Settings → Options → VMware Tools only works if VMware Tools (or `open-vm-tools`) is actually installed and running in that guest. Confirm with `Get-Service VMTools` (Windows) — if the service doesn't exist, that checkbox does nothing, and you must set the clock manually as above.

### 6.2 — "Duplicate agent name" Error During Re-Enrollment

**Symptom:** After clearing an agent's local `client.keys` (e.g., following a manager rebuild) and restarting it, enrollment fails on a loop with:
```
ERROR: Duplicate agent name: <hostname>. Unable to add agent (from manager)
```

**Root cause:** Clearing the agent's *local* identity doesn't remove the old registration record still sitting in the manager's own agent database under that same hostname.

**Fix:** Remove the stale entry on the manager side, then let the agent's automatic retry (usually every 60 seconds) succeed on its own:
- **Dashboard:** Agents management → find the old entry → delete/remove
- **Or CLI on the manager:**
  ```bash
  sudo /var/ossec/bin/manage_agents
  ```
  Choose the remove-agent option, select by ID or name.

You typically don't need to touch the agent VM at all after this — it will succeed on its next automatic retry.

### 6.3 — Agent Enrolled Successfully but Shows "Disconnected" Repeatedly

**Symptom:** Agent shows "Active" for a while, then flips to "Disconnected," particularly after the manager or host has been through any instability.

**Root cause:** Almost always one of the networking issues in [Section 5](#5-networking-issues-vmware--host-level) — the agent's persistent connection to `wazuh-remoted` on port 1514 dropped and hasn't re-established, usually due to the manager still recovering from a restart or a host-level network hiccup.

**Fix:** Confirm manager health and networking first (see 5.1), then just wait — agents retry their connection automatically. Manual re-enrollment is not needed unless the agent's identity was actually removed on the manager side (see 6.2).

### 6.4 — Wrong Manager IP Baked Into `ossec.conf` After Re-Running the Install Command

**Symptom:** You made a mistake entering the manager's IP in the "Deploy new agent" wizard, re-ran the install command with the correct IP, but the agent's log keeps showing it trying to reach the **old, wrong** IP.

**Root cause:** Re-running the installer on an already-installed agent does not reliably overwrite the existing `ossec.conf` — many installers try to preserve existing configuration on what looks like a "reinstall," silently ignoring the new value you provided.

**Fix:** Edit the config directly instead of trusting the installer to do it:
```powershell
$ossecConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"
Get-Content $ossecConf | Select-String "<address>"   # confirm current (wrong) value
(Get-Content $ossecConf -Raw) -replace '<address>.*?</address>', '<address>CORRECT_IP</address>' | Set-Content $ossecConf
Remove-Item "C:\Program Files (x86)\ossec-agent\client.keys" -ErrorAction SilentlyContinue
Restart-Service WazuhSvc
```
(Linux equivalent: edit `/var/ossec/etc/ossec.conf` directly with `sed` or a text editor, then `rm /var/ossec/etc/client.keys && sudo systemctl restart wazuh-agent`.)

---

## 7. Package Manager / Installation Issues

### 7.1 — `dpkg: error: dpkg frontend lock was locked by another process`

**Symptom:** Installing the Wazuh agent `.deb` package on Ubuntu fails with a dpkg lock error, even though you're not running any other install command yourself.

**Root cause:** Ubuntu's background `unattended-upgrades` service grabbed the package manager lock first — commonly the very first time a freshly imported/older VM image gets real internet access and starts catching up on pending updates.

**Fix — do NOT remove the lock file manually** (risks corrupting the package database). Identify and deal with the actual process holding it:
```bash
ps -p <pid-from-error-message> -f
```
If it's `unattended-upgrades` (or `apt`/`apt-get`), either wait for it to finish naturally, or if it's taking too long:
```bash
sudo kill <pid>
sleep 3
sudo fuser /var/lib/dpkg/lock-frontend   # should show no output once clear
```
Then retry the original install command.

---

## 8. Dashboard UI / Discover View Issues

### 8.1 — Discover Shows "No Results" Despite Data Existing (Confirmed via API)

**Symptom:** Threat Hunting / Discover view shows "No Results" for any query, even with a wide time range (90+ days) and no filters — but the Overview dashboard shows real alert counts, and a direct query to the indexer's API confirms thousands of real documents exist.

**Root cause:** The `wazuh-alerts-*` index pattern's configured **time field** was set to a plain string field called `timestamp` instead of the real date-typed field `@timestamp`. Since Discover's time-range filtering relies on the configured time field behaving like an actual date, filtering against a string field silently returns nothing, regardless of range or query.

**Diagnosis:** Stack/Dashboards Management → Index patterns → click `wazuh-alerts-*` → check the badge at the top: `Time field: 'timestamp'` (wrong) vs `Time field: '@timestamp'` (correct).

**Fix:** This field cannot be edited on an existing index pattern in this Wazuh version's UI — you must delete and recreate it:
1. Stack Management → Index patterns → select `wazuh-alerts-*` → delete
2. Create index pattern → type `wazuh-alerts-*` → on the time field dropdown, explicitly select **`@timestamp`**
3. Set it as default (star icon)
4. Hard-refresh your browser tab (`Ctrl+Shift+R`) — a stale reference to the deleted pattern's internal ID can otherwise persist in the current browser session even after recreating it under the same name

**If it still shows no results after all of that** — this was observed in testing to be a genuine UI/rendering bug in this specific Wazuh dashboard version's Discover/Data Explorer, unrelated to the time field. Don't keep chasing it. Use the **Indexer Management → Dev Tools** console instead (a raw query interface to the indexer, bypassing Discover's rendering entirely):
```
GET wazuh-alerts-*/_search
{
  "size": 3,
  "query": { "match_all": {} }
}
```
This confirms data health directly and is a perfectly valid (arguably more rigorous) way to verify your pipeline for portfolio/demo purposes.

**Note on the "API Console":** the dashboard has a separate tool, also sometimes reachable via "Dev Tools"-like navigation, called the **Wazuh API Console** — this talks to the *manager's* REST API on port 55000 (endpoints like `GET /agents`, `GET /manager/info`), not the indexer. If you query it with an OpenSearch-style `_search` request you'll get a 404 — that's not a bug, you're just in the wrong console. Look for **Indexer Management → Dev Tools** specifically for raw indexer queries.

---

## 9. VM-Level / Hypervisor Issues

### 9.1 — IDE Disk Controller I/O Bottleneck

**Symptom:** Chronic, wildly inconsistent slowness across the entire Wazuh server VM — indexer taking anywhere from 5 to 20+ minutes to start on different boots, dashboard taking a very long time on first load, and kernel hang warnings during shutdown (`inode_switch_wbs` blocked for more than 123 seconds, task `kworker` stuck in disk writeback).

**Root cause:** The Wazuh OVA imports into VMware with a legacy **IDE** virtual disk controller by default (it's primarily built/tested for VirtualBox). IDE emulation has a very shallow command queue depth with effectively no real parallelism — a poor match for OpenSearch's heavy, constant-fsync write workload. This was the single biggest, most disruptive root cause discovered across the whole build.

**Fix — change the controller to SCSI before first boot** (ideally) or at any point later (with more care):

**Before first boot (clean, low-risk):**
1. VM Settings → select the **Hard Disk (IDE)** entry → **Remove** (detaches only — does not delete the `.vmdk`)
2. **Add → Hard Disk → Use an existing virtual disk** → browse to the same `.vmdk` file
3. Choose **SCSI** as the controller type → Finish

**After the VM has already been used (higher risk — take a snapshot first):**
Same steps as above, but:
- **Take a VM snapshot before starting** — this is a real safety net if anything goes wrong
- After switching to SCSI, the VM may fail to find a bootable device on next power-on (blank screen, blinking cursor, then auto-powers-off) — this happens if the firmware's boot entry (especially on UEFI) was tied to the old IDE device path and doesn't automatically update
- If this happens, the fastest recovery is: **Snapshot Manager → Go to / Revert to the pre-change snapshot.** This is more reliable than trying to manually fix UEFI boot entries in an EFI shell, and preserves all your existing Wazuh configuration.
- If you don't have a snapshot and the VM won't boot after the controller change: try the boot menu (tap Esc/F2 repeatedly on power-on) and look for a "Hard Drive" or SCSI-referencing boot entry to select manually; if nothing works, revert to IDE (remove the SCSI disk, re-add the same file as IDE) — a working, slow VM is better than a fast, unbootable one.

**When to actually do this:** Do it on a brand-new import, before you've invested any configuration time — the risk/reward is completely different on a fresh VM (nothing to lose) versus a VM you've already spent hours configuring (real risk of an unbootable state). This guide's Phase 2 has you do it immediately after import, before first boot, specifically to avoid ever needing the risky "after the fact" version.

### 9.2 — Windows Defender Scanning VM Disk Files

**Symptom:** Same general symptom profile as 9.1 (chronic slowness), independent of or in addition to the disk controller issue.

**Root cause:** Real-time antivirus scanning intercepting every disk write into a VM's `.vmdk` file can add substantial overhead — potentially significant for a write-heavy workload like OpenSearch.

**Fix:**
1. Windows Security → Virus & threat protection → Manage settings → Add or remove exclusions
2. Add a **folder exclusion** for your VMware VMs directory

**Note:** in testing, adding this exclusion alone did *not* resolve the chronic timeouts — the IDE controller (9.1) turned out to be the dominant cause. Still worth doing as a one-time host setup step regardless, since it's a legitimate general performance improvement for any VM workload and costs nothing to configure.

### 9.3 — Unclean Shutdown / Kernel Hang During Power-Off

**Symptom:** `sudo shutdown -h now` (or VMware's guest shutdown) hangs for a very long time; the VM's console shows kernel messages like:
```
INFO: task kworker/X:X:PID blocked for more than 123 seconds.
```
with a call trace mentioning `inode_switch_wbs_work_fn` or similar disk-writeback functions.

**Root cause:** The kernel is waiting on a disk write/flush that isn't completing in a reasonable time — this is a symptom of the same underlying I/O bottleneck as 9.1, surfacing during shutdown instead of startup.

**Fix:** If it's been hung for many minutes with zero progress, it's safe to force power off via VMware's controls **only when the VM is about to be rebuilt/discarded anyway**. If you intend to keep using this VM, this recurring symptom is a strong signal to apply the SCSI controller fix (9.1) rather than repeatedly forcing shutdowns, which risks compounding filesystem inconsistency over time.

### 9.4 — Repeated Filesystem/State Corruption Leading to Full Rebuild

**Symptom:** After enough cycles of forced shutdowns, interrupted commands, and crash-loops, the VM reaches a point where even a completely fresh, empty indexer data directory still hangs identically on startup — ruling out data corruption as the cause, and pointing to something wrong at the VM/filesystem level itself.

**When to stop troubleshooting and rebuild:** If you've confirmed (a) DNS hardening is in place, (b) timeouts are generous, (c) a wiped data directory still exhibits the same hang, and (d) host resources are not under pressure — the filesystem itself has likely accumulated enough inconsistency from earlier forced shutdowns that further patching has poor ROI. A clean OVA re-import, done *with* the lessons in this guide applied from the start (especially the SCSI controller change before first boot), is faster than continuing to debug a VM with an unknown amount of accumulated damage.

**This is a normal, legitimate engineering call, not a failure** — real infrastructure work involves recognizing when to cut losses on a specific instance versus continuing to debug it indefinitely.

---

## 10. Active Response & Detection Issues

### 10.1 — VirusTotal Integration Placed on the Wrong Component

**Symptom:** FIM alerts fire correctly, but no VirusTotal enrichment ever appears.

**Root cause:** The `<integration>` block must go in the **manager's** `ossec.conf`, not the agent's. It's a common and easy mistake since FIM configuration itself (`<syscheck>`) *does* go on the agent, which can make it seem consistent to also put the integration there — but the integration daemon (`wazuh-integratord`) only runs on the manager.

**Fix:** Confirm the block is in `/var/ossec/etc/ossec.conf` on the **Wazuh manager VM**, not the Linux victim's agent config.

### 10.2 — Expecting Automatic Quarantine from the VirusTotal Integration

**Symptom / misunderstanding:** Expecting a file flagged as malicious by the VT integration to be automatically deleted or moved.

**Root cause:** Not a bug — this is how the integration is designed. It only **enriches** the FIM alert with a detection verdict (ratio, engine names). It does not take any remediation action on its own.

**If true auto-quarantine is wanted:** requires writing a custom Active Response script triggered off a rule matching "VT positives > 0" — a legitimate stretch goal, not something included by default.

---

## 11. Diagnostic Toolkit

The commands that solved nearly every issue in this guide. When something new goes wrong, work through these roughly in order:

```bash
# 1. Is the service actually running, and what does systemd think happened?
sudo systemctl status <service-name> -l --no-pager

# 2. What's in the systemd journal for it? (last 100 lines, no pager for easy scrolling)
sudo journalctl -u <service-name> -n 100 --no-pager

# 3. Watch it live while starting, rather than reconstructing after the fact
sudo journalctl -u <service-name> -f

# 4. Check the component's OWN dedicated log file — often has more detail than the journal
sudo tail -100 /var/log/wazuh-indexer/wazuh-cluster.log      # indexer
sudo tail -100 /var/ossec/logs/ossec.log                     # manager
sudo journalctl -u wazuh-dashboard -n 100 --no-pager          # dashboard (JSON-formatted lines)

# 5. Is it actually listening on the port it should be?
sudo ss -tlnp | grep <port>       # 9200=indexer, 55000=manager API, 443=dashboard, 1514/1515=agent comms

# 6. Is the process alive, and how much CPU/memory/IO is it actually using?
ps aux | grep -v grep | grep <process-name>     # exclude grep matching itself
top -bn1 -p <PID>

# 7. Kernel-level clues — actual crashes, OOM kills, disk hangs
sudo dmesg -T | tail -50
sudo dmesg | grep -i "killed process"     # OOM killer signature

# 8. Basic resource check
free -h
df -h

# 9. DNS/hostname resolution speed (should be near-instant)
time getent hosts $(hostname)

# 10. Confirm a systemd override actually took effect (more reliable than `systemctl show` for this)
sudo systemctl cat <service-name>

# 11. Direct indexer health check, bypassing the dashboard entirely
curl -k -u admin:'<password>' https://localhost:9200/_cluster/health?pretty

# 12. Direct data verification, bypassing Discover/dashboard rendering entirely
curl -k -u admin:'<password>' "https://localhost:9200/wazuh-alerts-*/_search?size=5&sort=@timestamp:desc&pretty"
```

**General principles that held true across nearly every issue in this project:**
- A service showing `active (running)` in systemd only means the process hasn't crashed — it does not mean the service has finished initializing or is actually functional yet.
- When something times out, don't assume it's frozen — check for actual progress (new log lines, growing CPU/memory usage) before concluding it's stuck versus just slow.
- Prefer direct, low-level verification (raw API queries, log files) over trusting a UI layer when something looks broken — UI bugs are real and can waste far more time than the underlying issue would have.
- Never interrupt a command that explicitly warns against it (the password tool) — the cost of corruption is much higher than the cost of waiting.
- Always shut down VMs gracefully from inside the guest, never via the hypervisor's force-power-off, except when a VM is already being discarded/rebuilt.
