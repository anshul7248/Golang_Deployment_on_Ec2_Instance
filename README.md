# 🚀 Golang Deployment on EC2 Instance with Apache2, PostgreSQL & letsencrypt

This guide provides a step-by-step process to deploy a **Golang API** on an **Ubuntu-based EC2 instance** using **Apache2** as a reverse proxy, **PostgreSQL** as the database, and **Certbot** for SSL configuration.

---

##  Prerequisites

Before starting, ensure you have:

-  An **AWS EC2 instance** (Ubuntu 20.04+ recommended)
-  A **domain name** (e.g., `api.bizztale.info`) pointing to your EC2 instance
-  A **Golang build file** ready for deployment
-  PostgreSQL credentials and `.env` configuration ready

---

## Step 1: Install and Configure PostgreSQL

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install postgresql postgresql-contrib -y
sudo systemctl status postgresql
sudo systemctl enable postgresql
```
```bash
sudo vim /etc/postgresql/14/main/postgresql.conf
```
listen_addresses = '*'

```bash
sudo vim /etc/postgresql/14/main/pg_hba.conf
```
host    all             all             0.0.0.0/0               md5

```bash
sudo systemctl restart postgresql
```

### Create Database
```bash
sudo -i -u postgres
psql
CREATE DATABASE testdb;
ALTER ROLE postgres WITH PASSWORD '1234';
\q
exit
```

## Step 2

```bash
sudo apt install certbot python3-certbot-apache -y
```
```bash
sudo vim /etc/apache2/sites-available/your_domain.conf
```
```bash
<VirtualHost *:80>
    ServerName your_domain
    ServerAlias www.your_domain
</VirtualHost>
```
```bash
sudo a2ensite your-domain.conf
sudo systemctl reload apache2
```

### Generate SSL certificates
```bash
sudo certbot --apache
```

## Step 3: Configure HTTP Virtual Host

```bash
sudo vim /etc/apache2/sites-available/your_domain-le-ssl.conf
```
```bash
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName api.bizztale.info
    ServerAlias www.api.bizztale.info

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/api.bizztale.info/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/api.bizztale.info/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf

    DocumentRoot /var/www/html/api.bizztale.info

    <Directory /var/www/html/api.bizztale.info/>
        Options FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    # Reverse proxy to backend service (Golang app)
    #ProxyPreserveHost On
    #ProxyPass /api/ http://127.0.0.1:8080/
    #ProxyPassReverse /api/ http://127.0.0.1:8080/
    #RequestHeader set X-Forwarded-Proto "https"
    #RequestHeader set X-Forwarded-Port "443"

    ErrorLog ${APACHE_LOG_DIR}/api_error.log
    CustomLog ${APACHE_LOG_DIR}/api_access.log combined
</VirtualHost>
</IfModule>
```

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod ssl
sudo a2enmod headers
sudo systemctl restart apache2
sudo a2ensite your_domain-le-ssl.conf
sudo systemctl reload apache2
```

Keep the proxy lines commented until the Go service is deployed.

## Step 4: Deploy the Golang Build

```bash
mkdir /home/ubuntu/ratelimiter-api
chmod 777 /home/ubuntu/ratelimiter-api
cd /home/ubuntu/ratelimiter-api
```
```bash
vim .env
```
```bash
DB_HOST=127.0.0.1
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=db_name
DB_PORT=5432
DB_SSLMODE=disable
DB_TIMEZONE=Asia/Kolkata
```
#### Now pull the build file on server
```bash
sudo scp -i your.pem_file /path/to/buildfile ubuntu@your_domain:/home/ubuntu/ratelimiter-api
```

#### Make the build file executable
```bash
chmod +x buildfile_name
```

## Step 5: Create a Systemd Service for the Go App

```bash
sudo vim /etc/systemd/system/ratelimiter.service
```

```bash
[Unit]
Description=My Golang App
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/ratelimiter-api
ExecStart=/home/ubuntu/ratelimiter-api/ratelimiter
EnvironmentFile=/home/ubuntu/ratelimiter-api/.env
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl restart ratelimiter.service
sudo systemctl enable ratelimiter.service
sudo systemctl status ratelimiter.service
```

## Step 6: Enable Reverse Proxy
```bash
sudo vim /etc/apache2/sites-available/your_domain-le-ssl.conf
```
#### Uncomment the below lines

ProxyPreserveHost On
ProxyPass /api/ http://127.0.0.1:8080/
ProxyPassReverse /api/ http://127.0.0.1:8080/
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

And run the below command after saving it 

```bash
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo systemctl restart apache2
```



