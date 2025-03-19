# CSV Upload & Processing Pipeline Journey
## Overview
This document outlines the journey of designing and implementing a secure and efficient CSV file upload and processing pipeline in Django. The system ensures that user-uploaded CSV files are validated, sanitized, and processed correctly while keeping the user informed in real-time via Django Channels.
## System Design & Flow
The pipeline consists of multiple steps, ensuring file integrity, security, and real-time notifications.

**1. User Uploads CSV File**

- The user uploads a CSV file through an API endpoint.

- The file is first stored in a Quarantine S3 Bucket.

- A record is created in the database with a pending status, s3_key for quarantine bucket.

  ```python
  class UploadedTemplateFile(models.Model):
    """Stores metadata for uploaded Suite & Phase Templates CSV files."""
    STATUS_CHOICES = [
        ("pending", "Pending"),
        ("sanitizing", "Sanitizing"),
        ("processed", "Processed"),
        ("failed", "Failed"),
    ]
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE, related_name="uploaded_templates")
    project = models.ForeignKey(ProjectProfile, on_delete=models.CASCADE, related_name="uploaded_templates")
    s3_key_quarantine = models.CharField(max_length=500, unique=True, db_index=True)  # S3 path in quarantine bucket
    s3_key_final = models.CharField(max_length=500, blank=True, null=True, unique=True, db_index=True)  # S3 path in final bucket
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="pending")  # Tracks processing status
    uploaded_at = models.DateTimeField(auto_now_add=True)
    uploaded_by = models.ForeignKey(UserAccount, on_delete=models.CASCADE, blank=True, null=True, related_name="uploaded_templates")

    def __str__(self):
        return f"File {self.s3_key_quarantine} (Org: {self.organization.id}, Project: {self.project.id}) - {self.status}"
  ```

**2. Lambda Monitors Quarantine Bucket & Sanitizes the File**

- An AWS Lambda function listens for new files in the Quarantine Bucket.

- The function reads and sanitizes the file:

  - Ensures proper formatting.

  - Removes any potentially harmful scripts or malformed data.

  - Standardizes column headers.

- If the file is valid, it is moved to the Final S3 Bucket.

**3. SQS Message Sent with Quarantine & Final S3 Keys**

- Once the sanitized file is successfully moved to the Final Bucket, the Lambda function sends an SQS message containing:

  ```python
  # Send message to SQS
  message_body = json.dumps({
      "quarantine_s3_key": quarantine_s3_key,
      "final_s3_key": final_s3_key
  })
  sqs_client.send_message(QueueUrl=SQS_QUEUE_URL, MessageBody=message_body)
  ```

  - quarantine_s3_key: The original file location.

  - final_s3_key: The sanitized file’s location in the Final Bucket.

- This ensures that Django can track and process the sanitized file correctly.

**4. SQS Message Handling in Django**

A Celery task (process_sqs_messages) is scheduled to poll SQS for messages.

- When a message is received:

  - The task retrieves the quarantine S3 key and the Final S3 Key.
  
  - Updates the database record to reflect that the file has been processed successfully.
  
  - Parses and processes the CSV data, creating SuiteTemplate and PhaseTemplate records in the database.
  
  - Any processing errors are logged.

**5. Django Channels for Real-time User Notifications**

- After successfully processing the CSV file, Django Channels sends a real-time WebSocket notification to the user who uploaded the file.

- This ensures that the user knows the upload and processing status without needing to refresh the page.

## Key Challenges & Solutions

**1. Missing or Malformed Data**

- Issue: CSV files had inconsistent headers (e.g., unexpected byte-order marks \ufeff).

- Solution: Stripped BOM characters and enforced standard column names before processing.

**2. AWS SQS Permissions Issue**

- Issue: The Celery task was failing due to AccessDenied when trying to read SQS messages.

- Solution: Updated the IAM Role to allow sqs:ReceiveMessage permission for the specific queue.

**3. WebSocket Notifications Not Reaching Users**

- Issue: Users were not receiving WebSocket notifications after CSV processing.

- Solution: Ensured that the correct user authentication ID was used when sending WebSocket messages

**4. Celery Setup in Django Project**

Celery is used in this project to handle asynchronous tasks, such as polling AWS SQS and processing CSV files. Below is the complete setup for integrating Celery into the Django project.

- Install Celery & Redis

  ```bash
  # django-celery-beat is for periodically tasks
  pip install celery redis django-celery-beat
  ```
- Configure Celery in Django (Create a celery.py under proejct folder)

- Update __init__.py to Import Celery

  ```python
  # amg_erp_system/__init__.py
  from __future__ import absolute_import, unicode_literals
  
  # Import celery
  from .celery import app as celery_app
  
  __all__ = ("celery_app",)
  ```

- Config Celery in Django Settings, add django-celery-beat in install apps, config celery broker url etc.
- Define Celery Task and register Celery Beat Task in celery.py
- Update celery_worker, celery_beat & redis services in docker-compose.yml
- Migrate django-celery-beat
