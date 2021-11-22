
# ECR Pull-through Cache 

ECR allows users to gradually manage their permissions. Pull-Through cache is a feature that does following things on customer’s behalf:

1. Pulls upstream tag to customer’s local desktop 
2. Creates repository with `repository-prefix/repository-name` name in customer's repo
3. Make the upstream tag available in recently created repository

This requires customers to have two permissions for themselves ecr:BatchImportUpstreamImage, ecr:CreateRepository

## Create and configure AWS Cloud9 environment
Log into the AWS Management Console and search for Cloud9 service in the search bar.

Click Cloud9 and create an AWS Cloud9 environment in the us-west-2 region based on Amazon Linux 2. Create an IAM role for Cloud9 workspace as explained [here](https://www.eksworkshop.com/020_prerequisites/iamrole/). Attache the IAM role to your workspace as explained [here](https://www.eksworkshop.com/020_prerequisites/ec2instance/). Turn off the AWS managed temporary credentials of the Cloud9 environment as explained [here](https://www.eksworkshop.com/020_prerequisites/workspaceiam/). 

## Clone the GitHub repository

Open a new terminal inside AWS Cloud9 IDE and run:
```bash
git clone https://github.com/ibuchh/reinvent2021-private
cd reinvent2021-private/
```
## Create Pull-through Cache Rules

```bash
aws configure add-model \
 --service-name ecr \
 --service-model file://ecr-sdk-model.json
 ```

```bash
export ECR_REPOSITORY_PREFIX="my-docker-hub"
export DOCKER_HUB_UPSTREAM_ENDPOINT="registry-1.docker.io"
export ECR_PUBLIC_UPSTREAM_ENDPOINT="public.aws.ecr"
export AWS_REGION="sa-east-1"
```

```bash
aws ecr create-pull-through-cache-rule \
--ecr-repository-prefix $ECR_REPOSITORY_PREFIX \
--upstream-registry-url $DOCKER_HUB_UPSTREAM_ENDPOINT \
--region $AWS_REGION
```

```bash
aws ecr describe-pull-through-cache-rules \
--region $AWS_REGION
```

*Note* - We currently support Docker Hub (registry-1.docker.io) and ECR Public (public.ecr.aws) as upstream registries, and the above examples use Docker Hub as the upstream registry.

## Docker Pull

After setting up a PTC rule, you should be able to test the pull-through cache flow.


```bash
ACCOUNT_ID=`aws sts get-caller-identity | jq --raw-output '.Account'`
```

Do a docker login against the gamma endpoint. 

```bash
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

## Pulls using image tag

```bash
docker pull $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY_PREFIX/library/nginx:latest
```

If the repository “$ECR_REPOSITORY_PREFIX/library/nginx” does not already exist in your registry, it will be automatically created for you.

## Describe the images

```bash
aws ecr describe-images --repository-name $ECR_REPOSITORY_PREFIX/library/nginx --region $AWS_REGION
```

Understand that the describe-iamges response might take a minute to reflect for 1GB image




