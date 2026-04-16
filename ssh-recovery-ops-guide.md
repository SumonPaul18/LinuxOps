# SSH Infrastructure Management: Ceph Clusters & OpenStack Recovery
### A Practical Field Guide for DevOps Engineers and System Administrators

**Version:** 1.0 (2026 Edition)  
**Author:** Sumon (IT Infrastructure & DevOps Specialist)  
**Target Audience:** System Administrators, DevOps Engineers, Cloud Architects  
**Environment:** Ubuntu 24.04 LTS, OpenSSH 9.x, Ceph Reef/Squid, OpenStack Caracal/Dalmatian  

---

## Table of Contents

1. [Introduction & Modern SSH Landscape](#1-introduction--modern-sh-landscape)
2. [Foundation: Building a Secure Ceph Cluster SSH Fabric](#2-foundation-building-a-secure-ceph-cluster-sh-fabric)
3. [Advanced Troubleshooting: The "Permission Denied" Detective Story](#3-advanced-troubleshooting-the-permission-denied-detective-story)
4. [Crisis Management: OpenStack Instance Key Recovery](#4-crisis-management-openstack-instance-key-recovery)
5. [Operational Excellence: Maintenance, Security & Lifecycle](#5-operational-excellence-maintenance-security--lifecycle)
6. [Appendix: Quick Reference Commands](#appendix-quick-reference-commands)

---

## 1. Introduction & Modern SSH Landscape

### The Real-World Context
In modern cloud infrastructure, SSH is not just a tool; it is the nervous system of your operations. Whether you are orchestrating a 100-node Ceph storage cluster or managing ephemeral OpenStack instances, your ability to access systems securely and reliably defines your success. 

Imagine this scenario: It’s 2 AM. Your Ceph cluster health is degraded because an OSD daemon failed to start on `ceph2`. You need to log in immediately to check logs. If your SSH keys are misconfigured, or if you’ve lost access to an OpenStack instance due to a lost private key, those precious minutes turn into hours of downtime. This guide is born from real battlefield experiences—like the time we spent four hours debugging a simple "Permission Denied" error only to find a single misplaced line in a cloud-init config file [[12]].

### Why This Guide Exists
Most documentation focuses on theory. This guide focuses on **action**. We will cover:
*   How to build a bulletproof passwordless SSH network for Ceph from scratch.
*   How to debug complex authentication failures involving `AllowUsers` and `AuthenticationMethods`.
*   How to recover access to an OpenStack instance when the key pair is lost, using only the VNC console.
*   How to maintain these systems with professional rigor.

### Prerequisites & Version Checks
Before diving in, ensure your environment matches our standards. We operate on the bleeding edge of stability.

*   **OS:** Ubuntu 24.04 LTS (Noble Numbat).
*   **SSH Server/Client:** OpenSSH 9.6p1 or higher (Ubuntu repos usually provide 9.6+, while upstream is at 9.9p2 as of early 2026) [[3]][[9]].
*   **Ceph:** Latest stable release (Reef or Squid).
*   **OpenStack:** Caracal or newer.

**Practical Step: Verifying Versions**
Always verify your tools before starting critical operations. Do not assume versions.

1.  **Check OpenSSH Version:**
    Run this on your admin workstation and target nodes:
    ```bash
    ssh -V
    ```
    *Expected Output:* `OpenSSH_9.6p1 Ubuntu-3ubuntu13.15, OpenSSL 3.0.13...`
    If you see version 8 or lower on a new Ubuntu 24.04 install, your package lists are outdated. Run `sudo apt update && sudo apt upgrade openssh-client openssh-server`.

2.  **Verify Official Sources:**
    Never install SSH from random PPAs unless necessary. The gold standard is the official OpenSSH release notes [[3]]. For Ubuntu, the `ubuntu-updates` repository is your trusted source [[9]]. If you need a feature from OpenSSH 9.9 specifically, consult the official release notes first to understand the security implications before compiling from source [[1]].

---

## 2. Foundation: Building a Secure Ceph Cluster SSH Fabric

### The Scenario
You have three bare-metal servers ready for a Ceph cluster: `ceph1` (Admin), `ceph2`, and `ceph3`. Ceph deployment tools (like `cephadm` or `ceph-deploy`) rely entirely on passwordless SSH to push configurations and start daemons [[14]]. If SSH fails, the cluster fails.

### Phase 1: The Clean Slate Strategy
**Story:** In a previous project, we tried to set up SSH on reused servers without cleaning old configurations. Result? Conflicting `known_hosts`, old authorized keys allowing ex-employees access, and weird permission errors. We learned: **Always start clean.**

#### Step 1.1: Wipe Old Configurations
On **all three nodes** (`ceph1`, `ceph2`, `ceph3`), log in via console or existing SSH and run:

```bash
# Backup existing config just in case (Professional habit)
sudo cp -r ~/.ssh ~/ssh-backup-$(date +%F)

# Remove the entire .ssh directory
rm -rf ~/.ssh

# Clear system-level known hosts if reusing IPs
sudo rm -f /etc/ssh/ssh_known_hosts
```

#### Step 1.2: Create the Dedicated Operations User
Never use `root` directly for daily operations. Create a dedicated user, e.g., `cephuser`.

1.  **Create User & Set Password:**
    ```bash
    sudo adduser cephuser
    # Follow prompts. Use a strong, complex password.
    ```

2.  **Grant Sudo Privileges:**
    Ceph installation requires root privileges. Add the user to the `sudo` group:
    ```bash
    sudo usermod -aG sudo cephuser
    ```

3.  **Enable Passwordless Sudo (For Automation):**
    Automation tools hate password prompts. Edit the sudoers file safely:
    ```bash
    sudo visudo
    ```
    Add this line at the bottom:
    ```text
    cephuser ALL=(ALL) NOPASSWD:ALL
    ```
    *Verification:* Switch to the user (`su - cephuser`) and run `sudo whoami`. It should return `root` without asking for a password.

### Phase 2: Generating and Distributing Keys
We will use **Ed25519** keys. They are faster, smaller, and more secure than RSA, and fully supported in OpenSSH 9.x [[3]].

#### Step 2.1: Generate Keys on Admin Node (`ceph1`)
Log in as `cephuser` on `ceph1`.

```bash
ssh-keygen -t ed25519 -C "ceph-cluster-admin-2026" -f ~/.ssh/id_ed25519
```
*   **Passphrase:** Leave it **empty** for automation purposes. (In high-security manual environments, use a passphrase with `ssh-agent`, but for cluster bootstrapping, empty is standard practice).
*   **Output:** You now have a private key (`id_ed25519`) and a public key (`id_ed25519.pub`).

#### Step 2.2: Distribute Keys to All Nodes
We need `ceph1` to access `ceph1`, `ceph2`, and `ceph3` without passwords.

```bash
# Copy key to itself (localhost)
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@ceph1

# Copy key to node 2
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@ceph2

# Copy key to node 3
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@ceph3
```
*Note:* You will be prompted for the `cephuser` password one time per node. This is the last time you’ll type it for SSH.

**What `ssh-copy-id` does behind the scenes:**
It appends your public key to `~/.ssh/authorized_keys` on the remote server and sets correct permissions automatically.

### Phase 3: The SSH Config Shortcut
Typing `ssh cephuser@192.168.68.249 -i ~/.ssh/id_ed25519` is tedious. Let’s simplify.

On `ceph1`, edit `~/.ssh/config`:
```bash
nano ~/.ssh/config
```

Add the following configuration:
```text
Host ceph1
    HostName 192.168.68.248
    User cephuser
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host ceph2
    HostName 192.168.68.249
    User cephuser
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host ceph3
    HostName 192.168.68.250
    User cephuser
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```
*Security Note:* `StrictHostKeyChecking no` is used here for lab/cluster bootstrapping to avoid interactive prompts. In a locked-down production environment, you would pre-populate `known_hosts` instead.

Set strict permissions on the config file:
```bash
chmod 600 ~/.ssh/config
```

### Phase 4: Verification & Operations
Now, test the fabric.

1.  **Basic Connectivity:**
    ```bash
    ssh ceph1 hostname
    ssh ceph2 hostname
    ssh ceph3 hostname
    ```
    *Success Criteria:* Returns the hostname immediately without any password prompt.

2.  **Sudo Test:**
    ```bash
    ssh ceph2 "sudo whoami"
    ```
    *Success Criteria:* Returns `root`.

3.  **Cluster Readiness:**
    Try a loop to check all nodes:
    ```bash
    for node in ceph1 ceph2 ceph3; do echo "Checking $node:"; ssh $node "uptime"; done
    ```

If any step fails, do not proceed to Ceph installation. Fix SSH first.

---

## 3. Advanced Troubleshooting: The "Permission Denied" Detective Story

### The Scenario
You followed the guide, but when you try `ssh ceph2`, you get:
`Permission denied (publickey,password).`
Or worse: `User cephuser from 192.168.68.248 not allowed because not listed in AllowUsers`.

This happened to us in a real deployment. The culprit? A hidden configuration file left by a cloud image.

### Step 3.1: The Art of Log Analysis
When SSH fails, the client tells you little. The **server** holds the truth.

1.  **Open Console Access:** Log into the problematic node (`ceph2`) via VNC/IPMI console. You cannot SSH in to debug SSH.
2.  **Monitor Logs in Real-Time:**
    ```bash
    sudo tail -f /var/log/auth.log
    ```
3.  **Trigger the Failure:** From your admin machine, try to SSH again.
4.  **Read the Clue:**
    *   *Clue A:* `Failed password for cephuser` → Wrong password or PAM issue.
    *   *Clue B:* `Connection closed by invalid user cephuser` → User doesn't exist.
    *   *Clue C:* `User cephuser ... not allowed because not listed in AllowUsers` → **The Smoking Gun.**
    *   *Clue D:* `Authentication refused: bad ownership or modes for file` → Permission error.

### Step 3.2: Solving the "AllowUsers" Mystery
**The Story:** Our log showed Clue C. We checked `/etc/ssh/sshd_config` and saw `# AllowUsers ceph2` commented out. We were confused. Why was it active?

**The Reality:** Modern Ubuntu uses modular configs. The main file includes snippets from `/etc/ssh/sshd_config.d/`.

1.  **Check Drop-in Directory:**
    ```bash
    ls -l /etc/ssh/sshd_config.d/
    ```
    You might see `50-cloud-init.conf` or `50-ubuntu.conf`.

2.  **Inspect the File:**
    ```bash
    sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```
    There it was: `AllowUsers ceph2`. This restricted access to only the user `ceph2`, but our user was named `cephuser`.

3.  **The Fix:**
    Edit the file:
    ```bash
    sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```
    Change it to:
    ```text
    AllowUsers cephuser
    ```
    Or comment it out entirely if you trust your firewall:
    ```text
    # AllowUsers ceph2
    ```

4.  **Restart SSH:**
    ```bash
    sudo systemctl restart sshd
    ```

### Step 3.3: Solving the "AuthenticationMethods" Conflict
**The Error:** `Disabled method "password" in AuthenticationMethods list "publickey,password"`.

**The Cause:** Someone tried to enforce Multi-Factor Authentication (MFA) by setting `AuthenticationMethods publickey,password` but didn’t properly enable password authentication in the chain, or the PAM module wasn’t ready.

**The Fix:**
1.  Check effective config:
    ```bash
    sudo sshd -T | grep authenticationmethods
    ```
2.  If it shows `publickey,password`, edit `/etc/ssh/sshd_config` (or the drop-in file) and **remove** or **comment out** the `AuthenticationMethods` line.
    ```text
    # AuthenticationMethods publickey,password
    ```
3.  Ensure `PasswordAuthentication yes` is set explicitly.
4.  Test config syntax before restarting:
    ```bash
    sudo sshd -t
    # If no output, syntax is OK.
    sudo systemctl restart sshd
    ```

### Step 3.4: The Permission Labyrinth
SSH is extremely strict about file permissions. If `~/.ssh` is writable by others, SSH refuses to use it.

**Checklist for the Target Node:**
```bash
# Home directory should not be world-writable
chmod 755 /home/cephuser

# .ssh directory must be 700
chmod 700 /home/cephuser/.ssh

# authorized_keys must be 600
chmod 600 /home/cephuser/.ssh/authorized_keys

# Private key (on client) must be 600
chmod 600 ~/.ssh/id_ed25519
```
**Ownership Check:** Ensure everything belongs to `cephuser`:
```bash
sudo chown -R cephuser:cephuser /home/cephuser/.ssh
```

---

## 4. Crisis Management: OpenStack Instance Key Recovery

### The Scenario
Disaster strikes. You lost your local private key for a critical OpenStack instance. The instance is running, but you can’t SSH in. OpenStack cannot "give" you the old key back for security reasons [[28]].

**The Solution:** We will use the **NoVNC Console** to reset root access, enable temporary password login, recover data, and inject a new key.

### Phase 1: Gaining Console Access
1.  Log in to the **OpenStack Dashboard (Horizon)**.
2.  Navigate to **Project > Compute > Instances**.
3.  Click **Console** next to your locked instance.
4.  You will see the login prompt.

### Phase 2: Resetting Root Password
Ubuntu images often disable root login by default. We need to enable it.

1.  **Login:** Use the default user (usually `ubuntu`) if you know the password, or if you have serial console access, reboot into **Recovery Mode** to reset passwords.
    *   *Alternative (if you have dashboard password reset capability):* Some OpenStack setups allow injecting a new admin password via the "Reset Password" action, but this relies on guest agents which may not be installed.
    *   *Manual Method (Guaranteed):* If you can’t log in at all, you must reboot into single-user mode via the console:
        *   Reboot instance from dashboard.
        *   In the console, hold `Shift` to enter GRUB menu.
        *   Edit the kernel line: append `init=/bin/bash` or `rw init=/sysroot/bin/sh` (for CentOS) / `rw single` (for Ubuntu).
        *   Boot. You will land in a root shell.
        *   Remount root as read-write: `mount -o remount,rw /`
        *   Reset password: `passwd root` (and `passwd ubuntu`).

*Assumption for this guide:* You have managed to get a root shell via Console/Recovery mode and have set a new root password.

### Phase 3: Enabling Password SSH (Temporary Bridge)
By default, Ubuntu disables password authentication for SSH (`PasswordAuthentication no`) and root login (`PermitRootLogin prohibit-password`). We must enable them temporarily to get back in via SSH.

1.  **Edit SSH Config:**
    ```bash
    nano /etc/ssh/sshd_config
    ```
2.  **Modify Directives:**
    Find and change these lines:
    ```text
    PermitRootLogin yes
    PasswordAuthentication yes
    ```
    *Crucial Check:* Look for files in `/etc/ssh/sshd_config.d/`. If `50-cloud-init.conf` exists, it often overrides the main config. Edit that file too if present [[25]].
    ```bash
    nano /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```
    Ensure it doesn't force `PasswordAuthentication no`.

3.  **Handle AuthenticationMethods:**
    If you see `AuthenticationMethods publickey,password`, remove it or comment it out. We just want simple password auth for now.

4.  **Restart SSH:**
    ```bash
    systemctl restart sshd
    ```

5.  **Verify Firewall:**
    Ensure port 22 is open in the OpenStack Security Group associated with the instance.

### Phase 4: Regaining Access & Injecting New Key
Now, from your local machine:
```bash
ssh root@<Instance-IP>
# Enter the new password you set.
```
**Success!** You are back in.

**Immediate Action: Restore Key-Based Auth**
Password auth is risky. Let’s fix it permanently.

1.  **Generate a New Key Pair (Local Machine):**
    ```bash
    ssh-keygen -t ed25519 -f ~/openstack-recovery-key -C "recovery-2026"
    ```

2.  **Inject the Public Key:**
    Copy the content of `~/openstack-recovery-key.pub`.
    On the instance (as root):
    ```bash
    mkdir -p /root/.ssh
    echo "<PASTE_PUBLIC_KEY_HERE>" >> /root/.ssh/authorized_keys
    chmod 700 /root/.ssh
    chmod 600 /root/.ssh/authorized_keys
    ```
    Also add it to the `ubuntu` user if needed:
    ```bash
    su - ubuntu
    mkdir -p ~/.ssh
    echo "<PASTE_PUBLIC_KEY_HERE>" >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    exit
    ```

3.  **Test New Key:**
    Logout and try connecting with the new key:
    ```bash
    ssh -i ~/openstack-recovery-key root@<Instance-IP>
    ```

### Phase 5: Locking Down (Hardening)
Once confirmed working:
1.  Edit `/etc/ssh/sshd_config` again.
2.  Set `PasswordAuthentication no`.
3.  Set `PermitRootLogin prohibit-password` (or `no` if using sudo).
4.  Restart SSH: `systemctl restart sshd`.

You have successfully recovered from a lost key scenario without rebuilding the instance.

---

## 5. Operational Excellence: Maintenance, Security & Lifecycle

### Daily Operations: The Health Check
A professional engineer doesn’t wait for things to break.

**Routine SSH Connectivity Check:**
Create a mental (or scripted) routine to verify your cluster fabric.
```bash
# Quick one-liner to check all nodes
for h in ceph1 ceph2 ceph3; do 
  echo -n "$h: "; 
  ssh -o BatchMode=yes -o ConnectTimeout=5 $h "echo UP" || echo "DOWN"; 
done
```
*   `BatchMode=yes`: Ensures the command fails immediately if it asks for a password (indicating a key issue).
*   `ConnectTimeout=5`: Prevents hanging if a node is unreachable.

### Maintenance: Key Rotation
Keys should be rotated every 6-12 months or when an administrator leaves the team.

**Rotation Procedure:**
1.  **Generate New Keys:** Create a new key pair on the admin station.
2.  **Dual Deployment:** Add the *new* public key to `authorized_keys` on all nodes *alongside* the old one. Do not remove the old one yet.
3.  **Test:** Verify you can log in with the new key.
4.  **Cleanup:** Once verified, remove the old public key line from `authorized_keys` on all nodes.
5.  **Revoke:** Securely delete the old private key from your backup stores.

### Security Hardening Checklist
Based on modern standards (OpenSSH 9.x) [[3]]:

*   **Disable Root Login:** Set `PermitRootLogin no` and use `sudo`.
*   **Disable Password Auth:** Set `PasswordAuthentication no` once keys are verified.
*   **Use Strong Ciphers:** OpenSSH 9.x disables weak algorithms by default. Do not re-enable `rsa-sha1` or `diffie-hellman-group1-sha1` unless absolutely necessary for legacy hardware.
*   **Fail2Ban:** Install `fail2ban` to ban IPs that show brute-force behavior in `auth.log`.
    ```bash
    sudo apt install fail2ban
    sudo systemctl enable fail2ban
    ```
*   **Audit Logs:** Regularly check `/var/log/auth.log` for unusual login patterns.

### Managing Configuration Drift
Over time, manual edits can cause configuration drift (e.g., one node has `AllowUsers`, another doesn’t).

**Strategy:**
*   Use configuration management tools (Ansible, Puppet) to enforce SSH configs.
*   Even without tools, keep a "Golden Config" file in a Git repository.
*   Periodically diff your live config against the golden standard:
    ```bash
    ssh ceph2 "sudo cat /etc/ssh/sshd_config" > live-ceph2.conf
    diff golden-sshd_config.conf live-ceph2.conf
    ```

### Disaster Recovery Planning
*   **Backup Keys:** Store encrypted backups of your private keys in a password manager or a secure offline vault. Losing your admin key means losing the cluster.
*   **Console Access:** Always ensure you have Out-of-Band (OOB) access (IPMI, iDRAC, or OpenStack VNC) before making SSH changes. If you lock yourself out remotely, OOB is your only lifeline.

---

## Appendix: Quick Reference Commands

| Task | Command | Notes |
| :--- | :--- | :--- |
| **Check SSH Version** | `ssh -V` | Verify OpenSSH 9.x+ |
| **Generate Ed25519 Key** | `ssh-keygen -t ed25519 -C "comment"` | Best practice for 2026 |
| **Copy Key to Remote** | `ssh-copy-id -i key.pub user@host` | Automates permission setting |
| **Test Connection (No Prompt)** | `ssh -o BatchMode=yes user@host` | Returns error if key fails |
| **Debug Connection** | `ssh -vvv user@host` | Verbose output for troubleshooting |
| **Check Effective SSH Config** | `sshd -T | grep <option>` | Shows what the server is actually using |
| **Restart SSH Service** | `sudo systemctl restart sshd` | Apply config changes |
| **Test Config Syntax** | `sudo sshd -t` | Run before restarting to prevent lockout |
| **View Auth Logs** | `sudo tail -f /var/log/auth.log` | Real-time failure monitoring |
| **Fix Permissions** | `chmod 700 ~/.ssh && chmod 600 ~/.ssh/*` | The "Magic Fix" for many errors |

---

### Final Thoughts from the Field
SSH is simple until it isn’t. The difference between a junior and a senior engineer is not knowing the commands, but understanding **where to look** when things go wrong (logs, drop-in configs, permissions) and having a **recovery plan** (console access, backup keys). Keep your keys safe, your configs version-controlled, and always test your recovery procedures before you actually need them.

*End of Guide.*