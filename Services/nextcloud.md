# Nextcloud

## Overview

Self-hosted cloud storage running as an LXC container on vmbr0.
Replaces Google Drive / iCloud for personal file storage and sync.
Accessible remotely via WireGuard VPN or through Nginx Proxy Manager
with a DuckDNS domain and SSL.

---

## Container Details

| Field    | Value              |
|----------|--------------------|
| CT ID    | 101                |
| IP       | 192.168.0.56        |
| RAM      | 512MB              |
| Storage  | 28GB           |
| Bridge   | vmbr0              |
| OS       | Debian 12          |
| Status   | ✅ Running          |

---

## Deployment

### 1. Create LXC Container

In Proxmox web UI:
- Template: Debian 12
- RAM: 512MB
- Storage: allocate based on expected data size
- Network: vmbr0, static IP 192.168.0.X
- Unprivileged: yes

### 2. Install Dependencies

```bash
apt update && apt upgrade -y
apt install -y apache2 mariadb-server libapache2-mod-php \
php php-gd php-json php-mysql php-curl php-mbstring \
php-intl php-imagick php-xml php-zip php-bcmath php-gmp \
unzip wget
```

### 3. Configure MariaDB

```bash
mysql_secure_installation
mysql -u root -p
```

```sql
CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> Replace `strongpassword` with a strong unique password.

### 4. Download and Install Nextcloud

```bash
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip -d /var/www/
chown -R www-data:www-data /var/www/nextcloud
chmod -R 755 /var/www/nextcloud
```

### 5. Configure Apache

Create `/etc/apache2/sites-available/nextcloud.conf`:

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud
    ServerName 192.168.0.X

    <Directory /var/www/nextcloud>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>
</VirtualHost>
```

```bash
a2ensite nextcloud.conf
a2enmod rewrite headers env dir mime
systemctl restart apache2
```

### 6. Complete Setup via Web UI

Navigate to `http://192.168.0.X` and complete the setup wizard:

| Field         | Value                     |
|---------------|---------------------------|
| Admin user    | [admin username]          |
| Admin pass    | [strong password]         |
| Data folder   | /var/www/nextcloud/data   |
| DB type       | MySQL/MariaDB             |
| DB user       | nextclouduser             |
| DB name       | nextcloud                 |
| DB host       | localhost                 |

---

## Configuration

### PHP Memory Limit

Edit `/etc/php/[version]/apache2/php.ini`:

```ini
memory_limit = 512M
upload_max_filesize = 512M
post_max_size = 512M
max_execution_time = 300
```

### Background Jobs

Set to Cron for reliability. Add to root crontab:

```bash
crontab -u www-data -e
```

```
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

---

## Access

| Interface  | URL                              |
|------------|----------------------------------|
| Local UI   | `http://192.168.0.X`             |
| Remote UI  | `https://[subdomain].duckdns.org` (once DuckDNS + Nginx PM deployed) |

---

## Hardening

- [ ] Enable HTTPS via Nginx Proxy Manager + DuckDNS SSL cert
- [ ] Disable SSH password auth (key only)
- [ ] Enable Nextcloud 2FA (TOTP app)
- [ ] Set `'overwrite.cli.url'` in `config.php` after adding domain
- [ ] Add trusted domain to `config.php`
- [ ] Enable Nextcloud brute-force protection (built-in)
- [ ] Regular backups of `/var/www/nextcloud/data` and MariaDB

---

## Backup

### Database

```bash
mysqldump -u root -p nextcloud > nextcloud-db-backup.sql
```

### Data Directory

```bash
tar -czf nextcloud-data-backup.tar.gz /var/www/nextcloud/data
```

> Automate with a cron job once the setup is stable.

---

## Integration

- **DuckDNS** — provides the domain for external access
- **Nginx Proxy Manager** — handles reverse proxy and SSL termination
- **WireGuard** — alternative secure access method without exposing
  Nextcloud directly to the internet
- **Heimdall** — will be added to the dashboard

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment
issues encountered.