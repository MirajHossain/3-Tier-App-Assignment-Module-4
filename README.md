# 3-Tier Application Deployment on AWS EC2 (Module 4 Assignment)

Project Overview
This project demonstrates a 3-tier architecture deployment on AWS using EC2 instances.

Architecture Layers:
- Presentation Layer → Nginx Web Server
- Application Layer → Backend Application
- Data Layer → PostgreSQL Database

AWS EC2 Instances:
nginx-server → Web Server (Ubuntu 26.04)
app-server → Backend (Ubuntu 26.04)
db-server → Database (Ubuntu 26.04)

Security Groups:
nginx: SSH 22, HTTP 80
app: SSH 22, TCP 3000 from nginx
db: SSH 22, PostgreSQL 5432 from app

Technologies Used:
AWS EC2, Ubuntu, Nginx, Node.js, PostgreSQL, Git

Deployment Steps:
1. Create EC2 instances
2. Configure security groups
3. Setup PostgreSQL on db-server
4. Deploy backend on app-server
5. Configure Nginx reverse proxy

Screen Shoot:
# Ec2 Instance Running
<img width="1596" height="482" alt="ec2_running" src="https://github.com/user-attachments/assets/7fc5ab15-6980-4d80-88d3-4059e3d44198" />


Author:
Miraj Hossain

