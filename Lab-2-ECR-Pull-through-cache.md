
# ECR Pull-through Cache Lab

## Introduction 

ECR Pull through Cache repositories removes undifferentiated effort to easily and safely access images from public container registries. Amazon ECR will fetch and store images from a publicly accessible registry as a source, and customers can download images from ECR without worrying about availability, security, throttling or impact to developer productivity. An ECR Pull through Cache repository keeps its image copies in sync with no additional solutions or tools required. Customers using Pull through Cache repositories automatically benefit from ECR’s highly available and scalable architecture and built in security capabilities such as AWS PrivateLink, Image Scanning, and encryption with AWS Key Management Service (KMS) keys.

## Create and configure AWS Cloud9 environment
Log into the AWS Management Console and search for Cloud9 service in the search bar.

Click Cloud9 and create an AWS Cloud9 environment in the us-west-2 region based on Amazon Linux 2. Create an IAM role for Cloud9 workspace as explained [here](https://www.eksworkshop.com/020_prerequisites/iamrole/). Attache the IAM role to your workspace as explained [here](https://www.eksworkshop.com/020_prerequisites/ec2instance/). Turn off the AWS managed temporary credentials of the Cloud9 environment as explained [here](https://www.eksworkshop.com/020_prerequisites/workspaceiam/). 

## Clone the GitHub repository

Open a new terminal inside AWS Cloud9 IDE and run:
```bash
git clone https://github.com/ibuchh/reinvent2021-private
cd reinvent2021-private/
```
## Configure the latest SDK

```bash
aws configure add-model \
 --service-name ecr \
 --service-model file://ecr-sdk-model.json
 ```
## Set environment variables

```bash
#Setting the prefix for all the auto-created repository names for pull-through cache repositories
export ECR_REPOSITORY_PREFIX="my-ecr-public"

#Setting ECR Public upstream registry endpoint
export ECR_PUBLIC_UPSTREAM_ENDPOINT="public.ecr.aws"

#Setting AWS region to be in IAD (us-east-1)
export AWS_REGION="sa-east-1"

```
ECR allows customers to granularly manage their permissions at resource level. Pull-Through cache assumes Service Linked Role and FAS token for following actions:

1. Creates repository with `$ECR_REPOSITORY_PREFIX/repository-name` name in customer's registry on pull
2. Make the upstream tag available in recently created repository

This requires customers to have two permissions for themselves ecr:BatchImportUpstreamImage, ecr:CreateRepository

## Set Amazon ECR policy

```bash
aws ecr put-registry-policy --policy-text file://example-registry-policy.json --region $AWS_REGION
```

## Pull-through Cache Rules

All the configurations for Pull-through cache feature are managed via pull-through rule. We will create, delete and describe the rules with following commands

## CreatePullThroughCacheRule

The following command will set the rule to make a pull from particular upstream registry along with EcrRepositoryPrefix, we will set the upstream registry to ECR Public and EcrRepositoryPrefix to be my-ecr-public in this lab. 


```bash
aws ecr create-pull-through-cache-rule \
--ecr-repository-prefix $ECR_REPOSITORY_PREFIX \
--upstream-registry-url $ECR_PUBLIC_UPSTREAM_ENDPOINT \
--region $AWS_REGION
```
We can list all the rules created with describe-pull-through-cache-rules command. 

```bash
aws ecr describe-pull-through-cache-rules \
--region $AWS_REGION
```

## Docker Pull

Once we set up the rule, any public repository from defined upstream registry can be pulled-through. For making a docker pull into ECR,

Let’s get the account id associated with the registry.

```bash
ACCOUNT_ID=`aws sts get-caller-identity | jq --raw-output '.Account'`
```

Pipe the docker login command with ecr get-login-password.s 

```bash
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

## Pull the upstream repository nginx using the latest tag

```bash
docker pull $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY_PREFIX/ubuntu/nginx:latest
```

If the repository “$ECR_REPOSITORY_PREFIX/library/nginx” does not already exist in your registry, pull-through cache will  automatically create it for you for the pull.

Amazon ECR syncs the upstream repositories in the common cache and syncs them further to customer’s account. This process might take around a minute for image of size 1gb. We can validate the sync to our account by running describe-images command and making sure that latest tag for nginx is present in the repository.

```bash
aws ecr describe-images --repository-name $ECR_REPOSITORY_PREFIX/ubuntu/nginx --region $AWS_REGION
```

Voila! This image can now be scanned, tagged and run on App runner just like any other Amazon ECR artifact!




