# SageMaker Error Logs Model Pipeline

This project implements an automated CI/CD pipeline that trains and deploys a machine learning model on AWS SageMaker whenever the `error_logs.csv` file is updated. The model analyzes error messages and predicts recommended actions.

## 🚀 Features

- **Automated Deployment**: GitHub Actions workflow triggers on changes to `error_logs.csv`
- **Infrastructure as Code**: Terraform manages all AWS resources
- **Remote State Management**: Terraform state stored securely in S3
- **ML Model Training**: Scikit-learn model trained on error logs data
- **SageMaker Endpoint**: Model deployed as a real-time inference endpoint
- **Automated Testing**: Endpoint tested automatically after deployment

## 📋 Prerequisites

- AWS Account with appropriate permissions
- GitHub repository with Actions enabled
- AWS CLI configured locally (for initial setup)
- Terraform installed locally (optional, for local development)

## 🛠️ Initial Setup

### 1. Create Terraform Backend S3 Bucket

Create an S3 bucket named `terraform-remote-sagemaker-ankit` for storing Terraform state:

```bash
# Create the bucket
aws s3api create-bucket --bucket terraform-remote-sagemaker-ankit --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning --bucket terraform-remote-sagemaker-ankit \
    --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption --bucket terraform-remote-sagemaker-ankit \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            }
        }]
    }'

# Block public access
aws s3api put-public-access-block --bucket terraform-remote-sagemaker-ankit \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 2. Configure GitHub Secrets

Add the following secrets to your GitHub repository:

1. Go to your repo on GitHub
2. Navigate to **Settings → Secrets and variables → Actions**
3. Click **"New repository secret"** and add each secret below:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | Your AWS IAM secret key |

#### Required IAM Permissions

The IAM user behind these keys needs the following permissions:

- **SageMaker** — train models, deploy/delete endpoints, manage notebook instances
- **S3** — read/write to the data bucket and the Terraform state bucket (`terraform-remote-sagemaker-ankit`)
- **VPC / EC2** — create VPC, subnets, internet gateway, route tables, security groups
- **IAM** — create roles and attach policies
- **CloudWatch Logs** — create log groups and write logs

> **Note:** The Terraform state bucket (`terraform-remote-sagemaker-ankit`) must already exist in your AWS account before the pipeline runs. See [Step 1](#1-create-terraform-backend-s3-bucket) above.

### 3. Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── sagemaker-deploy.yml    # Deployment workflow
│       └── sagemaker-destroy.yml   # Cleanup workflow
├── terraform/
│   ├── backend.tf                  # Terraform backend configuration
│   ├── main.tf                     # AWS infrastructure definition
│   └── outputs.tf                  # Terraform outputs
├── error_logs.csv                  # Training data (triggers pipeline)
├── README.md                       # This file
└── train_and_deploy.py             # SageMaker training/deployment script
```

## 🔄 How It Works

1. **Trigger**: When you update and push changes to `error_logs.csv`, the GitHub Actions workflow starts
2. **Infrastructure**: Terraform provisions AWS resources (VPC, S3, IAM roles, SageMaker)
3. **Data Upload**: The updated CSV file is uploaded to S3
4. **Model Training**: SageMaker trains a scikit-learn model on the error logs
5. **Deployment**: The trained model is deployed to a SageMaker endpoint
6. **Output**: The endpoint name is displayed in the GitHub Actions summary

## 📊 Using the Deployed Model

Once deployed, you can use the SageMaker endpoint to make predictions:

```python
import boto3
import json

# Initialize SageMaker runtime client
runtime = boto3.client('sagemaker-runtime', region_name='us-east-1')

# Your endpoint name (from GitHub Actions output)
endpoint_name = 'logs-error-endpoint-YYYYMMDD-HHMMSS'

# Make a prediction
response = runtime.invoke_endpoint(
    EndpointName=endpoint_name,
    ContentType='application/json',
    Body=json.dumps(["Error 404: Page not found"])
)

result = json.loads(response['Body'].read().decode())
print(f"Recommended action: {result[0]}")
```

## 🔧 Local Development

To run Terraform locally:

```bash
cd terraform
terraform init -backend-config="bucket=terraform-remote-sagemaker-ankit" \
               -backend-config="key=sagemaker-logs-poc/terraform.tfstate" \
               -backend-config="region=us-east-1"
terraform plan
terraform apply
```

To test the training script locally:

```bash
export SAGEMAKER_EXECUTION_ROLE="arn:aws:iam::YOUR_ACCOUNT:role/sagemaker-logs-poc-execution-role"
export S3_BUCKET="your-s3-bucket-name"
python train_and_deploy.py
```

## 📝 Updating the Model

To retrain and redeploy the model:

1. Update the `error_logs.csv` file with new training data
2. Commit and push to the main branch:
   ```bash
   git add error_logs.csv
   git commit -m "Update training data"
   git push origin main
   ```
3. Monitor the GitHub Actions workflow
4. Find the new endpoint name in the workflow summary

## 🔒 Security Considerations

- AWS credentials are stored as GitHub secrets
- Terraform state is encrypted in S3
- S3 buckets have public access blocked
- IAM roles follow least privilege principle
- VPC and security groups restrict network access

## 🧪 Testing and Cleanup

To clean up resources after testing:

1. Go to GitHub Actions → "Destroy SageMaker Infrastructure"
2. Click "Run workflow"
3. Type "destroy" in the confirmation field
4. Check "Also destroy SageMaker endpoints" (recommended)
5. Click "Run workflow"

This will:
- Delete all SageMaker endpoints containing "logs-error-endpoint"
- Destroy all Terraform-managed infrastructure (VPC, S3, IAM roles, etc.)
- Provide a summary when complete

## 💰 Cost Optimization

- SageMaker notebook instance: ml.t2.medium (~$0.0464/hour)
- Training instance: ml.m5.xlarge (~$0.23/hour, only during training)
- Inference endpoint: ml.m5.large (~$0.115/hour, continuous)
- S3 storage: Minimal cost for small datasets

**Remember to delete resources when not in use:**

```bash
cd terraform
terraform destroy
```

## 🐛 Troubleshooting

### GitHub Actions fails at Terraform init
- Verify the S3 bucket `terraform-remote-sagemaker-ankit` exists
- Check AWS credentials have necessary permissions

### Model training fails
- Check CloudWatch logs in AWS console
- Verify `error_logs.csv` format is correct
- Ensure S3 bucket permissions are set properly

### Endpoint deployment fails
- Check AWS service quotas for SageMaker
- Verify IAM role has necessary permissions
- Review CloudWatch logs for detailed errors

## 📚 Additional Resources

- [AWS SageMaker Documentation](https://docs.aws.amazon.com/sagemaker/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

## 📄 License

This project is provided as-is for demonstration purposes.