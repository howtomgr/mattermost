# Mattermost Installation Guide

Self-hosted team communication platform with features like team messaging, file sharing, and integrations. An open-source alternative to Slack.

## Prerequisites

- 64-bit Linux system (RHEL 7, CentOS 7, Oracle Linux 7, or Scientific Linux 7)
- Database server (MySQL 5.7+ or PostgreSQL 9.4+)
- NGINX proxy server (recommended for production)
- 4GB RAM minimum, 8GB recommended

## Quick Installation

```bash
# Download latest Mattermost server
wget https://releases.mattermost.com/X.X.X/mattermost-X.X.X-linux-amd64.tar.gz
tar -xvzf mattermost-*.tar.gz
sudo mv mattermost /opt

# Create storage directory
sudo mkdir /opt/mattermost/data

# Create system user
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
sudo chmod -R g+w /opt/mattermost
```

## Database Setup

### PostgreSQL (Recommended)
```bash
# Install PostgreSQL
sudo yum install postgresql94-server postgresql94-contrib
sudo /usr/pgsql-9.4/bin/postgresql94-setup initdb
sudo systemctl enable --now postgresql-9.4

# Create database and user
sudo -u postgres psql
postgres=# CREATE DATABASE mattermost;
postgres=# CREATE USER mmuser WITH PASSWORD 'secure_password';
postgres=# GRANT ALL PRIVILEGES ON DATABASE mattermost to mmuser;
postgres=# \q
```

### MySQL Alternative
```bash
# Install MySQL
wget http://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
sudo yum localinstall mysql57-community-release-el7-9.noarch.rpm
sudo yum install mysql-community-server
sudo systemctl enable --now mysqld

# Configure database
mysql -u root -p
mysql> CREATE DATABASE mattermost;
mysql> CREATE USER 'mmuser'@'%' IDENTIFIED BY 'secure_password';
mysql> GRANT ALL PRIVILEGES ON mattermost.* TO 'mmuser'@'%';
```

## Configuration

```bash
# Configure database connection in /opt/mattermost/config/config.json
# For PostgreSQL:
"DriverName": "postgres",
"DataSource": "postgres://mmuser:secure_password@localhost:5432/mattermost?sslmode=disable"

# For MySQL:
"DriverName": "mysql",
"DataSource": "mmuser:secure_password@tcp(localhost:3306)/mattermost?charset=utf8mb4,utf8"
```

## System Service Setup

```bash
# Create systemd service
sudo tee /etc/systemd/system/mattermost.service > /dev/null <<EOF
[Unit]
Description=Mattermost
After=network.target postgresql-9.4.service

[Service]
Type=notify
WorkingDirectory=/opt/mattermost
User=mattermost
ExecStart=/opt/mattermost/bin/mattermost
PIDFile=/var/spool/mattermost/pid/master.pid
TimeoutStartSec=3600
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable --now mattermost
```

## NGINX Proxy Setup (Production)

```bash
# Install NGINX
sudo yum install nginx
sudo systemctl enable --now nginx

# Configure proxy
sudo tee /etc/nginx/conf.d/mattermost.conf > /dev/null <<'EOF'
upstream backend {
   server 127.0.0.1:8065;
   keepalive 32;
}

server {
   listen 80;
   server_name your-domain.com;
   
   location / {
       proxy_pass http://backend;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
   }
   
   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_pass http://backend;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
   }
}
EOF

sudo nginx -t && sudo systemctl restart nginx
```

## Verification

```bash
# Test Mattermost is running
curl http://localhost:8065
sudo systemctl status mattermost

# Check logs if needed
sudo journalctl -u mattermost -f
```

## Usage

1. Open browser to `http://your-server-ip:8065` (or your domain if using NGINX)
2. Create your admin account (first user becomes system admin)
3. Configure team settings and invite users
4. Set up integrations and customize as needed

## Additional Resources

- [Official Documentation](https://docs.mattermost.com/)
- [Mattermost GitHub](https://github.com/mattermost/mattermost-server)
- [Troubleshooting Guide](https://docs.mattermost.com/install/troubleshooting.html)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.