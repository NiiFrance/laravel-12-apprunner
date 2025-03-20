Laravel running on AWS App Runner
==========================================

A production-level Laravel application running on AWS App Runner.

AWS App Runner is a service that allows your to build and deploy web applications without managing infrastructure. It scales your application to meet changes in traffic demand, load balances your traffic to ensure optimum performance for your application.

AWS App Runner builds and deploys applications automatically. This is true but that is half the story.

You still need to provision your infrastructure, like specify the CPU and memory allocations. If you're using containers, you need to build your application's image and upload to AWS Elastic Container Registry.

For production, you need to provision your App Runner service in a VPC and provision a VPC Connector for App Runner to connect to other private VPC resources. For your App Runner service to connect to the public internet, you need to provision a NAT Gateway.

These are related tasks you need to perform to make App Runner run your Laravel application. This is where this repository comes in to take those burden out of your shoulders.

This repository provides an Infrastructure-As-Code solution for your Laravel application to run on AWS App Runner. It handles all the App Runner related configuration so you can focus on building your application.

It also provides CI/CD solution to make deployments effortless.

## Quickstart
- Fork this repository.
- Clone your forked repository.
- Enable Github Action for your repository.
    - Click "Actions" on the repository tab.
    - Click "I understand my workflows, go ahead and enable them" button.
- Set your AWS Access Key ID, AWS Secret Access Key, AWS Account ID, and AWS Region as GitHub repository secrets.
- Follow these steps to setup the repository secrets; AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_ACCOUNT_ID and AWS_REGION.
    - Navigate to your repository's settings.
    - Under "Security", select "Secrets and variables".
    - Click "Actions" to navigate to the Actions secrets and variables page.
    - Select the "Secrets" tab, and click on "New repository secrets".

- Update the "APPRUNNER_SERVICE_NAME" field in the "iac/.env.example" file to your desired name.
- Commit and push your changes.
- This will trigger the CI/CD pipeline to:
    - Run the application tests.
    - Set up a repository in AWS Elastic Container Registry.
    - Build and upload the application's docker image to ECR.
    - Provision RDS and save RDS credentials in AWS Secret Manager.
    - Provision App Runner to pull your applications's docker image and run the application.

The infrastructure and application deployment take close to 20 minutes to provision. Once the deployment is done, get the App Runner URL from the "Deploy infra and application" Github action.

From the "Deploy infra and application" Github action, expand the "Depoly Apprunner" job and copy the value for `CdkApprunnerStack.serviceUrl` output. The App Runner URL would be similar to `jzcyikzimx.***.awsapprunner.com`.
Replace the *** with your AWS Region.

So `jzcyikzimx.***.awsapprunner.com` would be `jzcyikzimx.us-east-1.awsapprunner.com`


## Deep Dive
The most important thing you need to know is the infrastructure provisioned in your AWS Account. The deployment started out by:
1. Provisions an Elastic Container Registry (ECR) repository for your application images.
2. Provisions a Virtual Private Cloud (VPC). This VPC has the 10.0.0.0/16 CIDR. The VPC spans three availability zones.
3. A NAT Gateway is provision for all 3 availability zones in the VPC. This comes with an Elastic IP for each NAT Gateway.
3. Provisions an RDS instance running MySQL 8.0. This RDS instance runs from a private subnet, hence its not reachable from the public internet.
4. Stores the RDS credentials in AWS Secret Manager.
5. Provisions the App Runner service and runs it from private subnets within the VPC.

In essence there's a networking, data storage and app runner infrastructure that are provisioned in your AWS Account. Lets take a closer look at each infrastructure.

### Networking
As mentioned earlier, a VPC is provision in your AWS Account. Its given the CIDR of 10.0.0.0/16.
There are three availability zones in this VPC. There are three subnets in each availability zone, the public subnet, private subnet, and private subnet with a NAT Gateway attached. Each subnet has a CIDR of /20.

### Data Storage
An RDS instance running MySQL 8.0 is provisioned. The credentials of the RDS instance is stored securely in AWS Secret Manager and are injected into your applications as environment variables.
The RDS instance runs from a private subnet group in the VPC.

### App Runner
An App Runner service is provisioned to pull your application image from the ECR repository and run. The App Runner services is deployed in the private subnet with a NAT attached. This enables the App Runner service to connect to the public internet.
