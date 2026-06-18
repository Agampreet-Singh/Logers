# Logers

## Introduction

**Logers** is an Easy-rated Linux machine designed to demonstrate a realistic web application compromise followed by a custom privilege escalation technique.

The machine begins with enumeration of a publicly accessible web service hosting an **Emlog CMS** instance. Players must identify the correct virtual host, enumerate accessible resources, and obtain administrative access through a weak password that is intentionally crackable using the `rockyou.txt` wordlist. After authenticating, players exploit a vulnerable Emlog plugin upload functionality (CVE-2026-41517) to gain remote code execution.

Following the initial foothold, players must perform local enumeration to discover a backup configuration file containing credentials for a secondary system user. Finally, the machine concludes with a custom privilege escalation vulnerability involving insecure command execution inside a privileged maintenance script.

The machine focuses on the following skills:

* Linux enumeration
* Virtual host discovery
* Directory enumeration
* Password auditing with Hydra
* CMS exploitation
* PHP reverse shells
* Credential harvesting
* SSH access
* Sudo enumeration
* Command Injection
* Linux Privilege Escalation

---

# Info for HTB

## Access

### Passwords

| User        | Password                           |
| ----------- | ---------------------------------- |
| admin (CMS) | eminem                             |
| logging     | superlogging                       |
| root        | *ethicalhacker@logging* |

---

## Key Processes

### Nginx

Hosts the Emlog CMS website and is configured to serve the application only through the virtual host:

```
logers.htb
```

Direct access using the IP address returns a 404 response to encourage virtual host enumeration.

---

### PHP-FPM

Runs PHP 8.3 and executes the Emlog application.

---

### OpenSSH

Provides SSH access for the **logging** user after credentials are obtained from the backup configuration file.

---

### MySQL

Stores all Emlog CMS data including administrator credentials.

---

### Custom Privileged Script

```
/usr/local/bin/status_check.sh
```

The **logging** user is allowed to execute this script as root without a password.

The script contains an intentional command injection vulnerability which forms the intended privilege escalation path.

Source code is included with the submission.

---

## Automation / Crons

No cron jobs are required for the intended exploitation path.

No automated cleanup scripts are configured.

---

## Firewall Rules

No custom firewall rules are configured.

Only the intended services are exposed:

* TCP/22 (SSH)
* TCP/80 (HTTP)

---

## Docker

Docker is not used.

---

## Other

The machine intentionally requires virtual host discovery.

The web application is only accessible through:

```
logers.htb
```

Players are expected to add the hostname to their `/etc/hosts` file after discovering it during enumeration.

The administrator password is intentionally weak and is crackable within a few seconds using the `rockyou.txt` wordlist.

The privilege escalation vulnerability is a custom command injection inside a privileged maintenance script rather than a public GTFOBins technique.

---

# Writeup

This walkthrough documents the intended solution path for compromising the machine.

The attack path consists of four major phases:

1. Enumeration
2. Initial Foothold
3. Lateral Movement
4. Privilege Escalation

Every command shown below can be copied directly to reproduce the attack.

---

# Enumeration

The first step is to identify the services running on the target machine.

```bash
nmap -sC -sV 192.168.52.149
```

output:

```
![Nmap Scan](screenshots/01_nmap.png)
```

The scan reveals two services:

* SSH
* HTTP

The HTTP service also exposes a `robots.txt` entry indicating the existence of an administrative interface.

```
/admin/
```

---

## HTTP Enumeration

Browsing directly to the server IP does not expose the application.

![HTTP by IP](screenshots/02_ip.png)

This indicates that the application likely relies on a virtual host configuration.

Attempting to browse using the hostname initially fails because the local machine cannot resolve the domain.

![Hostname Failure](screenshots/03_domain_fail.png)

After adding the discovered hostname into `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add:

```
192.168.52.149    logers.htb
```

![Hosts File](screenshots/04_hosts.png)

Browsing again successfully loads the Emlog CMS homepage.

![Homepage](screenshots/05_homepage.png)

---

## Directory Enumeration

Directory enumeration identifies several interesting endpoints.

```bash
gobuster dir \
-u http://logers.htb \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Interesting results include:

```
![Gobuster](screenshots/06_gobuster.png)
```

```
/admin
/content
/posts
/users
```

The `/admin` endpoint presents an administrator login panel.

![Admin Login](screenshots/07_login.png)
