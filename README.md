# S3 to SFTP Transfer

AWS Lambda-based solution that automatically transfers files uploaded to S3 to an external SFTP server.

## Architecture

- **S3 Bucket**: Receives uploaded files and triggers Lambda via event notification
- **Lambda Function**: Downloads file from S3 and uploads to SFTP server (runs in VPC private subnets)
- **VPC**: Provides network isolation with 2 public and 2 private subnets across 2 AZs
- **NAT Gateways**: Enable Lambda in private subnets to access S3 and Secrets Manager
- **Secrets Manager**: Stores SFTP credentials securely
- **Paramiko Layer**: Python library for SFTP connectivity

## Prerequisites

- AWS CLI configured with appropriate credentials
- Python 3.9+ (for building the Paramiko layer)
- Docker (optional, for building layer in Amazon Linux environment)
- An external SFTP server (hostname/IP, port, username, password)

## Deployment Steps

### 1. Build and Publish Paramiko Lambda Layer

The `paramiko-layer.zip` is included in this repo. Publish it as a Lambda layer:

```powershell
aws lambda publish-layer-version `
  --layer-name paramiko-sftp-layer `
  --description "Paramiko and dependencies for SFTP" `
  --zip-file fileb://cloudformation/paramiko-layer.zip `
  --compatible-runtimes python3.9 python3.10 python3.11 python3.12
```

Note the `LayerVersionArn` from the output (e.g., `arn:aws:lambda:us-east-1:123456789012:layer:paramiko-sftp-layer:1`).

### 2. Deploy CloudFormation Stack

```powershell
aws cloudformation deploy `
  --template-file cloudformation/sftpS3trigger.yml `
  --stack-name sftp-transfer-stack `
  --parameter-overrides `
    ParamikoLayerArn=arn:aws:lambda:REGION:ACCOUNT:layer:paramiko-sftp-layer:1 `
    S3BucketName=my-unique-bucket-name `
  --capabilities CAPABILITY_NAMED_IAM
```

**Note**: Replace `my-unique-bucket-name` with a globally unique S3 bucket name.

### 3. Configure SFTP Credentials in Secrets Manager

Create a JSON file with your SFTP credentials:

```json
{
  "host": "sftp.example.com",
  "port": 22,
  "username": "your-sftp-user",
  "password": "your-sftp-password"
}
```

Upload to Secrets Manager:

```powershell
aws secretsmanager put-secret-value `
  --secret-id sftp-credentials-v3 `
  --secret-string file://sftp-credentials.json
```

**Important**: Do NOT commit `sftp-credentials.json` to git. It's already in `.gitignore`.

### 4. Test the Solution

Upload a test file to your S3 bucket:

```powershell
aws s3 cp test-file.txt s3://my-unique-bucket-name/
```

The Lambda function will automatically:
1. Be triggered by the S3 event
2. Download the file from S3
3. Retrieve SFTP credentials from Secrets Manager
4. Upload the file to your SFTP server

### 5. Monitor Execution

Check Lambda logs in CloudWatch:

```powershell
aws logs tail /aws/lambda/sftp-transfer-lambda-v3 --follow
```

## Stack Outputs

After deployment, retrieve important values:

```powershell
aws cloudformation describe-stacks `
  --stack-name sftp-transfer-stack `
  --query 'Stacks[0].Outputs' `
  --output table
```

Outputs include:
- `BucketName`: S3 bucket name
- `LambdaArn`: Lambda function ARN
- `SecretArn`: Secrets Manager secret ARN
- `VpcId`: VPC ID

## Cost Considerations

This stack creates resources that incur costs:
- **NAT Gateways**: ~$0.045/hour each (2 gateways = ~$65/month)
- **Elastic IPs**: Free when attached to running NAT gateways
- **Lambda**: Pay per invocation and duration
- **S3**: Storage and data transfer costs

**To minimize costs**: Delete the stack when not in use.

## Cleanup

To remove all resources:

```powershell
# Empty the S3 bucket first (required before deletion)
aws s3 rm s3://my-unique-bucket-name --recursive

# Delete the CloudFormation stack
aws cloudformation delete-stack --stack-name sftp-transfer-stack

# Delete the Lambda layer (optional)
aws lambda delete-layer-version `
  --layer-name paramiko-sftp-layer `
  --version-number 1
```

## Security Best Practices

1. **Secrets Management**: SFTP credentials are encrypted at rest in Secrets Manager
2. **VPC Isolation**: Lambda runs in private subnets with no direct internet access
3. **Least Privilege IAM**: Lambda role has minimum required permissions
4. **No Hardcoded Credentials**: All sensitive data in Secrets Manager or parameters

## Troubleshooting

### Lambda can't import paramiko
- Ensure you provided the correct `ParamikoLayerArn` when deploying
- Verify the layer was published successfully

### Lambda times out connecting to SFTP
- Verify your SFTP server is accessible from the internet
- Check security group rules on your SFTP server
- Ensure NAT Gateways are running and routes are correct

### S3 event not triggering Lambda
- Verify the S3 notification configuration was created (check stack outputs)
- Ensure Lambda has permission to be invoked by S3 (created automatically in template)

### Access denied to S3 or Secrets Manager
- Check Lambda execution role permissions in IAM
- Verify the bucket name matches the parameter used in deployment

## File Structure

```
.
├── cloudformation/
│   ├── sftpS3trigger.yml       # Main CloudFormation template
│   └── paramiko-layer.zip       # Paramiko Lambda layer
├── .gitignore                   # Excludes secrets and keys
└── README.md                    # This file
```

## Contributing

This is a proof-of-concept project. Enhancements welcome:
- Add support for key-based SFTP authentication
- Implement error handling and retry logic
- Add CloudWatch alarms for failures
- Support for multiple SFTP destinations

## License

MIT License - feel free to use and modify for your needs.