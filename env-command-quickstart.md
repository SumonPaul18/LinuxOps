# kolla-env-guide.md

# Professional Guide: Efficient Kolla OpenStack Environment Activation
### Streamlining `venv` and `admin-openrc` Workflows for DevOps Engineers

---

## 📑 Table of Contents
1. [Scenario Overview](#1-scenario-overview)
2. [Method 1: The Bash Function (Recommended)](#2-method-1-the-bash-function-recommended)
3. [Method 2: The Quick Alias](#3-method-2-the-quick-alias)
4. [Method 3: The Portable Script](#4-method-3-the-portable-script)
5. [Method 4: Automatic Session Loading](#5-method-4-automatic-session-loading)
6. [Verification & Troubleshooting](#6-verification--troubleshooting)

---

## 1. Scenario Overview
**The Problem:** Every time you log into your Kolla controller node, you must manually type two long commands to activate the Python virtual environment and load OpenStack credentials:
```bash
source /opt/venv-kolla/bin/activate
source /etc/kolla/admin-openrc.sh
```
**The Goal:** Create a "one-click" solution to load these environments instantly when needed, reducing typing errors and saving time during incident response or deployment tasks.

> **Note:** Since `source` modifies the *current* shell session, these solutions must be executed using `source` (or `.`) or defined as shell functions/aliases. Running them as standard executable scripts (`./script.sh`) will not work as the environment variables will vanish once the script ends.

---

## 2. Method 1: The Bash Function (Recommended)
**Best For:** Daily operations, clarity, and adding custom logic (like error checking). This is the most professional approach for a System Administrator.

### Step-by-Step Implementation

1.  **Open your shell configuration file:**
    ```bash
    nano ~/.bashrc
    ```

2.  **Add the following function at the end of the file:**
    ```bash
    # Function to load Kolla OpenStack Environment
    kolla-env() {
        if [ -f /opt/venv-kolla/bin/activate ]; then
            source /opt/venv-kolla/bin/activate
        else
            echo "Error: Kolla venv not found!"
            return 1
        fi

        if [ -f /etc/kolla/admin-openrc.sh ]; then
            source /etc/kolla/admin-openrc.sh
            echo "✅ Kolla Environment Activated Successfully."
            echo "Current Project: ${OS_PROJECT_NAME:-Unknown}"
        else
            echo "Error: admin-openrc.sh not found!"
            return 1
        fi
    }
    ```

3.  **Save and Exit:**
    Press `Ctrl+O`, `Enter`, then `Ctrl+X`.

4.  **Apply Changes:**
    ```bash
    source ~/.bashrc
    ```

### Real-World Usage
Instead of typing two lines, simply run:
```bash
kolla-env
```
*Output:* `✅ Kolla Environment Activated Successfully. Current Project: admin`

---

## 3. Method 2: The Quick Alias
**Best For:** Users who prefer the shortest possible command and do not need error checking logic.

### Step-by-Step Implementation

1.  **Edit `.bashrc`:**
    ```bash
    nano ~/.bashrc
    ```

2.  **Add the alias line:**
    ```bash
    alias kolla-init='source /opt/venv-kolla/bin/activate && source /etc/kolla/admin-openrc.sh'
    ```

3.  **Apply Changes:**
    ```bash
    source ~/.bashrc
    ```

### Real-World Usage
Run the short alias:
```bash
kolla-init
```
*Note: If the paths change in the future, you only need to update this one line in `.bashrc`.*

---

## 4. Method 3: The Portable Script
**Best For:** Storing configurations in Git, sharing with team members, or keeping a backup of your setup logic.

### Step-by-Step Implementation

1.  **Create a dedicated scripts directory:**
    ```bash
    mkdir -p ~/scripts
    ```

2.  **Create the script file:**
    ```bash
    nano ~/scripts/load-kolla.sh
    ```

3.  **Insert the content:**
    ```bash
    #!/bin/bash
    # Script: load-kolla.sh
    # Description: Loads Kolla Venv and OpenStack RC

    echo "Loading Kolla Environment..."
    source /opt/venv-kolla/bin/activate
    source /etc/kolla/admin-openrc.sh
    echo "Done."
    ```

4.  **Make it executable (Optional but good practice):**
    ```bash
    chmod +x ~/scripts/load-kolla.sh
    ```

### Real-World Usage
⚠️ **Critical:** You must still use `source` to run this file, or the environment won't persist.
```bash
source ~/scripts/load-kolla.sh
# OR the shorthand dot operator
. ~/scripts/load-kolla.sh
```

---

## 5. Method 4: Automatic Session Loading
**Best For:** Dedicated nodes where you *only* work on OpenStack/Kolla and never need a "clean" shell without these variables.

### Step-by-Step Implementation

1.  **Edit `.bashrc`:**
    ```bash
    nano ~/.bashrc
    ```

2.  **Paste the raw commands at the very bottom:**
    ```bash
    # Auto-load Kolla Environment on login
    source /opt/venv-kolla/bin/activate
    source /etc/kolla/admin-openrc.sh
    ```

3.  **Apply Changes:**
    ```bash
    source ~/.bashrc
    ```

### Real-World Usage
Simply log in via SSH. The environment is ready immediately.
```bash
ssh user@kolla-controller
# Prompt appears, environment is already active.
(openstack) user@controller:~$
```

---

## 6. Verification & Troubleshooting

Regardless of the method chosen, always verify that the environment is loaded correctly before running critical OpenStack commands.

### Verification Commands
Run these to confirm success:

1.  **Check Virtual Environment:**
    Your prompt should show `(venv-kolla)` or similar at the start.
    ```bash
    which python
    # Output should point to /opt/venv-kolla/bin/python
    ```

2.  **Check OpenStack Credentials:**
    ```bash
    env | grep OS_
    # Should list OS_USERNAME, OS_PASSWORD, OS_AUTH_URL, etc.
    ```

3.  **Test CLI Connectivity:**
    ```bash
    openstack token issue
    # Should return a valid token table without authentication errors.
    ```

### Common Issues
| Issue | Possible Cause | Solution |
| :--- | :--- | :--- |
| `command not found: openstack` | Venv not activated | Ensure you used `source` not `./` for scripts. |
| `Auth failed` | Wrong RC file or password | Check `/etc/kolla/admin-openrc.sh` contents. |
| Function/Alias not working | `.bashrc` not reloaded | Run `source ~/.bashrc` again. |

---
*Generated for DevOps Infrastructure Automation | Sumon's Lab*