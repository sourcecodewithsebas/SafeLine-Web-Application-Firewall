# SafeLine-Web-Application-Firewall
> A hands-on home lab that deploys a vulnerable web app (DVWA) on Ubuntu, attacks it from Kali Linux, and shows how SafeLine WAF detects and mitigates attacks (SQL injection, HTTP flood, custom deny rules, and more).

---

## 🎯 Goals

* Deploy DVWA (Damn Vulnerable Web App) on an Ubuntu VM.
* Launch a basic SQL injection attack from Kali Linux.
* Demonstrate SafeLine WAF protecting the application.
* Explore WAF features: HTTP flood defense, auth gateway, custom deny rules.

---

## 🧰 Prerequisites

* Host machine with ≥ 8 GB RAM and ≥ 50 GB free disk.
* VirtualBox installed (latest).
* Internet connection on host and VMs.
* Basic Linux command-line knowledge.
* Optional: ability to install packages and manage services on Ubuntu.

---

## 🏗️ Lab Topology

```
[ Kali VM ]  ──┐
               ├─ (LAN via bridged adapter) ── [ Host Network Router / Switch ]
[ Ubuntu VM (DVWA + SafeLine WAF) ] ──┘
```

* Kali: attacker machine (Kali Linux).
* Ubuntu: DVWA (Apache/PHP/MySQL) and SafeLine WAF (reverse proxy).
* All VMs use **Bridged Networking** so they share the host network and can resolve via /etc/hosts or DNS.

---
## 🛠️ Step-by-step setup

> NOTE: run commands on the appropriate VM (Kali or Ubuntu). Use `sudo` where needed.

### 1 — VirtualBox & VMs

1. Download & install VirtualBox: [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads)
2. Create two VMs:

   * **Kali Linux** (Name: `KaliLinux`) — 2+ GB RAM, 20 GB disk.
   * **Ubuntu Server** (Name: `UbuntuServer`) — 2+ GB RAM, 20+ GB disk.
3. In each VM: set **Network Adapter → Bridged Adapter** (choose host NIC).
4. (Optional) Install Guest Additions in each VM for better UX.

---

### 2 — Ubuntu: initial setup

```bash
# Update & essentials
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y net-tools git openssl
```

Get the VM IP:

```bash
ifconfig   # or `ip a`
```

---

### 3 — LAMP stack (Ubuntu)

Install Apache, PHP, and MySQL:

```bash
sudo apt-get install -y apache2 php php-mysql mysql-server
sudo mysql_secure_installation    # follow prompts, set strong root password
```

---

### 4 — Deploy DVWA

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA
sudo chmod -R 755 DVWA
```

Edit DVWA config (if needed):
`/var/www/html/DVWA/config/config.inc.php` — update DB settings:

```php
$DBMS = 'MySQL';
$db = 'dvwa';
$user = 'dvwa_user';
$pass = 'p@ssw0rd';
$host = 'localhost';
```

Create DB & user:

```sql
sudo mysql -u root -p
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
exit
```

Initialize DVWA in browser: `http://<Ubuntu-IP>/DVWA/setup.php` → **Create/Reset Database**

---

### 5 — Change DVWA to port 8080 (optional)

Edit Apache ports:

```bash
sudo nano /etc/apache2/ports.conf
# change Listen 80 → Listen 8080
```

Edit virtual host:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
# change <VirtualHost *:80> to <VirtualHost *:8080>
sudo systemctl restart apache2
```

Access DVWA at: `http://<Ubuntu-IP>:8080/DVWA`

---

### 6 — Add demo data for SQL injection

```sql
sudo mysql -u root -p
USE dvwa;
CREATE TABLE test_users (
  id INT NOT NULL AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  password VARCHAR(50) NOT NULL,
  PRIMARY KEY (id)
);
INSERT INTO test_users (username, password) VALUES
('alice','alice123'),('bob','bob123'),('admin','admin123');
exit
```

---

### 7 — Local DNS resolution (easy method)

/etc/hosts on **both** Ubuntu and Kali:

```
<Ubuntu-IP> dvwa.local
```

Now access: `http://dvwa.local:8080/DVWA`

(Optional) Set up BIND9 if you want a local DNS server — see docs for details.

---

### 8 — Self-signed SSL for DVWA

```bash
sudo mkdir -p /etc/ssl/dvwa
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/dvwa/dvwa.key \
  -out /etc/ssl/dvwa/dvwa.crt
```

(Answer prompt values or use `-subj` to automate.)

---

### 9 — Install SafeLine WAF (Ubuntu)

> Follow SafeLine instructions and the video link. The lab used the provider script.

**Automatic install (example):**

```bash
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

* Follow on-screen prompts.
* The installer will provide admin username/password and console URL (often port `9443`).

**Import SSL certificate into SafeLine**
In the SafeLine web UI → SSL Certificates: import:

* `/etc/ssl/dvwa/dvwa.crt` (certificate)
* `/etc/ssl/dvwa/dvwa.key` (private key)

**Onboard application:**

* Domain: `www.dvwa.local` (or chosen domain)
* Backend (reverse proxy): `http://<Ubuntu-IP>:8080`
* Remove port 80 from the virtual host; leave 443 only (WAF handles HTTPS).
* Attach the uploaded SSL certificate.

---

### 10 — Attack from Kali: SQL injection demo

1. Open browser on Kali → `http://dvwa.local:8080/DVWA` (or via WAF `https://www.dvwa.local/`).
2. Login DVWA (default: `admin` / `password`) — set Security to **low**.
3. Go to **SQL Injection** module and test payloads such as:

```
admin' OR '1'='1
' OR 1=1 --
```

4. Observe the vulnerability results in DVWA.

---

### 11 — Observe SafeLine WAF protection

* SafeLine should detect and block injection attempts (if WAF policy set to blocking).
* Check WAF logs and dashboards for blocked events and rules triggered.
* If blocking is enabled, you will see block pages or error responses.

---

### 12 — SafeLine advanced features (brief)

* **HTTP Flood defense**: enable DoS mitigation and set RPS thresholds. Test with `ab`, `siege`, or custom scripts from Kali.
* **Auth Sign-In**: enable the WAF authentication gateway to prompt visitors before accessing DVWA.
* **Custom Deny Rule**: block Kali IP (e.g., `192.168.x.x`) by creating a policy to block matching source IPs.

---








