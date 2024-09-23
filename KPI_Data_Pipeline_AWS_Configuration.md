# Background / Scenario

Our company uses an outdated system, and we aim to generate a daily KPI report using their API. The infrastructure consists of two EC2 instances: a Windows Server instance for API data extraction and a Linux instance hosting a PostgreSQL database.

## Process Overview:

- **Data Extraction (Windows Server):**
Using a .NET Framework application, KPI-related data is extracted from the API and saved as a CSV file daily at 12 PM. This process is automated using Windows Task Scheduler. The CSV files are stored in a designated "kpi raw data" folder on the Windows Server instance (since the API DLL is compatible with Windows OS only).

- **Data Loading and Storage (Python & PostgreSQL):**
A Python script (using watchdog) monitors the "kpi raw data" folder for new CSV uploads. When a new CSV is detected, the script is triggered to load the file and, using psycopg2, insert each row into the job_activities table in the PostgreSQL database on the Linux instance

## Backup Process:

- On the Linux instance, a shell script is scheduled (crontab) to run daily at 5 PM. This script backs up all records created that day by exporting them to a CSV file and uploading it to an S3 bucket for safe storage.

# AWS EC2 Instances Communication Configuration

In this setup, we need to ensure that the Windows Server instance can communicate with the PostgreSQL database hosted on the Linux instance. Below are the steps to configure communication between these two instances:

## Step 1. IAM Role for the Linux Server

Create the Linux Server with an IAM role attached that includes permissions for S3 and Secrets Manager. This ensures that the instance can securely access required resources like S3 for backups or Secrets Manager for credentials.

## Step 2. Security Group Configuration

The security groups control the traffic to and from your instances.

**Outbound Rules (Windows Server & Linux Server):**

- Outbound rules control traffic leaving the instance. Ensure the Linux instance can reach external services like S3 for backups.


**Inbound Rules (Linux Server):**

- Inbound rules define what traffic is allowed into the instance. 

- Configure the following rules:

1. SSH (Port 22):

    - Allow remote access to the Linux instance using SSH clients like Putty by permitting traffic on port 22. You’ll need to use the PEM or PPK key to authenticate.

2. PostgreSQL (Port 5432):

    - This is the default port for PostgreSQL. Configure the inbound rule to allow connections on **port 5432** but restrict access to only the Windows Server instance using its **Elastic IP** (a static public IP). This ensures that only the Windows instance can interact with the PostgreSQL database on the Linux server.


## Step 3. Installing PostgreSQL on the Linux Server

Once the Linux instance is set up, follow these steps to install PostgreSQL:

```bash
# Update packages
sudo apt-get update -y && sudo apt-get upgrade -y

# Install PostgreSQL
sudo apt install postgresql -y

```


**PostgreSQL User and Role Setup:**

Switch to the PostgreSQL user and create a new role with the required permissions:
```bash
sudo su postgres
psql -U postgres -c "CREATE ROLE ubuntu;"
psql -U postgres -c "ALTER ROLE ubuntu WITH LOGIN;"
psql -U postgres -c "ALTER USER ubuntu CREATEDB;"
psql -U postgres -c "ALTER USER ubuntu WITH PASSWORD 'password';"
exit
```

## Step 4. PostgreSQL Configuration for External Access

By default, PostgreSQL listens only on localhost. To allow external connections (i.e., from the Windows Server), you need to modify the PostgreSQL configuration files:

1. **Update** `postgresql.conf` to **bind to all IP addresses**:

    - Find and edit the `postgresql.conf` file:

        ```bash
        sudo find / -name "postgresql.conf"
        sudo nano /etc/postgresql/12/main/postgresql.conf
        ```

    - Modify the line `listen_addresses` to allow connections from all IP addresses:

        ```bash
        listen_addresses = '*'
        ```
2. **Update** `pg_hba.conf` **to allow connections from the Windows Server**:

    - Find and edit the `pg_hba.conf` file:

        ```bash
        sudo find / -name "pg_hba.conf"
        sudo nano /etc/postgresql/12/main/pg_hba.conf
        ```
    - Add the following line at the end of the file, replacing vm_ip with the Windows Server's IP address:
    ```
    host    all             all             vm_ip           md5
    ```

3. Restart PostgreSQL to apply the changes:

    ```bash
    sudo systemctl restart postgresql
    ```

## Step 5. PostgreSQL Authentication Method

If you encounter issues with the newly created user being denied access, ensure that the PostgreSQL configuration uses MD5 authentication instead of peer. Update the pg_hba.conf file as follows:

```bash
# "local" is for Unix domain socket connections only
local   all             all                                     md5
```

## Step 6. Testing the Connection (Windows Server)

To verify that the setup is working:

- Install pgAdmin on the Windows Server.

- Register a new server in pgAdmin using the IPv4 address of the Linux instance.

- Input the PostgreSQL username (ubuntu) and password to connect and ensure communication between the Windows and Linux instances.

[Reference Blog: How to Provision a Cheap PostgreSQL Database in AWS EC2](https://betterprogramming.pub/how-to-provision-a-cheap-postgresql-database-in-aws-ec2-9984ff3ddaea)
 
# Auto-Schedule Tasks on Windows and Linux OS

## Task Scheduler (Windows Server OS)

To automate tasks on the Windows Server (such as running the .NET application that extracts KPI data), follow these steps:

1. Open Task Scheduler.

2. Create a new task and specify the executable location (the location of the .exe file that runs the .NET application).

3. Set up the schedule (e.g., daily at 12 PM).

4. Configure the task to run with the required privileges (i.e., with the necessary permissions to access files and network resources).

## Cron Job (Linux OS)

For scheduling a backup task on the Linux instance, we will use the cron service to run a backup script at the specified time.

### Step 1: Schedule a Cron Job

To edit the cron schedule, use the following command:

```bash
crontab -e
```
Add a new line to schedule the task at 3 PM from Tuesday to Saturday:

```bash
# Scheduled Task on 3pm From Tuesday to Saturday
0 15 * * 2-6 /home/ubuntu/backup/backup_script.sh
```

### Step 2: Create a Shell Script for Backup

To automate the database backup and S3 upload, create a shell script as follows:

```bash
#! /usr/bin/bash
# The first line specifies the bash shell. Use 'which bash' to verify the path if needed.

# Variables
DB_NAME="morawarekpi"
DB_USER="amg_admin"
TABLE_NAME="job_activities"
S3_BUCKET="s3://moraware-kpi-database-backup"
BACKUP_PATH="/home/ubuntu/backup"
DATE=$(date +'%Y-%m-%d')
BACKUP_FILE="$BACKUP_PATH/filtered_backup_$DATE.csv"

# Export the data where created_at equals today's date
psql -U $DB_USER -d $DB_NAME -c "\COPY (SELECT * FROM $TABLE_NAME WHERE created_at::date = '$DATE') TO '$BACKUP_FILE' WITH CSV HEADER"

# Upload the backup to S3
aws s3 cp $BACKUP_FILE $S3_BUCKET

# Remove the backup file from local storage
rm $BACKUP_FILE
```
for tesing shell script: ./xxxx.sh or bash ./xxxxx.sh

## Step 3: Handle PostgreSQL Authentication

The pgpass file stores database credentials, allowing the psql command to authenticate without prompting for a password.

```bash
# Navigate to the home directory and edit the .pgpass file:
nano ~/.pgpass

# Add the following line, replacing the placeholders with your database information:
hostname:port:database:username:password
localhost:5432:your_database:your_username:your_password
```

Set the appropriate file permissions to secure it:
```bash
chmod 600 ~/.pgpass
```
## Step 4: Install AWS CLI for S3 Upload

To upload the backup file to S3, install the AWS CLI on the Linux instance. Since an IAM role is attached to the instance with the necessary permissions, no manual configuration is needed for AWS credentials.

```bash
sudo apt-get install awscli -y
```

Now, the backup script will run automatically at the scheduled time, securely connecting to PostgreSQL and uploading the backup to S3.

[Reference Blog: How I use cron in Linux](https://opensource.com/article/17/11/how-use-cron-linux)

[Reference Blog: Shell Scripting for Beginners – How to Write Bash Scripts in Linux](https://www.freecodecamp.org/news/shell-scripting-crash-course-how-to-write-bash-scripts-in-linux/)


# Watchdog & psycopg2

## Watchdog for New File Detection
```python
# Event handler for new file creation
class NewFileHandler(FileSystemEventHandler):
    def on_created(self, event):
        if event.is_directory:
            return

        # Process only CSV files
        if event.src_path.endswith('.csv'):
            print(f"New CSV file detected: {event.src_path}")
            
            # Wait for the file to be fully written
            file_size = -1
            while file_size != os.path.getsize(event.src_path):
                file_size = os.path.getsize(event.src_path)
                time.sleep(1)  # Wait and check every second
            
            # Process the CSV file
            import_csv_to_postgresql(event.src_path)

# Main function to monitor the folder
def monitor_folder(path):
    event_handler = NewFileHandler()
    observer = Observer()
    observer.schedule(event_handler, path, recursive=False)
    observer.start()
    print(f"Monitoring folder: {path}")

    try:
        while True:
            time.sleep(1)  # Keep the script running
    except KeyboardInterrupt:
        observer.stop()
    observer.join()

# Set the path to the folder you want to monitor
folder_to_monitor = r"C:/Projects/kpi_csv"

# Start monitoring the folder
monitor_folder(folder_to_monitor)
```

## psycopg2 for Database Connection and Record Upload

```python
# PostgreSQL connection setup
conn = psycopg2.connect(
    host="99.79.45.179",
    database="morawarekpi",
    user="amg_admin",
    password="password"
)
cur = conn.cursor()

# Function to upload CSV data into PostgreSQL
def import_csv_to_postgresql(csv_file):
    try:
        with open(csv_file, 'r') as f:
            reader = csv.reader(f)
            next(reader)  # Skip header row if present
            for row in reader:
                cur.execute(
                    "INSERT INTO job_activities (job_id, job_name, activity_type, assignee, total_sqft, activity_date) VALUES (%s, %s, %s, %s, %s, %s)",
                    row
                )
        conn.commit()
        print(f"CSV file {csv_file} successfully uploaded to PostgreSQL")
    except Exception as e:
        print(f"Error while processing CSV {csv_file}: {e}")
```