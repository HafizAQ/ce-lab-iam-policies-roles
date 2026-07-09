# Lab: IAM Policies and Roles - Advanced Identity Management

**Estimated Time:** 90 minutes
**Difficulty:** Intermediate
**Prerequisites:** IAM Users and Groups lab completed, AWS Console access 

## Learning Objectives

By the end of this lab, you will be able to:

- Create and attach custom IAM policies
- Understand policy structure and JSON syntax
- Create and configure IAM roles
- Implement role-based access for AWS services
- Test and validate IAM permissions
- Apply least privilege principle in policy design
- Troubleshoot permission issues

## Scenario

Your company needs more granular access control. The basic user groups are insufficient for the following requirements:

- Developers need read-only S3 access to specific buckets only
- EC2 instances need to access S3 without storing credentials
- Lambda functions need specific permissions to write logs
- You need to enable cross-account access for auditing

You'll implement these requirements using custom IAM policies and roles.

## Lab Overview

You will:

1. Create custom IAM policies with specific permissions
2. Attach policies to users and groups
3. Create service roles for EC2 and Lambda
4. Test permission boundaries
5. Implement least privilege access
6. Create assume-role trust relationships
7. Troubleshoot common IAM issues

## Prerequisites

- AWS Account with IAM administrative access
- Completed IAM Users and Groups lab
- Understanding of JSON basics
- AWS CLI configured (optional but recommended)

---

## Part 1: Understanding IAM Policy Structure

### Step 1: Policy Components

Every IAM policy contains these elements:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescriptiveStatementID",
      "Effect": "Allow",
      "Action": [
        "service:Action"
      ],
      "Resource": "arn:aws:service:region:account:resource"
    }
  ]
}
```

**Key Components:**

- **Version:** Policy language version (always "2012-10-17")
- **Statement:** Array of permission statements
- **Sid:** Optional statement ID (for organization)
- **Effect:** "Allow" or "Deny"
- **Action:** AWS service actions (e.g., "s3:GetObject")
- **Resource:** ARN of resources the actions apply to

**📋 Task 1:** In your solution file, explain each component in your own words.

---

## Part 2: Create Custom IAM Policies

### Step 2: S3 Read-Only Policy for Specific Bucket

Create a policy that allows read access only to a specific S3 bucket.

**Using AWS Console:**

1. Navigate to **IAM** → **Policies**
2. Click **Create policy**
3. Click **JSON** tab
4. Paste this policy (replace `YOUR-BUCKET-NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    },
    {
      "Sid": "ReadObjectsInBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

5. Click **Next: Tags** (skip tags)
6. Click **Next: Review**
7. Configure:
   - **Name:** `S3-ReadOnly-SpecificBucket`
   - **Description:** "Read-only access to specific S3 bucket"
8. Click **Create policy**

**📸 Screenshot 1:** Created custom S3 policy

**Using AWS CLI:**

```bash
# Create policy JSON file first
cat > s3-readonly-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    },
    {
      "Sid": "ReadObjectsInBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
EOF

# Create the policy
aws iam create-policy \
  --policy-name S3-ReadOnly-SpecificBucket \
  --policy-document file://s3-readonly-policy.json
```

### Step 3: EC2 Limited Access Policy

Create a policy for users who can start/stop but not terminate EC2 instances.

**Policy JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadAndStartStop",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyTerminate",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**Create this policy:**

1. IAM → Policies → Create policy
2. Name: `EC2-StartStop-Only`
3. Description: "Start, stop, and reboot EC2 but cannot terminate"

**📸 Screenshot 2:** EC2 custom policy

### Step 4: CloudWatch Logs Write Policy

Policy for applications to write logs only.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogsWrite",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/*"
    }
  ]
}
```

**Create this policy:**

- Name: `CloudWatch-Logs-Write-Only`

**📸 Screenshot 3:** CloudWatch Logs policy

---

## Part 3: Attach Policies to Users

### Step 5: Attach Custom Policy to User

Attach your S3 policy to a test user.

**Steps:**

1. Go to **IAM** → **Users**
2. Select a user (or create `test-developer`)
3. Click **Add permissions** → **Attach policies directly**
4. Search for your custom policy: `S3-ReadOnly-SpecificBucket`
5. Check the box next to it
6. Click **Next** → **Add permissions**

**Using CLI:**

```bash
# First, get the policy ARN
aws iam list-policies --scope Local --query 'Policies[?PolicyName==`S3-ReadOnly-SpecificBucket`].Arn' --output text

# Attach to user (replace ARN and username)
aws iam attach-user-policy \
  --user-name test-developer \
  --policy-arn arn:aws:iam::YOUR-ACCOUNT-ID:policy/S3-ReadOnly-SpecificBucket
```

**📸 Screenshot 4:** Policy attached to user

---

## Part 4: Create and Configure IAM Roles

### Step 6: Create EC2 Role for S3 Access

EC2 instances need to access S3 without storing credentials. Use IAM roles!

**Create role:**

1. Go to **IAM** → **Roles** → **Create role**
2. Select **AWS service** → **EC2**
3. Click **Next**
4. Search and select: `AmazonS3ReadOnlyAccess`
5. Click **Next**
6. Configure:
   - **Role name:** `EC2-S3-ReadOnly-Role`
   - **Description:** "Allows EC2 instances read access to S3"
7. Click **Create role**

**📸 Screenshot 5:** EC2 service role created

**Using CLI:**

```bash
# Create trust policy for EC2
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name EC2-S3-ReadOnly-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach S3 read policy
aws iam attach-role-policy \
  --role-name EC2-S3-ReadOnly-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 7: Create Lambda Execution Role

Lambda functions need specific permissions to run.

**Create role:**

1. IAM → Roles → Create role
2. Select **AWS service** → **Lambda**
3. Attach policies:
   - `AWSLambdaBasicExecutionRole` (for CloudWatch Logs)
   - Your custom `CloudWatch-Logs-Write-Only` (optional)
4. Name: `Lambda-Basic-Execution-Role`
5. Create role

**📸 Screenshot 6:** Lambda execution role

### Step 8: Create Cross-Account Access Role

Allow another AWS account to access your resources.

**Trust policy for cross-account:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::EXTERNAL-ACCOUNT-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-123"
        }
      }
    }
  ]
}
```

**Create role (Console):**

1. IAM → Roles → Create role
2. Select **Another AWS account**
3. Enter external account ID (use your own account ID for testing)
4. Check **Require external ID** → Enter: `unique-external-id-123`
5. Attach policy: `ReadOnlyAccess`
6. Name: `CrossAccount-ReadOnly-Role`
7. Create role

**📸 Screenshot 7:** Cross-account role

---

## Part 5: Test and Validate Permissions

### Step 9: Policy Simulator

Test policies before applying them.

**Steps:**

1. Go to **IAM** → **Policies**
2. Select your `S3-ReadOnly-SpecificBucket` policy
3. Click **Actions** → **Simulate**
4. Or navigate directly to: https://policysim.aws.amazon.com/
5. Select **User, Group, or Role**
6. Choose a test user
7. Select **Service:** S3
8. Select **Actions:** GetObject, PutObject, DeleteObject
9. Click **Run Simulation**

**Expected Results:**

- **GetObject:** ✅ Allowed
- **PutObject:** ❌ Denied
- **DeleteObject:** ❌ Denied

**📸 Screenshot 8:** Policy Simulator results

### Step 10: Test with AWS CLI

Verify permissions with actual AWS CLI commands.

**Test S3 read access:**

```bash
# Assuming you're using test-developer credentials
# List bucket (should work)
aws s3 ls s3://YOUR-BUCKET-NAME/

# Try to upload (should fail)
echo "test" > test.txt
aws s3 cp test.txt s3://YOUR-BUCKET-NAME/
# Expected: Access Denied

# Try to read (should work if objects exist)
aws s3 cp s3://YOUR-BUCKET-NAME/somefile.txt ./
```

**📋 Document test results in solution file**

---

## Part 6: Implement Least Privilege

### Step 11: Refine Policy Permissions

Review and tighten your policies.

**Example: Limit S3 access by IP address**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": "*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

**Example: Limit actions to specific time window**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {"aws:CurrentTime": "2026-01-14T00:00:00Z"},
        "DateLessThan": {"aws:CurrentTime": "2026-01-15T00:00:00Z"}
      }
    }
  ]
}
```

**📋 Task:** Create a policy with conditions for your use case

---

## Part 7: Troubleshooting

### Step 12: Common IAM Issues

**Issue 1: Access Denied**

```bash
# Check effective permissions
aws iam get-user-policy --user-name USERNAME --policy-name POLICY_NAME

# Simulate action
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT:user/USERNAME \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::bucket-name/object-key
```

**Issue 2: Role Not Assumable**

```bash
# Check trust relationship
aws iam get-role --role-name ROLE_NAME --query 'Role.AssumeRolePolicyDocument'

# Try assuming role
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT:role/ROLE_NAME \
  --role-session-name test-session
```

**📸 Screenshot 9:** Troubleshooting commands output

---

## Verification Checklist

- [ ] Created 3 custom IAM policies
- [ ] Attached policies to users
- [ ] Created EC2 service role
- [ ] Created Lambda execution role
- [ ] Created cross-account access role
- [ ] Tested policies with Policy Simulator
- [ ] Validated permissions with AWS CLI
- [ ] Implemented least privilege principles
- [ ] Troubleshot permission issues

---

## Cleanup

**Remove test resources:**

```bash
# Detach policies from users
aws iam detach-user-policy --user-name test-developer --policy-arn POLICY_ARN

# Delete custom policies
aws iam delete-policy --policy-arn POLICY_ARN

# Delete roles (first detach all policies)
aws iam detach-role-policy --role-name EC2-S3-ReadOnly-Role --policy-arn POLICY_ARN
aws iam delete-role --role-name EC2-S3-ReadOnly-Role
```

---

## Key Takeaways

✅ **IAM Policies** define granular permissions using JSON
✅ **Service Roles** enable AWS services to access resources securely
✅ **Least Privilege** minimizes security risks
✅ **Policy Simulator** helps test before deployment
✅ **Trust Relationships** control who can assume roles
✅ **Conditions** add extra security layers

---

## Next Steps

- Learn about IAM Permission Boundaries
- Explore AWS Organizations and SCPs
- Study IAM Access Analyzer
- Implement AWS SSO for enterprise access
- Practice writing complex conditional policies

**Congratulations!** You've mastered IAM Policies and Roles!
