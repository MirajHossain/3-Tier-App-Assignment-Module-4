# AWS 3-Tier Full Deployment Guide (VPC + EC2 + Nginx + Frontend)
Project Repository
https://github.com/MirajHossain/3-Tier-App-Assignment-Module-4.git
________________________________________
# 1. AWS Network Setup (VPC Architecture)
1.1 Create VPC
•	Name: 3tier-vpc
•	IPv4 CIDR: 10.0.0.0/16
•	Tenancy: Default
________________________________________
1.2 Create Subnets
Public Subnet (Web Tier)
•	CIDR: 10.0.1.0/24
•	AZ: us-east-1a
Private Subnet (App + DB Tier)
•	CIDR: 10.0.2.0/24
•	AZ: us-east-1b
________________________________________
1.3 Internet Gateway
•	Create IGW
•	Attach to 3tier-vpc
________________________________________
1.4 Route Table
Public Route Table:
0.0.0.0/0 → Internet Gateway
Associate with Public Subnet
________________________________________
 # 2. Security Groups
Web Server SG
•	SSH (22) → Your IP
•	HTTP (80) → 0.0.0.0/0
•	Backend (3000) → Open or App SG
App Server SG
•	SSH (22) → Web SG only
•	Port 3000 → Web SG only
DB Server SG
•	PostgreSQL (5432) → App SG only
________________________________________
 # 3. EC2 Setup (AWS)
Web Server (Frontend)
•	Ubuntu 24.04
•	Public Subnet
•	Web SG
•	Public IP enabled
App Server (Backend)
•	Ubuntu 24.04
•	Private Subnet
•	App SG
•	No public IP
DB Server (PostgreSQL)
•	Private Subnet
•	DB SG
•	No public IP
________________________________________
 # 4. Architecture Flow
User → Nginx (Web EC2)
          ↓
     Node.js (App EC2)
          ↓
     PostgreSQL (DB EC2)
________________________________________
 # 5. Frontend Deployment (Nginx)
5.1 Connect to Web EC2
ssh -i key.pem ubuntu@<EC2-IP>
5.2 Clone Project
cd /home/ubuntu
git clone https://github.com/MirajHossain/3-Tier-App-Assignment-Module-4.git
5.3 Frontend Build
cd frontend
sudo chown -R ubuntu:ubuntu .
npm install
npm run build
5.4 Deploy to Nginx
sudo cp -r dist/* /var/www/html/
________________________________________
 # 6. Nginx Configuration
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;
    index index.html index.htm;

    server_name _;

    location / {
        try_files $uri /index.html;
    }
}
________________________________________
# Restart Nginx
sudo nginx -t
sudo systemctl restart nginx
________________________________________
# 7. Backend Setup (Node.js)
cd backend
npm install
npm run dev
.env
PORT=3000
DATABASE_URL=postgresql://user:pass@db-private-ip:5432/bmidb
________________________________________
 # 8. Database Setup (PostgreSQL)
•	Install PostgreSQL on DB EC2
•	Create DB + user
•	Allow only App SG access
________________________________________
 # 9. Common Issues
403 Forbidden
•	Empty /var/www/html
•	Missing index.html
500 Error
•	Wrong nginx config
•	syntax error
npm EACCES
sudo chown -R ubuntu:ubuntu project
vite not found
npm install
________________________________________
 # Final Result
•	Web: Nginx + React
•	App: Node.js API
•	DB: PostgreSQL
✔ Full 3-tier architecture working ✔ Secure private DB ✔ Public web access
________________________________________
# Frontend Deployment Guide (React + Vite + Nginx on AWS EC2)
This guide includes FULL steps from SSH login to deployment of frontend on Nginx.
________________________________________
# 0. SSH into EC2 (First Step)
Windows (Git Bash / CMD)
chmod 400 /c/Users/Administrator/Downloads/mh-key-pair.pem
ssh -i "/c/Users/Administrator/Downloads/mh-key-pair.pem" ubuntu@13.126.148.73
Linux / Mac
chmod 400 mh-key-pair.pem
ssh -i mh-key-pair.pem ubuntu@13.126.148.73
________________________________________
# 1. System Update
sudo apt update -y
sudo apt upgrade -y
________________________________________
# 2. Install Nginx
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
Test:
curl http://localhost
________________________________________
# 3. Install Node.js (v18)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
Check:
node -v
npm -v
________________________________________
# 4. Clone Project
cd /var/www/html
sudo git clone https://github.com/MirajHossain/3-Tier-App-Assignment-Module-4.git
________________________________________
# 5. Set Permissions
sudo chown -R ubuntu:ubuntu /var/www/html/3-Tier-App-Assignment-Module-4
________________________________________
# 6. Go to Frontend
cd /var/www/html/3-Tier-App-Assignment-Module-4/frontend
________________________________________
# 7. Install Dependencies
npm install
________________________________________
# 8. Build Project
npm run build
# Output: dist/ folder will be created
________________________________________
# 9. Deploy to Nginx
sudo rm -rf /var/www/html/*
sudo cp -r dist/. /var/www/html/
________________________________________
# 10. Set Permissions
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
________________________________________
 # 11. Restart Nginx
sudo nginx -t
sudo systemctl restart nginx
________________________________________
 # 12. Access Application
http://13.126.148.73/
________________________________________



Screen Shoot:
# Ec2 Instance Running
<img width="1141" height="525" alt="WhatsApp Image 2026-05-10 at 4 02 56 PM" src="https://github.com/user-attachments/assets/d5f1f131-d209-4a91-985c-ad0a6251d32e" />
# Nginx Running
<img width="1905" height="456" alt="Nginx_Up_Running" src="https://github.com/user-attachments/assets/c700da26-b417-4adf-a303-54bfab35af9b" />
# Front End (Web Server) Running
<img width="1522" height="598" alt="Front End Running" src="https://github.com/user-attachments/assets/77eb8f2c-ca90-4049-9681-3fdb5dec162b" />
# App Sever Running
<img width="1204" height="1600" alt="WhatsApp Image 2026-05-10 at 5 26 10 PM" src="https://github.com/user-attachments/assets/ccd769a8-d251-49bc-8ade-1f4630666757" />






Author:
Miraj Hossain

