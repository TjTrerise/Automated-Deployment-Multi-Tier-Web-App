# IaC Deployment of A Multi-Tiered Application


## Project Overview

This project involves designing an automated multi-tiered application that utilises three different tiers to provide high availability. All services are provisioned using IaC, autoscaling groups and launch templates to create and deploy EC2 instances that are split over multiple availbility zones. The public and private tiers are supported by a load balancer (One external, one internal) to manage traffic between and verify system health. The MySQL database is launched on RDS in multi-AZ mode to ensure failover and the entire project is made secure by utilising security groups to prevent traffic into the VPC that is not on the specified ports.



## Features

- IaC to Automate construction of a scalable and highly available multi-tier web application.
- Launch templates and autoscaling groups that automatically maintain and provision working   gunicorn flask applications and a standard apache web servers.
- Database  MySQL database launched in multi-AZ mode to provide failover.
- Application load balancers used to control traffic between external and internal with health checks to confirm instances are healthy.

## Technologies Used

 - VPC: Virtual Private Cloud for network isolation.
 - EC2: Elastic Compute Cloud for virtual servers (Web Tier and App Tier instances).
 - RDS: Relational Database Service (MySQL) for managed database.
 - ALB: Application Load Balancer for traffic distribution.
 - IAM: Identity and Access Management for roles (e.g., Session Manager access).
 - Internet Gateway (IGW): For public subnet internet access.
 - NAT Gateway (NAT GW): For private subnet outbound internet access.
 - Security Groups: For network traffic filtering.
 - Route Tables: For network routing within the VPC.
 - Secrets Manager: Securely generate and store the database credentials
 - Parameter Store: Securly store reusable parameters 
 - Autoscaling: Automatically provision instances.
 

# Data Types Used

* YAML
* JSON

## Getting Started

To get started with deploying this AWS multi-tier Architecture, please ensure you meet the following prerequisites.

### Prerequisites 
 
* Fundamental knowledge of networking concepts such as:
 - IPv4.
 - Subnets.
 - Routing.
* An active AWS account with appropriate permissions to create the required resources.
* Access to the AWS console.
* Able to understand YAML.


## Deployment Steps

### Step 1: Create The CloudFormation Service Role For The Project

1. Allow CloudFormation to assume the role.
2. Allow IAM.
3. Allow EC2.
4. Allow S3.
5. Allow Secrets Manager.
6. Allow System Manager.
7. Allow RDS.
8. Allow Elastic Load Balancing.
9. Allow Autoscaling.
10. Allow CloudWatch

### Step 2: Create Main YAML file.

1. Add mappings for subnet CIDRs.
2. Add a resources section with references to the following stacks:
  - Secrets
  - Iam
  - VPC
  - Security groups
  - RDS
  - App tier launch template
  - Web tier launch template
  - Internal ALB
  - Internal target group
  - Autoscaling group for the app tier
  - External ALB
  - External target group
  - Autoscaling group for the web tier

### Step 3: Create Parameters and secrets Stack YAML File:

 1. Create the database secret.
 2. Create the database name parameter.
 3. Create the database engine parameter.
 4. Output the Database secret. 


### Step 4: Create The IAM Role Stack YAML File:

1. Create the IAM instance role.
2. Create the EC2 instance profile.
3. Create the secrets manager policy that allows the EC2 instances to retrieve secrets.
4. Output the EC2 instance profile. 

### Step 5: Create TVPC Stack YAML File:

1. Import the mappings for the public and private subnets as parameters.
2. Create the VPC.
3. Create the internet gateway and then attach it.
4. Create the first and second public subnets, map them to the parameters and ensure they deploy in different availability zones.
5. Create the first and second private subnets, map them to the parameters and ensure they deploy in different availability zones.
6. Request x2 elastic IPs for the NAT gateways.
7. Create the two NAT gateways, associae the elastic ips, and place them in seperate public subnets.
8. Create a single public route table.
9. Create two private route tables.
10. Associate both public subnets with the public route table.
11. Create indivdual associations for the private subnets and their corresponding route tables.
12. Create a route to the internet gateway for the public subnets.
13. Create a route for each of the private subnets to their corresponding NAT gateways.
14. Output the VPC ID, public and private subnets.

 ### Step 6: Create The Security Groups Stack YAML File:

* Import the VPC output. 
1. Create the external ALB security group, allow ingress traffic from port 80 and 8080.
2. Create the web tier security group, allow inbound port 80 traffic from the external ALB security group.
3. Create the internal ALB security group, allow inbound port 8080 traffic from the web tier security group.
4. Create the app tier security group, allow inbound port 8080 traffic from the internal ALB security group.
5. Create the RDS security group, allow inbound port 3306 traffic from app tier security group.
6. Export all the created security groups.  

### Step 7: Create the RDS Stack YAML File:

* Import the security group, secrets and VPC outputs.
1. Create the database subnet group.
2. Create the database, resolve the secret manager secrets and parameter store values for database variables.
3. Output the database and the database endpoint address.

### Step 7: Create the app tier launch template YAML File:

* Import the security group, secrets, IAM and database ouputs.
1. Define the app tier instance.
2. Allow the instance to resolve the latest version of the amazon linux instance
3. Assign the app tier security group.
4. Assign the instance profile.
5. Enter the user data:
```bash
  #!/bin/bash
   set -xe
   rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
   rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
   yum install -y mysql-community-client jq mysql git python3-pip
   python3 -m pip install --upgrade pip
   pip3 install Flask PyMySQL gunicorn cryptography
   mkdir -p /home/ec2-user/app
   SECRET_JSON=$(aws secretsmanager get-secret-value \
     --secret-id ${DBSecretId} \
     --query SecretString \
     --output text \
     --region ${AWS::Region})
   DB_USER=$(echo $SECRET_JSON | jq -r .username)
   DB_PASSWORD=$(echo $SECRET_JSON | jq -r .password)
   DB_NAME=$(aws ssm get-parameter \
     --name /mtwa/db/name \
     --with-decryption \
     --query Parameter.Value \
     --output text \
     --region ${AWS::Region})
   cat <<'EOF' > /home/ec2-user/app/app.py
   from flask import Flask
   import pymysql, os
   app = Flask(__name__)
   DB_HOST = os.environ['DB_HOST']
   DB_USER = os.environ['DB_USER']
   DB_PASSWORD = os.environ['DB_PASSWORD']
   DB_NAME = os.environ['DB_NAME']
   @app.route('/')
   def hello_world():
       try:
           conn = pymysql.connect(
               host=DB_HOST,
               user=DB_USER,
               password=DB_PASSWORD,
               database=DB_NAME,
               connect_timeout=5
           )
           cursor = conn.cursor()
           cursor.execute("SELECT VERSION()")
           db_version = cursor.fetchone()[0]
           cursor.close()
           conn.close()
           return f"<h1>App Tier - Connected to MySQL: {db_version}</h1>"
       except Exception as e:
           return f"<h1>Database connection failed: {e}</h1>", 500
   if __name__ == '__main__':
       app.run(host='0.0.0.0', port=8080)
   EOF
   
   cat <<EOF > /etc/systemd/system/flaskapp.service
   [Unit]
   Description=Gunicorn instance to serve my Flask app
   After=network.target
   [Service]
   User=ec2-user
   Group=ec2-user
   WorkingDirectory=/home/ec2-user/app
   ExecStart=/usr/local/bin/gunicorn --workers 4 --bind 0.0.0.0:8080 app:app
   Restart=always
   Environment="DB_HOST=${DatabaseEndpointId}"
   Environment="DB_USER=$DB_USER"
   Environment="DB_PASSWORD=$DB_PASSWORD"
   Environment="DB_NAME=$DB_NAME"
   
   [Install]
   WantedBy=multi-user.target
   EOF
   systemctl daemon-reload
   systemctl start flaskapp
   systemctl enable flaskapp
```
6. Export the app tier template and template version.

### Step 8: Create The Internal ALB Stack YAML File:

* Import the VPC and security group outputs.
1. Create the internal application load balancer.
2. Export the internal ALB DNS and ALB as outputs.

### Step 9: Create The Web Tier Launch Template Stack YAML File:

* Import the internal ALB, IAM and security group outputs. 
1. Define the app tier instance.
2. Allow the instance to resolve the latest version of the amazon linux instance
3. Assign the app tier security group.
4. Assign the instance profile.
5. Enter the user data:
```bash
  #!/bin/bash
  set -xe
  yum install httpd mod_ssl -y
  yum install mod_proxy_html -y

  systemctl start httpd
  systemctl enable httpd
  cat <<'EOF' > /etc/httpd/conf.d/my-app.conf
  <VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog /var/log/httpd/error_log
    CustomLog /var/log/httpd/access_log combined
    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass / http://${InternalAlbDNS}:8080/
    ProxyPassReverse / http://${InternalAlbDNS}:8080/
  </VirtualHost>
  EOF
  systemctl restart httpd
```
6. Output the web tier launch template and the template version.

### Step 10: Create The Internal ALB Target Group Stack YAML File:

* Import the internal ALB output.
1. Create the target group, protocol HTTP on port 8080.
2. Create the listener, protocol HTTP on port 8080 with the forward action.
3. Output the internal target group.

### Step 11: Create The App Tier Autoscaling Stack YAML File:

* Import the VPC, internal alb and app tier launch template outputs.
1. Create the app tier autoscaling group.

### Step 11: Create The External ALB Stack YAML File:

* Import the security group and vpc outputs.
1. Create the external application load balancer.
2. Output the external ALB.

### Step 12: Create The External ALB Target Group Stack YAML File:

* Import the VPC and External ALB stack.

1. Create the target group, protocol HTTP on port 80.
2. Create the listener, protocol HTTP on port 80 with the forward action.
3. Output the external target group.

### Step 13: Create the Web Tier Autoscaling Stack YAML File:

* Import the VPC, external ALB, and web tier launch template outputs. 

1. Create the web tier autoscaling group.

### Step 14: Test Architecture

* Connect to the apptier instance and check:
  1. Check logs to ensure user data ran correctly: `sudo cat /var/log/cloud-init-output.log` 
  2. Check Flask app status `sudo systemctl status flaskapp`
  3. Verify environmental variables(DB_HOST,USER,PASSWORD,NAME) `sudo cat /proc/$(pgrep gunicorn)/environ | tr '\0' '\n'`
  4. Attempt to connect to the database `mysql -h <database DNS> -u <database username> -p`

* Connect to the webtier instance and check:
 1. Check logs to ensure user data ran correctly: `cat /etc/httpd/conf.d/my-app.conf`
 2. Apache status: `sudo systemctl status httpd`
 3. Test reverse proxy: `curl http://<InternalAlbDNS>:8080` or go to the external ALB DNS - "Connected to MySQL will appear if successful".

* Verify ALB health checks.

## Usage

- You can deploy your applications and services within the private subnets in either region, ensuring they are not directly exposed to the internet while still having controlled outbound internet access via the NAT Gateways.
- The Transit Gateway peering connection enables secure communication between resources deployed in different regions. This is ideal for distributed applications, disaster recovery, or sharing data across regions.
- The architecture inherently supports high availability across services as it is deployed across multiple regions. 
- The Transit Gateway acts as a central hub for network connectivity, simplifying routing and management of inter-VPC and inter-region traffic.
- Individual CloudFormation templates can be used to create other VPCs and resources for other projects or use cases without having to go through the manually configuration. 


## Lessons Learned & Challenges Overcome

This project has further improved my ability and consistency utilising CloudFormation and IaC to make the manually created project more scalable. 
- The most difficult part of this project was creating the user data scripts within a launch template. Being able to successfully pull in the secrets and parameters and inject them was a fun to learn. It took a few attempts to correctly utilise the `<<EOF >`. I learned that by adding/removing the '' it changed how the variables were entered. Once realising that subtle difference my user data correctly injected the code. 
- In this project I have also refined my naming scheme. This is great for production deployments as all the resources will follow a naming scheme which makes managing and troubleshooting much easier. 

## Future Enhancements

- Implement a CloudWatch stack that will create log groups and dashboards for monitoring of resources.
- Utilise AWS WAF to attach to the ALB to give an extra layer of protection. 
- Implement read replicas and a proxy for the database instance to allow for large volumes of connections. 


## Visuals Of The Project

![App Tier User Data Logs](/docs/visual-guides/21-app-tier-log.png)
![Web Tier User Data Logs](/docs/visual-guides/21-web-tier-log.png)
![CloudFormation Events](/docs/visual-guides/20-successful-launch.png)
![Healthy External Targets](/docs/visual-guides/21-healthy-targets-external.png)
![Healthy Internal Targets](/docs/visual-guides/21-healthy-targets-internal.png)
![Flaskapp Running](/docs/visual-guides/21-flaskapp-status.png)
![HTTPD Service Running](/docs/visual-guides/21-httpd-status.png)
![The Database Instance](/docs/visual-guides/database.png)
![Launched Load Balancers](/docs/visual-guides/loadbalancers.png)
![Launched Instances](/docs/visual-guides/launched-instances.png)