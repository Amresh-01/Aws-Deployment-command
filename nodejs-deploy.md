# Deploy backand on AWS EC2 Ubuntu Machine with CI CD pipeline

backend ko isi folder mai rakhna hai. As production purpose.<br>
/var/www/express-app

### 1. Update & Upgrade System
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Node.js (LTS)
```bash
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
-p → create parent folders if not exist
/var/www/express-app → folder path

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
 cd ~
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
sudo ln -s /etc/nginx/sites-available/express-app/etc/nginx/sites-enabled/

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

Now Add this ecosystem.config.cjs FILE in your backend folder <br>

Agar Typescript use kar rahe hai tb <br>

script: ".dist/index.js" use krna hoga 

```bash
module.exports = {
  apps: [
    {
      name: "notes-backend",
      script: "index.js",

      // Use cluster mode for better performance
      exec_mode: "cluster",
      instances: "max", // use all CPU cores

      // Auto restart if crash
      autorestart: true,

      // Restart if memory exceeds limit
      max_memory_restart: "300M",

      // Restart delay to prevent rapid crashes
      restart_delay: 4000,

      // Watch should be OFF in production
      watch: false,

      // Timezone logs
      time: true,

      // Log files
      error_file: "/var/log/notes-backend/error.log",
      out_file: "/var/log/notes-backend/out.log",
      log_file: "/var/log/notes-backend/combined.log",
      log_date_format: "YYYY-MM-DD HH:mm Z",

      env: {
        NODE_ENV: "development",
        PORT: 8080
      },

      env_production: {
        NODE_ENV: "production",
        PORT: 8080
      }
    }
  ]
};
```

After this run your application using PM2 in the deployment script and use this command to start the server with PM2:

```bash
sudo pm2 start ecosystem.config.js

## Save the PM2
sudo pm2 save
```


#### Step 8: Create GitHub Secrets for Deployment

In your GitHub repository, go to Settings → Secrets and variables → Actions → New repository secret.

Add These Secrets

**EC2_HOST - Your EC2 public IP address:**<br>
**EC2_USERNAME - EC2 user (ubuntu for Ubuntu AMI):**<br>
**EC2_SSH_KEY - Your private SSH key content:**

```bash
# On your local machine, copy the entire key
cat ec2-deploy-key.pem

# Copy ALL content including:
# -----BEGIN RSA PRIVATE KEY-----
# ... key content ...
# -----END RSA PRIVATE KEY-----
```
IF Typescript project...

In your repository, create .github/workflows/deploy.yml

```yml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch: # manual trigger

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20.13.1"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build TypeScript
        run: npm run build

      - name: Create deployment package (without node_modules)
        run: |
          mkdir -p deploy
          cp -r dist deploy/
          cp -r src deploy/
          cp package*.json deploy/
          cp ecosystem.config.cjs deploy/
          # Copy .env.example if exists
          [ -f .env.example ] && cp .env.example deploy/ || echo "No .env.example found"
          tar -czf deploy.tar.gz -C deploy .

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-package
          path: deploy.tar.gz
          retention-days: 1

  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-package

      - name: Deploy to EC2 via SSH
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          # Create SSH key file
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          # Copy deployment package to EC2
          scp -i private_key.pem -o StrictHostKeyChecking=no \
            deploy.tar.gz ${EC2_USERNAME}@${EC2_HOST}:/tmp/

          # SSH into EC2 and deploy
          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'
            
            # Navigate to app directory
            cd /var/www/express-app

            # Fix ownership first (important!)
            echo "Fixing directory ownership..."
            sudo chown -R $USER:$USER /var/www/express-app
            
            
            # Backup current version
            if [ -d "dist" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mkdir -p backups
              tar -czf backups/backup_${timestamp}.tar.gz dist package.json ecosystem.config.cjs .env 2>/dev/null || true
              # Keep only last 5 backups
              ls -t backups/backup_*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
            fi
            
            # Extract new version (this will overwrite existing files)
            echo "Extracting new deployment package..."
            tar --overwrite -xzf /tmp/deploy.tar.gz -C /var/www/express-app
            rm /tmp/deploy.tar.gz

            # Fix ownership after extraction
            sudo chown -R $USER:$USER /var/www/express-app
            
            
            # Install production dependencies on the server
            echo ""
            echo "Installing production dependencies..."
            npm ci
            
            # Create .env file if needed (only on first deploy)
            if [ ! -f .env ]; then
              echo "Creating default .env file..."
              echo "NODE_ENV=production" > .env
              echo "PORT=8000" >> .env
            fi
            
            # Create logs directory
            mkdir -p logs
            
            # Zero-downtime reload with PM2
            if pm2 describe express-app > /dev/null 2>&1; then
              echo "Reloading PM2 app (zero-downtime)..."
              sudo pm2 reload ecosystem.config.cjs --update-env
            else
              echo "Starting new PM2 process..."
              sudo pm2 start ecosystem.config.cjs
            fi
            
            # Save PM2 process list
            sudo pm2 save
           
          EOF

          # Cleanup
          rm private_key.pem

      - name: Verify deployment
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
        run: |
          echo "Waiting for application to start..."
          sleep 10

          # Check if app responds via nginx (try multiple times)
          for i in {1..2}; do
            echo "Attempt $i of 2..."
            response=$(curl -s -o /dev/null -w "%{http_code}" http://${EC2_HOST} 2>/dev/null || echo "000")
            
            if [ "$response" = "200" ] || [ "$response" = "301" ] || [ "$response" = "302" ]; then
              echo "✅ Deployment successful! App is responding with status: $response"
              exit 0
            fi
            
            echo "Got response: $response, waiting 5 seconds..."
            sleep 5
          done

          echo "❌ Deployment verification failed after 2 attempts"
          echo "Please check PM2 logs on the server"
          exit 1

      - name: Rollback on failure
        if: failure()
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "🔄 Attempting to rollback to previous version..."

          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'
            
            cd /var/www/express-app

            # Fix ownership first (important!)
            sudo chown -R $USER:$USER /var/www/express-app
            
            # Find latest backup
            latest_backup=$(ls -t backups/backup_*.tar.gz 2>/dev/null | head -1)
            
            if [ -n "$latest_backup" ]; then
              echo "Found backup: $latest_backup"
              tar -xzf "$latest_backup" -C /var/www/express-app
              npm ci
              pm2 reload ecosystem.config.cjs
              echo "✅ Rolled back to previous version"
            else
              echo "⚠️ No backup found, cannot rollback"
            fi
          EOF

          rm private_key.pem
```
<br>
If Javascript Project....
<br>

```yml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: backened

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20.13.1"
          cache: "npm"
          cache-dependency-path: backened/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Create deployment package (without node_modules)
        run: |
          mkdir -p deploy
          cp -r src deploy/
          cp package*.json deploy/
          cp ecosystem.config.cjs deploy/
          [ -f .env.example ] && cp .env.example deploy/ || echo "No .env.example found"
          tar -czf deploy.tar.gz -C deploy .

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-package
          path: backened/deploy.tar.gz
          retention-days: 1

  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-package

      - name: Deploy to EC2 via SSH
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          scp -i private_key.pem -o StrictHostKeyChecking=no \
            deploy.tar.gz ${EC2_USERNAME}@${EC2_HOST}:/tmp/

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'

            cd /var/www/express-app

            sudo chown -R $USER:$USER /var/www/express-app

            if [ -d "src" ]; then
              timestamp=$(date +%Y%m%d_%H%M%S)
              mkdir -p backups
              tar -czf backups/backup_${timestamp}.tar.gz src package.json ecosystem.config.cjs .env 2>/dev/null || true
              ls -t backups/backup_*.tar.gz 2>/dev/null | tail -n +6 | xargs -r rm
            fi

            tar --overwrite -xzf /tmp/deploy.tar.gz -C /var/www/express-app
            rm /tmp/deploy.tar.gz

            sudo chown -R $USER:$USER /var/www/express-app

            npm ci

            if [ ! -f .env ]; then
              echo "NODE_ENV=production" > .env
              echo "PORT=8000" >> .env
            fi

            mkdir -p logs

            if pm2 describe express-app > /dev/null 2>&1; then
              sudo pm2 reload ecosystem.config.cjs --update-env
            else
              sudo pm2 start ecosystem.config.cjs
            fi

            sudo pm2 save

          EOF

          rm private_key.pem

      - name: Verify deployment
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
        run: |
          sleep 10

          for i in {1..2}; do
            response=$(curl -s -o /dev/null -w "%{http_code}" http://${EC2_HOST} 2>/dev/null || echo "000")

            if [ "$response" = "200" ] || [ "$response" = "301" ] || [ "$response" = "302" ]; then
              echo "Deployment successful: $response"
              exit 0
            fi

            sleep 5
          done

          echo "Deployment verification failed"
          exit 1

      - name: Rollback on failure
        if: failure()
        env:
          EC2_HOST: ${{ secrets.EC2_HOST }}
          EC2_USERNAME: ${{ secrets.EC2_USERNAME }}
          SSH_PRIVATE_KEY: ${{ secrets.EC2_SSH_KEY }}
        run: |
          echo "$SSH_PRIVATE_KEY" > private_key.pem
          chmod 600 private_key.pem

          ssh -i private_key.pem -o StrictHostKeyChecking=no \
            ${EC2_USERNAME}@${EC2_HOST} << 'EOF'

            cd /var/www/express-app
            sudo chown -R $USER:$USER /var/www/express-app

            latest_backup=$(ls -t backups/backup_*.tar.gz 2>/dev/null | head -1)

            if [ -n "$latest_backup" ]; then
              tar -xzf "$latest_backup" -C /var/www/express-app
              npm ci
              pm2 reload ecosystem.config.cjs
            fi

          EOF

          rm private_key.pem  
```

## If you want to change http to https domain then go by the following steps.

### Step 1.
Get Domain first then update your domain current ip with public ip which you have on aws.<br><br>
<b>And in security tab do this
| Type  | Port | Source    |
| ----- | ---- | --------- |
| HTTPS | 443  | 0.0.0.0/0 |


### Step2. 
 sudo nano /etc/nginx/sites-available/express-app
 
 ```bash
 server {
    listen 80;
    server_name api-amreshhhhhh.duckdns.org; #your domain name

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

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```
### Step 3.

sudo nginx -t <br>

sudo systemctl restart nginx

sudo certbot --nginx -d api-amreshhhhhh.duckdns.org (domain name)
