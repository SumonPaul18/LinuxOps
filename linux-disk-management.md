# 📚 Linux Disk Space Management: A Complete, Step-by-Step Guide for Real-World Scenarios

> **For:** SysAdmins, DevOps Engineers & Linux Users  
> **Goal:** Diagnose, resolve, and prevent 100% disk usage  
> **Style:** Practical • Real-life examples • Copy-paste ready  

---

## 📑 Table of Contents

1. [Introduction: Why Disk Full is a Critical Emergency](#1-introduction-why-disk-full-is-a-critical-emergency)  
2. [Step 1: Diagnose the Problem – Find What’s Eating Your Disk](#2-step-1-diagnose-the-problem--find-whats-eating-your-disk)  
3. [Step 2: Clean Up – Safely Free Up Space](#3-step-2-clean-up--safely-free-up-space)  
4. [Step 3: Verify & Monitor – Confirm Fix and Prevent Recurrence](#4-step-3-verify--monitor--confirm-fix-and-prevent-recurrence)  
5. [Advanced: Expand Disk if Physical Space is Available (LVM Example)](#5-advanced-expand-disk-if-physical-space-is-available-lvm-example)  
6. [Best Practices: Avoid 100% Disk Usage in the Future](#6-best-practices-avoid-100-disk-usage-in-the-future)  
7. [Real-Life Scenarios & Troubleshooting Tips](#7-real-life-scenarios--troubleshooting-tips)  

---

## 1. Introduction: Why Disk Full is a Critical Emergency

When your Linux root (`/`) filesystem hits **100% usage**, it’s not just “slow”—it’s a **system emergency**. Here’s what can break:

- **System logs stop writing** → You can’t debug issues  
- **Applications crash** (e.g., databases, web servers)  
- **SSH may hang or refuse login**  
- **Cron jobs fail silently**  
- **Package updates (`apt`) fail**  
- **Docker/Kubernetes pods go into `CrashLoopBackOff`**

> 💡 **Real-life example**: At your ISP (`dayclothingbd.com`), if the Zimbra mail server’s `/` fills up, **incoming/outgoing emails stop**, and the webmail interface fails. Users complain instantly.

**Your Goal:** Free space **fast** → **diagnose root cause** → **prevent recurrence**.

---

## 2. Step 1: Diagnose the Problem – Find What’s Eating Your Disk

> 🎯 **Rule**: Never delete blindly. Always identify the culprit first.

### 🔍 2.1 Check Overall Disk Usage

```bash
df -h
```

- Look for the `/` mount point.
- If **Use% = 100%**, proceed.
- Note the filesystem name (e.g., `/dev/mapper/ubuntu--vg-ubuntu--lv` → this is **LVM**).

### 🔍 2.2 Find Largest Directories (Top-Level)

```bash
sudo du -h --max-depth=1 / 2>/dev/null | sort -hr
```

**Example Output:**
```
45G     /var
8G      /home
1.2G    /usr
500M    /opt
...
```

👉 **Focus on `/var`** – it’s the #1 culprit (logs, caches, mail queues, containers).

### 🔍 2.3 Drill Down into the Largest Directory

If `/var` is huge:
```bash
sudo du -h --max-depth=1 /var 2>/dev/null | sort -hr
```

Common large subdirs:
- `/var/log` → system & app logs
- `/var/lib` → Docker, databases, package managers
- `/var/mail` or `/var/spool` → mail queues (critical for Zimbra!)
- `/var/cache` → APT, snap, or application cache

### 🔍 2.4 Find Large Individual Files (>100 MB)

```bash
sudo find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | awk '{print $9 ": " $5}' | sort -hr
```

**Real-life finds:**
- `/var/log/zimbra.log.1` → 20 GB (Zimbra not rotating logs)
- `/var/lib/docker/overlay2/.../merged/some-file` → Huge Docker layer
- `/home/user/large-dump.sql` → Forgotten database backup

> ✅ **Tip**: Add `-name "*.log"` to `find` to focus on logs:
> ```bash
> sudo find /var/log -type f -name "*.log" -size +50M -exec ls -lh {} \;
> ```

---

## 3. Step 2: Clean Up – Safely Free Up Space

> ⚠️ **WARNING**: Never `rm -rf /` or delete files you don’t understand.  
> ✅ **Always backup critical data before deleting**.

### 🧹 3.1 Clean System & Application Logs

#### A. Journalctl (systemd logs)
```bash
# Keep only last 3 days
sudo journalctl --vacuum-time=3d

# Or limit to 500 MB
sudo journalctl --vacuum-size=500M
```

#### B. Traditional logrotate (if misconfigured)
```bash
# Check /var/log for huge .log or .gz files
ls -lh /var/log/

# Truncate (NOT delete) active logs safely
sudo truncate -s 0 /var/log/syslog
sudo truncate -s 0 /var/log/kern.log

# For Zimbra-specific logs (if used)
sudo truncate -s 0 /var/log/zimbra.log
```

> 💡 **Why truncate?** Some apps (like Zimbra) keep log file handles open. Deleting the file won’t free space until the app restarts. `truncate` empties it instantly.

#### C. Force logrotate (if configured)
```bash
sudo logrotate -f /etc/logrotate.conf
```

### 🧹 3.2 Clean Package Manager Cache

#### APT (Debian/Ubuntu):
```bash
sudo apt clean          # Remove .deb files
sudo apt autoremove     # Remove old kernels & unused deps
```

#### YUM/DNF (RHEL/CentOS):
```bash
sudo dnf clean all
sudo dnf autoremove
```

### 🧹 3.3 Clean Temporary Files

```bash
# System temp
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# User temp (if safe)
rm -rf ~/.cache/*
```

### 🧹 3.4 Clean Docker (If Used)

```bash
# Remove stopped containers, unused networks, dangling images
docker system prune -f

# Remove ALL unused data (including volumes)
docker system prune -a --volumes -f

# Check Docker disk usage
docker system df
```

### 🧹 3.5 Clean Zimbra Mail Queues (Critical for Your ISP)

If you see non-existent users like `erm@reverie-bd.com` in queue:
```bash
# View mail queue
/opt/zimbra/postfix/sbin/postqueue -p

# Delete ALL queued mails (use with caution!)
/opt/zimbra/postfix/sbin/postsuper -d ALL

# Or delete by recipient
/opt/zimbra/postfix/sbin/postqueue -p | awk '/erm@reverie-bd.com/ {print $1}' | sudo /opt/zimbra/postfix/sbin/postsuper -d -
```

> ✅ **After cleanup**, restart Zimbra: `sudo su - zimbra -c "zmcontrol restart"`

### 🧹 3.6 Remove Old Kernels (Ubuntu/Debian)

```bash
# List installed kernels
dpkg --list | grep linux-image

# Keep current kernel (check with `uname -r`)
# Remove old ones, e.g.:
sudo apt purge linux-image-5.4.0-100-generic
```

---

## 4. Step 3: Verify & Monitor – Confirm Fix and Prevent Recurrence

### ✅ 4.1 Verify Disk Space is Freed

```bash
df -h /
```

You should now see **< 90% usage**.

### 🔔 4.2 Set Up Monitoring (Prevent Future Emergencies)

#### A. Simple cron alert (add to root crontab: `sudo crontab -e`)
```bash
# Check every 5 minutes; alert if / > 90%
*/5 * * * * df / | awk 'NR==2 {gsub(/%/,""); if ($5 > 90) print "ALERT: Disk usage is " $5 "%" | "mail -s \"Disk Alert\" sumonpaul267@gmail.com"}'
```

#### B. Use `ncdu` for interactive disk analysis (install it!)
```bash
sudo apt install ncdu
sudo ncdu /
```
> Navigate with arrow keys, delete with `d`.

---

## 5. Advanced: Expand Disk if Physical Space is Available (LVM Example)

> 📌 **Applies to your case**: Your `fdisk -l` shows **111.79 GB disk**, but LVM uses only **54 GB** → you have **~57 GB free**!

### 🔧 Steps to Expand LVM Logical Volume

```bash
# 1. Check free space in Volume Group
sudo vgdisplay

# 2. Extend Logical Volume to use ALL free space
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv

# 3. Resize filesystem (ext4)
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# 4. Verify
df -h /
```

> ✅ **Result**: Your `/` disk will grow from 54 GB → ~110 GB.  
> ⚠️ **No reboot needed**. Safe on live systems.

---

## 6. Best Practices: Avoid 100% Disk Usage in the Future

| Area | Best Practice |
|------|---------------|
| **Logs** | Configure `logrotate` for all apps (including Zimbra). Set max size & rotation count. |
| **Monitoring** | Use `cron` alerts or tools like `netdata`, `Prometheus+Node Exporter`. |
| **Separate Partitions** | Mount `/var`, `/home`, `/opt` on **separate disks/partitions**. So `/` filling won’t crash the whole system. |
| **Docker** | Limit container disk usage. Use external volumes for persistent data. |
| **Backups** | Never store backups on the same disk as the system. Use remote storage. |
| **Zimbra** | Regularly check mail queues. Set up spam filters to reduce fake user attacks. |

---

## 7. Real-Life Scenarios & Troubleshooting Tips

### 🚨 Scenario 1: “I deleted files, but `df` still shows 100%!”

- **Cause**: A running process still holds a handle to the deleted file (common with logs).
- **Fix**: 
  ```bash
  # Find the process
  sudo lsof +L1

  # Restart that service (e.g., rsyslog, zimbra, nginx)
  sudo systemctl restart rsyslog
  ```

### 🚨 Scenario 2: “My Zimbra mail queue is full of fake users like `erm@reverie-bd.com`”

- **Cause**: Your server is being used as an open relay or under spam attack.
- **Immediate Fix**: 
  ```bash
  /opt/zimbra/postfix/sbin/postsuper -d ALL
  ```
- **Long-term**: 
  - Ensure `smtpd_recipient_restrictions` in Postfix blocks unknown users.
  - Use Barracuda to filter spam before it hits Zimbra.
  - Monitor `/var/log/zimbra.log` for repeated failed deliveries.

### 🚨 Scenario 3: “I can’t even run `df` – command not found!”

- **Cause**: `/bin` or `/usr` is on a separate partition that’s full/unmounted.
- **Fix**: Boot from a Live USB and clean from there.

### 💡 Pro Tip: Create a “Disk Emergency” Alias

Add to `~/.bashrc`:
```bash
alias diskcheck='echo "=== Disk Usage ==="; df -h /; echo "=== Top 5 Directories ==="; sudo du -h --max-depth=1 / 2>/dev/null | sort -hr | head -5'
```

Run with just `diskcheck`.

---

## ✅ Final Checklist

- [ ] Ran `df -h` → confirmed 100% usage  
- [ ] Used `du`/`find` to find largest dirs/files  
- [ ] Cleaned logs, cache, temp, Docker, or mail queues  
- [ ] Verified with `df -h` → usage < 90%  
- [ ] (Optional) Expanded LVM if physical space available  
- [ ] Set up monitoring to prevent recurrence  

---
