# Deliverable ⓺ Scalable deployment: JWT Pizza Service

![course overview](../courseOverview.png)

Now that you have some experience with creating, registering, and deploying simple containers it is time to deploy JWT Pizza Service to the Cloud.

## Setup ECR

Follow the [AWS ECR instruction](../awsEcr/awsEcr.md) in order to get your AWS account setup to store the JWT Pizza Service container images in a Docker compliant Registry.

## Setup ECS

Follow the [AWS ECS instruction](../awsEcs/awsEcs.md) in order to get your AWS account setup to host the JWT Pizza Service container using Fargate.

## GitHub action: Upload container image

In order to automate building and uploading a Docker container image for the JWT Pizza service, you need increase the rights that the workflow has, and modify workflow script to build and push the image.

### Modify the IAM trust policy

We need to modify the `github-ci` IAM role so that it can create an OIDC connection when requests are made from the `jwt-pizza-service` GitHub repository.

1. Open the AWS IAM service console.
1. Choose `Roles`.
1. Select the `github-ci` role that you created when you setup `jwt-pizza` to deploy to S3.
1. Select the `Trust relationships` tab.
1. Press `Edit trust policy`.
1. Replace the `token.actions.githubusercontent.com:sub` value with the following. This allows both of your source repositories to make an OIDC connection.
   ```json
   "token.actions.githubusercontent.com:sub": [
   "repo:YOURGITHUBACCOUNTHERE/jwt-pizza:ref:refs/heads/main",
   "repo:YOURGITHUBACCOUNTHERE/jwt-pizza-service:ref:refs/heads/main"
   ],
   ```
1. Press the `Update policy` button.

### Enhance the IAM rights

Next you need to enhance the `github-ci` user rights so that they can push to ECR and initiate the deployment to ECS.

1. Open the AWS IAM service console.
1. Choose `Roles`.
1. Select the `github-ci` role that you created when you setup `jwt-pizza` to deploy to S3.
1. Select the `Permissions` tab.
1. Click on the `jwt-pizza-ci-deployment` policy.
1. Select `JSON` and press `Edit`.
1. Add the following statement in order to allow the use of ECR and ECS.

```json
{
  "Sid": "PushAndDeployImage",
  "Effect": "Allow",
  "Action": ["ecr:*", "ecs:*"],
  "Resource": ["*"]
}
```

1. Press `Next`.

### Modify the workflow script for upload

Previously the workflow stopped after the tests were done and the coverage badge was updated. Now, you need to do the following steps:

1. Create a distribution folder that will become our Docker container. This copies all of the source code files and the newly created Dockerfile
   ```yml
   - name: Create dist
     run: |
       mkdir dist
       cp Dockerfile dist
       cp -r src/* dist
       cp *.json dist
   ```
1. Authenticate to AWS using OIDC. This is the same authentication step that we took with the frontend deployment. Using OIDC makes it so we don't have to store any credentials to our AWS account.
   ```yml
   - name: Create OIDC token to AWS
     uses: aws-actions/configure-aws-credentials@v4
     with:
       audience: sts.amazonaws.com
       aws-region: us-east-2
       role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT }}:role/${{ secrets.CI_IAM_ROLE }}
   ```
1. Login to AWS ECR. We need to provide credentials to Docker so that it can push container images into the register. This [action](https://github.com/aws-actions/amazon-ecr-login) gets a temporary password from ECR using the OIDC credential we previously obtained.

   ```yml
   - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
   ```

1. Set up the Docker build and emulation. We need to build an ARM64 platform package. In order to do that in Linux we need to emulate an ARM based environment.

   ```yml
   - name: Set up machine emulation
      uses: docker/setup-qemu-action@v3

   - name: Set up Docker build
      uses: docker/setup-buildx-action@v3
   ```

1. Build and push the docker container. We used the `--push` parameter to automatically push the container image to ECR once it is done building.
   ```yml
   - name: Build and push container image
     env:
       ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
       ECR_REPOSITORY: 'jwt-pizza-service'
     run: |
       docker --version
       cd dist
       ls -la
       docker build --platform=linux/arm64 -t $ECR_REGISTRY/$ECR_REPOSITORY --push .
   ```

### Test upload

You should now be able to commit and push the workflow script to GitHub. This will trigger your container image to build and push to ECR. Once it is done you should see your image show up on the ECR images dashboard.

![ECR images](ecrImages.png)

## GitHub action: Deploy container

With a new image in the registry you can now automate the deployment of the container to Fargate.

1. Create a new task definition.
1. Update the service to the new task.
