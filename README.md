# WordPress Scalable Architecture with Docker, Nginx, and AWS

This project demonstrates how to deploy a scalable WordPress
architecture using Docker, Nginx, AWS EC2, Auto Scaling, and EFS.

------------------------------------------------------------------------

## Architecture Overview

-   **2 WordPress containers** running on the same EC2 instance
-   **MySQL container** running on a separate EC2 instance
-   **Nginx** used as a load balancer between the WordPress containers
-   **EFS** mounted to persist `wp-content/uploads`
-   **Auto Scaling Group** using a custom AMI

------------------------------------------------------------------------

## Prerequisites

-   AWS account
-   Two EC2 instances:
    -   WordPress + Nginx server
    -   MySQL database server
-   Docker and Docker Compose installed
-   EFS created and accessible from WordPress EC2

------------------------------------------------------------------------

## Step 1: Create EC2 Instances

Create two EC2 instances:

### 1. Application EC2 (Public Subnet)

This instance will host: - Two WordPress containers - Nginx load
balancer

### 2. Database EC2 (Private Subnet)

This instance will host: - MySQL container running via Docker Compose -
No direct internet access

Install Docker and Docker Compose on both machines.

Example for Amazon Linux:

``` bash
sudo yum update -y
sudo yum install docker -y
sudo service docker start
sudo usermod -aG docker ec2-user
```

------------------------------------------------------------------------

## Step 2: Run MySQL with Docker Compose (Database EC2)

On the database EC2, create a `docker-compose.yml`:

``` yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    container_name: mysql_wp
    restart: always
    environment:
      MYSQL_DATABASE: ${WORDPRESS_DB_NAME}
      MYSQL_USER: ${WORDPRESS_DB_USER}
      MYSQL_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${WORDPRESS_DB_ROOT_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

Start the database:

``` bash
docker compose up -d
```

### Security Group Rule

Allow port **3306** only from the **application EC2 security group**.

------------------------------------------------------------------------

## Step 3: Create Docker Compose for WordPress and Nginx (App EC2)

Create a `docker-compose.yml` that contains:

-   Two WordPress containers
-   One Nginx container
-   Each WordPress container on a different port
-   Both connected to the external MySQL database

The second WordPress container must start only after the first becomes
healthy.

Start the services:

``` bash
docker compose up -d
```

------------------------------------------------------------------------

## Step 4: Configure Nginx Load Balancer

Create an `nginx.conf` file that:

-   Defines an upstream block with the two WordPress containers
-   Listens on port 80
-   Forwards traffic to the upstream servers

This distributes traffic between both WordPress containers.

------------------------------------------------------------------------

## Step 4: Persist Uploads Directory with EFS

Mount the EFS file system to the application EC2:

``` bash
sudo mkdir -p /mnt/efs
sudo mount -t nfs4 <EFS_DNS_NAME>:/ /mnt/efs
```

In the Docker Compose file, map:

    /mnt/efs/wp-content/uploads

to:

    /var/www/html/wp-content/uploads

This ensures uploaded files are shared and persistent.

------------------------------------------------------------------------

## Step 5: Create an AMI

After verifying everything works:

1.  Go to EC2 console.
2.  Select the application EC2 instance.
3.  Click **Create Image (AMI)**.
4.  Wait until the AMI becomes available.

------------------------------------------------------------------------

## Step 6: Create Launch Template

1.  Go to **Launch Templates**.
2.  Create a new launch template.
3.  Select the AMI created earlier.
4.  Configure instance type, security groups, and key pair.

------------------------------------------------------------------------

## Step 7: Create Auto Scaling Group

1.  Create an Auto Scaling Group using the launch template.
2.  Set desired capacity to **1 instance**.
3.  Attach it to the appropriate subnets.

------------------------------------------------------------------------

