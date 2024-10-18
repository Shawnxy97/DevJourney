# Git Flow Setup & Github Actions Setup

## Roadmap

- [Git Flow Setup](#git-flow-setup)
- [Git Actions Setup](#git-actions-setup)
- [Container Deployment on AWS EC2](#container-deployment-on-aws-ec2)
## Scenarios

When working on a project that involves local development and deployment to AWS, it's best to follow a structured branching strategy. The **Git Flow** workflow allows for smooth handling of multiple features, hotfixes, and releases. In this setup:

- The master branch (or main) is the production branch used for stable, ready-to-deploy code.
- The develop branch is the development branch where ongoing development takes place.
- Feature branches are used for individual feature development, and once features are complete, they are merged into the develop branch.
- Once the develop branch is ready for a new release, it gets merged into the master branch via a pull request.

By structuring your workflow this way, you maintain a clean separation between development work and stable production code, which is essential for managing both local development and AWS deployment.

## Git Flow Setup

Here are the steps to set up Git Flow for your project:

1. **Create the `develop` branch**:

- If your current repository only has a master branch, you can create a develop branch from master (which will be your production branch).
- Run the following command to create and switch to the develop branch:

  ```bash
  git checkout -b develop
  ```

This creates the develop branch as a copy of your current master branch.


2. **Initialize Git Flow:**

- After creating the develop branch, you can set up Git Flow, which will manage your branches and releases automatically.

- Run the following command to initialize Git Flow:

  ```bash
  git flow init
  ```


Git Flow will then help you manage feature branches, release branches, and hotfix branches following a structured workflow.

- Create a new feature branch
```bash
git flow feature start feature_name
```

- Finish a new feature branch
```bash
git flow feature finish feature_name
```

[Git Flow Reference](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)


## Git Actions Setup

To automate your deployment process, GitHub Actions can be set up to trigger on certain branch events, such as pushing changes to the develop branch for development builds or the master branch for production deployment.

### Using Git Actions for Container Build and Push to AWS ECR

1. In the branch you want to setup the git actions, create .github/workflows/a-workflow-file.yml

2. A yml file should contain the following parts:
```
name: Build and Push Docker to ECR

on:
  push:
    branches:
      - develop

env:                 
  ECR_REPOSITORY: kpi-report-project-dev  # Set this to your Amazon ECR repository name

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-22.04
    environment: production  

    steps:
    # Step 1: Checkout the code from GitHub
      - name: Checkout
        uses: actions/checkout@v4

      # Step 2: Configure AWS credentials
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ca-central-1

      # Step 3: Log in to Amazon ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      # Step 4: Build, tag, and push the Docker image to ECR
      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

```
#### Prompt:
From my understanding:

first the name is the name of the ci/cd workflow , on is the trigger, means that when push something to develop, the following jobs will be triggered. env, set some enviroment variables? jobs: the tasks will do in sequence after the tigger is hit. deploy: deploy on this os, and set the enviroment as production steps: first checkoout, what does checkout means here, as in github, checkout is to switch to a branch then configure aws crendentials what does uses command do? @v1? with is the variables, secrets we need to give the uses command to have access to the credentail then log in to ecr, id? uses? build image, i can only understand that there are 2 commands one is to build the container, and the other one, is to push to the ecr?

### Response:
You're correct in your understanding of how the CI/CD workflow works, but let me break down each part in detail to clarify things further.

#### Overall Workflow Breakdown

* **name**: This defines the name of the GitHub Actions workflow.
* **on**: This section defines the event that triggers the workflow. In your case, it triggers on a **push** to the `develop` branch.
* **env**: This is where you define environment-wide variables that will be accessible to the whole workflow. In your case, it’s used to set the ECR repository name, but you can use it for other variables as well.

#### Detailed Breakdown of `jobs` and `steps`

##### **jobs:**

* **`jobs`** defines the various tasks or jobs that will be executed when the workflow is triggered. Each job runs in isolation and can have multiple steps.

##### **deploy:**

* This is the name of the job (`deploy`). You can name it anything you want (e.g., `build-and-deploy`).
* **`runs-on: ubuntu-22.04`**: This specifies the virtual machine (runner) where the job will run. In this case, it’s using the latest **Ubuntu 22.04** machine that GitHub provides.

##### **environment: production**

* This specifies that the environment for this job is `production`. It ties in with GitHub’s **environments** feature, where you can define secrets, variables, or approval rules.

#### Breakdown of **steps**

Each job consists of multiple steps. The steps are executed in sequence and can run specific actions or shell commands.

##### 1\. **Checkout Step**

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

* **`name: Checkout`**: This is just the name of the step. You can name it anything.

* **`uses: actions/checkout@v4`**: The `uses` keyword means this step is using a **GitHub Action** from the marketplace. In this case, it's using the `actions/checkout` action.

  * **What does `checkout` do here?**

    * It **checks out** the repository's code to the runner (the virtual machine where the job is running). This is necessary because your CI/CD pipeline needs the code from the repository to build and push the Docker image.
    * In Git, `checkout` usually means switching branches, but in the context of GitHub Actions, it means fetching the code of the repository at the specific point (commit or branch) where the workflow is triggered.
  * **`@v4`**: This refers to the version of the action being used. In this case, it's version `v4` of the `actions/checkout` action. You could specify another version, but the latest stable version is often preferred.

##### 2\. **Configure AWS Credentials Step**

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ca-central-1
```

* **`uses: aws-actions/configure-aws-credentials@v1`**: This is another action from the GitHub marketplace. It sets up the AWS credentials required to interact with AWS services.
  * **What does `@v1` mean?** It means the action is using version `v1` of the `aws-actions/configure-aws-credentials` action.
* **with**: This is where you pass variables (inputs) to the action.
  * **`aws-access-key-id`** and **`aws-secret-access-key`**: These are the AWS credentials (stored in GitHub secrets) that allow GitHub Actions to authenticate with AWS.
  * **`aws-region: ca-central-1`**: This is the region where your AWS services (like ECR) are hosted. It’s set directly in this example.

##### 3\. **Login to Amazon ECR Step**

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1
```

* **`name: Login to Amazon ECR`**: This step logs into your Amazon ECR (Elastic Container Registry).

* **`id: login-ecr`**: This is an identifier you can assign to this step so you can reference outputs from this step in future steps.

* **`uses: aws-actions/amazon-ecr-login@v1`**: This uses a pre-built action to log into ECR. This action authenticates the Docker client to your ECR registry, allowing it to push images.

##### 4\. **Build, Tag, and Push Docker Image to ECR**

```yaml
- name: Build, tag, and push image to Amazon ECR
  id: build-image
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

* **`name: Build, tag, and push image to Amazon ECR`**: This is the name of the step.

* **`id: build-image`**: The ID for this step so that the results can be referenced later if needed.

* **`env`**: These are environment variables that are specific to this step.

  * **`ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}`**: The registry URL (e.g., `123456789012.dkr.ecr.ca-central-1.amazonaws.com`) output from the `login-ecr` step.
  * **`IMAGE_TAG: ${{ github.sha }}`**: The image tag is set to the current Git commit SHA, ensuring each image is uniquely tagged with the commit that triggered the build.

* **`run`**: This keyword specifies the shell commands that will be run in this step:

  * **`docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .`**:
    * **`docker build`**: This command builds the Docker image using the `Dockerfile` in your repository.
    * **`-t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG`**: This tags the Docker image using the ECR registry URL, the repository name, and the Git commit SHA as the image tag.
  * **`docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG`**: This pushes the Docker image to your ECR repository.

#### Summary:

* **The name** defines the workflow name, and **on** defines the trigger (in this case, a push to the `develop` branch).
* **env** is where environment-wide variables like your ECR repository are defined.
* **jobs** contains tasks that are executed when the workflow is triggered.
  * **Steps**:
    * **Checkout**: Fetches your repository’s code.
    * **Configure AWS credentials**: Sets up AWS credentials to interact with AWS services.
    * **Login to ECR**: Logs Docker into your ECR registry.
    * **Build and Push Docker Image**: Builds a Docker image and pushes it to ECR.




[Deploying to Amazon Elastic Container Service](https://docs.github.com/en/actions/use-cases-and-examples/deploying/deploying-to-amazon-elastic-container-service)

[Using secrets in Github Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository)

[About GitHub Marketplace for apps](https://docs.github.com/en/apps/github-marketplace/github-marketplace-overview/about-github-marketplace-for-apps) - the pre-define actions/workflows can be found in marketplace

## Container Deployment on AWS EC2

From AWS ECR to AWS EC2

Once the Docker container is built and pushed to AWS Elastic Container Registry (ECR), it can be deployed to an Amazon EC2 instance. The following steps will guide you through this process.

**1. Building and Pushing the Docker Image to ECR**

Before deploying on EC2, ensure that your Docker container is successfully built and pushed to ECR. The command format for pushing to ECR is as follows:
```bash
docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .

# docker container name example: aws_account_id.dkr.ecr.ca-central-1.amazonaws.com/repository_name:commit_id
```

**2. Preparing the EC2 Instance**

a. IAM Role Setup 

Ensure that the EC2 instance has an IAM role attached with the required permissions. Specifically, this role must have the necessary permissions to interact with the Elastic Container Registry.

b. Logging into the Docker Private Registry (ECR)

Before pulling images from ECR, you need to log in to your Docker registry hosted on AWS ECR. This can be done using the following command, which retrieves an authentication token and uses it to log in to your Docker client:
```bash
aws ecr get-login-password --region your-region | sudo docker login --username AWS --password-stdin aws_account_id.dkr.ecr.your-region.amazonaws.com

# example:aws ecr get-login-password --region ca-central-1 | sudo docker login --username AWS --password-stdin aws_account_id.dkr.ecr.ca-central-1.amazonaws.com
```

c. Pulling the Docker Image from ECR
Once authenticated, you can pull the desired Docker image from ECR:

```bash
sudo docker pull image_URI
```

**3. Running the Docker Container on EC2**

Now that the image is pulled to your EC2 instance, you can deploy the container.

a. Stopping and Removing the Previous Container

If a previous version of the application is running, stop and remove the container before running the new one:

```bash
sudo docker ps  # Lists all running containers
sudo docker stop kpi_report_project_dev  # Stops the running container
sudo docker rm kpi_report_project_dev  # Removes the stopped container
```

b. Running the Latest Docker Container

Finally, you can run the latest version of your container that was just pulled from ECR. The -d flag runs the container in detached mode, and -p maps port 8002 on the host to port 8000 in the container:
```
sudo docker run -d -p 8002:8000 --env-file .env --name kpi_report_project_dev image_URI
```

- `-d`: runs the container in the background
- `--env-file`: uses enviroment variables from a .env file
- `--name`: assigns a name to the container
