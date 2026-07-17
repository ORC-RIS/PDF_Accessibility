# Complete Deployment Guide: Using Existing VPC

## Overview

This guide walks you through deploying the PDF-to-PDF Accessibility Remediation solution using an existing VPC, explaining what gets deployed and how to use it.

---

## What Gets Deployed

When you deploy with an existing VPC, the following AWS resources are created:

### 1. Storage Layer
- **S3 Bucket** (`pdfaccessibility-*`)
  - Encrypted with S3-managed keys
  - Versioning enabled
  - Folders: `pdf/`, `temp/`, `result/`

### 2. Container Infrastructure (Uses Your VPC)
- **ECS Cluster** - Runs in your specified private subnets
- **2 ECR Repositories** - Stores Docker images:
  - Adobe Autotag container
  - Alt-text generator container
- **2 ECS Fargate Task Definitions**:
  - Adobe Autotag Task (1024 MB, 256 CPU)
  - Alt-text Generator Task (1024 MB, 256 CPU)

### 3. Serverless Functions
- **PDF Splitter Lambda** (Python 3.12, 1024 MB)
  - Triggered by S3 uploads to `pdf/` folder
  - Splits PDFs into 200-page chunks
- **PDF Merger Lambda** (Java 21, 1024 MB)
  - Merges processed chunks back together
- **Title Generator Lambda** (Python 3.12, 1024 MB)
  - Generates accessible titles using Bedrock
- **Pre-Remediation Checker Lambda** (Python 3.12, 512 MB)
  - Audits original PDF accessibility
- **Post-Remediation Checker Lambda** (Python 3.12, 512 MB)
  - Audits final PDF accessibility

### 4. Orchestration
- **Step Functions State Machine**
  - Orchestrates the entire workflow
  - Runs pre/post audits in parallel with remediation
  - Max execution time: 150 minutes
  - Handles up to 100 concurrent chunk processing

### 5. Monitoring
- **CloudWatch Dashboard** (timestamped name)
  - File status tracking
  - Lambda logs
  - ECS task logs
  - Step Function execution logs
- **CloudWatch Log Groups** (1-month retention):
  - `/aws/lambda/*` - All Lambda functions
  - `/ecs/pdf-remediation/*` - ECS tasks
  - `/aws/states/*` - Step Functions

### 6. Security
- **IAM Roles** (4 roles):
  - ECS Task Execution Role
  - ECS Task Role (with Bedrock, S3, Comprehend, Secrets Manager permissions)
  - Lambda Execution Roles (per function)
- **Secrets Manager Secret**:
  - `/myapp/client_credentials` - Adobe API credentials

### 7. What Does NOT Get Created (Using Existing VPC)
- ❌ No new VPC
- ❌ No NAT Gateway (uses yours)
- ❌ No VPC Endpoints (uses yours if available)
- ❌ No Internet Gateway (uses yours)

---

## Deployment Process

### Step 1: Prerequisites

Ensure you have:
1. **AWS Account** with appropriate permissions
2. **Existing VPC** with:
   - At least 2 private subnets in different AZs
   - NAT Gateway OR VPC Endpoints (ECR, S3, Bedrock)
   - Internet connectivity for private subnets
3. **Adobe PDF Services API credentials**:
   - Client ID
   - Client Secret
   - Get from: https://acrobatservices.adobe.com/

### Step 2: Find Your Private Subnet IDs

**Option A: AWS Console**
```
1. Go to VPC Console → Subnets
2. Filter by your VPC ID
3. Look for subnets with route to NAT Gateway
4. Note down at least 2 subnet IDs in different AZs
```

**Option B: AWS CLI**
```bash
# List all subnets in your VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-xxxxx" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' \
  --output table

# Find subnets with NAT Gateway routes
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-xxxxx" \
  --query 'RouteTables[?Routes[?NatGatewayId!=`null`]].Associations[].SubnetId' \
  --output text
```

### Step 3: Run Deployment

```bash
# Clone the repository
git clone https://github.com/ASUCICREPO/PDF_Accessibility.git
cd PDF_Accessibility

# Make deployment script executable
chmod +x deploy.sh

# Run deployment
./deploy.sh
```

### Step 4: Interactive Prompts

You'll be prompted for:

```
1. Solution Selection
   → Choose: 1 (PDF-to-PDF Remediation)

2. Adobe API Credentials
   → Enter Client ID: your-client-id
   → Enter Client Secret: your-client-secret

3. VPC Configuration
   → Use existing VPC? y
   → Enter VPC ID: vpc-abc123
   
4. Subnet Selection (NEW!)
   → [Script displays available subnets]
   → Enter private subnet IDs: subnet-abc123,subnet-def456
   
5. Deployment Progress
   → [Script monitors CodeBuild deployment]
   → Typically takes 3-5 minutes
```

### Step 5: Deployment Complete

You'll see:
```
🎊 Deployment Complete!
📊 Deployment Summary:
   ✅ PDF-to-PDF Remediation: pdfaccessibility-123456789012-us-east-1
   
🔍 Monitor builds in AWS Console:
   https://console.aws.amazon.com/codesuite/codebuild/projects
```

---

## How to Use the Solution

### Processing a PDF

**Step 1: Upload PDF to S3**

```bash
# Using AWS CLI
aws s3 cp your-document.pdf s3://pdfaccessibility-*/pdf/

# Or use AWS Console:
# 1. Go to S3 Console
# 2. Find bucket: pdfaccessibility-*
# 3. Upload to pdf/ folder
```

**Step 2: Automatic Processing Begins**

The system automatically:
1. **Splits PDF** (Lambda) → Chunks uploaded to `temp/`
2. **Parallel Processing** (Step Functions):
   - **Branch A: Remediation**
     - Adobe Autotag (ECS) → Adds accessibility tags
     - Alt-text Generation (ECS) → Generates image descriptions
     - PDF Merger (Lambda) → Combines chunks
     - Title Generator (Lambda) → Creates accessible title
     - Post-Audit (Lambda) → Validates improvements
   - **Branch B: Pre-Audit** (Lambda) → Baseline accessibility report
3. **Final Output** → Uploaded to `result/`

**Step 3: Monitor Progress**

**Option A: CloudWatch Dashboard**
```
1. Go to CloudWatch Console → Dashboards
2. Find: PDF_Processing_Dashboard-[timestamp]
3. View:
   - File status
   - Processing logs
   - Error messages
```

**Option B: Step Functions Console**
```
1. Go to Step Functions Console
2. Find: PdfAccessibilityRemediationWorkflow
3. View execution graph and status
```

**Option C: S3 Bucket**
```
Check folder structure:
pdf/                    ← Your uploaded file
temp/
  └── [filename]/
      ├── chunks/       ← Split PDF pages
      ├── output_autotag/ ← Adobe processing
      └── images/       ← Extracted images
result/
  └── COMPLIANT_[filename].pdf ← Final output!
```

**Step 4: Download Results**

```bash
# Using AWS CLI
aws s3 cp s3://pdfaccessibility-*/result/COMPLIANT_your-document.pdf ./

# Or use AWS Console:
# 1. Navigate to result/ folder
# 2. Download COMPLIANT_* file
```

---

## Architecture Flow with Existing VPC

```
┌─────────────────────────────────────────────────────────────┐
│                     Your Existing VPC                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Private Subnet 1 (us-east-1a)                       │   │
│  │  ┌────────────────┐  ┌────────────────┐             │   │
│  │  │ ECS Task:      │  │ ECS Task:      │             │   │
│  │  │ Adobe Autotag  │  │ Alt-text Gen   │             │   │
│  │  └────────────────┘  └────────────────┘             │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Private Subnet 2 (us-east-1b)                       │   │
│  │  ┌────────────────┐  ┌────────────────┐             │   │
│  │  │ ECS Task:      │  │ ECS Task:      │             │   │
│  │  │ Adobe Autotag  │  │ Alt-text Gen   │             │   │
│  │  └────────────────┘  └────────────────┘             │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Internet Access via:                                        │
│  • NAT Gateway (in your public subnet)                       │
│  OR                                                           │
│  • VPC Endpoints (ECR, S3, Bedrock)                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    AWS Services (Outside VPC)                │
│                                                              │
│  S3 Bucket          Step Functions       Lambda Functions   │
│  ┌──────────┐      ┌──────────────┐    ┌────────────────┐  │
│  │ pdf/     │──┬──▶│ Orchestrator │◀──▶│ PDF Splitter   │  │
│  │ temp/    │  │   └──────────────┘    │ PDF Merger     │  │
│  │ result/  │  │          │            │ Title Gen      │  │
│  └──────────┘  │          ▼            │ Pre-Audit      │  │
│                │   ┌──────────────┐    │ Post-Audit     │  │
│                └──▶│ ECS Tasks    │    └────────────────┘  │
│                    │ (in your VPC)│                         │
│                    └──────────────┘                         │
│                                                              │
│  Bedrock AI         Secrets Manager    CloudWatch           │
│  ┌──────────┐      ┌──────────────┐    ┌────────────────┐  │
│  │ Nova Pro │      │ Adobe Creds  │    │ Dashboard      │  │
│  │ Nova Lite│      └──────────────┘    │ Logs           │  │
│  └──────────┘                           └────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Processing Details

### What Each Component Does

**1. PDF Splitter Lambda**
- Triggered when you upload to `pdf/` folder
- Splits PDF into 200-page chunks
- Uploads chunks to `temp/[filename]/`
- Starts Step Functions execution

**2. Adobe Autotag ECS Task**
- Downloads chunk from S3
- Calls Adobe PDF Services API:
  - Auto-tags for accessibility
  - Extracts text, tables, figures
  - Generates TOC
  - Detects language
- Uploads tagged PDF to `temp/[filename]/output_autotag/`
- Creates SQLite DB with image metadata

**3. Alt-text Generator ECS Task**
- Downloads tagged PDF and images
- Reads SQLite DB for image context
- Calls AWS Bedrock (Nova Pro):
  - Generates WCAG 2.1-compliant alt text
  - Considers page context
  - Handles equations, charts, diagrams
- Updates PDF with alt text
- Uploads to `temp/[filename]/FINAL_*`

**4. PDF Merger Lambda**
- Collects all processed chunks
- Merges into single PDF
- Preserves accessibility tags
- Uploads merged PDF

**5. Title Generator Lambda**
- Analyzes PDF content
- Calls AWS Bedrock (Nova Lite)
- Generates descriptive title
- Updates PDF metadata

**6. Pre/Post Audit Lambdas**
- Scan PDF for accessibility issues
- Generate compliance reports
- Compare before/after improvements
- Upload reports to S3

---

## Monitoring and Troubleshooting

### Check Processing Status

**CloudWatch Dashboard Widgets:**

1. **File Status** - Overall progress
   ```
   Processing → Autotagging → Alt-text → Merging → Complete
   ```

2. **Split PDF Lambda Logs** - Initial processing
   ```
   Look for: "Uploaded chunk_X to S3"
   ```

3. **Step Function Execution Logs** - Workflow status
   ```
   Look for: "Execution started" / "Execution succeeded"
   ```

4. **Adobe Autotag Logs** - Tagging progress
   ```
   Look for: "Adobe Autotag completed successfully"
   ```

5. **Alt Text Generation Logs** - AI processing
   ```
   Look for: "Alt text generation complete: X succeeded"
   ```

6. **PDF Merger Logs** - Final assembly
   ```
   Look for: "PDF merge completed"
   ```

### Common Issues

**Issue 1: ECS Tasks Fail to Start**
```
Symptom: "Task failed to start" in Step Functions
Cause: Subnets don't have internet access
Solution:
  1. Verify NAT Gateway is attached to route table
  2. Check route: 0.0.0.0/0 → nat-xxxxx
  3. Or add VPC Endpoints for ECR, S3, Bedrock
```

**Issue 2: Adobe API Errors**
```
Symptom: "Adobe Autotag API failed" in logs
Cause: Invalid credentials or quota exceeded
Solution:
  1. Verify credentials in Secrets Manager
  2. Check Adobe API quota/limits
  3. Ensure credentials are active
```

**Issue 3: Bedrock Throttling**
```
Symptom: "All Bedrock requests failed" in alt-text logs
Cause: Too many concurrent requests
Solution:
  1. Request quota increase for Bedrock
  2. Reduce concurrent chunk processing
  3. Wait and retry
```

**Issue 4: PDF Not in Result Folder**
```
Symptom: Processing complete but no output
Cause: Check Step Functions for failures
Solution:
  1. Go to Step Functions Console
  2. Find failed step
  3. Check CloudWatch logs for that step
  4. Look for error messages
```

### Cost Estimation

**With Existing VPC (No NAT Gateway Cost):**

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| S3 Storage | 100 GB | ~$2.30 |
| Lambda Invocations | 1,000 PDFs | ~$5-10 |
| ECS Fargate | 1,000 PDFs × 2 tasks | ~$20-30 |
| Step Functions | 1,000 executions | ~$0.25 |
| Bedrock API | 1,000 PDFs × 50 images | ~$50-100 |
| CloudWatch Logs | 10 GB | ~$5 |
| **Total** | | **~$82-147/month** |

**Note:** Bedrock costs vary based on:
- Number of images per PDF
- Model used (Nova Pro vs Nova Lite)
- Token usage

---

## Advanced Configuration

### Adjust Chunk Size

Edit `lambda/pdf-splitter-lambda/main.py`:
```python
# Change from 200 to desired page count
chunks = split_pdf_into_pages(pdf_file_content, pdf_file_key, s3, bucket_name, 100)
```

### Change AI Models

Edit `alt-text-generator-container/alt_text_generator.js`:
```javascript
// Line 30-33
const MODEL_ID_ALT_TEXT = "us.amazon.nova-pro-v1:0";  // For images
const MODEL_ID_LINK_ALT_TEXT = "us.amazon.nova-lite-v1:0";  // For links
```

### Adjust Concurrency

Edit `app.py`:
```python
# Line 254 - Change max_concurrency
pdf_chunks_map_state = sfn.Map(self, "ProcessPdfChunksInParallel",
    max_concurrency=50,  # Reduce from 100 to 50
    ...
)
```

---

## Cleanup

To remove all deployed resources:

```bash
# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name PDFAccessibility

# Delete S3 bucket (after emptying it)
aws s3 rm s3://pdfaccessibility-* --recursive
aws s3 rb s3://pdfaccessibility-*

# Delete ECR repositories
aws ecr delete-repository --repository-name cdk-* --force

# Delete Secrets Manager secret
aws secretsmanager delete-secret --secret-id /myapp/client_credentials --force-delete-without-recovery
```

---

## Summary

✅ **Deployed Resources:**
- 1 S3 Bucket
- 1 ECS Cluster (in your VPC)
- 2 ECR Repositories
- 2 ECS Task Definitions
- 5 Lambda Functions
- 1 Step Functions State Machine
- 1 CloudWatch Dashboard
- Multiple IAM Roles
- 1 Secrets Manager Secret

✅ **Uses Your Existing:**
- VPC
- Private Subnets
- NAT Gateway (or VPC Endpoints)
- Internet connectivity

✅ **Processing Capability:**
- Handles PDFs of any size
- Processes up to 100 chunks in parallel
- Generates WCAG 2.1-compliant alt text
- Provides before/after accessibility reports

✅ **Cost Savings:**
- No new NAT Gateway (~$32-40/month saved)
- No new VPC infrastructure
- Pay only for processing resources

The solution is now ready to process PDFs and make them accessible!
