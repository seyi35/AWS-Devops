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
### 1. **Create Key Pairs**  
   - Used for secure SSH access into EC2 instances.
     
     <img width="1392" height="647" alt="key pair" src="https://github.com/user-attachments/assets/3629fdaf-7ad8-4e33-81d5-1663959f9cb7" />

### 2. **Create Security Groups**  
   - Create ELB-SG first, then configure Inbound rules to Allow both HTTP and HTTPS on port 80 and 443 respectively from Anywhere IPv4 and IPv6.
     <img width="1827" height="630" alt="ELG SG" src="https://github.com/user-attachments/assets/19fee883-35bc-4220-b0d0-9a2955dfc0f8" />
    
   - Next, create vprofile-app-SG. Open port 8080 to accept connections from ELB-SG and ssh connection from "my ip".
     <img width="1835" height="636" alt="APP SG" src="https://github.com/user-attachments/assets/70c81b17-4bcc-4e19-be68-8832da77af07" />

   - Finally, create backend-SG and open port 3306 for MySQL, 11211 for Memcached, 5672 for RabbitMQ server ssh connection from "my ip" just incase you need to troubleshoot something.
     We also need to open communication within the SecGrp for backend services to communicate with each other.
     <img width="1841" height="637" alt="Backend SG" src="https://github.com/user-attachments/assets/b2b0a410-888e-4bef-bbd7-8ec0b1869bb3" />


### 3. **Provision Backend & Application EC2 Instances with UserData**  
 ### DB Instance:
   - Create DB instance with below details. 
      
               Name: vprofile-db01
               Project: vprofile
               AMI: Amazon linux 2023
               InstanceType: t2.micro
               SecGrp: vprofile-backend-SG
               UserData: mysql.sh
   - Once instance is ready, SSH into the server and check if userdata script is executed also, check status of mariadb.
     
                 ssh -i vprofile-prod-key.pem ec2-user@<public_ip_of_instance>
                 sudo -i 
                 systemctl status mariadb
     
     <img width="898" height="265" alt="Mariadb service running" src="https://github.com/user-attachments/assets/d095e167-7e9c-4332-b069-53731e112f10" />

   - Lets also check the Database. `mysql -u admin -p accounts` 
  
     <img width="752" height="530" alt="database " src="https://github.com/user-attachments/assets/9f210be5-69fb-49c1-a876-ff857c832e9b" />
  ### Memcached Instance:
   - Create Memcached instance with below details.
     
         Name: vprofile-mc01
         Project: vprofile
         AMI:  Amazon linux 2023
         InstanceType: t2.micro
         SecGrp: vprofile-backend-SG
         UserData: memcache.sh
   - Once instance is ready, SSH into the server and check if userdata script is executed, also check status of memcache service.
     
         ssh -i vprofile-prod-key.pem ec2-user@<public_ip_of_instance>
         sudo -i 
         systemctl status memcached.service
     <img width="936" height="312" alt="memcache service" src="https://github.com/user-attachments/assets/3db999df-278f-4b40-9ddf-47cc6046bebc" />
  ### RabbitMQ Instance:
   - Create RabbitMQ instance with below details.

            Name: vprofile-rmq01
            Project: vprofile
            AMI: Amazon Linux 2023
            InstanceType: t2.micro
            SecGrp: vprofile-backend-SG
            UserData: rabbitmq.sh
   - Once instance is ready, SSH into the server and check the status of rabbitmq service.
     
         ssh -i vprofile-prod-key.pem ec2-user@<public_ip_of_instance>
         sudo -i
         systemctl status rabbitmq-server
      <img width="1293" height="448" alt="rmq service running" src="https://github.com/user-attachments/assets/dbe814a8-e4d4-4d67-a841-86a299910778" />
  
  ### Application (Tomcat) Instance:
   - Create Tomcat instance with below details.

         Name: vprofile-app01
         Project: vprofile
         AMI: Ubuntu 24
         InstanceType: t2.micro
         SecGrp: vprofile-app-SG
         UserData: tomcat_ubuntu.sh
   - Once instance is ready, SSH into the server and check the status of tomcat10 service.
        
         ssh -i vprofile-prod-key.pem ubuntu@<public_ip_of_instance>
         sudo -i
         systemctl status tomcat10

### 4. **Configure Route53 Private Hosted Zone**  
   - Our backend stack is running. Next we will created a hosted zone `vprofile.in` and update the Private IPs of our backend services in Route53 Private DNS Zone to enable service-to-service communication using hostnames . Lets note down Private IP addresses.
     
         rmq01 172.31.27.36
         db01 172.31.31.90
         mc01 172.31.23.70
   - Create vprofile.in Private Hosted zone in Route53. We will pick Default VPC in N.Virginia region.

     <img width="1865" height="622" alt="route 53" src="https://github.com/user-attachments/assets/c2c6d810-4d52-4550-926b-f5108dfba200" />
     
   - Create records for our backend services. The purpose of this activity is to also record names in our application.properties file.
     
         Simple Routing -> Define Simple Record
         Value/Route traffic to: IP address or another value    
     <img width="1252" height="623" alt="route 53 records" src="https://github.com/user-attachments/assets/fac0aa6a-5b8d-4c57-9831-41c1f7cc6865" />
     
   - SSH into the App server and pinged all 3 servers `ping -c 4 db01.vprofile.in` to ensure communication has been established.
     
     <img width="1016" height="702" alt="Verify that the IPs are been resolved" src="https://github.com/user-attachments/assets/3d1b70f0-fb9b-4721-aae0-480e2fdfb80c" />


### 5. **Build Artifact locally with Maven**  
   - Clone the repository.
     
         git clone https://github.com/hkhcoder/vprofile-project.git
      
   - Before we create our artifact, we need to make changes to our application.properties file under /src/main/resources directory for below lines.
     
         jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&
         memcached.active.host=mc01.vprofile.in
         rabbitmq.address=rmq01.vprofile.in
   - We will go to the vprofile-project root directory where the pom.xml file exists. Then we will execute below command to create our artifact vprofile-v2.war:

           mvn install
     <img width="738" height="121" alt="mvn install" src="https://github.com/user-attachments/assets/e67902c4-3fe7-4975-8e2f-796d861d52b7" />
     <img width="1430" height="245" alt="mvn install target" src="https://github.com/user-attachments/assets/7fa7f88c-063c-44bf-bd28-286fa7b23b8e" />
     
   - After you run the command, a target found will be created in it the vprofile-v2.war file as you can see above.


### 6. **Upload Artifact to S3**  
   - We will upload our artifact to s3 bucket from AWS CLI, and our App/Tomcat server will get the same artifact from the S3 bucket.
   - We will also create an IAM user for authentication to be used from AWS CLI.
     
     <img width="1320" height="575" alt="IAM USER" src="https://github.com/user-attachments/assets/dcea456c-861d-4b6e-83f1-53e7677f4e6f" />
     
   - Next, we will configure our AWS CLI to use IAM user credentials.
     
         aws configure
         AccessKeyID: 
         SecretAccessKey:
         region: us-east-1
         format: json
   - Create a bucket. Note: S3 buckets are global, so the naming must be UNIQUE!

           aws s3 mb s3://vprofile-artifacts-storev2
   - Go to the target directory (or use absolute) path and copy the artifact to the bucket with below command. Then verify by listing objects in the bucket.

         aws s3 cp vprofile-v2.war s3://vprofile-artifacts-storage-rd
         aws s3 ls vprofile-artifact-storage-rd 
           
     <img width="637" height="67" alt="artifact push to s3" src="https://github.com/user-attachments/assets/4927cdcf-099b-4b6c-9716-68df5f16809d" />

   - We can verify the same from the AWS Console.

     <img width="1246" height="568" alt="bucket creation, delete later" src="https://github.com/user-attachments/assets/64bc5947-6b13-4681-8d83-574220a602fa" />


### 7. **Deploy Artifact on Tomcat Server**  
   - To download our artifact onto the Tomcat server, we need to create IAM role for Tomcat. Once the role is created, we will attach it to our app01 server.  

         Type: EC2
         Name: s3-admin
         Policy: s3FullAccess

   - Before we log in to our server, we need to ensure that SSH access on port 22 has been added to our vprofile-app-SG.
   - Then connect to app011 Ubuntu server.

         ssh -i "vprofile-prod-key.pem" ubuntu@<public_ip_of_server>
         sudo su -
         systemctl stop tomcat10
   - We will delete the ROOT (where the default Tomcat app files are stored) directory under /var/lib/tomcat10/webapps/

         cd /var/lib/tomcat10/webapps/
         rm -rf ROOT
   - Next, we will deploy our artifact from S3 using AWS CLI commands. First, we need to install AWS CLI. We will initially download our artifact to the /tmp directory, then we will copy it under /var/lib/tomcat10/webapps/ directory as ROOT.war. Since this is the default app directory, Tomcat will extract the compressed file.
     
         snap install aws-cli --classic
         aws s3 ls s3://vprofile-artifacts-storev2
         aws s3 cp s3://vprofile-artifacts-storev2/vprofile-v2.war /tmp/vprofile-v2.war
         cp /tmp/vprofile-v2.war /var/lib/tomcat10/webapps/ROOT.war
         systemctl start tomcat10
     <img width="560" height="127" alt="restart tomcat service" src="https://github.com/user-attachments/assets/1fba858a-0486-4daa-8c9a-1af8b24bc2d7" />

   - As you can see, after the service was restarted, the compressed file was extracted.
   - We can also verify the application.properties file has the latest changes.
     
         cat /var/lib/tomcat10/webapps/ROOT/WEB-INF/classes/application.properties

     
### 8. **Setup Elastic Load Balancer (ELB)**  
   - Before creating LoadBalancer, first we need to create a Target Group.
     
            Instances
            Target Grp Name: vprofile-elb-tg
            protocol-port: HTTP:8080
            healthcheck path: /
            Advanced health check settings
            Override: 8080
            Healthy threshold: 5
            Register targets
            Available instance: app01 (Include as pending below)
   
   - Now we will create our Load Balancer.
     
            vprofile-prod-elb
            Application Load Balancer
            Internet Facing
            Select all AZs
            SecGrp: vprofile-elb-secGrp
            Listeners: HTTP, HTTPS
            Select the certificate for HTTPS
            Target Group: vprofile-elb-tg

### 9. **Application Verification**  
   - Accessed application via Elastic Load Balancer endpoint(public URL).
     
       <img width="1543" height="896" alt="Login page" src="https://github.com/user-attachments/assets/12228c0d-1024-4195-9fbc-9f55d1240018" />
       <img width="1817" height="836" alt="Login page 2" src="https://github.com/user-attachments/assets/131971cd-92aa-4797-9991-fa60ed3eca51" />


### 10. **Configure Auto Scaling Group (ASG)**  
   - We will create an AMI from our App Instance.
       <img width="1682" height="677" alt="AMI " src="https://github.com/user-attachments/assets/979e997a-79c5-4d3b-ac57-bbaefc60d98b" />
       
   - Next we will create a Launch template using the AMI created in above step for our ASG.
     
         Name: vprofile-las-app-LT
         AMI: vprofile-las-app-ami
         InstanceType: t2.micro
         IAM Profile: S3-admin
         SecGrp: vprofile-app-SG
         KeyPair: vprofile-prod-key
     <img width="1832" height="597" alt="Launch Template" src="https://github.com/user-attachments/assets/4e06438b-7f06-4c22-97ad-8793189b94e3" />

   - Our Launch template is ready, now we can create our ASG.
     
         Name: vprofile-app-ASG
         ELB healthcheck
         Add ELB
         Min:1
         Desired:2
         Max:4
         Target Tracking-CPU Utilization 50
   - If we terminate any instances we will see ASG will create a new one using LT that we created.
     
     <img width="1524" height="472" alt="AUTOSCALING (2)" src="https://github.com/user-attachments/assets/39a7912a-e6ed-4595-ae34-d709a6bf227e" />
     <img width="1537" height="447" alt="Autoscaling" src="https://github.com/user-attachments/assets/92e409b2-71f7-4711-8f01-231e49a87cff" />

### 11. **Clean-up**  
   - Deleted resources: ASG, ELB, EC2 instances, Route53, S3 bucket, IAM roles, and Key Pairs.  

---

## ⚡ Challenges Faced
- Forgot `#!/bin/bash` in UserData → scripts didn’t execute.  
- Incorrect Security Group outbound rules → blocked communication between backend services.  
- Using public IPs in `application.properties` instead of Route53 hostnames caused failures when IPs changed.  
- Bucket naming issues → S3 bucket names must be **globally unique**.  

---

## ✅ Lessons Learned  
- Route53 Private Hosted Zones simplify service discovery.  
- Auto Scaling requires ELB health checks for reliable recovery.  
- Always test UserData scripts by checking service status after provisioning.  

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



