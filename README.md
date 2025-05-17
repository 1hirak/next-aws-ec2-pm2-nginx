## Complete Guide to Setting Up a Next.js Application on AWS EC2 with Nginx and SSL

This guide provides step-by-step instructions to deploy a Next.js application on an AWS EC2 instance, configure Nginx as a reverse proxy, secure it with SSL using Certbot, and manage the application with PM2.

### Prerequisites

- **AWS EC2 Instance:** Launched with Ubuntu.
- **Domain Name:** Registered and configured (e.g., `www.domain.com`).
- **GitHub Repository:** Contains your Next.js application (e.g., `git@github.com:username/nextjs-app.git`).
- **Key Pair:** Downloaded and available for SSH (e.g., `keypair.pem`).

---

### Step 1: Set Up the EC2 Instance

1. **Launch an EC2 Instance:**
   
   - Use an Ubuntu AMI (e.g., Ubuntu 20.04 LTS).
   - Select an instance type (e.g., `t2.micro` for testing).
   - Download the key pair file (e.g., `keypair.pem`) during setup.

2. **Get the Public IP:**
   
   - Check the public IP in the AWS EC2 dashboard or run:
     
     ```bash
     curl ifconfig.me
     ```

3. **Configure Security Groups:**
   
   - Set inbound rules:
     - **Port 22 (SSH):** Restrict to your IP (e.g., `your-ip/32`).
     - **Port 80 (HTTP):** Allow from `0.0.0.0/0`.
     - **Port 443 (HTTPS):** Allow from `0.0.0.0/0`.

---

### Step 2: Set Up Domain and Hosted Zones

1. **Configure DNS:**
   - Use AWS Route 53 or your domain registrar.
   - Create a hosted zone for `domain.com`.
   - Add an **A Record** pointing `www.domain.com` to the EC2 public IP (`99.98.97.96`).

---

### Step 3: Connect to the EC2 Instance

1. **Navigate to Key Pair Folder:**
   
   - On your local machine:
     
     ```bash
     cd /path/to/keypair/folder
     ```

2. **SSH into the Instance:**
   
   - Connect using:
     
     ```bash
     ssh -i keypair.pem ubuntu@99.98.97.96
     ```
   
   - Ensure proper permissions:
     
     ```bash
     chmod 400 keypair.pem
     ```

---

### Step 4: Install Required Software

1. **Add Nginx PPA:**
   
   ```bash
   sudo add-apt-repository ppa:ondrej/nginx
   ```

2. **Install Node.js, npm, Nginx, and Dependencies:**
   
   ```bash
   curl -sL https://deb.nodesource.com/setup_20.x | sudo -E bash -
   sudo apt-get install -y nodejs nginx nginx-extras certbot python3-certbot-nginx libnginx-mod-http-brotli-filter libnginx-mod-http-brotli-static
   ```

3. **Install Yarn:**
   
   ```bash
   sudo npm install --global yarn
   ```

4. **Install PM2:**
   
   ```bash
   sudo npm install pm2 -g
   ```

5. **Setup Swap:**
   
   ```bash
   sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile && sudo sed -i '2i/swapfile swap swap defaults 0 0 ' /etc/fstab && sudo sysctl vm.swappiness=10 && sudo sed -i '1ivm.swappiness=10 ' /etc/sysctl.conf && sudo timedatectl set-timezone Asia/Kolkata
   ```

---

### Step 5: Link GitHub to the Instance

To access your private GitHub repository:

1. **Generate an SSH Key Pair:**
   
   ```bash
   ssh-keygen -t rsa -b 4096 -C "email@domain.com"
   ```
   
   - Accept the default location (`~/.ssh/id_rsa`).
   - Skip the passphrase by pressing Enter twice (or set one if desired).

2. **Copy the Public Key:**
   
   ```bash
   cat ~/.ssh/id_rsa.pub
   ```
   
   - Copy the output.

3. **Add the Key to GitHub:**
   
   - Go to GitHub > **Settings** > **SSH and GPG keys** > **New SSH key**.
   - Paste the key, set it as an **Authentication Key**, and save.

---

### Step 6: Clone and Build the Application

1. **Clone the Repository:**
   
   ```bash
   sudo git clone git@github.com:username/nextjs-app.git
   cd nextjs-app
   ```

2. **Install Dependencies and Build:**
   
   ```bash
   sudo yarn install --prefer-online
   sudo yarn build
   ```

---

### Step 7: Run the Application with PM2

1. **Start the Application:**
   
   ```bash
   pm2 start yarn --name "nextjs-app" -- start -- -p 3000
   ```

2. **Test the Application:**
   
   ```bash
   curl http://127.0.0.1:3000
   ```

---

### Step 8: Configure Nginx

1. **Start and Enable Nginx:**
   
   ```bash
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

2. **Create a Dummy SSL Certificate:**
   
   ```bash
   sudo mkdir -p /etc/nginx/ssl
   sudo openssl req -x509 -nodes -days 365 \
     -newkey rsa:2048 \
     -keyout /etc/nginx/ssl/dummy.key \
     -out    /etc/nginx/ssl/dummy.crt \
     -subj "/CN=default"
   ```

3. **Configure the Default Nginx Server:**
   
   - Edit the default config:
     
     ```bash
     sudo nano /etc/nginx/sites-available/default
     ```
   
   - Replace with:
     
     ```nginx
     server {
         listen 80 default_server;
         listen [::]:80 default_server;
         server_name _;
         return 444;
     }
     
     server {
         listen 443 ssl default_server;
         listen [::]:443 ssl default_server;
         server_name _;
         ssl_certificate     /etc/nginx/ssl/dummy.crt;
         ssl_certificate_key /etc/nginx/ssl/dummy.key;
         return 444;
     }
     ```

4. **Link to Sites-Enabled:**
   
   ```bash
   sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
   ```

---

### Step 9: Set Up Nginx for the Application

1. **Create a Configuration File:**
   
   ```bash
   sudo nano /etc/nginx/sites-available/www.domain.com.conf
   ```

2. **Add the Configuration:**
   
   ```bash
   upstream nextjs_upstream_3000 {
    server 127.0.0.1:3000;
   }
   server {
    listen      80;
    server_name www.domain.com;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options            "SAMEORIGIN"                                 always;
    add_header X-Content-Type-Options     "nosniff"                                    always;
    add_header X-XSS-Protection           "1; mode=block"                              always;
    add_header Referrer-Policy            "strict-origin-when-cross-origin"            always;
    add_header Content-Security-Policy      "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'self';" always;
    client_max_body_size 10m;
   
    proxy_set_header Upgrade             $http_upgrade;
    proxy_set_header Connection          "upgrade";
    proxy_set_header Host                $host;
    proxy_set_header X-Real-IP           $remote_addr;
    proxy_set_header X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto   $scheme;
    proxy_cache_bypass  $http_upgrade;
   
    access_log /var/log/nginx/nextjs-app.access.log;
    error_log /var/log/nginx/nextjs-app.error.log;
   
    location / {
        proxy_pass              http://nextjs_upstream_3000;
        proxy_buffer_size       128k;
        proxy_buffers           4 256k;
        proxy_busy_buffers_size 256k;
    }
   
    location ~ /\.(ht|git) {
        deny all;
    }
   
    gzip              on;
    gzip_comp_level   6;
    gzip_types
        text/plain
        text/html
        text/css
        text/xml
        text/javascript
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/rss+xml
        application/atom+xml
        image/svg+xml
        application/font-woff
        application/font-woff2
        application/vnd.ms-fontobject
        font/ttf
        font/otf
        audio/mpeg
        audio/ogg
        audio/mp4
        video/mp4
        video/webm
        video/ogg;
   
    brotli            on;
    brotli_comp_level 6;
    brotli_types
        text/plain
        text/html
        text/css
        text/xml
        text/javascript
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/rss+xml
        application/atom+xml
        image/svg+xml
        application/font-woff
        application/font-woff2
        application/vnd.ms-fontobject
        font/ttf
        font/otf
        audio/mpeg
        audio/ogg
        audio/mp4
        video/mp4
        video/webm
        video/ogg
        application/octet-stream;
   
   }
   ```

3. **Link to Sites-Enabled:**
   
   ```
   sudo ln -s /etc/nginx/sites-available/www.domain.com.conf /etc/nginx/sites-enabled/
   ```

4. **Test and Reload Nginx:**
   
   ```
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

### Step 10: Install SSL with Certbot

1. **Run Certbot:**
   
   ```bash
   sudo certbot
   ```
   
   - Select `www.domain.com` and enable HTTP-to-HTTPS redirection when prompted.

2. **Test Renewal:**
   
   ```bash
   sudo certbot renew --dry-run
   ```

---

### Step 11: Ensure PM2 Restarts on Reboot

1. **Set Up PM2 Startup:**
   
   ```bash
   sudo pm2 save
   sudo pm2 startup
   ```

---

### Extra Notes

---

- **Environment Variables:**
  
  - Add a `.env` file in `nextjs-app/`:
    
    ```
    API_KEY=your_api_key
    DATABASE_URL=your_database_url
    ```
  - Or set in PM2:
    
    ```bash
    pm2 start yarn --name "nextjs-app" -- start --env API_KEY=your_api_key -- -p 3000
    ```

- **PM2 Commands:**
  
  - List processes: `pm2 ls`
  - Monitor: `pm2 monit`
  - View logs: `pm2 logs nextjs-app`
  - View last 100 lines: `pm2 logs nextjs-app --lines 100`
  - Stop all: `pm2 stop all`
  - Delete all: `pm2 delete all`
  - Stop specific: `pm2 stop <id>`
  - Delete specific: `pm2 delete <id>`

- **Nano Shortcuts:**
  
  - `Alt + \`: Go to start.
  - `Ctrl + 6`: Start selection.
  - `Alt + /`: Go to end.
  - `Ctrl + K`: Cut selection.
  - `Alt + 6`: Copy selection.

- **System Configuration:**
  
  - **Recreate Nginx Config:**
    
    ```bash
    sudo rm /etc/nginx/sites-available/www.domain.com.conf
    sudo rm /etc/nginx/sites-enabled/jobs.domain.com.conf
    sudo nano /etc/nginx/sites-available/www.domain.com.conf
    sudo ln -s /etc/nginx/sites-available/www.domain.com.conf /etc/nginx/sites-enabled/
    sudo systemctl reload nginx
    sudo nginx -t
    ```

- **Resource Monitoring:**
  
  - **System Monitor:**
    
    ```bash
    sudo apt-get install bpytop
    bpytop
    ```
  
  - **Storage Monitor:**
    
    ```bash
    sudo apt install ncdu
    cd / && ncdu
    ```

- **Environment Optimization:**
  
  - **Cleanup Node Modules:**
    
    ```bash
    sudo npm install -g npkill
    ```
    
    *OR*
    
    ```bash
    sudo yarn global add npkill
    ```
    
    ```bash
    npkill
    ```
  
  - **Yarn Process Improvement:**
    
    ```bash
    sudo yarn config set enableGlobalCache false
    sudo yarn install && sudo yarn cache clean && cd / && sudo rm -rf ~/.cache/yarn
    sudo yarn install --prefer-online
    ```

- **Version Control:**
  
  - **Git Notes:**
    
    ```bash
    
    git log
    git remote -v
    git remote remove <remote-name>
    git remote add <remote-name> <remote-url>

    git add .
    git commit -m "message"
    git commit -a -m "commit"
    git push origin main
    ```

---
