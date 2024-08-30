# Methods to Secure Secrets during Development

Securing API keys, database passwords, and other confidential information is essential throughout the entire project development process, from initial development to production deployment. Leaking these secrets can expose your application to significant risks, including unauthorized access and data breaches.

Below are several methods to protect confidential information:

## 1. **Environment Variables**
   - **Usage**: Store sensitive information in environment variables rather than hardcoding them into your source code.
   - **Implementation**:
     - Create a `.env` file at the root of your project and add your secrets:
       ```bash
       DATABASE_URL=your_database_url
       SECRET_KEY=your_secret_key
       API_KEY=your_api_key
       ```
     - Use a library like `python-dotenv` in Python projects to load these variables:
       ```python
       from dotenv import load_dotenv
       import os

       load_dotenv()

       DATABASE_URL = os.getenv('DATABASE_URL')
       ```
     - Ensure your `.env` file is added to `.gitignore` to prevent it from being pushed to version control.

## 2. **AWS Secrets Manager**
   - **Usage**: Manage your credentials and API keys securely throughout their lifecycle.
   - **Implementation**:
     1. **Create an IAM Role/User**:
        - The choice between an IAM role or user depends on the environment where you will be accessing secrets. 
            - For Local Development: You typically create an IAM user with the necessary permissions, then configure your local AWS CLI with the user's credentials.
            - For AWS Services (e.g., EC2, Lambda): You should create an IAM role with the appropriate permissions and attach it to the service. This allows the service to automatically assume the role and access secrets securely without hardcoding credentials.
     2. **Grant Permissions**:
        - Ensure the IAM role/user has permissions to `GetSecretValue` and `PutSecretValue` in AWS Secrets Manager.
     3. **Connect to AWS Secrets Manager**:
        - Use the following Python code to retrieve a secret:
          ```python
          import boto3
          from botocore.exceptions import ClientError
          
          def get_secret():
              secret_name = "your_secret_name"
              region_name = "your_region"
              
              # Create a Secrets Manager client
              session = boto3.session.Session()
              client = session.client(
                  service_name='secretsmanager',
                  region_name=region_name
              )

              try:
                  get_secret_value_response = client.get_secret_value(
                      SecretId=secret_name
                  )
              except ClientError as e:
                  # For a list of exceptions thrown, see
                  # https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html
                  raise e

              secret = get_secret_value_response['SecretString']
              return secret
          ```
     - This code snippet demonstrates how to securely retrieve secrets from AWS Secrets Manager using the Boto3 library.
