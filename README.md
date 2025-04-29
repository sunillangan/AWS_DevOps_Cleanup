# AWS_DevOps_Cleanup

Project Title: Automated EC2 Backup & Termination System with Approval Workflow

Overview

This project automates the process of:
	1.	Receiving EC2 termination requests via an API
	2.	Backing up EC2 instances as AMIs before termination
	3.	Validating via AWS Config or EC2 API
	4.	Logging the outcome and sending notifications

Perfect project for DevOps + AWS learners, job seekers, and portfolio building.

⸻

Architecture Components
	•	API Gateway: Receives termination request (Private/Public)
	•	Lambda Function: Executes logic to validate, backup, and terminate EC2
	•	AWS Config: (Optional) Searches for instance metadata
	•	SNS/SES: Sends email notification on success/failure
	•	IAM Roles: With least privilege for Lambda
	•	CloudWatch Logs: For audit trail

⸻

Step-by-Step Guide

⸻

Step 1: Create IAM Role for Lambda

Create a role with the following policies:
	•	AmazonEC2FullAccess (fine-tune later)
	•	AWSConfigUserAccess
	•	AmazonSNSFullAccess or SES permissions
	•	CloudWatchLogsFullAccess

Attach this role to your Lambda function.

⸻

Step 2: Create Lambda Function

Function Name: ec2-termination-handler

Runtime: Python 3.12
Timeout: 3 mins

code : import boto3, json, os
from datetime import datetime

ec2 = boto3.client('ec2')
sns = boto3.client('sns')
config = boto3.client('config')

SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN')

def lambda_handler(event, context):
    try:
        payload = json.loads(event['body']) if 'body' in event else event
        instance_id = payload['instance_id']
        owner = payload['owner']
        reason = payload['reason']
        
        # Validate instance
        instance = ec2.describe_instances(InstanceIds=[instance_id])
        if not instance:
            raise Exception("Instance not found")

        # Backup (Create AMI)
        timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
        ami_name = f"Backup-{instance_id}-{timestamp}"
        ami = ec2.create_image(
            InstanceId=instance_id,
            Name=ami_name,
            Description=f"Auto-backup before termination",
            NoReboot=True
        )
        
        # Tag AMI
        ec2.create_tags(
            Resources=[ami['ImageId']],
            Tags=[
                {'Key': 'CreatedBy', 'Value': owner},
                {'Key': 'BackupReason', 'Value': reason}
            ]
        )
        
        # Terminate instance
        ec2.terminate_instances(InstanceIds=[instance_id])
        
        message = f"""✅ EC2 instance {instance_id} terminated.
AMI backup created: {ami['ImageId']}
Owner: {owner}
Reason: {reason}
"""
        print(message)
        if SNS_TOPIC_ARN:
            sns.publish(Subject="EC2 Termination Success", Message=message, TopicArn=SNS_TOPIC_ARN)

        return {"statusCode": 200, "body": json.dumps({"status": "success", "ami_id": ami['ImageId']})}

    except Exception as e:
        error_msg = f"❌ Termination failed: {str(e)}"
        print(error_msg)
        if SNS_TOPIC_ARN:
            sns.publish(Subject="EC2 Termination Failed", Message=error_msg, TopicArn=SNS_TOPIC_ARN)
        return {"statusCode": 500, "body": json.dumps({"status": "error", "message": str(e)})}

Environment Variables:
	•	SNS_TOPIC_ARN: <your-sns-topic-arn>

⸻

Step 3: Create SNS Topic for Notifications
	1.	Create a new SNS topic ec2-termination-alerts
	2.	Subscribe your email or webhook
	3.	Use the ARN in Lambda

⸻

Step 4: Setup API Gateway (HTTP API or REST API)
	•	Create a new HTTP API
	•	Integration: Lambda Function
	•	Enable CORS
	•	Endpoint: /terminate-ec2

payload sample : {
  "instance_id": "i-0abcd1234efgh5678",
  "owner": "virender",
  "reason": "Old testing instance"
}

Step 5: Deploy the API & Test

Use Postman or curl:curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/terminate-ec2 \
-H "Content-Type: application/json" \
-d '{"instance_id":"i-0abcd1234efgh5678","owner":"virender","reason":"cleanup"}'

Step 6: Monitor and Audit
	•	Use CloudWatch Logs to monitor Lambda output
	•	Use SNS Email Alerts for success/failure logs
	•	(Optional) Store logs in S3 using Lambda destination or CloudTrail

⸻

Bonus (Optional Enhancements)
	•	Add ServiceNow/Jira ticket integration
	•	Add Slack webhook for DevOps alerts
	•	Integrate with Step Functions for multi-step flows
	•	Protect API Gateway with API Key / Cognito / IAM Auth

What to Write in Your Resume:

Project: Automated EC2 Backup & Termination System using AWS Lambda, API Gateway, and SNS
	•	Built an event-driven solution to backup and terminate EC2 instances securely
	•	Implemented AMI backup with tagging and email-based alerting using SNS
	•	Integrated CloudWatch for logging and AWS Config for resource validation
	•	Tools used: Lambda (Python), EC2, SNS, API Gateway, CloudWatch, IAM
