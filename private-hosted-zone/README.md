## Overview
This solution creates a Route53 private hosted zone as well as resources to automatically create DNS records in that hosted zone for EC2 instances created in workload accounts. Essentially, the hosted zone is created in a networking account along with a Lambda that responds to EC2 start and terminate events to automatically create/remote DNS entries for EC2 instances using the instance's name tag. For this to work, a CloudFormation stack set is run in all workload accounts that creates an EventBridge rule that listens for EC2 start/terminate events and forwards those events to the default EventBridge bus in the networking account which trigger the DNS record creation Lambda. The entire solution is deployed using an AWS CodePipeline that is created using CloudFormation. It is assumed that the CloudFormation templates that make up this solution are checked into a GitHub repository and a connection has been established between AWS and this GitHub repository. For instructions on how to set up this connection, see this documentation: https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html.

## Prerequisites

1. This solution assumes you are using a GitHub repository to house these CloudFormation templates. In order to set up the solution, you first need to setup a connection between the AWS networking account and this GitHub repository. Be sure to use the "Install a new app" option when setting up the connection. This installs the "AWS Connector for GitHub" app in your GitHub account which allows it to access your repositories using OAuth. For more information see: https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html.

2. You must have CloudTrail enabled for automatic DNS record creation to work because the EventBridge rule in workload accounts listens for CloudTrail `RunInstances` and `TerminateInstances` events. It is recommended that you create an organization trail so that all events across the organization are captured.

3. This solution uses stack sets to deploy resources across multiple member accounts. You enter the account ID for the account from which to run stack sets and the account IDs of member (workload) accounts as parameters to the `1-pipeline.yml` template. If you have access to the management account, it is easiest to have the stack sets run in the management account because it eliminates the need to create IAM roles that are required if you aren't using the management account. If you are going to be deploying stack sets from an account other than the management account, there are two required IAM roles that need to be created. To create these roles, run the templates in the `stack-set-required-roles` folder. The `stack-set-admin-role.yml` template needs to be run in the account that will run the stack sets and the `stack-set-execution-role.yml` template needs to be run in all accounts where the stack sets will be deploying resources. For more information see: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html.

## Deployment Steps

1. Log into the networking account and run the `1-pipeline.yml` template. This will create and run an AWS CodePipeline deployment pipeline that will run the other CloudFormation templates to create all the resources required in the networking and workload accounts. The `1-pipeline.yml` template requires the following parameters:

- DomainName - The domain name used by the private hosted zone.
- DnsRecordFailureNotificationEmailAddress - Email address where DNS record creation failure notices will be sent.
- OrganizationId - AWS Organization ID.
- PrimaryRegion - Primary region used by the AWS Organization (where ControlTower is deployed).
- NetworkAccountId - AWS account ID of the networking account where the private hosted zone will be created.
- WorkloadRegions - AWS regions where EC2 instances will be run that should have DNS entries automatically created.
- WorkloadAccounts - AWS account IDs of accounts where EC2 instances will be run that should have DNS entries automatically created.
- ConnectionArn - ARN of the AWS CodeConnections connection to the GitHub repository where the CloudFormation templates for this solution reside.
- Repository - Repository path for the GitHub repository where the CloudFormation templates for this solution reside, including the GitHub account name in the format "account/repository".
- BranchName - The branch that the created pipeline will pull from when it's triggered. Pushing changes to this branch will automatically trigger the pipeline.

## Troubleshooting
If you're not seeing DNS entries created for new EC2 instances:

1. Make sure you wait several minutes for the DNS record to be created. Sometimes it can take a bit for the events to propagate from the workload account to the network account.

2. Verify that CloudTrail is enabled in the account where the EC2 instance was created.

3. Look at EventBridge in the account where the EC2 instance was created and verify the EventBridge rule that listens for RunInstances and TerminateInstances is being triggered. The primary reason this rule wouldn't be triggered is because CloudTrail isn't enabled.

4. Look at EventBridge in the networking account and verify the rule that listens for RunInstances and TerminateInstances events from workload accounts is being triggered. If the event isn't making it from the workload account to the network account's EventBridge, the most likely reason is that the workload account isn't allowed to send events to the network account's default EventBridge bus. There is a resource named `NetworkAccountDefaultEventBusPolicy` that is created in the `2-network-account-primary-region.yml` CloudFormation template that should provide these permissions. Go to CloudFormation in the networking account and verify that this policy has been created by that template by looking at the Resources created by the template.

5. Look at Lambda in the networking account and verify the Create DNS record Lambda is being invoked. If the Lambda isn't being invoked, verify that EVentBridge in the networking account has permission to invoke the Lambda function. There is a resource named `CreateDnsRecordLambdaEventBridgePermission` in the `2-network-account-primary-region.yml` template that should create the correct permissions. Verify these permissions are being created.

6. Look at the CloudWatch logs for the Create DNS record Lambda to investigate the reason for record creation failure. The most likely reason for a failure is that the Lambda doesn't have the correct permissions to call DescribeInstances in the workloads account to get information about the EC2 instance. There is a role named `GetInstanceDetailsLambdaCrossAccountRole` that is created by the `3-all-accounts-primary-region-stackset.yml` template that should provide the correct permissions. Verify this role exists in the workloads account and that the Lambda in the networking account has permission to assume the role.