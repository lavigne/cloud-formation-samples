## Overview
This solution creates a Route53 private hosted zone as well as resources to automatically create DNS records in that hosted zone for EC2 instances created in workload accounts. Essentially, the hosted zone is created in a networking account along with a Lambda that responds to EC2 start and terminate events to automatically create/remote DNS entries for EC2 instances. For this to work, a CloudFormation stack set is run in all workload accounts that creates an EventBridge rule that listens for EC2 start/terminate events and forwards those events to the default EventBridge bus in the networking account to trigger the DNS record creation Lambda. The entire solution is deployed using an AWS CodePipeline that is created using CloudFormation. It is assumed that the CloudFormation templates that make up this solution are checked into a GitHub repository and a connection has been established between AWS and this GitHub repository. For instructions on how to set up this connection, see this documentation: https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html.

## Prerequisites

1. This solution assumes you are using a GitHub repository to house these CloudFormation templates. In order to set up the solution, you first need to setup a connection between the AWS networking account and this GitHub repository. Be sure to use the "Install a new app" option when setting up the connection. This installs the "AWS Connector for GitHub" app in your GitHub account which allows it to access your repositories using OAuth. For more information see: https://docs.aws.amazon.com/codepipeline/latest/userguide/connections-github.html.

2. You must have CloudTrail enabled for automatic DNS record creation to work because the EventBridge rule in workload accounts listens for CloudTrail `RunInstances` and `TerminateInstances` events. It is recommended that you create an organization trail so that all events across the organization are captured.

3. This solution uses stack sets to deploy resources across multiple member accounts. You enter the account ID for the account from which to run stack sets and the account IDs of member (workload) accounts as parameters to the `1-pipeline.yml` template. If you have access to the management account, it is easiest to have the stack sets run in the management account because it eliminates the need to create IAM roles that are required if you aren't using the management account. If you are going to be deploying stack sets from an account other than the management account, there are two required IAM roles that need to be created. To create these roles, run the templates in the `stack-set-required-roles` folder. The `stack-set-admin-role.yml` template needs to be run in the account that will run the stack sets and the `stack-set-execution-role.yml` template needs to be run in all accounts where the stack sets will be deploying resources. For more information see: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html.

## Deployment Steps

1. Log into the networking account and run the `1-pipeline.yml` template. This will create and run an AWS CodePipeline deployment pipeline that will run the other CloudFormation templates to create all the resources required in the networking and workload accounts. The `1-pipeline.yml` template requires the following parameters:

- BranchName
- ConnectionArn
- DnsRecordFailureNotificationEmailAddress
- DomainName
- NetworkAccountId
- OrganizationId
- PrimaryRegion
- Regions

## Troubleshooting
If you're not seeing DNS entries created for new EC2 instances, first be sure that CloudTrail is enabled in the account where the EC2 instance was created.