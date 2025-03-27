
---
**Dynamic Website Deployment on AWS**  

**Project Overview**  
This repository contains the architecture diagram and deployment scripts for hosting a dynamic website on Amazon Web Services (AWS). The solution is designed for high availability, fault tolerance, scalability, and secure deployment using various AWS services and DevOps principles.

---

**Architecture**  
The dynamic website is hosted on EC2 instances within a multi-tier infrastructure that includes the following components:

- **Virtual Private Cloud (VPC):** A custom VPC with public and private subnets spanning two Availability Zones (AZs) to ensure fault tolerance and high availability.

- **Internet Gateway:** Provides internet access for instances in the public subnet.

- **Security Groups:** Virtual firewalls configured to control inbound and outbound traffic for instances and services.

- **Public Subnets:** Used for hosting infrastructure components such as the NAT Gateway and Application Load Balancer (ALB).

- **Private Subnets:** Used to securely host the web servers (EC2 instances) and data layer resources.

- **EC2 Instance Connect Endpoint:** Enables secure, browser-based access to instances in both public and private subnets.

- **Application Load Balancer:** Distributes incoming HTTP/HTTPS traffic across EC2 instances in multiple Availability Zones.

- **Auto Scaling Group:** Automatically adjusts the number of EC2 instances based on traffic patterns, ensuring resilience and elasticity.

- **Amazon RDS:** Hosts the MySQL database used by the web application.

- **Amazon S3:** Stores application code and SQL migration files for deployment and automation.

- **AWS Certificate Manager (ACM):** Issues and manages SSL/TLS certificates to secure application communications over HTTPS.

- **Amazon SNS:** Sends alerts for events related to the Auto Scaling Group.

- **Amazon Route 53:** Manages DNS routing for the registered domain name.

---

**Deployment Scripts**

**Flyway Migration Script**  
This script handles database initialization using Flyway and a migration SQL script stored in Amazon S3.

```bash
#!/bin/bash

S3_URI=s3://aosnote-sql-files/V1__shopwise.sql
RDS_ENDPOINT=dev-rds-db.cu2idoemakwo.us-east-1.rds.amazonaws.com
RDS_DB_NAME=applicationdb
RDS_DB_USERNAME=azeezs
RDS_DB_PASSWORD=azeezs123

# Update all packages
sudo yum update -y

# Download and extract Flyway
sudo wget -qO- https://download.red-gate.com/maven/release/com/redgate/flyway/flyway-commandline/10.9.1/flyway-commandline-10.9.1-linux-x64.tar.gz | tar -xvz 

# Create a symbolic link to make Flyway accessible globally
sudo ln -s $(pwd)/flyway-10.9.1/flyway /usr/local/bin

# Create the SQL directory for migrations
sudo mkdir sql

# Download the migration SQL script from AWS S3
sudo aws s3 cp "$S3_URI" sql/

# Run Flyway migration
flyway -url=jdbc:mysql://"$RDS_ENDPOINT":3306/"$RDS_DB_NAME" \
  -user="$RDS_DB_USERNAME" \
  -password="$RDS_DB_PASSWORD" \
  -locations=filesystem:sql \
  migrate
```

---

**EC2 Web Server Setup Script**  
This script installs and configures the web server stack, downloads the application files from Amazon S3, and prepares the environment for hosting the dynamic website.

```bash
#!/bin/bash

# Update software packages
sudo yum update -y

# Install Apache and start the service
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# Install PHP and required extensions
sudo dnf install -y \
php \
php-pdo \
php-openssl \
php-mbstring \
php-exif \
php-fileinfo \
php-xml \
php-ctype \
php-json \
php-tokenizer \
php-curl \
php-cli \
php-fpm \
php-mysqlnd \
php-bcmath \
php-gd \
php-cgi \
php-gettext \
php-intl \
php-zip

# Install MySQL 8 server
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Enable .htaccess via mod_rewrite
sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

# Define environment variable for S3 bucket
S3_BUCKET_NAME=aosnote-shopwise-web-files

# Download and extract application files
sudo aws s3 sync s3://"$S3_BUCKET_NAME" /var/www/html
cd /var/www/html
sudo unzip shopwise.zip
sudo cp -R shopwise/. /var/www/html/
sudo rm -rf shopwise shopwise.zip

# Set appropriate file permissions
sudo chmod -R 777 /var/www/html
sudo chmod -R 777 storage/

# Manually configure environment variables
sudo vi .env

# Restart Apache to apply changes
sudo service httpd restart




