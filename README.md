## Nginx Research & Configuration Project
# Project Overview
This project demonstrates comprehensive research and practical implementation of Nginx web server capabilities. It covers advanced configurations including reverse proxy, load balancing, SSL/TLS, caching, PHP integration, and security hardening.

# Architecture
![alt text](https://raw.githubusercontent.com/zaeemattique/InnovationLab-Task2/refs/heads/main/Task2%20Architecture%20(1).jpg)

# Prerequisites
AWS Account with appropriate permissions
Basic understanding of Linux commands
SSH key pair for EC2 instance access

# Infrastructure Deployment

**Manual AWS Setup**

**VPC Configuration**
Create VPC with CIDR 10.0.0.0/16
Name: Task2-VPC-Zaeem
No IPv6 CIDR block

**Subnets Configuration**
Public Subnet: 10.0.1.0/24 in us-west-2a
Private Subnet: 10.0.2.0/24 in us-west-2a

**Internet & NAT Gateways**
Create and attach Internet Gateway to VPC
Create NAT Gateway in public subnet with Elastic IP

**Route Tables**
Public Route Table: Route 0.0.0.0/0 → Internet Gateway
Private Route Table: Route 0.0.0.0/0 → NAT Gateway

**EC2 Instances**
Public Instance: Amazon Linux 2023, t3.micro
Security Groups: Allow SSH (22) and HTTP (80)
Auto-assign public IP enabled
Private Instance: Backend server in private subnet

# Nginx Installation & Configuration

SSH into public EC2 instance
```
sudo dnf update -y
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

# Configuration Tasks
**Serve Static Content**
Default configuration serves content from /usr/share/nginx/html/index.html

**Reverse Proxy Configuration**
```
nginx
server {
    listen 80;
    server_name _;
    
    location / {
        proxy_pass http://10.0.2.231;  # Private instance IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Load Balancing**
```
upstream backend {
    # round robin (default)
    server 10.0.2.231;
    server 10.0.2.201;
    
    # Alternative algorithms:
    # least_conn;
    # ip_hash;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
    }
}
```

**SSL/TLS with Self-Signed Certificate**
```
# Generate self-signed certificate
sudo openssl req -x509 -newkey rsa:2048 \
    -keyout /etc/ssl/private/nginx-selfsigned.key \
    -out /etc/ssl/certs/nginx-selfsigned.crt \
    -days 365 -nodes
```
```
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location / {
        proxy_pass http://backend;
    }
}
```

**Caching Implementation**
```
# Create cache directory
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m 
                 max_size=1g inactive=60m use_temp_path=off;

server {
    location / {
        proxy_pass http://backend;
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_bypass $http_cache_control;
        add_header X-Proxy-Cache $upstream_cache_status;
    }
}
```

**WebSocket Support**
```
location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
}
```

**PHP Integration with PHP-FPM**
```
# Install PHP-FPM
sudo dnf install php-fpm php-mysql -y
nginx
server {
    root /var/www/html;
    index index.php;

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/var/run/php-fpm/www.sock;
    }
}
```

**URL Rewriting & Redirects**
```
# Permanent redirect (301)
location /old-page {
    return 301 /new-page;
}

# URL rewriting
location / {
    rewrite ^/blog/(.*)$ /posts/$1 last;
}
```

**Security Headers**
```
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options SAMEORIGIN;
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "no-referrer-when-downgrade";
add_header Content-Security-Policy "default-src 'self'";
```

**Custom Error Pages**
```
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

location = /404.html {
    root /usr/share/nginx/html;
    internal;
}
```

**Multiple Sites with Server Blocks**
```
# Site 1
server {
    listen 80;
    server_name example1.com;
    root /var/www/example1;
}

# Site 2
server {
    listen 80;
    server_name example2.com;
    root /var/www/example2;
}
```

**Modular Configuration with Include**
```
# Main nginx.conf
http {
    include /etc/nginx/mime.types;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

**Using Nginx Variables**
```
location / {
    add_header X-Client-IP $remote_addr;
    add_header X-Host-Name $host;
    add_header X-Request-URI $request_uri;
}

```
**Rate Limiting & Connection Limits**
```
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=5r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        location / {
            limit_req zone=one burst=10 nodelay;
            # ... other directives
        }

        location /downloads/ {
            limit_conn addr 2;
            # ... other directives
        }
    }
}
```

**Custom Logging**
```
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;
```

# Management Commands
```
# Test configuration
sudo nginx -t

# Reload configuration
sudo systemctl reload nginx

# Restart service
sudo systemctl restart nginx

# Check status
sudo systemctl status nginx

# View logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

# Testing & Validation
Access WordPress
Navigate to http://<PUBLIC_IP> in browser

Verify Nginx welcome page

Test Reverse Proxy
Access public instance IP

Verify content from private backend instance

Validate SSL
Access https://<PUBLIC_IP> (will show security warning for self-signed cert)

Check Load Balancing
Monitor which backend server handles each request

# Security Considerations
Use Let's Encrypt for production SSL certificates instead of self-signed

Restrict SSH access to specific IP ranges

Implement proper firewall rules

Regular security updates for Nginx and OS

# Notes
All configurations tested on Amazon Linux 2023
Nginx configuration files located in /etc/nginx/
Main configuration file: /etc/nginx/nginx.conf
Site configurations typically in /etc/nginx/conf.d/ or /etc/nginx/sites-available/

# Author
Zaeem Attique Ashar
