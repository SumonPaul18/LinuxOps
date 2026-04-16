# Guide: Mastering SSH Key Authentication for Ceph Clusters

## Table of Contents
1. [The Real-World Problem](#1-the-real-world-problem)
2. [Scenario A: The "Pro" Way (SSH Config)](#2-scenario-a-the-pro-way-ssh-config)
3. [Scenario B: The "Standard" Way (Default Keys)](#3-scenario-b-the-standard-way-default-keys)
4. [Scenario C: The "Quick" Way (One-Time Command)](#4-scenario-c-the-quick-way-one-time-command)
5. [Troubleshooting: When It Still Asks for Password](#5-troubleshooting-when-it-still-asks-for-password)
6. [Final Verification](#6-final-verification)

---

## 1. The Real-World Problem

**Context:**
You are setting up a Ceph Cluster. You generated a specific key pair named `ceph_id_rsa` for security and organization.
- **Works:** `ssh -i ~/.ssh/ceph_id_rsa ceph4@192.168.68.20`
- **Fails:** `ssh ceph4@192.168.68.20` (Asks for password)

**Why?**
SSH clients are lazy. By default, they only look for files named `id_rsa`, `id_ed25519`, etc. They ignore `ceph_id_rsa` unless you explicitly tell them to use it.

---

## 2. Scenario A: The "Pro" Way (SSH Config)

**Best For:** Production environments, managing multiple nodes (ceph1, ceph2, ceph3...), and keeping custom key names.

### Step 1: Create/Edit Config
On your admin node (`ceph1`), edit the SSH config file:

```bash
nano ~/.ssh/config
```

### Step 2: Add Host Definitions
Paste this configuration. This maps the short name `ceph4` to the IP and the specific key.

```ssh-config
# Ceph Cluster Nodes
Host ceph4
    HostName 192.168.68.20
    User ceph4
    IdentityFile ~/.ssh/ceph_id_rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host ceph2
    HostName 192.168.68.21
    User ceph4
    IdentityFile ~/.ssh/ceph_id_rsa

Host ceph3
    HostName 192.168.68.22
    User ceph4
    IdentityFile ~/.ssh/ceph_id_rsa
```

### Step 3: Secure the Config File
SSH will refuse to use this file if permissions are wrong.

```bash
chmod 600 ~/.ssh/config
```

### Step 4: Use It
Now, you don't need to type the IP or the key path. Just type:

```bash
ssh ceph4
```

---

## 3. Scenario B: The "Standard" Way (Default Keys)

**Best For:** Simple setups where you don't want to maintain a config file.

### Step 1: Rename Your Key
Rename your custom key to the default name `id_rsa`. SSH picks this up automatically.

```bash
# Backup existing default keys if any
mv ~/.ssh/id_rsa ~/.ssh/id_rsa.old 2>/dev/null
mv ~/.ssh/id_rsa.pub ~/.ssh/id_rsa.old.pub 2>/dev/null

# Rename your Ceph key to default
cp ~/.ssh/ceph_id_rsa ~/.ssh/id_rsa
cp ~/.ssh/ceph_id_rsa.pub ~/.ssh/id_rsa.pub
```

### Step 2: Fix Permissions
```bash
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### Step 3: Use It
Now standard commands work:

```bash
ssh ceph4@192.168.68.20
```

---

## 4. Scenario C: The "Quick" Way (One-Time Command)

**Best For:** Testing connectivity or quick debugging. Not recommended for daily scripts.

### Command
Always pass the `-i` flag:

```bash
ssh -i ~/.ssh/ceph_id_rsa ceph4@192.168.68.20
```

---

## 5. Troubleshooting: When It Still Asks for Password

If you did the above and it **still** asks for a password, the issue is on the **Remote Server (`ceph4`)**. SSH rejects keys with bad permissions.

### Step 1: Check Remote Permissions
Log in to `ceph4` (using password) and run these exact commands:

```bash
# 1. Home directory must not be writable by others
chmod 755 /home/ceph4

# 2. .ssh folder must be private
chmod 700 /home/ceph4/.ssh

# 3. authorized_keys must be private
chmod 600 /home/ceph4/.ssh/authorized_keys

# 4. Ensure ownership is correct
chown -R ceph4:ceph4 /home/ceph4/.ssh
```

### Step 2: Check SELinux (CentOS/RHEL/Rocky Only)
If you are on RHEL-based systems, SELinux might block the key. Run this on `ceph4`:

```bash
restorecon -R -v /home/ceph4/.ssh
```

### Step 3: Restart SSH Service
Apply changes on `ceph4`:

```bash
sudo systemctl restart sshd
```

---

## 6. Final Verification

To see exactly what is happening, run this debug command from `ceph1`:

```bash
ssh -v ceph4
```

**Success Indicators:**
- Look for: `debug1: Offering public key: .../ceph_id_rsa`
- Look for: `debug1: Server accepts key`
- Look for: `Authenticated ... using "publickey"`

**Failure Indicators:**
- If you see: `Next authentication method: password`, then re-check **Section 5**.

---

### Quick Reference Table

| Scenario | Command to Connect | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **SSH Config** | `ssh ceph4` | Clean, professional, scalable | Requires initial setup |
| **Default Key** | `ssh ceph4@IP` | No config needed | Clutters default key folder |
| **Explicit Flag** | `ssh -i key user@IP` | Good for testing | Typing heavy, error-prone |

***