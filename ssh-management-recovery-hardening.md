# Ultimate Practical Guide: SSH Management, Recovery & Hardening
### For Ceph Clusters and OpenStack Cloud Infrastructure

**Version:** 2.0 (2026 Edition)  
**Author:** Sumon, IT Infrastructure & DevOps Specialist  
**Target Audience:** System Administrators, DevOps Engineers, Cloud Architects  
**Language:** Simple English (Professional Technical Narrative)  

---

## Table of Contents

1. [SSH Fundamentals for Modern Infrastructure](#1-ssh-fundamentals-for-modern-infrastructure)
2. [Building a Secure 3-Node Ceph Cluster with Passwordless SSH](#2-building-a-secure-3-node-ceph-cluster-with-passwordless-ssh)
3. [Advanced Troubleshooting: Solving Real-World SSH Connection Failures](#3-advanced-troubleshooting-solving-real-world-ssh-connection-failures)
4. [OpenStack Instance Recovery: Regaining Access When Keys Are Lost](#4-openstack-instance-recovery-regaining-access-when-keys-are-lost)
5. [Security Hardening and Operational Best Practices](#5-security-hardening-and-operational-best-practices)
6. [Daily Operations, Maintenance, and Key Rotation](#6-daily-operations-maintenance-and-key-rotation)

---

## 1. SSH Fundamentals for Modern Infrastructure

### Story from the Field
Imagine you are managing a data center in Dhaka with servers located in a remote facility in Chittagong. Ten years ago, administrators used tools like Telnet. If you typed your password, anyone listening on the network could steal it instantly. Today, we use **SSH (Secure Shell)**. It creates an encrypted tunnel where even if someone captures your data packets, they only see gibberish. In modern DevOps, SSH is not just for logging in; it is the backbone of automation tools like Ansible, Terraform, and CI/CD pipelines. If SSH fails, your entire automation chain breaks.

### Requirements
Before starting any SSH operation, ensure you have the following:
*   **Operating System:** Ubuntu 24.04 LTS (or latest stable RHEL/CentOS).
*   **Network Connectivity:** Static IP addresses for all nodes. Firewalls must allow TCP Port 22.
*   **User Privileges:** Root access or a user with `sudo` privileges.
*   **Official Documentation Reference:** Always verify OpenSSH versions against the [OpenBSD Official Site](https://www.openssh.com/) or your OS vendor's security advisories.

### How to Verify and Install Latest SSH
Do not assume SSH is installed or up-to-date. Follow this practical workflow:

**Step 1: Check Current Version**
Run the following command to see your installed version:
```bash
ssh -V
```
*Output Example:* `OpenSSH_9.6p1 Ubuntu-3ubuntu13.1, OpenSSL 3.0.13 30 Jan 2024`

**Step 2: Verify Against Official Sources**
Visit the official Ubuntu Security Notices or OpenSSH website. If your version is significantly old (e.g., below 8.0), plan an update. Never install SSH from random third-party repositories; always use the official OS package manager to ensure stability and security patches.

**Step 3: Install or Update**
On Ubuntu/Debian systems:
```bash
sudo apt update
sudo apt install openssh-server openssh-client -y
```
On RHEL/CentOS/Rocky Linux:
```bash
sudo dnf install openssh-server openssh-clients -y
```

**Step 4: Ensure Service is Running**
```bash
sudo systemctl status sshd
```
If it is not active, enable and start it:
```bash
sudo systemctl enable --now sshd
```

### Verification
Confirm the server is listening on the correct port:
```bash
sudo ss -tlnp | grep :22
```
You should see `LISTEN` on `0.0.0.0:22` or your specific IP.

### Operations Guide: Basic Usage
Once installed, test local connectivity:
```bash
ssh localhost
```
If this works, your SSH daemon is functional. For remote connections, the standard format is:
```bash
ssh username@ip-address
```

### Management and Maintenance
*   **Log Monitoring:** Regularly check `/var/log/auth.log` (Ubuntu) or `/var/log/secure` (RHEL) for failed login attempts.
*   **Configuration Backups:** Before making any changes to `/etc/ssh/sshd_config`, always create a backup:
    ```bash
    sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)
    ```
*   **Syntax Checking:** After editing config files, always test the syntax before restarting the service to avoid locking yourself out:
    ```bash
    sudo sshd -t
    ```
    If no output appears, the syntax is correct.

---

## 2. Building a Secure 3-Node Ceph Cluster with Passwordless SSH

### Story from the Field
We were deploying a 3-node Ceph storage cluster. The automation script (Ansible) needed to push configurations to all three nodes simultaneously. However, the script kept failing because it was waiting for a password prompt on every single command. Manually typing passwords for hundreds of commands is impossible. We needed **Passwordless SSH**. Furthermore, during setup, one node (`ceph2`) kept rejecting connections despite having the correct key. The issue? A hidden configuration rule called `AllowUsers` that was blocking our new admin user. This guide walks you through setting this up correctly from scratch, including how to clean up old messes.

### Requirements
*   **Nodes:** 3 Servers (e.g., `ceph1` as Admin, `ceph2` & `ceph3` as Storage).
*   **OS:** Ubuntu 24.04 LTS on all nodes.
*   **Network:** All nodes must be able to ping each other by IP.
*   **User Strategy:** Create a dedicated user (e.g., `cephuser`) on all nodes. Do not use `root` directly for daily tasks.

### How to Setup Passwordless SSH (Step-by-Step)

#### Phase 1: Complete Cleanup (Start Fresh)
If these servers were used before, old keys and configs will cause conflicts.
**Action on ALL Nodes (ceph1, ceph2, ceph3):**
1.  Login via console or existing SSH.
2.  Remove old SSH configurations for your user:
    ```bash
    rm -rf ~/.ssh
    ```
3.  Verify removal:
    ```bash
    ls -la ~/.ssh
    # Should return "No such file or directory"
    ```

#### Phase 2: User Creation and Privileges
**Action on ALL Nodes:**
1.  Create the dedicated user:
    ```bash
    sudo adduser cephuser
    ```
    (Follow prompts to set a temporary strong password).
2.  Grant sudo privileges:
    ```bash
    sudo usermod -aG sudo cephuser
    ```
3.  **Critical for Automation:** Configure passwordless sudo. Edit the sudoers file safely:
    ```bash
    sudo visudo
    ```
    Add this line at the bottom:
    ```text
    cephuser ALL=(ALL) NOPASSWD:ALL
    ```
    Save and exit. This allows automation tools to run admin commands without stopping for a password.

#### Phase 3: Generate SSH Keys (Admin Node Only)
**Action on `ceph1` (Admin Node) ONLY:**
1.  Switch to the new user:
    ```bash
    su - cephuser
    ```
2.  Generate a modern Ed25519 key (faster and more secure than RSA):
    ```bash
    ssh-keygen -t ed25519 -C "ceph-cluster-admin-key"
    ```
    *   **Prompt:** "Enter file in which to save the key..." -> Press **Enter** (accept default).
    *   **Prompt:** "Enter passphrase..." -> Press **Enter** twice (leave empty for automation).
    *   *Note:* In high-security production environments, you might use a passphrase and an SSH agent, but for cluster automation, an empty passphrase is standard practice provided the private key is kept secret.

#### Phase 4: Distribute Keys to All Nodes
**Action on `ceph1` (Admin Node) ONLY:**
Use the `ssh-copy-id` tool to send your public key to all nodes (including itself).

1.  Copy to itself:
    ```bash
    ssh-copy-id cephuser@<IP-of-ceph1>
    ```
2.  Copy to node 2:
    ```bash
    ssh-copy-id cephuser@<IP-of-ceph2>
    ```
3.  Copy to node 3:
    ```bash
    ssh-copy-id cephuser@<IP-of-ceph3>
    ```
    *   **Process:** You will be asked for the `cephuser` password one time for each node. After successful entry, the key is installed.

#### Phase 5: Configure SSH Client for Ease of Use
**Action on `ceph1` (Admin Node) ONLY:**
Create a config file so you don't have to type IPs every time.

1.  Create/Edit the config file:
    ```bash
    nano ~/.ssh/config
    ```
2.  Add the following content:
    ```text
    Host ceph1
        HostName <IP-of-ceph1>
        User cephuser
        IdentityFile ~/.ssh/id_ed25519
        StrictHostKeyChecking no

    Host ceph2
        HostName <IP-of-ceph2>
        User cephuser
        IdentityFile ~/.ssh/id_ed25519
        StrictHostKeyChecking no

    Host ceph3
        HostName <IP-of-ceph3>
        User cephuser
        IdentityFile ~/.ssh/id_ed25519
        StrictHostKeyChecking no
    ```
3.  Secure the config file:
    ```bash
    chmod 600 ~/.ssh/config
    ```

### Verification
Test the setup from `ceph1`:
```bash
ssh ceph1 hostname
ssh ceph2 hostname
ssh ceph3 hostname
```
**Success Criteria:** The command should return the hostname immediately without asking for a password.

### Operations Guide: Using the Cluster
Now you can run commands remotely instantly:
```bash
# Check disk space on all nodes
ssh ceph2 df -h

# Restart a service on node 3
ssh ceph3 sudo systemctl restart ceph-osd.target
```

### Management and Maintenance
*   **Adding New Nodes:** When adding `ceph4`, simply generate keys on `ceph1` (if not already done) and run `ssh-copy-id cephuser@ceph4`. Then add an entry to `~/.ssh/config`.
*   **Revoking Access:** If an admin leaves, remove their public key from the `~/.ssh/authorized_keys` file on **all** nodes. Do not just remove it from one node.
*   **Permission Checks:** If SSH suddenly stops working, 90% of the time it is a permission issue. Ensure `~/.ssh` is `700` and `authorized_keys` is `600`.

---

## 3. Advanced Troubleshooting: Solving Real-World SSH Connection Failures

### Story from the Field
During the Ceph setup mentioned above, `ssh-copy-id` to `ceph2` kept failing with "Permission denied," even though the password was correct. We checked logs (`/var/log/auth.log`) and found a cryptic message: `User cephuser from ... not allowed because not listed in AllowUsers`. It turned out the cloud image used to create the VM had a security restriction allowing only the default `ubuntu` user. We had to hunt down this setting in a secondary config folder (`/etc/ssh/sshd_config.d/`) which overrides the main config. This section teaches you how to diagnose and fix such hidden issues.

### Requirements
*   **Access:** Console access (VNC/noVNC) to the server is mandatory. If SSH is broken, you cannot fix it via SSH.
*   **Tools:** `grep`, `tail`, `journalctl`, `sshd` binary.
*   **Mindset:** Assume nothing. Check logs first.

### How to Diagnose and Fix Common Issues

#### Scenario A: "Permission denied (publickey,password)"
**Symptoms:** You type the correct password, but it rejects you. Or you have a key, but it says publickey failed.

**Step 1: Check the Logs (The Golden Rule)**
On the **Server** (target machine), open a terminal (via Console) and watch the logs in real-time:
```bash
sudo tail -f /var/log/auth.log
```
Now try to connect from your client. Watch the output.
*   If it says `Failed password`, your password is wrong or keyboard layout is incorrect.
*   If it says `Connection closed by ... [preauth]`, the server rejected you before you could even try.

**Step 2: Check for "AllowUsers" Restrictions**
This is a common hidden trap.
1.  Check the main config:
    ```bash
    sudo grep -i "allowusers" /etc/ssh/sshd_config
    ```
2.  **Crucial Step:** Check the override directory (common in Ubuntu 22.04/24.04):
    ```bash
    ls /etc/ssh/sshd_config.d/
    sudo cat /etc/ssh/sshd_config.d/*.conf
    ```
    Look for lines like `AllowUsers ubuntu` or `AllowUsers root`.
3.  **Fix:** Edit the file containing the restriction and add your user or comment out the line:
    ```bash
    sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
    # Change: AllowUsers ubuntu
    # To:     AllowUsers ubuntu cephuser
    # Or comment it: # AllowUsers ubuntu
    ```
4.  Restart SSH:
    ```bash
    sudo systemctl restart sshd
    ```

#### Scenario B: "Authentication Methods: Publickey" (Password Not Accepted)
**Symptoms:** The server says `server sent: publickey` and never offers password login.

**Step 1: Verify Active Configuration**
Run this command on the server to see what SSH is *actually* using (ignoring comments):
```bash
sudo sshd -T | grep passwordauthentication
```
If it returns `no`, password login is disabled.

**Step 2: Find the Culprit**
It might be disabled in `/etc/ssh/sshd_config` OR in `/etc/ssh/sshd_config.d/`.
1.  Check main config: `grep PasswordAuthentication /etc/ssh/sshd_config`
2.  Check override configs: `grep -r PasswordAuthentication /etc/ssh/sshd_config.d/`
3.  **Fix:** Set it to `yes` in the file where it is defined as `no`.
    ```text
    PasswordAuthentication yes
    ```
4.  **Important:** Check for `AuthenticationMethods`. If you see `AuthenticationMethods publickey,password`, it means you need BOTH a key AND a password. For simple password login, remove or comment out this line.
5.  Test config and restart:
    ```bash
    sudo sshd -t
    sudo systemctl restart sshd
    ```

#### Scenario C: "Connection Refused"
**Symptoms:** Immediate failure, no password prompt.

**Step 1: Check Service Status**
```bash
sudo systemctl status sshd
```
If inactive, start it.

**Step 2: Check Firewall**
*   **UFW (Ubuntu):** `sudo ufw status`. Ensure port 22 is allowed.
*   **Firewalld (RHEL):** `sudo firewall-cmd --list-all`.
*   **Cloud Security Groups:** If on OpenStack/AWS, check the dashboard. The Security Group MUST have an Inbound Rule for TCP Port 22 from your IP.

### Verification
After applying fixes:
1.  Run `sudo sshd -T | grep -E "password|permitroot"` to confirm settings are `yes`.
2.  Try connecting from a new terminal window (do not reuse an old session).
3.  Use verbose mode for final confirmation:
    ```bash
    ssh -v cephuser@<IP>
    ```
    Look for `debug1: Authentication succeeded`.

### Management and Maintenance
*   **Log Rotation:** Ensure log files do not fill up the disk. Standard Linux logrotate handles this, but verify `/etc/logrotate.d/rsyslog` or similar.
*   **Fail2Ban:** Install `fail2ban` to automatically ban IPs that try too many wrong passwords. This is essential for any server exposed to the internet.
    ```bash
    sudo apt install fail2ban
    sudo systemctl enable --now fail2ban
    ```

---

## 4. OpenStack Instance Recovery: Regaining Access When Keys Are Lost

### Story from the Field
A junior engineer deleted his local laptop containing the only private key for a critical OpenStack instance. Panic ensued. The instance was running a vital database. He couldn't SSH in. The "Reset Password" button in some panels doesn't work if the OS isn't configured for it. The solution? Use the **OpenStack Console (noVNC)**. It provides direct screen access to the VM, bypassing SSH entirely. We booted into the console, reset the root password, enabled password authentication temporarily, recovered access, injected a new key, and then re-secured the server. This is your disaster recovery playbook.

### Requirements
*   **OpenStack Dashboard Access:** You must have login credentials for the Horizon dashboard.
*   **Console Access:** The "Console" tab must be functional in the dashboard.
*   **Physical/VNC Access:** Ability to type commands directly into the VM screen.

### How to Recover Access (Step-by-Step)

#### Phase 1: Gain Console Access and Reset Root Password
1.  Log in to OpenStack Dashboard.
2.  Go to **Project** -> **Compute** -> **Instances**.
3.  Click on the locked instance name.
4.  Click the **Console** tab. You will see the VM's screen.
5.  **Login:** If it asks for login, try the default user (e.g., `ubuntu`). If you don't know the password, you may need to reboot into single-user mode (advanced) or hope cloud-init set a known user.
    *   *Assumption:* Let's assume you can log in as `ubuntu` with sudo, or you have reset the root password via GRUB (if console login is also locked).
    *   *Simpler Path:* If you can log in as a sudo user:
        ```bash
        sudo -i
        passwd root
        ```
        Set a strong temporary password.

#### Phase 2: Enable Password Authentication in SSH
By default, Ubuntu disables root login and password auth. We must enable them temporarily.

1.  Edit the SSH config:
    ```bash
    nano /etc/ssh/sshd_config
    ```
2.  Find and change these lines:
    ```text
    PermitRootLogin yes
    PasswordAuthentication yes
    ```
3.  **CRITICAL CHECK:** Check the override directory again!
    ```bash
    ls /etc/ssh/sshd_config.d/
    nano /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```
    If `PasswordAuthentication no` exists here, change it to `yes`. This file often overrides the main config.
4.  Check for `AuthenticationMethods`. If present, comment it out to avoid conflicts.
5.  Validate and Restart:
    ```bash
    sshd -t
    systemctl restart sshd
    ```

#### Phase 3: Connect and Inject New Key
1.  From your local machine, connect using the new root password:
    ```bash
    ssh root@<Instance-IP>
    ```
    (Enter the temporary password you set).
2.  **Generate a New Key Pair Locally:**
    On your laptop (not the server):
    ```bash
    ssh-keygen -t ed25519 -f ~/recovery-key-2024
    ```
3.  **Copy the Public Key to Server:**
    Display your new public key on your laptop:
    ```bash
    cat ~/recovery-key-2024.pub
    ```
    Copy the entire output.
4.  **Paste into Server:**
    Back in your SSH session to the server:
    ```bash
    mkdir -p /root/.ssh
    nano /root/.ssh/authorized_keys
    ```
    Paste the key. Save and exit.
    Fix permissions:
    ```bash
    chmod 700 /root/.ssh
    chmod 600 /root/.ssh/authorized_keys
    ```

#### Phase 4: Re-Secure the Server
Never leave password authentication enabled permanently.

1.  Test the new key first! Open a **new** terminal window on your laptop:
    ```bash
    ssh -i ~/recovery-key-2024 root@<Instance-IP>
    ```
    Ensure it logs in without a password.
2.  Once confirmed, go back to the server config:
    ```bash
    nano /etc/ssh/sshd_config
    # AND
    nano /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```
    Set:
    ```text
    PermitRootLogin prohibit-password
    PasswordAuthentication no
    ```
3.  Restart SSH:
    ```bash
    systemctl restart sshd
    ```

### Verification
Try to connect with the password. It should now fail. Try with the key. It should succeed.
```bash
ssh root@<IP> 
# Should say "Permission denied (publickey)" - This is GOOD!

ssh -i ~/recovery-key-2024 root@<IP>
# Should login successfully - This is GOOD!
```

### Management and Maintenance
*   **Update OpenStack Keypair:** Go to the OpenStack Dashboard -> Compute -> Key Pairs. Import your new public key (`recovery-key-2024.pub`) so future instances can use it automatically.
*   **Document the Incident:** Record why the key was lost and store the new private key in a secure password manager or encrypted vault.
*   **Snapshot:** Before making major recovery changes in the future, take an OpenStack Snapshot of the instance.

---

## 5. Security Hardening and Operational Best Practices

### Story from the Field
We once audited a server that had been online for 3 years. It allowed root login with a password. Our automated scanner tried 10,000 common passwords and guessed "Admin123!" within minutes. The server was compromised. We immediately migrated it to Key-Only authentication, changed the SSH port, and installed Fail2Ban. This section details how to lock down your servers so that even if someone finds your IP, they cannot break in.

### Requirements
*   **Working SSH Key Access:** Ensure you have a working key before hardening, or you will lock yourself out.
*   **Console Access:** Keep the OpenStack/VNC console open as a backup while applying changes.

### How to Harden SSH Configuration

#### Step 1: Disable Root Login and Passwords
Edit `/etc/ssh/sshd_config` (and check `.d/` overrides):

```text
# Disable direct root login (force users to sudo)
PermitRootLogin prohibit-password

# Disable password authentication entirely
PasswordAuthentication no

# Enable public key authentication
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no
```

#### Step 2: Change Default Port (Optional but Recommended)
Script kiddies scan port 22 constantly. Moving to a non-standard port reduces noise in logs.
```text
Port 2244
```
*Note: Remember to update your Firewall and Security Groups to allow this new port.*

#### Step 3: Limit Users
Only allow specific users to login via SSH.
```text
AllowUsers cephuser adminuser
```
This blocks all other users, even if they have a valid key.

#### Step 4: Set Timeouts
Disconnect idle sessions to free resources and reduce hijacking risk.
```text
ClientAliveInterval 300
ClientAliveCountMax 2
```

#### Step 5: Apply and Test
1.  Check syntax: `sudo sshd -t`
2.  Restart: `sudo systemctl restart sshd`
3.  **IMMEDIATE TEST:** Do not close your current terminal. Open a **new** terminal and try to connect. If it works, you are safe. If not, use the Console to fix it.

### Verification
Run a security scan using `nmap` from an external machine:
```bash
nmap -p 22,2244 <Server-IP>
```
Check if only the intended port is open. Attempt a password login; it must fail immediately.

### Management and Maintenance
*   **Key Rotation Policy:** Plan to rotate SSH keys every 6-12 months. Generate new keys, distribute them, and remove old ones from `authorized_keys`.
*   **Audit Logs:** Weekly review of `/var/log/auth.log`. Look for "Invalid user" or "Failed password" spikes.
*   **Software Updates:** Regularly run `sudo apt update && sudo apt upgrade` to patch SSH vulnerabilities.

---

## 6. Daily Operations, Maintenance, and Key Rotation

### Story from the Field
In a large cluster, managing keys manually becomes a nightmare. "Who has access?" "Did we remove the intern's key?" "Is this key expired?" We implemented a strict operational routine: a quarterly key rotation day, centralized logging, and a "break-glass" procedure for emergencies. This section outlines the daily and monthly habits of a professional SysAdmin.

### Requirements
*   **Documentation:** A secure record of who holds which keys.
*   **Automation Tools:** Ansible or similar for mass key deployment (optional but recommended).
*   **Backup Storage:** Encrypted storage for private keys.

### How to Perform Routine Operations

#### Daily Operations
1.  **Check Service Health:**
    ```bash
    systemctl is-active sshd
    ```
2.  **Monitor Logs:**
    Quickly scan for brute force attempts:
    ```bash
    sudo grep "Failed password" /var/log/auth.log | tail -10
    ```
    If you see hundreds of attempts from one IP, consider blocking it manually or checking Fail2Ban.

#### Monthly Maintenance
1.  **Verify User Access:**
    Review `/home/<user>/.ssh/authorized_keys` on critical servers. Remove keys belonging to employees who have left the organization.
2.  **Update SSH Configs:**
    Ensure all servers have consistent hardening settings. Use a configuration management tool to drift-check.
3.  **Backup Keys:**
    Ensure your own administrative private keys are backed up to an encrypted USB drive or a secure password manager.

#### Key Rotation Procedure (Quarterly)
**Scenario:** Rotating keys for the `cephuser`.

1.  **Generate New Keys (Admin Laptop):**
    ```bash
    ssh-keygen -t ed25519 -f ~/ceph-key-2026-Q1
    ```
2.  **Deploy New Keys:**
    Use `ssh-copy-id` or Ansible to push the *new* public key to all servers.
    ```bash
    ssh-copy-id -i ~/ceph-key-2026-Q1.pub cephuser@ceph1
    # Repeat for all nodes
    ```
3.  **Test New Keys:**
    Connect using the new key explicitly:
    ```bash
    ssh -i ~/ceph-key-2026-Q1 cephuser@ceph1
    ```
    Ensure it works perfectly.
4.  **Remove Old Keys:**
    Login to each server and edit `~/.ssh/authorized_keys`. Delete the line containing the OLD key comment (e.g., `...-2025-Q4`).
5.  **Revoke Old Private Keys:**
    Securely delete the old private key from your laptop and notify all admins to do the same.

### Verification
After rotation, attempt to connect with the **old** key. It must fail.
```bash
ssh -i ~/old-key cephuser@ceph1
# Expected: Permission denied (publickey)
```

### Management and Maintenance
*   **Emergency Access:** Maintain a "Break-Glass" key pair stored in a physical safe or a highly restricted digital vault. This key is only used when normal access methods fail.
*   **Training:** Ensure all team members know how to generate keys and use `ssh-copy-id`. Avoid sharing private keys via email or chat (a common bad practice).
*   **Compliance:** If you are in a regulated industry (Banking, Health), keep logs of key rotation dates and personnel changes for auditors.

---

**End of Document**
*This guide represents current best practices as of 2026. Always refer to the latest official documentation for your specific OS version.*
