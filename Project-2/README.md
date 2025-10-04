# 🚀 Lift and Shift Project on AWS

This project demonstrates a **Lift and Shift migration** of a Java-based web application to the AWS Cloud.  
The goal was to replicate a traditional on-premise stack by provisioning backend services, deploying the application to EC2 instances, and managing high availability with Load Balancers and Auto Scaling Groups.

---

## 📌 Project Overview
- **Cloud Provider:** AWS  
- **Architecture Style:** Lift and Shift (IaaS approach)  
- **Application Stack:** Java, Tomcat, Maven  
- **Backend Services:** MariaDB, RabbitMQ, Memcached  
- **Key AWS Services:** EC2, S3, IAM, Route53, ELB, ASG  

---
Architecture on DataCenter:

<img width="1096" height="562" alt="Architecture" src="https://github.com/user-attachments/assets/c24aaf84-de72-4556-8523-605198f1c8f7" />


Architecture on AWS:

<img width="790" height="463" alt="Architecture on AWS" src="https://github.com/user-attachments/assets/397a4a91-de8b-4f7b-bfff-955b005856d9" />

---

## 🛠️ Flow of Execution
1. **Create Key Pairs**  
   - Used for secure SSH access into EC2 instances.
     
     <img width="1392" height="647" alt="key pair" src="https://github.com/user-attachments/assets/3629fdaf-7ad8-4e33-81d5-1663959f9cb7" />

2. **Create Security Groups**  
   - Create ELB-SG first, then configure Inbound rules to Allow both HTTP and HTTPS on port 80 and 443 respectively from Anywhere IPv4 and IPv6.
     <img width="1827" height="630" alt="ELG SG" src="https://github.com/user-attachments/assets/19fee883-35bc-4220-b0d0-9a2955dfc0f8" />
    
   - Next, create vprofile-app-SG. Open port 8080 to accept connections from ELB-SG and ssh connection from "my ip".
     <img width="1835" height="636" alt="APP SG" src="https://github.com/user-attachments/assets/70c81b17-4bcc-4e19-be68-8832da77af07" />

   - Finally, create backend-SG and open port 3306 for MySQL, 11211 for Memcached, 5672 for RabbitMQ server ssh connection from "my ip" just incase you need to troubleshoot something.
     We also need to open communication within the SecGrp for backend services to communicate with each other.
     <img width="1841" height="637" alt="Backend SG" src="https://github.com/user-attachments/assets/b2b0a410-888e-4bef-bbd7-8ec0b1869bb3" />


3. **Provision Backend & Application EC2 Instances with UserData**  
   DB Instance:
   - Create DB instance with below details. 
      
               Name: vprofile-db01
               Project: vprofile
               AMI: Amazon linux 2023
               InstanceType: t2.micro
               SecGrp: vprofile-backend-SG
               UserData: mysql.sh
   - Once instance is ready, SSH into the server and check if userdata script is executed also, check status of mariadb.
     
                 ssh -i vprofile-prod-key.pem ubuntu@<public_ip_of_instance>
                 sudo su 
                 systemctl status mariadb
     
     <img width="898" height="265" alt="Mariadb service running" src="https://github.com/user-attachments/assets/d095e167-7e9c-4332-b069-53731e112f10" />

   - Lets also check the Database. `mysql -u admin -p accounts` 
  
     <img width="752" height="530" alt="database " src="https://github.com/user-attachments/assets/9f210be5-69fb-49c1-a876-ff857c832e9b" />

  

4. **Configure Route53 Private Hosted Zone**  
   - Created private DNS records (`db01.vprofile.in`, `mc01.vprofile.in`, `rmq01.vprofile.in`)  
   - Enabled service-to-service communication using hostnames instead of IPs.  

5. **Build Artifact with Maven**  
   - Generated `.war` file locally (`vprofile-v2.war`).  
   - Updated `application.properties` with backend hostnames.  

6. **Upload Artifact to S3**  
   - Created S3 bucket: `s3://vprofile-artifacts-storev2`.  
   - Uploaded artifact using AWS CLI.  
   - Configured IAM Role for EC2 → S3 access.  

7. **Deploy Artifact on Tomcat Server**  
   - Pulled artifact from S3 inside EC2 instance.  
   - Deployed as `ROOT.war` in Tomcat webapps directory.  

8. **Setup Elastic Load Balancer (ELB)**  
   - Created Target Group for app instances.  
   - Configured ALB to forward HTTP traffic.  

9. **Application Verification**  
   - Accessed application via ELB endpoint.  

10. **Configure Auto Scaling Group (ASG)**  
    - Created AMI from application server.  
    - Built Launch Template & ASG (min=1, desired=2, max=4).  
    - Configured scaling policy (CPU utilization at 50%).  

11. **Clean-up**  
    - Deleted resources: ASG, ELB, EC2 instances, Route53, S3 bucket, IAM roles, and Key Pairs.  

---

## 📸 Documentation Capture
To make this guide practical and repeatable:
- **Screenshots:**  
  - Key Pairs, Security Groups, EC2 setup, Route53 records, S3 uploads, ELB health checks, ASG scaling events.  
- **Diagrams:**  
  - High-level architecture (Client → ELB → App → Backend).  
  - Traffic flow between services (SG rules).  

---

## ⚡ Challenges Faced
- Forgot `#!/bin/bash` in UserData → scripts didn’t execute.  
- Incorrect Security Group outbound rules → blocked communication between backend services.  
- Using public IPs in `application.properties` instead of Route53 hostnames caused failures when IPs changed.  
- Bucket naming issues → S3 bucket names must be **globally unique**.  

---

## ✅ Lessons Learned
- IAM Roles > Access Keys (safer for EC2-S3 communication).  
- Route53 Private Hosted Zones simplify service discovery.  
- Auto Scaling requires ELB health checks for reliable recovery.  
- Always test UserData scripts by checking service status after provisioning.  

---

## 🖼️ Architecture Diagram
*(Insert diagram here — e.g., draw.io, Lucidchart, or Excalidraw)*  

---

## 📂 Repository Structure
```bash
├── scripts/
│   ├── mysql.sh
│   ├── memcached.sh
│   ├── rabbitmq.sh
│   ├── tomcat_ubuntu.sh
├── docs/
│   ├── screenshots/
│   ├── architecture-diagram.png
├── README.md

