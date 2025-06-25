# Testing Cross-Account Access Monitor

## Method 1: Cross-Account Role Assumption Testing

This is the most realistic testing method as it simulates actual cross-account access scenarios that would occur in production environments.

### Prerequisites

- Two AWS accounts (or access to two AWS accounts):
  - **Target Account**: Where the monitoring solution is deployed
  - **Source Account**: External account that will access resources
- AWS CLI configured for both accounts
- Appropriate permissions in both accounts

### Step 1: Deploy the Monitoring Solution

First, ensure the cross-account access monitor is deployed in your target account:

```bash
# Deploy the monitoring stack
aws cloudformation create-stack \
  --stack-name cross-account-monitor \
  --template-body file://cross-account-access-monitor.yaml \
  --parameters ParameterKey=NotificationEmail,ParameterValue=your-email@example.com \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Wait for deployment to complete
aws cloudformation wait stack-create-complete \
  --stack-name cross-account-monitor
```

### Step 2: Create Cross-Account Test Role

In the **target account** (where monitoring is deployed), create a role that can be assumed from the external account:

```bash
# Replace EXTERNAL-ACCOUNT-ID with the actual source account ID
EXTERNAL_ACCOUNT_ID="987654321098"
TARGET_ACCOUNT_ID="123456789012"

# Create the cross-account role
aws iam create-role \
  --role-name CrossAccountTestRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::'$EXTERNAL_ACCOUNT_ID':root"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
          "StringEquals": {
            "sts:ExternalId": "test-external-id"
          }
        }
      }
    ]
  }'

# Create a custom policy for testing
aws iam create-policy \
  --policy-name CrossAccountTestPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "dynamodb:ListTables",
          "dynamodb:DescribeTable",
          "dynamodb:GetItem",
          "s3:ListBucket",
          "s3:GetObject",
          "ec2:DescribeInstances",
          "rds:DescribeDBInstances",
          "lambda:ListFunctions"
        ],
        "Resource": "*"
      }
    ]
  }'

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name CrossAccountTestRole \
  --policy-arn arn:aws:iam::'$TARGET_ACCOUNT_ID':policy/CrossAccountTestPolicy
```

### Step 3: Create Test User in Source Account

In the **source account**, create a user that can assume the cross-account role:

```bash
# Create test user
aws iam create-user --user-name CrossAccountTestUser

# Create policy allowing role assumption
aws iam create-policy \
  --policy-name AssumeTestRolePolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::'$TARGET_ACCOUNT_ID':role/CrossAccountTestRole"
      }
    ]
  }'

# Attach policy to user
aws iam attach-user-policy \
  --user-name CrossAccountTestUser \
  --policy-arn arn:aws:iam::'$EXTERNAL_ACCOUNT_ID':policy/AssumeTestRolePolicy

# Create access keys for the test user
aws iam create-access-key --user-name CrossAccountTestUser
```

### Step 4: Configure AWS CLI Profile

Create a new AWS CLI profile for the test user:

```bash
# Add profile for test user (use the access keys from previous step)
aws configure set aws_access_key_id YOUR_TEST_USER_ACCESS_KEY --profile test-user
aws configure set aws_secret_access_key YOUR_TEST_USER_SECRET_KEY --profile test-user
aws configure set region us-east-1 --profile test-user
```

### Step 5: Perform Cross-Account Access Test

Now perform the actual cross-account access that should trigger the monitoring:

```bash
# Step 5a: Assume the cross-account role
aws sts assume-role \
  --role-arn arn:aws:iam::'$TARGET_ACCOUNT_ID':role/CrossAccountTestRole \
  --role-session-name TestCrossAccountAccess \
  --external-id test-external-id \
  --profile test-user

# This will return temporary credentials - save them as environment variables
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

# Step 5b: Perform actions that should trigger alerts
# Test DynamoDB access
aws dynamodb list-tables --region us-east-1

# Test S3 access (if you have S3 buckets)
aws s3 ls

# Test EC2 access
aws ec2 describe-instances --region us-east-1

# Test RDS access
aws rds describe-db-instances --region us-east-1

# Test Lambda access
aws lambda list-functions --region us-east-1
```

### Step 6: Verify Monitoring Results

#### Check Email Notifications
- You should receive email notifications within 5-10 minutes
- Each monitored service action should generate a separate alert

#### Monitor CloudWatch Logs
```bash
# Check Lambda function logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/CrossAccountAccessProcessor \
  --start-time $(date -d '10 minutes ago' +%s)000 \
  --filter-pattern "Cross-account access"

# Check EventBridge rule execution
aws logs filter-log-events \
  --log-group-name /aws/events/rule/CrossAccountAccessDetection \
  --start-time $(date -d '10 minutes ago' +%s)000
```

#### Verify CloudTrail Events
```bash
# Check CloudTrail for the cross-account events
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time $(date -d '1 hour ago' --iso-8601) \
  --end-time $(date --iso-8601)
```

### Expected Test Results

#### Successful Test Indicators:
1. **Email Notifications**: You should receive emails with details like:
   ```
   Subject: Cross-Account Access Alert - Account 123456789012
   
   CROSS-ACCOUNT ACCESS DETECTED
   - Source Account: 987654321098
   - Target Account: 123456789012
   - Event: ListTables
   - Service: dynamodb.amazonaws.com
   ```

2. **CloudWatch Logs**: Lambda function should show successful processing:
   ```
   [INFO] Processing cross-account access event
   [INFO] Source account: 987654321098
   [INFO] Target account: 123456789012
   [INFO] Notification sent successfully
   ```

3. **CloudTrail Events**: Should show the cross-account API calls with external account ID

### Troubleshooting

#### No Email Notifications Received
```bash
# Check SNS subscription status
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:us-east-1:'$TARGET_ACCOUNT_ID':CrossAccountAccessAlerts

# Test SNS topic directly
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:'$TARGET_ACCOUNT_ID':CrossAccountAccessAlerts \
  --subject "Test Alert" \
  --message "Testing SNS functionality"
```

#### Lambda Function Not Triggering
```bash
# Check EventBridge rule status
aws events describe-rule --name CrossAccountAccessDetection

# Check Lambda function permissions
aws lambda get-policy --function-name CrossAccountAccessProcessor
```

#### CloudTrail Not Capturing Events
```bash
# Verify CloudTrail status
aws cloudtrail get-trail-status --name CrossAccountAccessTrail

# Check CloudTrail configuration
aws cloudtrail describe-trails --trail-name-list CrossAccountAccessTrail
```

### Test Cleanup

After successful testing, clean up the test resources:

```bash
# In target account - remove test role and policy
aws iam detach-role-policy \
  --role-name CrossAccountTestRole \
  --policy-arn arn:aws:iam::'$TARGET_ACCOUNT_ID':policy/CrossAccountTestPolicy

aws iam delete-role --role-name CrossAccountTestRole
aws iam delete-policy --policy-arn arn:aws:iam::'$TARGET_ACCOUNT_ID':policy/CrossAccountTestPolicy

# In source account - remove test user and policy
aws iam detach-user-policy \
  --user-name CrossAccountTestUser \
  --policy-arn arn:aws:iam::'$EXTERNAL_ACCOUNT_ID':policy/AssumeTestRolePolicy

aws iam delete-access-key \
  --user-name CrossAccountTestUser \
  --access-key-id YOUR_TEST_USER_ACCESS_KEY

aws iam delete-user --user-name CrossAccountTestUser
aws iam delete-policy --policy-arn arn:aws:iam::'$EXTERNAL_ACCOUNT_ID':policy/AssumeTestRolePolicy
```

### Advanced Testing Scenarios

#### Test Organization Member Exclusion
If you provided an Organization ID during deployment:

1. Test with an account that's part of your organization (should NOT trigger alerts)
2. Test with an external account (should trigger alerts)

#### Test Different Services
Repeat the testing process for each monitored service:
- DynamoDB operations
- S3 bucket access
- EC2 instance management
- RDS database queries
- Lambda function invocations

#### Test Different Identity Types
Test with different AWS identity types:
- IAM User from external account
- IAM Role from external account  
- Root user from external account

This comprehensive testing approach will validate that your cross-account access monitoring solution is working correctly and will alert you to unauthorized external access attempts.
