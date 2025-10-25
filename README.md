# Snipe-IT Docker Deployment Guide

This guide provides step-by-step instructions for deploying Snipe-IT using Docker on Ubuntu servers for client locations.

## Prerequisites

- Ubuntu 20.04 LTS or later
- Docker Engine 20.10+ 
- Docker Compose 2.0+
- Minimum 2GB RAM
- Minimum 10GB free disk space
- Root or sudo access

## Preparing Deployment Package

### Creating the Deployment Zip

Before deploying to client sites, prepare your deployment package:

```bash
# In your development environment, create the deployment zip
zip -r snipeit-deployment.zip . -x "*.git*" "node_modules/*" "*.log" "*.tmp" ".env.local"

# Or exclude specific files/directories
zip -r snipeit-deployment.zip . \
  -x "*.git*" \
  -x "node_modules/*" \
  -x "*.log" \
  -x "*.tmp" \
  -x ".env.local" \
  -x "backups/*" \
  -x "logs/*"
```

**Files to include in the zip:**
- `docker-compose.yml`
- `Dockerfile.snipeit`
- `Dockerfile.mariadb`
- `mariadb.cnf`
- `snipeit/` directory (Snipe-IT application files)
- `README.md`
- Any custom configuration files

**Files to exclude:**
- `.git/` directory
- `node_modules/` (if any)
- Log files (`*.log`)
- Temporary files (`*.tmp`)
- Local environment files (`.env.local`)
- Backup directories
- Development-specific files

## Quick Start

### 1. Install Docker and Docker Compose

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add current user to docker group (optional, for non-root usage)
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Deploy Project Files

**Option A: Zipped Deployment (Recommended for Client Sites)**

```bash
# Create project directory
sudo mkdir -p /opt/snipeit
cd /opt/snipeit

# Upload and extract the zip file
# Upload snipeit-deployment.zip to the server
sudo unzip snipeit-deployment.zip
sudo chown -R $USER:$USER /opt/snipeit

# Verify all required files are present
ls -la
# Should show: docker-compose.yml, Dockerfile.snipeit, Dockerfile.mariadb, mariadb.cnf, snipeit/, README.md
```

**Option B: Git Clone**

```bash
# Create project directory
sudo mkdir -p /opt/snipeit
cd /opt/snipeit

# Clone from repository
git clone <your-repository-url> .
```

**Option C: Manual File Upload**

```bash
# Create project directory
sudo mkdir -p /opt/snipeit
cd /opt/snipeit

# Upload all project files manually to /opt/snipeit/
```

### 3. Configure Environment Variables

Edit the `docker-compose.yml` file to match your client's requirements:

```bash
sudo nano docker-compose.yml
```

**Important Configuration Changes:**

```yaml
services:
  mariadb:
    environment:
      MYSQL_ROOT_PASSWORD: your_secure_root_password
      MYSQL_DATABASE: snipeit
      MYSQL_USER: snipeit
      MYSQL_PASSWORD: your_secure_db_password

  snipeit:
    environment:
      APP_KEY: base64:your_generated_app_key
      APP_URL: http://your-domain.com  # Change to client's domain
      DB_HOST: mariadb
      DB_PORT: 3306
      DB_DATABASE: snipeit
      DB_USERNAME: snipeit
      DB_PASSWORD: your_secure_db_password
    ports:
      - "80:80"  # Change port if needed
```

### 4. Generate Application Key

```bash
# Generate a new APP_KEY for production
docker run --rm snipe/snipe-it:latest php artisan key:generate --show
```

Copy the generated key and update it in `docker-compose.yml`.

### 5. Deploy the Application

```bash
# Build and start containers
sudo docker-compose up -d

# Check container status
sudo docker-compose ps

# View logs
sudo docker-compose logs -f
```

### 6. Complete Initial Setup

1. **Access the application**: Open `http://your-server-ip` in a web browser
2. **Follow the setup wizard**:
   - Create admin user
   - Configure organization settings
   - Complete initial configuration

## Production Deployment Steps

### 1. Server Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install firewall
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443

# Install fail2ban for security
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 2. SSL Certificate Setup (Recommended)

```bash
# Install Certbot
sudo apt install -y certbot

# Generate SSL certificate (replace with your domain)
sudo certbot certonly --standalone -d your-domain.com

# Update docker-compose.yml to include SSL
# Add SSL configuration to snipeit service
```

### 3. Database Backup Configuration

```bash
# Create backup directory
sudo mkdir -p /opt/snipeit/backups

# Add backup script
sudo nano /opt/snipeit/backup.sh
```

Backup script content:
```bash
#!/bin/bash
BACKUP_DIR="/opt/snipeit/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Database backup
docker exec snipeit-mariadb mysqldump -u root -p$MYSQL_ROOT_PASSWORD snipeit > $BACKUP_DIR/snipeit_$DATE.sql

# Keep only last 7 days of backups
find $BACKUP_DIR -name "snipeit_*.sql" -mtime +7 -delete

echo "Backup completed: snipeit_$DATE.sql"
```

```bash
# Make backup script executable
sudo chmod +x /opt/snipeit/backup.sh

# Add to crontab for daily backups
sudo crontab -e
# Add this line:
# 0 2 * * * /opt/snipeit/backup.sh
```

### 4. Monitoring and Logs

```bash
# Create log rotation configuration
sudo nano /etc/logrotate.d/snipeit
```

Log rotation content:
```
/opt/snipeit/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
}
```

## Client-Specific Configuration

### 1. Domain Configuration

```bash
# Update APP_URL in docker-compose.yml
APP_URL: https://client-domain.com

# Configure reverse proxy (if using Nginx)
sudo apt install -y nginx
sudo nano /etc/nginx/sites-available/snipeit
```

Nginx configuration:
```nginx
server {
    listen 80;
    server_name client-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name client-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/client-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/client-domain.com/privkey.pem;
    
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 2. Email Configuration

Update `docker-compose.yml` with client's email settings:

```yaml
environment:
  MAIL_MAILER: smtp
  MAIL_HOST: smtp.client-domain.com
  MAIL_PORT: 587
  MAIL_USERNAME: snipeit@client-domain.com
  MAIL_PASSWORD: email_password
  MAIL_FROM_ADDR: snipeit@client-domain.com
  MAIL_FROM_NAME: "Client Company IT"
```

## Maintenance Commands

### Container Management

```bash
# Stop containers
sudo docker-compose down

# Start containers
sudo docker-compose up -d

# Restart containers
sudo docker-compose restart

# View logs
sudo docker-compose logs -f snipeit
sudo docker-compose logs -f mariadb

# Update application
sudo docker-compose pull
sudo docker-compose up -d
```

### Database Management

```bash
# Access database
sudo docker exec -it snipeit-mariadb mysql -u root -p

# Backup database
sudo docker exec snipeit-mariadb mysqldump -u root -p$MYSQL_ROOT_PASSWORD snipeit > backup.sql

# Restore database
sudo docker exec -i snipeit-mariadb mysql -u root -p$MYSQL_ROOT_PASSWORD snipeit < backup.sql
```

### Application Maintenance

```bash
# Clear application cache
sudo docker exec snipeit-app php artisan cache:clear

# Clear configuration cache
sudo docker exec snipeit-app php artisan config:clear

# Run database migrations
sudo docker exec snipeit-app php artisan migrate

# Generate new application key
sudo docker exec snipeit-app php artisan key:generate --show
```

## Troubleshooting

### Common Issues

1. **Container won't start**:
   ```bash
   sudo docker-compose logs
   sudo docker-compose down
   sudo docker-compose up -d
   ```

2. **Database connection issues**:
   ```bash
   sudo docker exec snipeit-app php artisan migrate:status
   ```

3. **Permission issues**:
   ```bash
   sudo chown -R www-data:www-data /opt/snipeit/snipeit/
   sudo chmod -R 755 /opt/snipeit/snipeit/
   ```

4. **Memory issues**:
   ```bash
   # Check system resources
   free -h
   df -h
   ```

### Health Checks

```bash
# Check container health
sudo docker-compose ps

# Check application health
curl -f http://localhost:8080/health

# Check database connectivity
sudo docker exec snipeit-app php artisan migrate:status
```

## Security Checklist

- [ ] Change default passwords
- [ ] Enable SSL/TLS
- [ ] Configure firewall
- [ ] Set up fail2ban
- [ ] Regular security updates
- [ ] Database backups
- [ ] Monitor logs
- [ ] Restrict file permissions

## Support and Maintenance

### Regular Maintenance Tasks

1. **Weekly**:
   - Check container status
   - Review logs for errors
   - Verify backups

2. **Monthly**:
   - Update system packages
   - Update Docker images
   - Review security logs

3. **Quarterly**:
   - Full system backup
   - Security audit
   - Performance review

### Contact Information

For technical support or questions about this deployment:
- Email: support@yourcompany.com
- Phone: +1-XXX-XXX-XXXX

## File Structure

```
/opt/snipeit/
├── docker-compose.yml          # Main configuration
├── Dockerfile.snipeit         # Snipe-IT container definition
├── Dockerfile.mariadb         # MariaDB container definition
├── mariadb.cnf               # Database configuration
├── snipeit/                  # Snipe-IT application files
├── backups/                  # Database backups
├── logs/                     # Application logs
└── README.md                 # This file
```

## Backup Functionality

The Docker setup includes proper mysqldump support for Snipe-IT's backup functionality:

- **mysqldump**: Installed and configured in the Snipe-IT container
- **Symlink**: `/usr/local/bin/mysqldump` points to `/usr/bin/mysqldump`
- **Database Access**: Can connect to the MariaDB container for backups
- **Web Interface**: Backup functionality available in Snipe-IT admin panel

## Version Information

- Snipe-IT Version: Latest
- Docker Compose Version: 2.0+
- MariaDB Version: 10.6
- PHP Version: 8.2

---

**Note**: This deployment guide assumes a standard Ubuntu server environment. Adjust configurations based on specific client requirements and infrastructure.
