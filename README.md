# Task 3: Automated AWS Infrastructure Provisioning Using Terraform and GitHub Actions
## 1. Project Overview
This project demonstrates how to provision AWS infrastructure using Terraform and automate infrastructure validation, planning, deployment, and manual cleanup using GitHub Actions.

AWS Region: ap-south-1 (Mumbai)
## 2. Project Objectives

- Provision a custom VPC with a public subnet and internet connectivity.
- Configure a security group for the required network traffic.
- Deploy an EC2 instance inside the VPC.
- Allow EC2 to access S3 using an IAM role instead of permanent AWS credentials.
- Automate Terraform formatting, validation, planning, and application using GitHub Actions.
- Manage infrastructure destruction through a separate, manually triggered workflow.

## 3. AWS Architecture and Resources

| Resource | Configuration |
|---|---|
| VPC | CIDR `10.0.0.0/16`, tag `task3-vpc` |
| Public Subnet | CIDR `10.0.1.0/24`, public IPv4 assignment enabled |
| Internet Gateway | Attached to the custom VPC |
| Route Table | Default route `0.0.0.0/0` through the Internet Gateway |
| Security Group | Allows inbound HTTP on TCP port 80 and outbound traffic |
| EC2 Instance | Instance type `t3.micro`, tag `task3-projectttt` |
| IAM Role | Existing role `task3-ec2-s3-role`, referenced by Terraform |
| IAM Instance Profile | `task3-ec2-profile`, managed by Terraform |
| S3 Bucket | Existing bucket `terraform-state-plann`, referenced as a data source and used for Terraform state |

**Note:** Terraform does not create the S3 bucket or IAM role. Both already exist and are referenced by the configuration.

## 4. Terraform Remote State and Locking

Terraform stores its state remotely using the S3 backend.

- **Bucket:** `terraform-state-plann`
- **State Key:** `task3/terraform.tfstate`
- **Region:** `ap-south-1`
- **Encryption:** Enabled in the backend configuration
- **State Locking:** Enabled using `use_lockfile = true`

**Important:**
- Preserve the S3 state bucket to avoid losing Terraform state.
- A one-time cleanup step was previously used to remove the old `aws_s3_bucket.app` resource from Terraform state without deleting the actual bucket.
- After confirming that `aws_s3_bucket.app` no longer exists in the Terraform state, remove the temporary cleanup step from the normal workflow.
## 5. EC2-to-S3 Access
The EC2 instance receives AWS permissions through the task3-ec2-s3-role IAM role attached via the task3-ec2-profile instance profile. Permanent AWS access keys should not be stored on the EC2 instance.

The role has been configured manually with these permissions for the existing bucket:

s3:ListBucket on arn:aws:s3:::terraform-state-plann

s3:GetObject on arn:aws:s3:::terraform-state-plann/*

s3:PutObject on arn:aws:s3:::terraform-state-plann/*

Example commands
Run these from the EC2 instance after connecting to it and confirming the AWS CLI is installed:

# Confirm the AWS identity provided to the instance
aws sts get-caller-identity

# List bucket objects
aws s3 ls s3://terraform-state-plann/

# Upload a file
aws s3 cp ./example.txt s3://terraform-state-plann/example.txt

# Download a file
aws s3 cp s3://terraform-state-plann/example.txt ./downloaded-example.txt
These operations depend on the IAM role's effective permissions, bucket policies, and network access. They do not automatically upload Terraform plans; an explicit upload command is needed for that.

Security note: This lab uses the same bucket for Terraform state and EC2 file transfers. Because Terraform state can contain sensitive infrastructure information, a production setup should use a separate application bucket and narrowly scoped permissions.

## 6. GitHub Repository Secrets
Configure these GitHub repository secrets:

AWS_ACCESS_KEY_ID

AWS_SECRET_ACCESS_KEY

AWS_REGION — set to ap-south-1

Never commit AWS credentials to the repository. For production, use least-privilege permissions and consider GitHub Actions OIDC instead of long-lived access keys.
## 7. GitHub Actions Workflows

### 7.1 Normal Workflow: `terraform.yml`

The workflow triggers on:
- Pushes to the `terraform-changes` and `main` branches.
- Pull requests targeting `main`.

The Plan job performs:
- `terraform init`
- `terraform fmt -check -recursive`
- `terraform validate`
- `terraform plan`

The Apply job runs only after Plan succeeds and only on a push to `main`. It uses the GitHub Environment named `approval`.

**Approval Requirements:**
- Configure required reviewers under **Repository Settings → Environments → approval**.
- Review the Terraform plan for unexpected resource destruction or replacement before approving deployment.

### 7.2 Manual Destroy Workflow: `terraform-destroy.yml`

The Destroy workflow uses `workflow_dispatch`, so pull requests and merges do not trigger it automatically.

**Steps to run Destroy:**
1. Open the repository's **Actions** tab.
2. Select **Terraform Destroy Demo**.
3. Click **Run workflow**.
4. Enter `DESTROY` in the confirmation field.
5. Approve the environment if required.

The workflow generates a destroy plan and applies the saved plan. Environment approval happens before the job starts, not after the plan is generated. Review the destroy plan output carefully.

Resources managed by Terraform may be deleted, including:
- EC2 instance
- VPC and subnet
- Internet Gateway
- Route table and route table association
- Security group
- IAM instance profile

The exact resources depend on the current Terraform state and AWS dependencies.

The existing S3 state bucket (`terraform-state-plann`) and manually created IAM role (`task3-ec2-s3-role`) are referenced rather than managed as Terraform resources. They should remain. Verify that neither appears in the destroy plan before proceeding.

## 8. Deployment Procedure

1. Commit Terraform changes to the `terraform-changes` branch.
2. Open a pull request targeting `main`.
3. Verify that formatting, validation, and Plan succeed.
4. Review the plan for unexpected changes.
5. Merge the pull request when ready.
6. Approve the deployment through the `approval` environment if required reviewers are configured.
7. Verify the EC2 instance and networking resources in AWS.
8. Test EC2-to-S3 access using the example commands documented above.

## 9. Destroy Demonstration Procedure

1. Confirm that the infrastructure is no longer needed for the demonstration.
2. Open **Actions → Terraform Destroy Demo → Run workflow**.
3. Enter `DESTROY` and run the workflow.
4. Review the destroy plan and confirm that the S3 state bucket and manually created IAM role are not listed for deletion.
5. Allow the destroy step to proceed only if the plan is correct.
6. Verify that the EC2 instance and Terraform-managed networking resources have been deleted.
7. Confirm that `terraform-state-plann` and `task3-ec2-s3-role` remain.

**Important:** Do not run Destroy if you still need the infrastructure. To recreate it, run the normal deployment workflow again.

## 10. Security Considerations

- Inbound HTTP access is allowed from `0.0.0.0/0` on port 80.
- Inbound SSH is not configured. Prefer AWS Systems Manager Session Manager when properly configured.
- The EC2 instance has a public IP address and internet connectivity through an Internet Gateway.
- Allowing the EC2 IAM role to access the Terraform state bucket is a security trade-off in this lab.
- The Terraform configuration does not install or start a web server, so opening the EC2 public IP in a browser may not display a website.
- Always review the Terraform plan carefully before running Apply or Destroy.

## 11. Cost Considerations
Actual costs depend on the AWS account, region, runtime, storage, public IPv4 usage, and data transfer. A t3.micro may qualify for applicable free-tier benefits depending on account eligibility, but it should not be assumed to be free. EC2 runtime, public IPv4 usage, data transfer, and S3 storage or requests may incur charges. Check current AWS pricing and the account billing dashboard before and after the lab.

## 12. Completion Checklist

- [x] AWS infrastructure configured using Terraform.
- [x] Terraform CI/CD pipeline implemented using GitHub Actions.
- [x] Remote state management configured with S3.
- [x] IAM role-based access configured for EC2 and S3.
- [x] Deployment approval and manual Destroy workflows implemented.
- [x] AWS resource provisioning and destruction verified.
- [x] Security practices and cost considerations documented.

## 13. Key Learnings

- Learned to provision and manage AWS infrastructure using Terraform.
- Understood Terraform state management using an S3 backend and state locking.
- Automated Terraform formatting, validation, planning, and deployment using GitHub Actions.
- Learned to configure IAM roles and instance profiles for secure EC2 access to S3.
- Understood the importance of reviewing Terraform plans, approval gates, and manual destruction of infrastructure.
