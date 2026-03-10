# SSH Hardening & Firewall Configuration
### Cybersecurity Homelab Series

---

##  Overview

**Date:** March 10, 2026
**Environment:** Ubuntu VM (VirtualBox on MacBook M4)
**Difficulty:** Beginner - Intermediate
**Time Taken:** ~75 minutes

In a previous exercise I scanned Ubuntu from Kali and found an open SSH port broadcasting its version, a banner leaking OS information, and zero firewall protection. In this exercise I fix all of that. By the end, I re-run the exact same nmap scans and compare the results side by side — watching the attack surface shrink in real time.

---

##  Objectives

- Audit the existing SSH configuration
- Generate a modern Ed25519 SSH key pair
- Harden the SSH configuration — disable root login, password auth, X11 forwarding
- Suppress the SSH service banner
- Configure UFW firewall with a default deny policy
- Re-run nmap scans to verify hardening effectiveness
- Understand the before/after attack surface difference

---

##  Tools & Environment

| Item | Detail |
|------|--------|
| Host Machine | MacBook M4 |
| VM Platform | VirtualBox |
| Target VM | Ubuntu (192.168.4.113) |
| Scanning VM | Kali Linux (192.168.4.198) |
| Tools Used | sshd, ssh-keygen, ufw, nmap |
| Working Directory | ~/lab/ex04 |

---

##  Step-by-Step Walkthrough

### Step 1 — Auditing the Current SSH Configuration

Before changing anything, document the current state. This gives us a clean before/after comparison.

```bash
sudo cat /etc/ssh/sshd_config | grep -v "^#" | grep -v "^$"
```

Output:
```
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_* COLORTERM NO_COLOR
Subsystem    sftp    /usr/lib/openssh/sftp-server
```



**Analysing the current config:**

| Setting | Value | Risk |
|---------|-------|------|
| PermitRootLogin | Not set (default allows) | 🔴 High |
| PasswordAuthentication | Not set (default allows) | 🔴 High |
| X11Forwarding | yes | 🟡 Medium |
| Banner | Not set | 🟡 Medium |
| MaxAuthTries | Not set (default 6) | 🟡 Medium |

Three critical settings missing entirely — meaning SSH is running with permissive defaults. This is the state most freshly installed Ubuntu servers are in.

---

### Step 2 — Generating an SSH Key Pair

Before disabling password authentication, we must set up key-based authentication — otherwise we lock ourselves out completely.

```bash
ssh-keygen -t ed25519 -C "homelab-key"
```

Output:
```
Generating public/private ed25519 key pair.
Your identification has been saved in /home/mdubuntu/.ssh/id_ed25519
Your public key has been saved in /home/mdubuntu/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:7MqE7zuVaZsTmo7z5mmHWKgCpGlRh5AtX7kru2NV974 homelab-key
+--[ED25519 256]--+
| .+ . .          |
| o + +           |
|  + o .          |
| o . . o .       |
|o..  .o So.      |
|+. ..+..*  .     |
|o  .=o.*.+.      |
|. .+o=Bo=  .     |
| ...+O@+ . E.    |
+----[SHA256]-----+
```



**Understanding the key pair:**

| File | Type | Purpose |
|------|------|---------|
| `~/.ssh/id_ed25519` | Private key | Keep secret — never share |
| `~/.ssh/id_ed25519.pub` | Public key | Put on any server you want to access |

Think of it as a padlock and key — the public key is the padlock you put on servers, the private key is the key only you hold.

**Why Ed25519 over RSA?**
Ed25519 is a modern elliptic curve algorithm that is faster, more secure, and produces shorter keys than the traditional RSA algorithm. It's the current recommended standard for SSH keys.

> **Why key-based auth is more secure than passwords:** An Ed25519 private key would take longer than the age of the universe to brute force with current technology. A password like "Password123" can be cracked in under a second.

---

### Step 3 — Configuring Authorized Keys

Added the public key to the authorized_keys file:

```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Verified the file contents:
```bash
cat ~/.ssh/authorized_keys
```

Output:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK9LYV4iPeTnS9Wc6yglUd5jn/IvQbZYpTi3kfGy4A3+ homelab-key
```

Verified the permissions:
```bash
ls -la ~/.ssh/
```

Output:
```
drwx------  2 mdubuntu mdubuntu 4096  .ssh/
-rw-------  1 mdubuntu mdubuntu   93  authorized_keys
-rw-------  1 mdubuntu mdubuntu  399  id_ed25519
-rw-r--r--  1 mdubuntu mdubuntu   93  id_ed25519.pub
```



SSH enforces strict permission requirements:
- `.ssh/` directory must be `700` (owner only)
- `authorized_keys` must be `600` (owner read/write only)
- Private key must be `600`

If permissions are too open, SSH refuses to use the keys entirely — a security feature, not a bug.

---

### Step 4 — Hardening the SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Added the following hardening block at the bottom of the file:

```bash
# Hardening — Exercise 04
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 20
Banner none
DebianBanner no
AllowUsers mdubuntu
```



**Breaking down every hardening setting:**

| Setting | Value | Why |
|---------|-------|-----|
| `PermitRootLogin no` | no | Blocks direct root SSH — attackers always try root first |
| `PasswordAuthentication no` | no | Disables password login — keys only |
| `PubkeyAuthentication yes` | yes | Enables SSH key authentication |
| `X11Forwarding no` | no | Disables graphical forwarding — unnecessary attack surface |
| `MaxAuthTries 3` | 3 | Disconnects after 3 failed attempts (default is 6) |
| `LoginGraceTime 20` | 20s | Gives connecting user only 20 seconds to authenticate |
| `Banner none` | none | Suppresses the banner that leaked version info |
| `DebianBanner no` | no | Strips Ubuntu OS identification from SSH handshake |
| `AllowUsers mdubuntu` | mdubuntu | Whitelist — only this user can SSH in |

Restarted SSH to apply changes:

```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

Output:
```
Active: active (running)
Server listening on 0.0.0.0 port 22
status=0/SUCCESS
```



---

### Step 5 — Configuring UFW Firewall

Checked current firewall status:
```bash
sudo ufw status
```
Output: `Status: inactive` — no firewall protection at all.

Added rules before enabling to avoid lockout:

```bash
sudo ufw allow 22/tcp    # Allow SSH
sudo ufw deny 23/tcp     # Block Telnet
sudo ufw deny 21/tcp     # Block FTP
sudo ufw deny 80/tcp     # Block HTTP
```

**Why these specific ports?**

| Port | Service | Reason to Block |
|------|---------|----------------|
| 23 | Telnet | Completely unencrypted — passwords sent in plain text |
| 21 | FTP | Unencrypted file transfer — credentials visible in traffic |
| 80 | HTTP | Unencrypted web traffic — we're not running a web server |

Enabled the firewall:

```bash
sudo ufw enable
```

Output: `Firewall is active and enabled on system startup`

Verified all rules:
```bash
sudo ufw status verbose
```

Output:
```
Status: active
Default: deny (incoming), allow (outgoing)

To          Action    From
22/tcp      ALLOW IN  Anywhere
23/tcp      DENY IN   Anywhere
21/tcp      DENY IN   Anywhere
80/tcp      DENY IN   Anywhere
```



**The most important line:**
```
Default: deny (incoming)
```
Everything not explicitly allowed is automatically blocked. This is a **default deny** policy — the gold standard in firewall configuration.

---

### Step 6 — Verifying Hardening with Nmap

Switched to Kali and re-ran the same scans from Exercise 3.

**Basic scan:**
```bash
nmap 192.168.4.113
```

Output:
```
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
Nmap done: 1 IP address (1 host up) scanned in 4.88 seconds
```



**Banner grab scan:**
```bash
nmap -sV --script=banner 192.168.4.113
```

Output:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 (protocol 2.0)
|_banner: SSH-2.0-OpenSSH_10.0p2
```



---

## Before & After Comparison

| Metric | Exercise 3 (Before) | Exercise 4 (After) |
|--------|--------------------|--------------------|
| Port status | 999 closed | 999 filtered ✅ |
| Banner | Ubuntu-5ubuntu5 exposed | OS stripped ✅ |
| OS identification | Linux confirmed | Less certain ✅ |
| Scan time | 0.70 seconds | 4.88 seconds ✅ |
| Password auth | Enabled | Disabled ✅ |
| Root login | Allowed | Blocked ✅ |
| Firewall | Inactive | Active (default deny) ✅ |
| Auth method | Password | Keys only ✅ |

---

##  Mistakes & Fixes

### Mistake — Banner still showing after initial hardening
**What happened:** Added `Banner none` to sshd_config but the banner still appeared in nmap scans:
```
|_banner: SSH-2.0-OpenSSH_10.0p2 Ubuntu-5ubuntu5
```
**Root cause:** nmap's `-sV` flag extracts version info from the SSH protocol handshake itself, not just the traditional banner. Ubuntu's OpenSSH also adds OS identification separately.

**Fix:** Added `DebianBanner no` to sshd_config which suppresses Ubuntu's OS-specific identification string.

**Result:** Banner reduced from `SSH-2.0-OpenSSH_10.0p2 Ubuntu-5ubuntu5` to `SSH-2.0-OpenSSH_10.0p2`

**Lesson:** Hardening is iterative. Security tools like nmap are sophisticated — a single config change rarely achieves perfect results. Test, observe, adjust.

---

##  Key Takeaways

1. **Audit before you harden** — document the current state so you can measure improvement
2. **Set up keys before disabling passwords** — order of operations matters or you lock yourself out
3. **Default deny is the gold standard** — allow only what's needed, block everything else
4. **Hardening is iterative** — test after every change and adjust accordingly
5. **Filtered ports are better than closed** — no response gives attackers less information than a rejection
6. **Scan time as a security metric** — the firewall forced nmap from 0.70s to 4.88s. At scale that matters

---

##  Real-World Relevance

| Skill Practiced | Real-World Application |
|----------------|----------------------|
| SSH hardening | Server security, cloud instance hardening |
| Key-based authentication | Industry standard for server access |
| UFW firewall configuration | Network access control |
| Default deny policy | Enterprise firewall design |
| Before/after verification | Security audit methodology |

Every cloud server deployed at a company goes through this exact hardening process  or should. SSH misconfiguration is one of the most common causes of server compromise. These settings are the baseline minimum for any production system.

---



---

**Connect with me:** [Your LinkedIn] | [Your GitHub] | [Your Twitter/X]
