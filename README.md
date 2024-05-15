# Deploy Streamlit App on ECS

## Description

This project is dedicated to providing an all-encompassing solution for hosting Streamlit applications on AWS by leveraging AWS CloudFormation and a robust Continuous Integration/Continuous Deployment (CI/CD) pipeline. By combining the power of infrastructure as code and automated deployment workflows, developers can effortlessly host and update their Streamlit apps, ensuring a seamless and efficient user experience.  

This repository provides base code for Streamlit application's and is not production ready. It is your responsibility as a developer to test and vet the application according to your security guidlines.

## Prerequisite

An AWS Account, to deploy the infrastructure. You can find more instructions to create your account [here](https://aws.amazon.com/free).

## Table of contents
1. [Architecture CICD deployment](#architecture-cicd-deployment)
	1. [Steps to Deploy Hello World App](#steps-to-deploy-hello-world-app-cicd-deployment)
	2. [Steps to Customize Web App](#steps-to-customize-web-app-cicd-deployment)
	3. [Streamlit Secrets Management](#streamlit-secrets-management-cicd-deployment)
	4. [Invoking AWS Services from Web App](#invoking-aws-services-from-web-app-cicd-deployment)
	5. [Clean Up](#clean-up-cicd-deployment)
2. [Architecture Development deployment](#Architecture-development-deployment)
    1. [Steps to Deploy Hello World App](#steps-to-deploy-hello-world-app-development-deployment)
	2. [Clean Up](#clean-up-development-deployment)

## Architecture CICD deployment

![architecture-cicd](/architecture-cicd.png)

1. Developer manually deploys [codepipeline.yaml](/codepipeline.yaml) stack, [infrastructure.yaml](/infrastructure.yaml) is deployed as nested stack.
2. Lambda triggers the CodeBuild project. 
3. The CodeBuild project zip's this repository content into app.zip file.
4. CodeBuild copies app.zip into S3 bucket. 
5. app.zip PUT event triggers the CodePipeline and triggers the CodeBuild stage.
6. This CodeBuild is responsible for creating a container image using the DockerFile and pushing this image into ECR. 
7. Deploy stage is trigged.
8. Cloudformation stage deploys the [deploy.yaml](/deploy.yaml) stack. This stack takes the new docker image URI as input. This stage creates the Hello world app. Follow steps [here](#steps-to-deploy-hello-world-app-cicd-deployment).
9. After successfull creation of [deploy.yaml](/deploy.yaml) stack, Cloudfront invalidate cache stage is triggered.
10. Developer Customize's the Web App, zip's new content and uploads it into Amazon S3. This triggers the CodePipeline which results in new Docker image. These docker images replaces the old Fargate tasks. Follow steps [here](#steps-to-customize-web-app-cicd-deployment) to customize app.

> [!NOTE]  
> Steps 2, 3 and 4 are run only once when Codepipeline.yaml is created. To Trigger the changes to the Streamlit web applicaiton manually follow steps [here](#steps-to-customize-web-app-cicd-deployment). 

## Steps to Deploy Hello World App CICD deployment

### Step :one: Clone the forked repository
```
git clone https://github.com/aws-samples/aws-streamlit-deploy-cicd.git
```

### Step :two: Deploy codePipeline.yaml

Create a CloudFormation Stack using the [codepipeline.yaml](/codepipeline.yaml) file.

|   Region   | codepipeline.yaml |
| ---------- | ----------------- |
| us-east-1  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=StreamlitDeploy&templateURL=https://ws-assets-prod-iad-r-iad-ed304a55c2ca1aee.s3.us-east-1.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/codepipeline.yaml)|
| us-west-2  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=StreamlitDeploy&templateURL=https://ws-assets-prod-iad-r-cmh-8d6e9c21a4dec77d.s3.us-east-2.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/codepipeline.yaml)|


### Step :three: Viewing the app

After the successful completion of CodePipeline, the `deploy.yaml` cloudFormation stack is deployed. Get the CloudFront URL from the `Output` of the stack named `<StackName>deploy<EnvironmentName>`. Paste it in the browser to view the Hellow World app.

## Steps to Customize Web App CICD deployment

### Step :one: Replace Web App Content  

In order to customize the web app change the content of `app.py` file.

> [!CAUTION]   
> 1. Do not rename the `app.py` file
> 2. Make sure to declare all packages in requirements.txt

### Step :two: Zip the Repository  

First commit all changes

```
git add .
git commit -m "All Changes"
```

Then zip the current repository using the following command:
```
git archive --format=zip --output=app.zip HEAD
```

This will create an `app.zip` file.

### Step :three: Upload app.zip  

Upload the zip file into `CodeS3Bucket` either using S3 management console or AWS CLI. 

```
aws s3 cp app.zip s3://<CodeS3Bucket-Name>
```

> [!IMPORTANT]
> You have access to CodeS3Bucket name from `Outputs` of `codepipeline.yaml` cloudFormation stack

### Step :four: Deploy Further Revisions of the Web App

Repeat `Step 2` and `Step 3` with modified content.  

## Streamlit Secrets Management (CICD deployment)

> [!IMPORTANT]   
> It is crucial to .gitignore files containing confidential information

### Step :one: Create Parameter in SSM Parameter store  

> [!TIP]   
> 1. For the purpose of simplicity we are using [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html). However, when dealing with sensitive secrets such as Database credentials, the best practice is to use [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/).
> 2. If you decide to use AWS Secrets Manager for storing credentials make sure to follow stops [here](#invoking-aws-services-from-web-app) to give Fargate appropriate permissions. 

To get more information about creating secure parameters using SSM Parameter store visit [link](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-create-console.html).

> [!CAUTION]
> 1. Start the parameter path with /streamlitapp
> 2. Use SecureString parameter type while creating the parameters to encrypt the parameters

### Step :two: Use boto3 for accessing Secrets within your streamlit app

```
from boto3.session import Session
ssm = Session().client("ssm")

USERNAME =  ssm.get_parameter(Name='/streamlitapp/EnvironmentName/USERNAME',WithDecryption=True)["Parameter"]["Value"]
```

> [!IMPORTANT]   
> Replace EnvironmentName with value passed in infrastructure.yaml 

## Invoking AWS Services from Web App (CICD deployment)

Inorder to give permission to the web app to invoke AWS services add appropriate policies to `
StreamlitECSTaskRole-<EnvironmentName>`. 

For instance, if you want to invoke Anthropic Claude V2 Bedrock model from the Streamlit app add the following policy to `
StreamlitECSTaskRole-<EnvironmentName>`:

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "Statement1",
			"Effect": "Allow",
			"Action": [
				"bedrock:InvokeModel"
			],
			"Resource": [
				"arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-v2"
			]
		}
	]
}
```

For invoking Bedrock Agents from Streamlit app add the following policy to `
StreamlitECSTaskRole-<EnvironmentName>`:  

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "Statement1",
			"Effect": "Allow",
			"Action": [
				"bedrock:InvokeAgent"
			],
			"Resource": [
				"arn:aws:bedrock:{Region}:{Account}:agent-alias/{AgentId}/{AgentAliasId}"
			]
		}
	]
}
```

> [!CAUTION]   
> Replace {Region}, {Account}, {AgentId}, and {AgentAliasId} with valid values in the above policy

## Clean Up CICD deployment
- Open the CloudFormation console.
- Select the stack `codepipeline.yaml` you created then click **Delete** twice. Wait for the stack to be deleted.
- Delete the nested stack `<StackName>-Infra-<StackId>` created by `codepipeline.yaml`. Please ensure that you refrain from deleting this stack if there are any additional web deployments utilizing this repository within the specified region of your current work environment.
- Delete the role `StreamlitCfnRole-<EnvironmentName>` manually.


## Architecture Development deployment
![architecture-development](/architecture-development.png)
## Steps to Deploy Hello World App Development deployment

> [!NOTE]  
> Optionally, you can deploy the Virtual Private Cloud (VPC) infrastructure using the provided [infrastructure.yaml](/infrastructure.yaml) file, or utilize the default VPC. The required infrastructure components, including Amazon CloudFront, an Application Load Balancer, and Amazon Elastic Container Service (ECS) on AWS Fargate instances, will be deployed within the chosen VPC environment.

|   Region   | infrastructure.yaml |
| ---------- | ----------------- |
| us-east-1  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=StreamlitDeployInfra&templateURL=https://ws-assets-prod-iad-r-iad-ed304a55c2ca1aee.s3.us-east-1.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/infrastructure.yaml)|
| us-west-2  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=StreamlitDeployInfra&templateURL=https://ws-assets-prod-iad-r-pdx-f3b3f9f1a7d6a3d0.s3.us-west-2.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/infrastructure.yaml)|


### Step :one: Deploy [development.yaml](/development.yaml).

|   Region   | development.yaml |
| ---------- | ----------------- |
| us-east-1  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=StreamlitDeploy&templateURL=https://ws-assets-prod-iad-r-iad-ed304a55c2ca1aee.s3.us-east-1.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/development.yaml)|
| us-west-2  | [![launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=StreamlitDeploy&templateURL=https://ws-assets-prod-iad-r-pdx-f3b3f9f1a7d6a3d0.s3.us-west-2.amazonaws.com/0a9f7588-a2c4-4484-b051-6658ce32605c/streamlit-deploy/development.yaml)|

### Step :two: Viewing the app

After the successful completion of `development.yaml`. Get the CloudFront URL from the `Output` of the stack. Paste it in the browser to view the web application.

## Clean Up Development deployment
- Open the CloudFormation console.
- Select the stack `infrastructure.yaml` you created then click **Delete** twice. Wait for the stack to be deleted.
- Select the stack `development.yaml` you created then click **Delete** twice. Wait for the stack to be deleted.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

