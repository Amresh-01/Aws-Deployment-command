# Deploy backand on AWS EC2 Ubuntu Machine

backend ko isi folder mai rakhna hai.<br>
/var/www/express-app

### 1. Update & Upgrade System
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Node.js (LTS)
```bash
# Install npm
sudo apt-get install npm -y

# Install n (Node version manager)
sudo npm i -g n

# Install latest LTS version of Node.js
sudo n lts
```

### 3. Install & Configure Nginx
```bash
# Install Nginx
sudo apt install nginx -y
```

> **Note:** Agar system restart ho to Nginx automatically start ho jaye, neeche diye commands run karo.
```bash
# Start Nginx
sudo systemctl start nginx

# Enable Nginx on system boot
sudo systemctl enable nginx

# Check Nginx status
sudo systemctl status nginx

# make folder on ubuntu machine
mkdir → make directory
-p → create parent folders if not exist
/var/www/express-app → folder path for your app

sudo mkdir -p /var/www/express-app

```


### 4. Clone the project from git on the Ec2 ubuntu machine 
```bash
# Clone Repo
 sudo git clone https://github.com/Amresh-01/Notes-App.git
 backend folder mai
 sudo npm i
```


### 5. Configure Nginx as Reverse Proxy
```bash
 sudo nano /etc/nginx/sites-available/express-app

 
 server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;  # Replace with your IP or domain

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```
#### 6. Enable the Configuration

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/express-app /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

#### Step 7: Install PM2 to manage the Node.js application

```bash
sudo npm install -g pm2

## Generate the start script using PM2
sudo pm2 startup
```

After this run your application using PM2 in the deployment script and use this command to start the server with PM2:

```bash
sudo pm2 start ecosystem.config.js

## Save the PM2
sudo pm2 save
```