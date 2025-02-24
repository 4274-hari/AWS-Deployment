# AWS-Deployment
Deployment Guide: React Frontend & Node.js Backend on AWS EC2 with MongoDB & S3

# Overview

This guide outlines the steps to deploy a React frontend and a Node.js/Express backend on an AWS EC2 instance running Amazon Linux 2023. The backend connects to MongoDB and serves the frontend via Nginx, with an S3 bucket used for static assets.

# Prerequisites

-AWS EC2 instance (Amazon Linux 2023)

-MongoDB installed on EC2

-AWS S3 bucket for storing assets

-Node.js & npm installed

-PM2 for process management

-Nginx for reverse proxy

-Domain name (optional) for configuring HTTPS

# Step 1: Set Up EC2 Instance

  1.Launch an EC2 instance (Amazon Linux 2023 recommended).

  2.Configure security groups to allow:

        SSH (port 22)
        HTTP (port 80)
        HTTPS (port 443)
        Custom port (e.g., 5000 for API) *custom port should be configure after running your backend in instance.

  3.Enter into root user:

        sudo su 

  4.Update packages:

        sudo dnf update -y


# Step 2: Install Node.js, npm, PM2, and git

    curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
    sudo dnf install -y nodejs
    node -v   # Verify Node.js installation
    npm -v    # Verify npm installation
    sudo npm install -g pm2
    sudo dnf install -y git

# Step 3: Install and Configure MongoDB

  *Install MongoDB from the Official MongoDB Repository

  1. If you have already added the mongodb-repo run this to remove it:

             sudo rm -f /etc/yum.repos.d/mongodb*.repo

  (or)     
  

  1. If you are adding the MongoDB Repository for the first time 

            cat <<EOF | sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo
            [mongodb-org-7.0]
            name=MongoDB Repository
            baseurl=https://repo.mongodb.org/yum/amazon/2023/mongodb-org/7.0/x86_64/
            gpgcheck=1
            enabled=1
            gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
            EOF

  2. Install MongoDB

            sudo dnf install -y mongodb-org

  3.Enable and Start the MongoDB Service

            sudo systemctl enable mongod
            sudo systemctl start mongod
 
  4.Verify the Installation

            mongod --version

# Step 4: Clone Repository and Install Dependencies

    git clone https://github.com/your-repo.git
    cd your-repo
    npm install

  Start the backend with PM2:
    
        pm2 start server.js
        pm2 save
        pm2 startup

# Step 5: Build and Deploy React Frontend

  1.If you not have enough memory to build the app, you need to create a swapfile for memory based on your need.

        sudo fallocate -l 4G /swapfile  # Create a 4GB file
        sudo chmod 600 /swapfile        # Set correct permissions
        sudo mkswap /swapfile           # Format it as swap
        sudo swapon /swapfile           # Enable swap

  Checking Swap Status

        swapon --show
        free -h

  2.Navigate to the React app folder:

        cd client
        npm install
        NODE_OPTIONS="--max-old-space-size=4096" npm run build

  3. Copy build files to the Nginx root directory:

         sudo mkdir -p /var/www/html

         mv from_build_path /var/www/html/ #(or)
     
         sudo cp -r build/* /var/www/html/

  5. After build remove the swapfile 

         sudo swapoff -a

         sudo rm -f /swapfile


 (or)

  1.If you have enough memory then Navigate to the React app folder:

        cd client
        npm install
        npm run build

  2. Copy build files to the Nginx root directory:

         sudo cp -r build/* /var/www/html/

  3. If you are using Backend as seperate file move the folder

         mv from_directory /var/www/
         cd /var/www/Backend

        Start the backend with PM2:
    
            pm2 start server.js
            pm2 save
            pm2 startup


# Step 6: Configure Nginx as Reverse Proxy

  1. Install Nginx:

         sudo dnf install nginx -y 

  2. Modify Nginx configuration:

         sudo nano /etc/nginx/nginx.conf

        Replace contents with:

           server {
                 listen 80;
                 server_name your-domain.com;
             location / {
                 root /var/www/html;
                 index index.html index.htm;
                 try_files $uri /index.html;
              }

             location /api/ {
                proxy_pass http://localhost:5000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
             }
             location /uploads/ {
                alias /mnt/s3bucket/uploads/;
                autoindex on;
                allow all;
            }

         }

  3. Restart Nginx:

         sudo systemctl restart nginx

# Step 7: Configure AWS S3 for Static Assets

  1.Create an S3 bucket in AWS.

  2.Upload files via AWS CLI:
      
       aws s3 cp uploads s3://your-bucket-name/ --recursive

  3.Get AWS Credentials:
  Create an Access Key from AWS IAM:
            1.Go to AWS IAM Console → Users → Your User.
            2.Click Security Credentials → Create Access Key.
            3.Copy Access Key ID and Secret Access Key.

  Now, store your credentials in a file:

            echo "AWS_ACCESS_KEY_ID:AWS_SECRET_ACCESS_KEY" > ~/.passwd-s3fs
            chmod 600 ~/.passwd-s3fs

  4.Mount the S3 Bucket 

        git clone https://github.com/s3fs-fuse/s3fs-fuse.git
        cd s3fs-fuse

  Create a mount point for the S3 bucket:

            sudo mkdir -p /mnt/s3bucket

  Mount the S3 bucket:

            s3fs your-bucket-name /mnt/s3bucket -o passwd_file=~/.passwd-s3fs -o allow_other

  To make it persistent on reboot, add this line to /etc/fstab:

            your-bucket-name /mnt/s3bucket fuse.s3fs _netdev,allow_other,use_cache=/tmp,passwd_file=~/.passwd-s3fs 0 0
            
  5.Give permission 

        sudo chmod -R 755 /mnt/s3bucket/uploads  # Read & execute for others, but only write for owner
        sudo chown -R nginx:nginx /mnt/s3bucket/uploads  # Give ownership to the Nginx server (if using Nginx)


# Step 8: Enable SSL (Optional)

  Use Certbot to enable HTTPS:

       sudo dnf install certbot python3-certbot-nginx -y
       sudo certbot --nginx -d your-domain.com

# Step 9: Monitoring and Maintenance

  1. Check logs:

         pm2 logs

  2. Restart the backend:

         pm2 restart backend-app

  3. Check Nginx status:

         sudo systemctl status nginx

# Conclusion
 
  Your React app with a Node.js backend is now successfully deployed on an AWS EC2 instance using MongoDB and an S3 bucket. 🚀
