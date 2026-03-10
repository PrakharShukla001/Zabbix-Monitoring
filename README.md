# 🖥️ Zabbix 4.0 Server Setup
### Complete Installation Guide — Commands, Configs & Setup

---

> **Environment**
> | Parameter | Value |
> |---|---|
> | Zabbix Version | 4.0 LTS |
> | OS | CentOS 7 / RHEL 7 |
> | Database | MariaDB 5.5 |
> | Web Server | Apache (httpd) |
> | Server IP | `192.168.182.139` |
> | Zabbix Port | `10051` |
> | Web URL | `http://192.168.182.139/zabbix` |

---

## Table of Contents

1. [Install Zabbix Repository](#step-1--install-zabbix-repository)
2. [Install Packages](#step-2--install-zabbix-server-frontend--agent)
3. [Database Setup](#step-3--mariadb-database-setup)
4. [Configure Zabbix Server](#step-4--configure-zabbix-server)
5. [Configure PHP / Apache](#step-5--configure-php-for-zabbix-frontend)
6. [Start & Enable Services](#step-6--start--enable-services)
7. [Firewall Configuration](#step-7--firewall-configuration)
8. [SELinux Configuration](#step-8--selinux-configuration)
9. [Web Frontend Installation](#step-9--web-frontend-installation)
10. [Agent Configuration](#zabbix-agent-configuration)
11. [Post-Install Dashboard](#dashboard--post-installation)
12. [Troubleshooting](#troubleshooting)

---

## Step 1 — Install Zabbix Repository

```bash
# Install Zabbix 4.0 repository for CentOS 7
rpm -ivh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm

# Clean and update yum cache
yum clean all
yum makecache
```

---

## Step 2 — Install Zabbix Server, Frontend & Agent

```bash
# Install Zabbix server (MySQL backend), web frontend, and agent
yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent

# Install MariaDB
yum install -y mariadb-server mariadb

# Start and enable MariaDB
systemctl start mariadb
systemctl enable mariadb
```

---

## Step 3 — MariaDB Database Setup

```bash
# Secure MariaDB installation
mysql_secure_installation

# Login as root
mysql -u root -p
```

Inside the **MariaDB shell**:

```sql
-- Create the Zabbix database
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;

-- Create user and grant privileges
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost IDENTIFIED BY 'your_password';

-- Apply changes
FLUSH PRIVILEGES;

EXIT;
```

**Import Zabbix schema:**

```bash
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
```

---

## Step 4 — Configure Zabbix Server

```bash
vi /etc/zabbix/zabbix_server.conf
```

**Key parameters:**

| Parameter | Value |
|---|---|
| `DBHost` | `localhost` |
| `DBName` | `zabbix` |
| `DBUser` | `zabbix` |
| `DBPassword` | `your_password` |
| `ListenPort` | `10051` |
| `LogFile` | `/var/log/zabbix/zabbix_server.log` |
| `PidFile` | `/var/run/zabbix/zabbix_server.pid` |
| `AlertScriptsPath` | `/usr/lib/zabbix/alertscripts` |
| `ExternalScripts` | `/usr/lib/zabbix/externalscripts` |

**Full `/etc/zabbix/zabbix_server.conf` snippet:**

```ini
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/var/run/zabbix/zabbix_server.pid
SocketDir=/var/run/zabbix
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=your_password
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
Timeout=4
AlertScriptsPath=/usr/lib/zabbix/alertscripts
ExternalScripts=/usr/lib/zabbix/externalscripts
LogSlowQueries=3000
```

---

## Step 5 — Configure PHP for Zabbix Frontend

```bash
vi /etc/httpd/conf.d/zabbix.conf
```

**Contents of `/etc/httpd/conf.d/zabbix.conf`:**

```apache
Alias /zabbix /usr/share/zabbix

<Directory "/usr/share/zabbix">
    Options FollowSymLinks
    AllowOverride None
    Require all granted

    <IfModule mod_php5.c>
        php_value max_execution_time 300
        php_value memory_limit 128M
        php_value post_max_size 16M
        php_value upload_max_filesize 2M
        php_value max_input_time 300
        php_value max_input_vars 10000
        php_value always_populate_raw_post_data -1
        php_value date.timezone Asia/Kolkata
    </IfModule>
</Directory>
```

> **⚠️ Note:** Set `date.timezone` to your local timezone.
> Examples: `Asia/Kolkata` (India) · `America/New_York` (US East) · `Europe/London` (UK)

---

## Step 6 — Start & Enable Services

```bash
# Start services
systemctl start zabbix-server
systemctl start zabbix-agent
systemctl start httpd

# Enable on boot
systemctl enable zabbix-server
systemctl enable zabbix-agent
systemctl enable httpd

# Verify all services
systemctl status zabbix-server
systemctl status zabbix-agent
systemctl status httpd
systemctl status mariadb
```

---

## Step 7 — Firewall Configuration

```bash
# Open Zabbix server port
firewall-cmd --permanent --add-port=10051/tcp

# Open Zabbix agent port
firewall-cmd --permanent --add-port=10050/tcp

# Open HTTP
firewall-cmd --permanent --add-service=http

# Apply changes
firewall-cmd --reload

# Verify
firewall-cmd --list-all
```

---

## Step 8 — SELinux Configuration

```bash
# Allow Zabbix connections
setsebool -P httpd_can_network_connect 1
setsebool -P httpd_can_connect_zabbix 1

# Check SELinux status
sestatus

# Temporarily disable (NOT recommended for production)
setenforce 0
```

---

## Step 9 — Web Frontend Installation

Navigate to: **`http://192.168.182.139/zabbix/setup.php`**

| Step | Action |
|---|---|
| 1. Welcome | Click **Next step** |
| 2. Pre-requisites | All items must show **OK** — fix any failures before continuing |
| 3. Configure DB | `DBHost=localhost` · `DBName=zabbix` · `DBUser=zabbix` · `DBPassword=<your_password>` |
| 4. Server Details | `Host=localhost` · `Port=10051` · `Name=Zabbix Server` |
| 5. Summary | Review all settings and confirm |
| 6. Install | ✅ Success — config file created |

> **✅ Success message:** `Configuration file '/etc/zabbix/web/zabbix.conf.php' created successfully.`

---

## Zabbix Agent Configuration

```bash
vi /etc/zabbix/zabbix_agentd.conf
```

```ini
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0

# Allow connections from Zabbix server
Server=192.168.182.139

# Active checks
ServerActive=192.168.182.139

# Must match the hostname configured in Zabbix web frontend
Hostname=Zabbix server
```

---

## Dashboard — Post-Installation

Default login credentials:

| Setting | Value |
|---|---|
| URL | `http://192.168.182.139/zabbix` |
| Username | `Admin` |
| Password | `zabbix` |

> **🔒 Security:** Change the default password immediately after first login.
> Go to **Administration → Users → Admin**

**System summary after login:**

| Parameter | Value |
|---|---|
| Zabbix server is running | ✅ Yes (`localhost:10051`) |
| Hosts (enabled / disabled / templates) | 92 (1 / 0 / 91) |
| Items (enabled / disabled / not supported) | 76 (70 / 0 / 6) |
| Triggers (enabled / disabled) | 46 (46 / 0) |
| Users (online) | 2 (1 online) |

> **⚠️ Common Alert:** *"Zabbix agent on Zabbix server is unreachable for 5 minutes"*
> Fix: ensure `zabbix-agent` is running and `Server=` in `zabbix_agentd.conf` matches the server IP.

---

## Troubleshooting

### Check Logs

```bash
tail -f /var/log/zabbix/zabbix_server.log
tail -f /var/log/zabbix/zabbix_agentd.log
tail -f /var/log/httpd/error_log
```

### Check Service Status & Ports

```bash
systemctl status zabbix-server
systemctl status zabbix-agent
netstat -tlnp | grep 10051
netstat -tlnp | grep 10050
```

### Test Database Connection

```bash
mysql -u zabbix -p zabbix -e 'SELECT COUNT(*) FROM hosts;'
```

### Fix Agent Unreachable

```bash
# Verify agent config points to correct server IP
grep '^Server' /etc/zabbix/zabbix_agentd.conf
# Expected: Server=192.168.182.139

# Restart agent
systemctl restart zabbix-agent
```

---

## Quick Command Reference

| Task | Command |
|---|---|
| Install repo | `rpm -ivh https://repo.zabbix.com/.../zabbix-release-4.0-1.el7.noarch.rpm` |
| Install packages | `yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent` |
| Install MariaDB | `yum install -y mariadb-server mariadb` |
| Login to DB | `mysql -u root -p` |
| Create DB | `CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;` |
| Import schema | `zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz \| mysql -uzabbix -p zabbix` |
| Start all services | `systemctl start zabbix-server zabbix-agent httpd mariadb` |
| Enable on boot | `systemctl enable zabbix-server zabbix-agent httpd mariadb` |
| Open firewall | `firewall-cmd --permanent --add-port=10051/tcp && firewall-cmd --reload` |
| View server log | `tail -f /var/log/zabbix/zabbix_server.log` |

---

*Zabbix 4.0 LTS — CentOS 7 / RHEL 7 — MariaDB*
