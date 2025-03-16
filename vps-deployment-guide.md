
# FileVault VPS Deployment Guide

This guide provides detailed instructions for deploying the FileVault application on a Virtual Private Server (VPS).

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [Application Installation](#application-installation)
4. [Configuration](#configuration)
5. [Running the Application](#running-the-application)
6. [Setting Up a Production Environment](#setting-up-a-production-environment)
7. [Securing Your Deployment](#securing-your-deployment)
8. [Maintenance](#maintenance)
9. [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, ensure you have the following:

- A VPS with at least 1GB RAM and 1 CPU core
- A domain name (optional but recommended)
- Basic knowledge of Linux commands
- SSH access to your VPS

## Server Setup

### Step 1: Update Your Server

Log in to your VPS via SSH and update the system:

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 2: Install Required Dependencies

Install Node.js (v20), npm, and other required dependencies:

```bash
# Install Node.js and npm
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install additional dependencies
sudo apt install -y git build-essential

# Verify installations
node -v
npm -v
```

### Step 3: Set Up a Non-Root User (Recommended for Security)

```bash
# Create a new user
sudo adduser filevault

# Add user to sudo group
sudo usermod -aG sudo filevault

# Switch to the new user
su - filevault
```

## Application Installation

### Step 1: Clone the FileVault Repository

```bash
# Create a directory for the application
mkdir -p ~/filevault
cd ~/filevault

# Clone from your repository or copy files to this directory
# If using git:
# git clone https://your-repository-url.git .


## Database Setup for FileVault

FileVault uses PostgreSQL as its database. Here's a detailed guide to set up and configure the database for your application.

### Step 1: Install PostgreSQL

```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Verify installation
sudo systemctl status postgresql
```

### Step 2: Create Database and User

```bash
# Connect to PostgreSQL as postgres user
sudo -u postgres psql

# In the PostgreSQL shell, create a new user and database
CREATE USER filevault WITH PASSWORD 'your_secure_password';
CREATE DATABASE filevault_db OWNER filevault;
GRANT ALL PRIVILEGES ON DATABASE filevault_db TO filevault;

# Exit PostgreSQL shell
\q
```

### Step 3: Configure Database Connection

Update your `.env` file with the PostgreSQL connection string:

```bash
# Open .env file
nano .env
```

Add the following line to your `.env` file:

```
DATABASE_URL=postgres://filevault:your_secure_password@localhost:5432/filevault_db
```

### Step 4: Initialize Database Schema

The application uses Drizzle ORM to manage the database schema. Run the following command to initialize your database with the required tables:

```bash
# Install Drizzle CLI globally if not already installed
sudo npm install -g drizzle-kit

# Run database migration
npm run db:push
```

This will create the necessary tables defined in your schema:
- `users` - Stores user credentials
- `folders` - Manages folder structure and hierarchy
- `files` - Tracks file metadata and storage paths

### Step 5: Database Maintenance

For regular database maintenance, consider setting up automatic backups:

```bash
# Create a backup directory
mkdir -p ~/filevault-backups

# Create a backup script
cat > ~/backup-db.sh << 'EOL'
#!/bin/bash
BACKUP_DIR=~/filevault-backups
TIMESTAMP=$(date +"%Y%m%d-%H%M%S")
BACKUP_FILE="$BACKUP_DIR/filevault-$TIMESTAMP.sql"

# Create database backup
pg_dump -U filevault filevault_db > $BACKUP_FILE

# Compress the backup
gzip $BACKUP_FILE

# Keep only the last 7 backups
find $BACKUP_DIR -name "filevault-*.sql.gz" -type f -mtime +7 -delete
EOL

# Make the script executable
chmod +x ~/backup-db.sh

# Set up a daily cron job for backups
(crontab -l 2>/dev/null; echo "0 2 * * * ~/backup-db.sh") | crontab -
```

### Step 6: Database Connection Pooling

For better performance, implement connection pooling in your application:

```javascript
// Example implementation in your server/db.ts file
import { Pool, neonConfig } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-serverless';
import ws from "ws";
import * as schema from "@shared/schema";

neonConfig.webSocketConstructor = ws;

if (!process.env.DATABASE_URL) {
  throw new Error(
    "DATABASE_URL must be set. Did you forget to provision a database?",
  );
}

// Create a connection pool with max 10 connections
export const pool = new Pool({ 
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

export const db = drizzle({ client: pool, schema });
```

### Additional Database Dependencies

Make sure the following dependencies are installed for database functionality:

```bash
npm install @neondatabase/serverless drizzle-orm drizzle-zod pg
npm install -D drizzle-kit pg-connection-string @types/pg
```

### Troubleshooting Common Database Issues

1. **Connection Issues**:
   - Verify PostgreSQL is running: `sudo systemctl status postgresql`
   - Check firewall settings: `sudo ufw status`
   - Test database connection: `psql -U filevault -h localhost -d filevault_db`

2. **Migration Failures**:
   - Check schema file for errors
   - Ensure DATABASE_URL is correct in your .env file
   - Run migrations with verbose flag: `npx drizzle-kit push --verbose`

3. **Performance Issues**:
   - Consider adding indexes to frequently queried columns
   - Increase connection pool size for higher traffic
   - Monitor query performance with: `EXPLAIN ANALYZE SELECT * FROM files;`

# If transferring files manually, use SCP or SFTP to upload your files
```

### Step 2: Install Dependencies

In your project directory:

```bash
npm install
```

## Configuration

### Step 1: Create Storage Directory

```bash
# Create storage directory
mkdir -p storage/root
chmod 755 storage
chmod 755 storage/root
```

### Step 2: Environment Setup (If Needed)

If your application requires environment variables:

```bash
# Create a .env file (if not already created)
touch .env

# Edit the .env file
nano .env
```

Add the following configuration (adjust as needed):

```
# Database configuration (if using)
DATABASE_URL=your_database_connection_string

# Server configuration
PORT=5000
NODE_ENV=production
```

## Running the Application

### Step 1: Build the Application

```bash
# Build the client and server
npm run build
```

### Step 2: Start the Application

For testing purposes, you can run:

```bash
npm run start
```

Your application should now be accessible at http://your-server-ip:5000

## Setting Up a Production Environment

For a more robust setup, we'll use PM2 to manage the Node.js process:

### Step 1: Install PM2

```bash
sudo npm install -g pm2
```

### Step 2: Create a PM2 Configuration File

Create a file named `ecosystem.config.js`:

```bash
nano ecosystem.config.js
```

Add the following content:

```javascript
module.exports = {
  apps: [{
    name: "filevault",
    script: "dist/index.js",
    env: {
      NODE_ENV: "production",
      PORT: "5000"
    },
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: "1G"
  }]
}
```

### Step 3: Start the Application with PM2

```bash
pm2 start ecosystem.config.js
```

### Step 4: Set Up PM2 to Start on Boot

```bash
# Generate startup script
pm2 startup

# Save the current PM2 configuration
pm2 save
```

## Setting Up a Web Server (Nginx)

Nginx will serve as a reverse proxy to your Node.js application:

### Step 1: Install Nginx

```bash
sudo apt install -y nginx
```

### Step 2: Configure Nginx as a Reverse Proxy

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/filevault
```

Add the following configuration (replace `yourdomain.com` with your actual domain name):

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://0.0.0.0:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Configure for large file uploads
    client_max_body_size 100M;
}
```

### Step 3: Enable the Configuration

```bash
# Create a symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/filevault /etc/nginx/sites-enabled/

# Test the Nginx configuration
sudo nginx -t

# If the test is successful, restart Nginx
sudo systemctl restart nginx
```

## Securing Your Deployment

### Step 1: Set Up SSL with Let's Encrypt

If you have a domain pointing to your server:

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain an SSL certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Follow the interactive prompts
```

### Step 2: Set Up Firewall

```bash
# Install UFW if not already installed
sudo apt install -y ufw

# Set up basic rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'

# Enable the firewall
sudo ufw enable
```

## Maintenance

### Updating the Application

To update your application:

```bash
# Navigate to your application directory
cd ~/filevault

# Pull the latest changes (if using git)
# git pull

# Install dependencies
npm install

# Build the application
npm run build

# Restart the application
pm2 restart filevault
```

### Monitoring

Monitor your application's performance:

```bash
# View PM2 status
pm2 status

# View application logs
pm2 logs filevault

# Monitor resources
pm2 monit
```

## Troubleshooting

### Checking Logs

```bash
# Application logs
pm2 logs filevault

# Nginx access and error logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# System logs
sudo journalctl -u nginx
```

### Common Issues and Solutions

1. **Application not starting:**
   - Check logs with `pm2 logs filevault`
   - Ensure all dependencies are installed
   - Verify the build was successful

2. **Cannot connect to the application:**
   - Check if the application is running with `pm2 status`
   - Verify Nginx is running with `sudo systemctl status nginx`
   - Check firewall settings with `sudo ufw status`

3. **File upload issues:**
   - Check Nginx client_max_body_size configuration
   - Ensure storage directory permissions are set correctly

## Conclusion

You've now successfully deployed the FileVault application on a VPS. The application should be accessible at your domain or server IP address with proper security measures in place.

Remember to regularly update both your application and the server's operating system to maintain security and stability.
