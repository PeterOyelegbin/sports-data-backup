# Sports Data Backup
Sports Data Backup automates the retrieval of sports highlights, stores data in S3 and DynamoDB, processes videos, and operates on a schedule using ECS Fargate and EventBridge. It employs templated JSON files with environment variable injection for straightforward configuration and deployment.
![BackupHighlights](https://github.com/user-attachments/assets/64014e8b-bcc6-4bf8-ad62-29188c2a0907)

## Prerequisites
Before running the scripts, ensure you have the following:
1. **Create Rapidapi Account:** We will be using NCAA (USA College Basketball) highlights since it's included for free in the basic plan. [Sports Highlights API Endpoint](https://rapidapi.com/highlightly-api-highlightly-api-default/api/sport-highlights-api/playground/apiendpoint_16dd5813-39c6-43f0-aebe-11f891fe5149)

2. **Verify prerequisites are installed**
   - Docker should be pre-installed, docker --version
   - AWS CLI should be pre-installed, aws --version
   - Python3 should be pre-installed, python3 --version
   - Install gettext package: envsubst is a command-line utility for environment variable substitution in shell scripts and text files. [Install Guide](https://www.drupal.org/docs/8/modules/potion/how-to-install-setup-gettext)

3. **Retrieve AWS Account ID:** Copy your AWS Account ID Once logged in to the AWS Management Console. Click on your account name in the top right corner, you will see your account ID. Copy and save this somewhere safe because you will need to update codes in the labs later

4. **Retrieve Access Keys and Secret Access Keys:** Create an access key if you do not have an existing access key in the IAM dashboard Under Users

## Step-by-Step Procedure 
1. **Clone The Repo**
```bash
git clone https://github.com/PeterOyelegbin/sports-data-backup.git
cd sports-data-backup/src
```

2. **Setup VPC and Update .env File**
   * Run the script
     ```bash
     bash resources/vpc_setup.sh
     ```
   * You will see variables in the output, copy and paste these variables into Subnet_ID and Security_Group_ID in the .env file
     ![vpc_setup](images/vpc_setup.png)

  * Update the .env File
    Search & Replace the following values:
    - Your-AWS-Account-ID
      ```bash
      aws sts get-caller-identity --query "Account" --output text
      ```
    - Your-RAPIDAPI-Key
    - Your-AWS-Access-Key
    - Your-AWS-Secret-Access-key
    - S3_BUCKET_NAME=your-alias
    - Your-MediaConvert-Endpoint
      ```bash
      aws mediaconvert describe-endpoints
      ```
    - SUBNET_ID=subnet-<Your-SubnetId> 
    - SECURITY_GROUP_ID=sg-<Your-SecurityGroupId>

3. **Create DynamoDB Table**
In the CLI, run the following command to create an on-demand table
```bash
aws dynamodb create-table \
    --table-name SportsHighlights \
    --attribute-definitions AttributeName=id,AttributeType=S \
    --key-schema AttributeName=id,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```
![vpc_setup](images/vpc_setup.png)

4. **Load Environment Variables**
```bash
set -a
source .env
set +a
```
Optional - Verify the variables are loaded
```bash
echo $AWS_LOGS_GROUP
echo $TASK_FAMILY
echo $AWS_ACCOUNT_ID
```

5. **Generate Final JSON Files from Templates**
   * ECS Task Definition
     ```bash
     envsubst < taskdef.template.json > taskdef.json
     ```
   * S3/DynamoDB Policy
     ```bash
     envsubst < s3_dynamodb_policy.template.json > s3_dynamodb_policy.json
     ```
   * ECS Target
     ```bash
     envsubst < ecsTarget.template.json > ecsTarget.json
     ```
   * ECS Events Role Policy
     ```bash
     envsubst < ecsEventsRole-policy.template.json > ecsEventsRole-policy.json
     ```
   * Optional - Open the generated files using cat or a text editor to confirm that all placeholders have been correctly replaced

6. **Build and Push Docker Image**
   * Create an ECR Repo
     ```bash
     aws ecr create-repository --repository-name sports-backup
     ```
   * Log In To ECR
     ```bash
     aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
     ```
   * Build the Docker Image
     ```bash
     docker build -t sports-backup .
     ```
   * Tag the Image for ECR
     ```bash
     docker tag sports-backup:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
     ```
   * Push the Image
     ```bash
     docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/sports-backup:latest
     ```
   * If sudo is needed to push the docker image, copy your login credentials to enable the root
     ```bash
     sudo cp ~/.docker/config.json /root/.docker
     ```

7. **Create AWS Resources**
   * Register the ECS Task Definition
     ```bash
     aws ecs register-task-definition --cli-input-json file://taskdef.json --region ${AWS_REGION}
     ```
   * Create the CloudWatch Logs Group
     ```bash
     aws logs create-log-group --log-group-name "${AWS_LOGS_GROUP}" --region ${AWS_REGION}
     ```
   * Ensure CloudWatch logs capture ECS task execution logs
     ```bash
     aws logs describe-log-groups --log-group-name-prefix "${AWS_LOGS_GROUP}"
     ```
   * Set log retention duration
     ```bash
     aws logs put-retention-policy --log-group-name "${AWS_LOGS_GROUP}" --retention-in-days 10
     ```
   * Attach the S3/DynamoDB Policy to the ECS Task Execution Role
     ```bash
     aws iam put-role-policy \
      --role-name ecsTaskExecutionRole \
      --policy-name S3DynamoDBAccessPolicy \
      --policy-document file://s3_dynamodb_policy.json
     ```
   * Set up the ECS Events Role
     Create the Role with Trust Policy
     ```bash
     aws iam create-role --role-name ecsEventsRole --assume-role-policy-document file://ecsEventsRole-trust.json
     ```
   * Attach the Events Role Policy
     ```bash
     aws iam put-role-policy --role-name ecsEventsRole --policy-name ecsEventsPolicy --policy-document file://ecsEventsRole-policy.json
     ```

8. **Create an EventBridge Rule to Schedule the Task**
   * Create the Rule
     ```bash
     aws events put-rule --name SportsBackupScheduleRule --schedule-expression "rate(1 day)" --region ${AWS_REGION}
     ```
  * Add the Target
     ```bash
     aws events put-targets --rule SportsBackupScheduleRule --targets file://ecsTarget.json --region ${AWS_REGION}
     ```

9. **Manually Test ECS Task**
```bash
aws ecs run-task \
  --cluster sports-backup-cluster \
  --launch-type FARGATE \
  --task-definition ${TASK_FAMILY} \
  --network-configuration "awsvpcConfiguration={subnets=[\"${SUBNET_ID}\"],securityGroups=[\"${SECURITY_GROUP_ID}\"],assignPublicIp=\"ENABLED\"}" \
  --region ${AWS_REGION}
```
![vpc_setup](images/vpc_setup.png)

## What I Learned
1. Using templates to generate JSON files
2. Integrating DynamoDB to store data backup
3. Cloudwatcher for logging

## Future Enhancements
1. Integrate exporting a table from DynamoDB to an S3 bucket
2. Configure an automated backup
3. Creating batch processing of the entire JSON file (importing more than 10 videos)
