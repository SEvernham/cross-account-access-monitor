# Cross-Account Access Monitor

This CloudFormation template creates a monitoring solution that detects when external AWS accounts access resources in your linked account and sends email notifications via Amazon SNS.

## What It Does

The solution monitors for cross-account access patterns and sends email alerts when:
- An IAM user, role, or root user from a different AWS account accesses resources in your account
- Access occurs to monitored services like DynamoDB, S3, EC2, RDS, or Lambda
- The accessing account is not part of your AWS Organization (if Organization ID is provided)

## Architecture Components

1. **Amazon CloudTrail** - Captures API calls and events across your AWS account
2. **Amazon EventBridge** - Filters and routes relevant cross-account access events
3. **AWS Lambda** - Processes events and determines if notification should be sent
4. **Amazon SNS** - Sends email notifications to specified recipients
5. **Amazon S3** - Stores CloudTrail logs with lifecycle management

## Prerequisites

- AWS account with appropriate permissions to create the resources
- Valid email address for receiving notifications
- (Optional) AWS Organization ID if you want to exclude organization members from alerts

## Installation Instructions

### Method 1: AWS Console (Recommended)

1. **Download the Template**
   - Save the `cross-account-access-monitor.yaml` file to your local machine

2. **Open AWS CloudFormation Console**
   - Navigate to the [CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
   - Ensure you're in the correct AWS region

3. **Create Stack**
   - Click "Create stack" → "With new resources (standard)"
   - Select "Upload a template file"
   - Choose the `cross-account-access-monitor.yaml` file
   - Click "Next"

4. **Configure Parameters**
   - **Stack name**: Enter a descriptive name (e.g., `cross-account-monitor`)
   - **NotificationEmail**: Enter your email address for alerts
   - **OrganizationId**: (Optional) Enter your AWS Organization ID to exclude org members
   - **MonitoredServices**: Comma-separated list of services to monitor (default: dynamodb,s3,ec2,rds,lambda)
   - Click "Next"

5. **Configure Stack Options**
   - Add tags if desired (recommended: Environment, Owner, Purpose)
   - Leave other options as default
   - Click "Next"

6. **Review and Create**
   - Review all settings
   - Check "I acknowledge that AWS CloudFormation might create IAM resources"
   - Click "Create stack"

7. **Confirm Email Subscription**
   - Check your email for an SNS subscription confirmation
   - Click the confirmation link to start receiving notifications

### Method 2: AWS CLI

```bash
# Deploy the stack
aws cloudformation create-stack \
  --stack-name cross-account-monitor \
  --template-body file://cross-account-access-monitor.yaml \
  --parameters ParameterKey=NotificationEmail,ParameterValue=your-email@example.com \
               ParameterKey=OrganizationId,ParameterValue=o-xxxxxxxxxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Check deployment status
aws cloudformation describe-stacks \
  --stack-name cross-account-monitor \
  --query 'Stacks[0].StackStatus'
```

## Configuration Options

### Parameters

- **NotificationEmail** (Required): Email address to receive alerts
- **OrganizationId** (Optional): AWS Organization ID to exclude member accounts from alerts
- **MonitoredServices** (Optional): Comma-separated list of AWS services to monitor

### Monitored Services

By default, the solution monitors these services:
- Amazon DynamoDB
- Amazon S3
- Amazon EC2
- Amazon RDS
- AWS Lambda

You can customize this list by modifying the `MonitoredServices` parameter.

## How It Works

1. **Event Capture**: CloudTrail logs all API calls in your account
2. **Event Filtering**: EventBridge rules filter for cross-account access events
3. **Processing**: Lambda function analyzes events to determine if they're from external accounts
4. **Notification**: SNS sends email alerts for confirmed cross-account access

## Sample Alert Email

```
Subject: Cross-Account Access Alert - Account 123456789012

CROSS-ACCOUNT ACCESS DETECTED

Alert Time: 2025-06-25 13:00:00 UTC

ACCESS DETAILS:
- Source Account: 987654321098
- Target Account: 123456789012
- Event: GetItem
- Service: dynamodb.amazonaws.com
- Event Time: 2025-06-25T12:59:45Z

USER IDENTITY:
- Type: AssumedRole
- Principal ID: AIDACKCEVSQ6C2EXAMPLE
- ARN: arn:aws:sts::987654321098:assumed-role/CrossAccountRole/user
- User Name: N/A

NETWORK DETAILS:
- Source IP: 203.0.113.12
- User Agent: aws-cli/2.0.0

RECOMMENDATION:
Please review this access to ensure it is authorized...
```

## Cost Considerations

### Detailed Cost Breakdown

**CloudTrail**
- Data Events: ~$0.10 per 100,000 data events
- Management Events: First copy of management events is free per region
- Typical Usage: 50,000-200,000 events per month for moderately active accounts
- Estimated Cost: $0.50 - $2.00/month

**Lambda**
- Invocations: $0.20 per 1 million requests
- Duration: $0.0000166667 per GB-second
- Typical Usage: 100-1,000 invocations per month (depending on cross-account activity)
- Estimated Cost: $0.01 - $0.10/month

**SNS**
- Email Notifications: $0.50 per 1 million notifications
- Typical Usage: 10-100 notifications per month
- Estimated Cost: <$0.01/month

**S3 Storage**
- CloudTrail Logs: $0.023 per GB/month (Standard storage)
- Typical Usage: 1-5 GB per month for logs
- 90-day lifecycle: Automatic deletion keeps costs controlled
- Estimated Cost: $0.02 - $0.12/month

**EventBridge**
- Custom Rules: $1.00 per million events processed
- Typical Usage: Same as CloudTrail events
- Estimated Cost: $0.05 - $0.20/month

### Monthly Cost Estimates by Usage Level

| Usage Level | Account Activity | Monthly Cost |
|-------------|------------------|--------------|
| **Light** | Small account, few cross-account events | $1 - $3 |
| **Moderate** | Typical enterprise account | $3 - $8 |
| **Heavy** | High-activity account, frequent cross-account access | $8 - $15 |

### Cost Optimization Tips

1. **Adjust CloudTrail Data Events**: Only monitor specific S3 buckets or DynamoDB tables if needed
2. **Lifecycle Management**: The 90-day S3 lifecycle rule keeps storage costs low
3. **Event Filtering**: Customize monitored services to reduce unnecessary processing
4. **Organization Filtering**: Use Organization ID to reduce false positives and processing

### One-Time Setup Costs
- No upfront costs - all resources are pay-as-you-go
- Can be deleted anytime without penalties
- Most organizations see costs in the $5-10/month range

## Security Best Practices

1. **Least Privilege**: The solution uses minimal required permissions
2. **Encryption**: CloudTrail logs are encrypted at rest
3. **Access Control**: S3 bucket blocks public access
4. **Monitoring**: All components generate CloudWatch logs

## Troubleshooting

### No Alerts Received
1. Check email subscription confirmation
2. Verify CloudTrail is enabled and logging events
3. Check Lambda function logs in CloudWatch
4. Ensure EventBridge rule is enabled

### False Positives
1. Add your Organization ID to exclude member accounts
2. Modify the Lambda function to add custom filtering logic
3. Adjust monitored services list

### Missing Events
1. Verify CloudTrail covers all regions (multi-region trail enabled)
2. Check EventBridge rule event pattern matches your use case
3. Ensure monitored services are included in the configuration

## Customization

### Adding New Services
Modify the EventBridge rule's event pattern to include additional AWS services:

```yaml
EventPattern:
  source:
    - aws.dynamodb
    - aws.s3
    - aws.your-new-service
```

### Custom Filtering Logic
Edit the Lambda function code to add custom business logic for determining when to send alerts.

### Integration with Security Tools
The SNS topic can be integrated with:
- AWS Security Hub
- Third-party SIEM tools
- Slack/Teams notifications
- Custom webhook endpoints

## Cleanup

To remove all resources:

```bash
aws cloudformation delete-stack --stack-name cross-account-monitor
```

Note: The S3 bucket may need to be emptied manually before stack deletion.

## Support

For issues or questions:
1. Check AWS CloudFormation events for deployment issues
2. Review CloudWatch logs for Lambda function errors
3. Verify IAM permissions for all components
4. Ensure all required parameters are correctly configured

## License

This solution is provided as-is for educational and operational purposes. Review and test thoroughly before deploying in production environments.
