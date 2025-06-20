# EC2 Auto-Recovery with Dockerized Web Service (AWS + Lambda + CloudWatch)

This document addresses a critical issue when running a Dockerized Django + Celery application on a small AWS EC2 instance used for development. In some cases, the instance becomes unresponsive due to CPU overload or high volumes of accidental frontend requests. 
This can lead to EC2-level network failures (such as DHCP timeouts caused by low memory), making the instance unreachable.

To solve this, the system uses AWS-native tools—**CloudWatch**, **SNS**, and **Lambda**—to automatically detect failures and trigger recovery actions that restore the instance and restart all containers with minimal downtime.

---

## Problem

Occasional EC2 instance outages (especially on small instances like `t2.micro`) led to:

- Instance unreachability due to network stack/DHCP failure
 
  ![image](https://github.com/user-attachments/assets/c43a3fe5-1b4d-42c3-b2ae-f6a7d1b6ec7c)

- Manual intervention required to stop → start EC2
- Docker containers not recovering automatically after restart
- Database entanglement inside the same EC2, risking data loss during rebuild

---

## Solution Overview

I implemented an **automated recovery flow** using:

- **CloudWatch Alarm** on `StatusCheckFailed_System`
- **SNS Topic** to broadcast recovery events
- **Lambda Function** to stop → wait → start the EC2
- **Docker Compose** services with `restart: always`
- **Systemd service** to auto-run Docker Compose at boot
- **Database split to a dedicated EC2 instance**, decoupling stateful services

---

## Recovery Workflow

```
[EC2 Web Server]
↓
[CloudWatch Alarm - StatusCheckFailed_System >= 1 for 5 mins]
↓
[SNS Topic - ec2-auto-recovery-topic]
↓
[Lambda Function - stop → wait → start EC2 instance]
↓
[EC2 reboots, systemd triggers docker-compose up -d]
↓
[All containers restart:always]
```

---

## CloudWatch Alarm Setup

Enter EC2 instance detail page, Actions -> Monitor and troubleshoot -> Manage CloudWatch alarms

![image](https://github.com/user-attachments/assets/9e922a76-3486-4c8a-b172-117d978ef089)

---

## SNS Setup

![image](https://github.com/user-attachments/assets/d442adc6-c8cf-425c-8bd0-2c08345e19ea)

This will invoke the Lambda function whenever CouldWatch Alarm sends a message via SNS.

---

## Lambda Function Setup

- The Lambda function must have permission to **stop and start EC2 instances**.  Attach the EC2 policy to its execution role.

- Timeout: Stopping and starting an EC2 instance can take time.To prevent the Lambda function from timing out mid-process:
  - Set the Lambda timeout to at least 3 minutes (e.g., 180 seconds in the Lambda configuration)

```python
import boto3

INSTANCE_ID = 'i-instance_id'  
REGION = 'region_name'            

def lambda_handler(event, context):
    ec2 = boto3.client('ec2', region_name=REGION)
    
    try:
        print(f"Stopping instance {INSTANCE_ID}...")
        ec2.stop_instances(InstanceIds=[INSTANCE_ID])
        waiter = ec2.get_waiter('instance_stopped')
        waiter.wait(InstanceIds=[INSTANCE_ID])
        
        print(f"Starting instance {INSTANCE_ID}...")
        ec2.start_instances(InstanceIds=[INSTANCE_ID])
        
        return {
            'statusCode': 200,
            'body': f'Restarted instance {INSTANCE_ID}'
        }

    except Exception as e:
        print(f"Error: {e}")
        raise

```
---

## Docker Compose Restart Setup

In `docker-compose.yml`, every container includes:

```yaml
restart: always
```

Example:
```yaml
  web:
    build: .
    command: daphne -b 0.0.0.0 -p 8000 amg_erp_system.asgi:application
    volumes:
      - .:/app
      - static_volume:/app/staticfiles  # Share static files with nginx
    ports:
      - "8000:8000"
    depends_on:
      - redis
    env_file:
      - .env
    environment:
    - DJANGO_SETTINGS_MODULE=amg_erp_system.settings
    restart: always
```




