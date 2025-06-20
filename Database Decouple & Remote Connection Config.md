# Database Decouple & Remote Connection Configuration

This document is to offload PostgreSQL database from the main Docker-based development environment to a dedicated AWS EC2 instance for better performance, modularity, and scalability.

## Background Problem

As the development system expanded, hosting all services (web backend, PostgreSQL, Redis, Celery workers, etc.) in a single small EC2 instance (e.g., t2.micro) via docker-compose caused frequent:

- CPU exhaustion

- Memory spikes

- SSH connection failures

- DHCP renewal failures

- Complete server freezes or crashes

This was mainly due to limited resources being shared by heavy services like PostgreSQL and Celery.

## Objectives

- Launch a separate EC2 instance dedicated to PostgreSQL.

- Enable remote database access from both:

  - The web application (on another EC2 instance via private IP)
  
  - The local development environment (via public IP + security group)

- Modify docker-compose.yml and .env to remove the db service and point all services to the new database host.
  
- Clean up old Docker containers, volumes, and images to free up space.

## Step-by-Step Setup

### Create a New EC2 Instance for DB

> Note: While AWS offers RDS (Relational Database Service), it is expensive for development use. A self-managed EC2 instance running PostgreSQL is a more cost-effective solution.

- Launch a new EC2 instance:
  
  - Recommended AMI: Ubuntu 22.04 LTS
  
- Attach an IAM Role:
  
  - For dev environment, give full access to S3 and basic EC2 monitoring (optional)
    
- Connect via SSH:

  ```
  ssh -i <your-key.pem> ubuntu@<public-ip>
  ```
- Upgrade packages and install PostgreSQL:

  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install postgresql postgresql-contrib -y
  ```
- Enable PostgreSQL service to auto-start on reboot:

  ```bash
  sudo systemctl enable postgresql
  sudo systemctl start postgresql
  ```
  
### Config the database for Remote Connections

- Modify postgresql.conf:

  ```bash
  sudo nano /etc/postgresql/14/main/postgresql.conf
  ```

  - Find `listen_addresses' and change:
    ```conf
    listen_addresses = '*' # By default, database only listen to local requests, update it to listen on all requests
    ```
- Modify pg_hba.conf:

  ```bash
  sudo nano /etc/postgresql/14/main/pg_hba.conf
  ```

  - Add entries to allow remote hosts:
 
    ```conf
    # Allow web instance (private IP)
    host all all private_ip/32 md5
    
    # Allow your laptop (public IP)
    host all all <YOUR_LAPTOP_PUBLIC_IP>/32 md5
    ```
  
- Restart postgresql service

  ```bash
  sudo systemctl restart postgresql
  ```
  
- Set Up Security Groups

  In AWS Console → EC2 → DB Instance → Security Groups

  - Inbound Rule (Config who can send requests to the db instance)
    - Port: 5432
    - Type: Postgresql
    - Source:
      - Private IP of web server instance
      - Public IP of local laptop ip
     
  **POLP**: Only allow specific IPs to minimize exposure. Avoid 0.0.0.0/0 for PostgreSQL in production.
  
- Test Remote Connections

  From **web instance (private IP)**:

  ```bash
  psql -h <PRIVATE_IP_of_DB_instance> -U amg_admin -d amg_database
  ```

  From **your laptop(public IP)**:
  
  ```bash
  psql -h <PUBLIC_IP_of_DB_instance> -U amg_admin -d amg_database
  ```

  > If authentication fails, check:
  
    - Password (you may need to run sudo -u postgres psql to set it)
    
    - PostgreSQL config files
    
    - Security group settings
    
    - Use telnet <public-ip> 5432 to test if the port is reachable
  

