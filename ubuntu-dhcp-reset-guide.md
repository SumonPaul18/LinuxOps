YES.

### File Name
`ubuntu-dhcp-reset-guide.md`

---

# Ubuntu Server 24.04: DHCP Release & Renew Practical Guide

## 1. Introduction & Problem Analysis
In **Ubuntu Server 24.04 Minimal**, the traditional `dhclient` command is often not installed by default because modern Ubuntu versions use **Netplan** (with `systemd-networkd` or `NetworkManager`) as the primary network configuration tool.

When you run `sudo dhclient -r enp6s18` and get `command not found`, it means the ISC DHCP client package is missing. However, you do not necessarily need to install it. You can achieve the same result (releasing and renewing the IP) using native `ip` commands or by restarting the Netplan service.

This guide provides **three practical methods** to reset your DHCP lease, ranked from "Modern Standard" to "Legacy Method."

---

## 2. Pre-Requisites: Identify Your Interface
Before running any reset commands, confirm your interface name and current IP status.

```bash
# Check interface name and IP address
ip addr show

# Check if the interface is managed by NetworkManager or systemd-networkd
networkctl status
```
*Look for your interface (e.g., `enp6s18`). Note its current state.*

---

## 3. Method 1: The Modern Standard (Using Netplan)
**Best for:** Production servers, Cloud instances, and standard Ubuntu 24.04 setups.
Since Ubuntu 24.04 uses Netplan, the cleanest way to release and renew an IP is to apply the Netplan configuration again. This forces the backend (`systemd-networkd`) to restart the connection.

### Step 1: Apply Netplan Configuration
This command effectively restarts the network interface defined in your YAML files.

```bash
sudo netplan apply
```

### Step 2: Verify the New IP
Check if the interface has received a fresh IP address from the DHCP server.

```bash
ip addr show enp6s18
```

> **Note:** If `netplan apply` does not seem to refresh the lease immediately, proceed to **Method 2** to manually flush the IP before applying.

---

## 4. Method 2: Manual Flush & Renew (Using `ip` and `systemd`)
**Best for:** Immediate release without restarting the whole network stack.
This method manually removes the current IP (Release) and triggers the DHCP discovery process (Renew).

### Scenario A: If using `systemd-networkd` (Default for Server)

#### Step 1: Release (Flush) the Current IP
Manually remove the IP address assigned to the interface. This acts as a "Release."

```bash
# Replace 'enp6s18' with your actual interface name
sudo ip addr flush dev enp6s18
```

#### Step 2: Renew (Trigger DHCP Discovery)
Restart the specific network service for that interface to request a new IP.

```bash
# Restart the systemd-networkd service
sudo systemctl restart systemd-networkd

# OR, restart only the specific link if supported
sudo networkctl reload
```

#### Step 3: Verification
Wait 5-10 seconds and check the new IP.

```bash
ip addr show enp6s18
journalctl -u systemd-networkd -n 20 --no-pager
```

### Scenario B: If using `NetworkManager`
If your minimal server has NetworkManager installed:

```bash
# Down the interface (Release)
sudo nmcli connection down "Your-Connection-Name"

# Up the interface (Renew)
sudo nmcli connection up "Your-Connection-Name"
```
*(Find connection name using `nmcli connection show`)*

---

## 5. Method 3: Installing Legacy `dhclient` (Optional)
**Best for:** Specific troubleshooting where you strictly require the `dhclient` tool behavior.
If you specifically need the `dhclient` command for scripts or habit, you must install the `isc-dhcp-client` package.

### Step 1: Update Package List
Ensure you have internet access (even via static IP or current DHCP) to download the package.

```bash
sudo apt update
```

### Step 2: Install ISC DHCP Client
```bash
sudo apt install isc-dhcp-client -y
```

### Step 3: Execute Release and Renew
Now the commands you originally tried will work.

```bash
# 1. Release the IP
sudo dhclient -r enp6s18

# 2. Verify IP is gone
ip addr show enp6s18

# 3. Renew the IP
sudo dhclient -v enp6s18
```
*The `-v` flag shows verbose output so you can see the DHCP handshake (DISCOVER, OFFER, REQUEST, ACK).*

---

## 6. Real-World Scenarios & Troubleshooting

### Scenario A: VM Cloning / Machine ID Change Context
*Context:* You recently changed the `/etc/machine-id` on a cloned VM (as discussed in previous steps) and the DHCP server is still assigning the old IP because it recognizes the old Client ID.

**Solution:**
1. Flush the IP: `sudo ip addr flush dev enp6s18`
2. Remove old DHCP lease files (Critical step):
   ```bash
   sudo rm /var/lib/systemd/network/*.lease
   # Or if using dhclient
   sudo rm /var/lib/dhcp/dhclient.enp6s18.leases
   ```
3. Restart network: `sudo netplan apply` or `sudo systemctl restart systemd-networkd`
4. The DHCP server will now treat this as a completely new client and assign a new IP.

### Scenario B: Stuck in "Configuring" State
If the interface hangs while trying to get an IP:
```bash
# Check detailed logs
journalctl -u systemd-networkd -f

# Forcefully bring interface down and up
sudo ip link set enp6s18 down
sleep 2
sudo ip link set enp6s18 up
sudo systemctl restart systemd-networkd
```

---

## 7. Official References & Further Reading

| Topic | Official Documentation Link |
| :--- | :--- |
| **Netplan Configuration** | [Ubuntu Netplan Docs](https://netplan.io/) |
| **systemd-networkd** | [Freedesktop systemd-networkd](https://www.freedesktop.org/software/systemd/man/latest/systemd-networkd.service.html) |
| **ISC DHCP Client** | [Debian isc-dhcp-client Package](https://packages.debian.org/search?keywords=isc-dhcp-client) |
| **IP Command Manual** | [Linux ip-route man page](https://man7.org/linux/man-pages/man8/ip.8.html) |

---

## 8. Summary Checklist

| Action | Command (Modern) | Command (Legacy) |
| :--- | :--- | :--- |
| **Release IP** | `sudo ip addr flush dev <iface>` | `sudo dhclient -r <iface>` |
| **Renew IP** | `sudo netplan apply` | `sudo dhclient <iface>` |
| **Check Status** | `ip addr show <iface>` | `ip addr show <iface>` |
| **View Logs** | `journalctl -u systemd-networkd` | `tail -f /var/log/syslog` |

**Recommendation:** For Ubuntu 24.04 Server, stick to **Method 1 (Netplan)** or **Method 2 (IP Flush + Systemd Restart)**. Only install `isc-dhcp-client` if you have a specific legacy requirement.