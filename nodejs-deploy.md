# EC2 Ubuntu Machine Setup

## 1. Update & Upgrade System
```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Install Node.js (LTS)
```bash
# Install npm
sudo apt-get install npm -y

# Install n (Node version manager)
sudo npm i -g n

# Install latest LTS version of Node.js
sudo n lts
```

## 3. Install & Configure Nginx
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


## 4. Clone the project from git on the Ec2 ubuntu machine 
```bash
# Clone Repo
 sudo git clone https://github.com/Amresh-01/Notes-App.git
 backend folder mai
 sudo npm i
```


## 5. Configure Nginx as Reverse Proxy
```bash
 sudo nano /etc/nginx/sites-available/express-app
```