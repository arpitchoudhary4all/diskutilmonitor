# diskutilmonitor
Multi-Account AWS EC2 Disk Utilization Monitoring

1. Introduction

1.1 Purpose

This document provides detailed technical specifications for implementing a comprehensive disk utilization monitoring solution across multiple AWS accounts using Ansible. The solution addresses the requirement to collect, aggregate, and report disk utilization metrics from EC2 instances distributed across three AWS accounts without relying on additional monitoring tools.

1.2 Background

The enterprise has grown through acquisitions and now operates across three separate AWS accounts, each containing numerous EC2 instances. The Chief Technology Officer (CTO) has identified potential disk space issues as a critical operational risk and requires a solution that provides visibility across the entire infrastructure. As the organization uses Ansible for configuration management, this existing tool has been designated for implementation of the monitoring solution.

1.3 Scope

The monitoring solution encompasses:
- All EC2 instances across three AWS accounts
- Disk utilization metrics for all mounted volumes
- Centralized access management
- Data aggregation and reporting
- Scalability provisions for future account additions

2. Solution Architecture

2.1 Architectural Overview

The solution is built around a central monitoring account that hosts the Ansible control node. This account serves as the hub for monitoring operations, with the following key components:
- Ansible Control Node: Manages the execution of playbooks and tasks
- Cross-Account IAM Roles: Enable secure access across accounts
- Systems Manager (SSM): Facilitates agent-less command execution
- S3 Bucket: Serves as the central repository for collected metrics
- Dashboard Generator: Transforms collected data into consumable reports

The architecture follows a hub-and-spoke model, with the monitoring account acting as the hub and individual AWS accounts as spokes. This design ensures centralized control while maintaining account isolation and security.

2.2 Data Flow

- Ansible control node starts the monitoring process
- The playbook assumes the necessary cross-account IAM roles via AWS STS
- For each account, the playbook discovers all EC2 instances
- For each running instance, disk utilization data is collected via SSM
- Results are formatted as JSON and stored temporarily on the control node
- Aggregated results are uploaded to the central S3 bucket
- Dashboard generator processes the data from S3 to create reports
- Reports are made available through the reporting system

Prerequisites and Setup

3.1 AWS Account Prerequisites

Each AWS account in the monitoring scope requires:

    A. IAM Role Configuration:
        - Role Name: DiskMonitoringRole (consistent across all accounts)
        - Trust Relationship: Trust the monitoring account
        - Permissions:
            ec2:DescribeInstances
            ssm:SendCommand
            ssm:GetCommandInvocation
    B. Systems Manager Setup:
        - SSM Agent installed on all EC2 instances
        - Appropriate instance profiles attached to EC2 instances
        - Network access to SSM endpoints
    C. S3 Bucket:
        - A central S3 bucket in the monitoring account
        - Bucket policy allowing cross-account write access

3.2 IAM Role Configuration

Create the following IAM role in each target account:
    Role Name: DiskMonitoringRole
    
    Trust Relationship Policy:

    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::MONITORING-ACCOUNT-ID:role/AnsibleExecutionRole"
        },
        "Action": "sts:AssumeRole",
        "Condition": {}
        }
    ]
    }

    Permissions Policy:

    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Action": [
            "ec2:DescribeInstances",
            "ssm:SendCommand",
            "ssm:GetCommandInvocation"
        ],
        "Resource": "*"
        },
        {
        "Effect": "Allow",
        "Action": [
            "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::company-disk-metrics/*"
        }
    ]
    }

3.3 Ansible Control Node Setup

    A. Install Required Packages:
            sudo apt update
            sudo apt install -y python3 python3-pip jq
            pip3 install ansible boto3 botocore
    B. Install Required Ansible Collections:
            ansible-galaxy collection install amazon.aws
            ansible-galaxy collection install community.aws
    C. Configure AWS Credentials:
            mkdir -p ~/.aws
            cat > ~/.aws/credentials << EOF
            [monitoring]
            aws_access_key_id = YOUR_ACCESS_KEY
            aws_secret_access_key = YOUR_SECRET_KEY
            region = us-east-1
            EOF

            cat > ~/.aws/config << EOF
            [profile monitoring]
            region = us-east-1
            output = json
            EOF
    D. Create Directory Structure:
            mkdir -p ~/disk-monitoring/{tasks,vars,templates,files}

4. Implementation Details

4.1 File Structure

    disk-monitoring/
    ├── multi_account_disk_monitoring.yml  # Main playbook
    ├── tasks/
    │   ├── process_account.yml           # Account processing tasks
    │   └── check_disk_space.yml          # Disk space checking tasks
    ├── vars/
    │   └── aws_accounts.yml              # Account definition variables
    └── templates/
        └── report_template.j2            # Report template

4.2 Main Playbook (multi_account_disk_monitoring.yml)

The main playbook orchestrates the entire monitoring process and is divided into three distinct plays:

1. **Setup Prerequisites Play**
   - Creates the result directory for storing collected data
   - Cleans previous results to ensure fresh data collection
   - Verifies AWS credentials are properly configured
   - Displays execution information including target accounts

2. **Collection Play**
   - Iterates through each AWS account defined in vars/aws_accounts.yml
   - Includes the process_account.yml task file for each account
   - Maintains a consistent loop structure for easy tracking

3. **Aggregation and Reporting Play**
   - Counts and displays collection statistics
   - Combines all individual JSON results into a single file
   - Uploads the combined results to the S3 bucket with timestamp
   - Generates an HTML summary report using the report_template.j2
   - Creates "latest" pointers for easy access to the most recent report
   - Displays the URL where the report can be accessed

Key features of the main playbook:
- Clear separation of setup, collection, and reporting phases
- Consistent error handling and status reporting
- Use of AWS environment variables for secure credential management
- Timestamped directories for historical data retention

4.3 Task Files

4.3.1 Process Account Task (process_account.yml)

This task file handles the processing of a single AWS account:

- **Role Assumption**
  - Uses the community.aws.sts_assume_role module to assume the DiskMonitoringRole
  - Creates a secure session with temporary credentials
  - Sets up environment variables for AWS API access

- **Instance Discovery**
  - Creates an account-specific results directory
  - Uses the amazon.aws.ec2_instance_info module to discover all EC2 instances
  - Displays the count of instances found in the account

- **Task Delegation**
  - Includes the check_disk_space.yml task for each discovered instance
  - Passes account information to the disk space checking task
  - Uses loop controls for clear labeling and tracking

4.3.2 Check Disk Space Task (check_disk_space.yml)

This task file handles the disk space checking for a single EC2 instance:

- **Instance Verification**
  - Checks if the instance is in a running state
  - Tests if the SSM agent is available on the instance
  - Uses a block/rescue structure for robust error handling

- **Disk Data Collection**
  - Uses the amazon.aws.aws_ssm_send_command module to execute the df command
  - Collects disk utilization data for all mounted volumes
  - Parses and transforms the raw data into structured format

- **Utilization Analysis**
  - Flags volumes with utilization above warning and critical thresholds
  - Categorizes instances based on their most severe volume status
  - Creates a comprehensive JSON document with all relevant metrics

- **Result Storage**
  - Saves the instance results as a JSON file in the results directory
  - Includes account and instance metadata for context
  - Handles errors gracefully with a rescue block

4.4 Variables and Templates

4.4.1 AWS Accounts Variables (aws_accounts.yml)

This file defines the AWS accounts to be monitored and global configuration settings:

- Account definitions including name, ID, region, and description
- Result directory path for temporary storage
- S3 bucket name for permanent storage
- Warning and critical thresholds for disk utilization
- Report retention period in days

4.4.2 Report Template (report_template.j2)

This Jinja2 template generates an HTML report with the following features:

- Responsive design with CSS styling
- Summary section with counts of critical, warning, and OK instances
- Detailed tables for critical and warning instances
- Visual utilization bars for easy identification of issues
- Complete instance summary table
- Contact information and generation timestamp

5. Security Considerations

5.1 Cross-Account Access

The solution uses IAM roles with the principle of least privilege. The cross-account access is strictly controlled through:

    - Limited IAM Role Permissions: Each account has a role with only the necessary permissions to collect disk metrics.
    - Trust Relationships: Only the specific role in the monitoring account can assume the roles in the target accounts.
    - Session Duration: Role sessions are short-lived and automatically expire.

5.2 Data Storage Security

The metrics data stored in S3 should be secured through:

    - Bucket Encryption: Enable default encryption on the S3 bucket using AWS KMS.
    - Access Policies: Restrict access to the bucket only to necessary principals.
    - Lifecycle Policies: Implement appropriate retention and transition policies.

Example S3 Bucket Policy:

    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::MONITORING-ACCOUNT-ID:role/AnsibleExecutionRole"
        },
        "Action": [
            "s3:PutObject",
            "s3:GetObject",
            "s3:ListBucket"
        ],
        "Resource": [
            "arn:aws:s3:::company-disk-metrics",
            "arn:aws:s3:::company-disk-metrics/*"
        ]
        }
    ]
    }

5.3 Credential Management

The Ansible control node should manage credentials securely:

    - No Hardcoded Credentials: Use AWS profiles and environment variables.
    - Temporary Credentials: Always use temporary credentials obtained via STS.
    - Regular Rotation: Regularly rotate any long-term credentials.

6. Operation and Maintenance

6.1 Scheduling the Monitoring Process

Configure a cron job on the Ansible control node to run the playbook on a regular schedule:

    # Create a cron job to run the monitoring playbook every 6 hours
    echo "0 */6 * * * cd /path/to/disk-monitoring && ansible-playbook multi_account_disk_monitoring.yml >> /var/log/disk-monitoring.log 2>&1" | crontab -
    6.2 Log Management
    Implement appropriate log rotation for the monitoring logs:
    Bash
    Copy code
    cat > /etc/logrotate.d/disk-monitoring << EOF
    /var/log/disk-monitoring.log {
        daily
        rotate 14
        compress
        missingok
        notifempty
        create 0644 root root
    }
    EOF

6.3 Dashboard Access

The HTML report generated by the playbook can be accessed via the S3 bucket's website hosting feature or through a dedicated web server. To enable website hosting on the S3 bucket:

    aws s3 website s3://company-disk-metrics/ --index-document latest/disk_utilization_report.html

6.4 Alerting Integration

For critical disk utilization alerts, consider implementing additional steps in the playbook to send notifications:

    - name: Send alert for critical instances
    mail:
        subject: "CRITICAL: EC2 Disk Utilization Alert"
        body: "The following instances have critical disk utilization:\n{{ critical_volumes | to_nice_yaml }}"
        to: it-ops@company.com
        from: monitoring@company.com
    when: critical_volumes | length > 0

7. Scalability and Future Enhancements

7.1 Adding New AWS Accounts

To add a new AWS account to the monitoring solution:
    - Update Account Variables: Add the new account details to the vars/aws_accounts.yml file.
    - Create IAM Role: Create the DiskMonitoringRole in the new account with the same trust policy and permissions.
    - Verify Access: Test the role assumption from the monitoring account.

No changes to the core playbook structure are required.

7.2 Handling Large Scale

For environments with hundreds or thousands of EC2 instances:
    - Parallel Execution: Modify the playbook to use async tasks for parallel processing.
    - Regional Distribution: Consider deploying multiple Ansible control nodes in different regions.
    - Result Chunking: Implement batch processing of results to avoid memory constraints.

Example of parallel processing:
        - name: Process each AWS account in parallel
        community.general.shell: "ansible-playbook process_single_account.yml -e account_id={{ item.id }} -e account_name={{ item.name }} -e account_region={{ item.region }}"
        async: 3600
        poll: 0
        loop: "{{ aws_accounts }}"
        register: account_jobs

        - name: Wait for all accounts to be processed
        async_status:
            jid: "{{ item.ansible_job_id }}"
        loop: "{{ account_jobs.results }}"
        register: job_result
        until: job_result.finished
        retries: 300
        delay: 10

7.3 Enhanced Metrics and Monitoring

Future enhancements could include:
    - Historical Trending: Implement time-series analysis of disk utilization patterns.
    - Predictive Analytics: Add forecasting to predict when volumes will reach capacity.
    - Integration with AWS CloudWatch: Publish metrics to CloudWatch for unified monitoring.
    - Auto-remediation: Implement automated responses to critical disk space issues, such as log rotation, cleanup, or volume expansion.

8. Troubleshooting Guide

8.1 Common Issues and Solutions

Failed Role Assumption
Symptom: Error message about being unable to assume a role.
Solution:
    - Verify the role exists in the target account
    - Check the trust relationship configuration
    - Confirm the role has the necessary permissions
    - Validate that the role name is consistent across accounts

    # Manually test role assumption
    aws sts assume-role --role-arn "arn:aws:iam::TARGET_ACCOUNT_ID:role/DiskMonitoringRole" --role-session-name "TestSession" --profile monitoring

Missing SSM Agent
Symptom: Unable to execute commands on some instances.
Solution:
    - Verify the SSM agent is installed and running on the instance
    - Check that the instance has the necessary IAM profile
    - Confirm the instance can communicate with the SSM service endpoints

S3 Access Denied
Symptom: Unable to upload results to the S3 bucket.
Solution:
    - Verify the bucket exists and the assumed role has permissions
    - Check bucket policy and ACLs
    - Ensure no VPC endpoint policies are restricting access

8.2 Diagnostic Steps
    A. Enable Verbose Output: Run the playbook with increased verbosity:
        ansible-playbook multi_account_disk_monitoring.yml -vvv
    B. Check AWS API Logs: Enable AWS CloudTrail and check for API errors related to the assumed roles.
    C. Inspect SSM Command History: Review command execution history in the AWS Systems Manager console.
    D. Validate JSON Files: Check the generated JSON files for proper formatting:
        find /tmp/disk-utilization-results -name "*.json" -exec jq . {} \; -exec echo "" \;

9. Conclusion

This documentation provides a comprehensive guide to implementing a multi-account AWS EC2 disk utilization monitoring solution using Ansible. The solution addresses the requirements for centralized access management, data aggregation, and scalability while leveraging existing Ansible infrastructure.

The architecture is designed to be:
    - Secure: Using cross-account roles with least privilege
    - Scalable: Easy addition of new accounts with minimal configuration
    - Maintainable: Modular design with clear separation of concerns
    - Actionable: Providing clear reports and alerts for operation teams

By following this implementation guide, organizations can achieve comprehensive visibility into disk utilization across their entire AWS infrastructure without requiring additional monitoring tools or services.