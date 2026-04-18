GuardDuty → EventBridge → Lambda → SNS → Email Alert

  -------------------------------------------------------
  STEP 1: IAM role for this setup
  -------------------------------------------------------
  1. Go to IAM. 
  2. Click Role--> create role.
  3. Select Lambda service
  4. Add lambda basic execution role
  5. create role

Add SNS - write inline policy to same IAM role 
  3. In IAM, click “Add Permissions”.
  4. Click “Create Inline Policy”.
  5. Select JSON tab.

  Paste the following policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "*"
    }
  ]
}

  Save the policy.



  -------------------------------------------------------
  STEP 2: ENABLE GUARDDUTY
  -------------------------------------------------------
  1. Log in to AWS Console. 2. Search for GuardDuty. 3.
  Click “Get Started”. 4. Click “Enable GuardDuty”.

  GuardDuty will begin monitoring: - CloudTrail logs -
  VPC Flow Logs - DNS Logs
  -------------------------------------------------------

 STEP 3: CREATE SNS TOPIC

1.  Open AWS Console.
2.  Search for SNS (Simple Notification Service).
3.  Click “Create Topic”.

Configuration: Type: Standard Name: guardduty-alert-topic

Click “Create Topic”.

  -------------------------------------------------------
  STEP 4: CREATE EMAIL SUBSCRIPTION
  -------------------------------------------------------
  1. Open the SNS topic you created. 
  2. Click “Create Subscription”.

  Configuration: 
  Protocol: Email Endpoint:
  your-email@example.com

  3. Click Create Subscription. 
  4. Check your email and  confirm the subscription.
  -------------------------------------------------------

STEP 4: CREATE LAMBDA FUNCTION

1.  Open AWS Lambda.
2.  Click “Create Function”.

Configuration: 
Function Name: guardduty-alert-function 
Runtime: Python3.x 
Execution Role: select IAM role created earlier.

Click Create Function.


  -------------------------------------------------------

STEP 6: ADD ENVIRONMENT VARIABLE

1.  Open Lambda function.
2.  Go to Configuration → Environment Variables.
3.  Add:

Key: SNS_TOPIC_ARN Value:

  -------------------------------------------------------
  STEP 7: ADD LAMBDA CODE
  -------------------------------------------------------


import json
import boto3
import os

sns = boto3.client("sns")
topic_arn = os.environ["SNS_TOPIC_ARN"]

def lambda_handler(event, context):

    detail = event.get("detail", {})

    severity = detail.get("severity", "Unknown")
    finding_type = detail.get("type", "Unknown")
    title = detail.get("title", "GuardDuty Alert")
    description = detail.get("description", "")

    message = f"""
GuardDuty Security Alert

Title: {title}
Type: {finding_type}
Severity: {severity}

Description: {description}
"""

    sns.publish(
        TopicArn=topic_arn,
        Subject="GuardDuty Security Alert",
        Message=message
    )

    return {
        "status": "Alert Sent"
    }
  -------------------------------------------------------

STEP 8: CREATE EVENTBRIDGE RULE

1.  Open EventBridge.
2.  Click Rules.
3.  Click “Create Rule”.

Rule Name: guardduty-findings-rule

Rule Type: Rule with event pattern

  ---------------------------
  STEP 9: ADD EVENT PATTERN
  ---------------------------

{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [
      {
        "numeric": [">=", 4]
      }
    ]
  }
}
This triggers the rule only when severity >= 4.

  -------------------------------------------------------
  STEP 10: ADD TARGET
  -------------------------------------------------------
  Target Type: Lambda Function

  Select: guardduty-alert-function

  Click Create Rule.
  -------------------------------------------------------

STEP 11: TEST THE AUTOMATION

Method 1: Lambda Test

Open Lambda → Test

Use event:

{
  "detail": {
    "severity": 8,
    "type": "UnauthorizedAccess:IAMUser/MaliciousIPCaller",
    "title": "Test GuardDuty Finding",
    "description": "Testing security automation pipeline"
  }
}
Method 2: EventBridge Test Event

Send this event through EventBridge:

{
  "version": "0",
  "id": "abcd1234-5678-90ab-cdef-EXAMPLE11111",
  "detail-type": "GuardDuty Finding",
  "source": "aws.guardduty",
  "account": "123456789012",
  "time": "2026-03-16T10:00:00Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "severity": 8,
    "type": "UnauthorizedAccess:IAMUser/MaliciousIPCaller",
    "title": "Security Test"
  }
}
  --------------------
  FINAL ARCHITECTURE
  --------------------

GuardDuty 
    ↓
EventBridge Rule 
    ↓
 Lambda Function 
    ↓
 SNS Topic 
    ↓
 Email Alert
 -----------------------------------------------------------------------------------------------------------
What exactly you’re billed for
1. VPC Flow Logs analysis
Tracks network traffic in your VPC
Pricing depends on GB of logs analyzed


2. DNS Logs analysis
Monitors DNS queries for suspicious domains
Charged per million DNS queries

3. CloudTrail Events analysis
Looks at API activity (like IAM, EC2 actions)
Charged per million events analyzed

4. EKS Audit Logs (if enabled)
Monitors Kubernetes API activity
Charged per million audit log events

5. Malware Protection (EBS scanning)
Scans EBS volumes for malware
Charged per GB scanned

6. Runtime Monitoring (EKS / EC2)
Detects threats during runtime
Charged per vCPU per hour

