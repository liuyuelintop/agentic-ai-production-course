# Complete AWS Deployment Tutorial: From Docker Build to Production

> **Language Versions:** **English** (current) | [中文](./aws-docker-deployment-guide-zh.md)

This is a hands-on AWS deployment tutorial that explains every shell command and configuration step in detail.

## Table of Contents
1. [Docker Local Build and Testing](#docker-local-build-and-testing)
2. [AWS CLI Configuration and Authentication](#aws-cli-configuration-and-authentication)
3. [ECR Container Image Push](#ecr-container-image-push)
4. [App Runner Service Deployment](#app-runner-service-deployment)
5. [Monitoring, Debugging and Troubleshooting](#monitoring-debugging-and-troubleshooting)
6. [Update Deployment Process](#update-deployment-process)
7. [AWS Fundamentals](#aws-fundamentals)

---

## Docker Local Build and Testing

### Environment Variable Loading Mechanism

Before starting the build, we need to understand how environment variables are loaded on different operating systems:

#### Mac/Linux Systems
```bash
export $(cat .env | grep -v '^#' | xargs)
```

**Command Breakdown:**
- `cat .env`: Read the .env file contents
- `grep -v '^#'`: Filter out comment lines starting with # (-v means reverse match)
- `xargs`: Convert input into command line arguments
- `export`: Set variables as environment variables for child processes

**Execution Process:**
1. Read .env file: `CLERK_SECRET_KEY=sk_test_xxx`
2. After filtering comments: `CLERK_SECRET_KEY=sk_test_xxx`
3. Export command executes: `export CLERK_SECRET_KEY=sk_test_xxx`

#### Windows PowerShell Systems
```powershell
Get-Content .env | ForEach-Object {
    if ($_ -match '^(.+?)=(.+)$') {
        [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2])
    }
}
```

**Command Breakdown:**
- `Get-Content .env`: PowerShell command to read files
- `ForEach-Object`: Execute operation on each line
- `$_ -match '^(.+?)=(.+)$'`: Regex to match key-value pair format
- `$matches[1]`: Captured variable name
- `$matches[2]`: Captured variable value
- `[System.Environment]::SetEnvironmentVariable()`: .NET method to set environment variables

### Docker Build Command Explained

#### Mac/Linux Build
```bash
docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**Parameter Details:**
- `docker build`: Docker image build command
- `\`: Line continuation character for readability
- `--build-arg`: Pass build-time arguments to Dockerfile
  - These arguments are only available during build process, not saved in final image
  - Used to pass sensitive information needed during build
- `"$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY"`:
  - Double quotes ensure special characters like spaces are handled correctly
  - $ symbol references previously set environment variable
- `-t consultation-app`:
  - Tag the built image with a name
  - If no version specified, defaults to `latest`
- `.`: Build context, points to current directory

**What Happens During Build:**
1. Docker reads Dockerfile from current directory
2. Passes `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` as ARG
3. Executes multi-stage build:
   - Stage 1: Node.js environment builds frontend static files
   - Stage 2: Python environment configures backend service
4. Finally generates image named `consultation-app:latest`

### Docker Run Command Explained

#### Mac/Linux Run
```bash
docker run -p 8000:8000 \
  -e CLERK_SECRET_KEY="$CLERK_SECRET_KEY" \
  -e CLERK_JWKS_URL="$CLERK_JWKS_URL" \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  consultation-app
```

**Parameter Details:**
- `docker run`: Run container command
- `-p 8000:8000`: Port mapping
  - Format: `host-port:container-port`
  - Maps local port 8000 to container port 8000
  - Allows access via `localhost:8000`
- `-e`: Set environment variables
  - These variables are available during container runtime
  - Different from build-time `--build-arg`
- `consultation-app`: Image name to run

**Why Separate Build-time and Runtime Variables?**
- **Build-time variables (--build-arg)**: Configure application during build (e.g., frontend public key)
- **Runtime variables (-e)**: Configure application during container execution (e.g., backend secrets)

### Common Build Issues and Solutions

#### Issue 1: Environment Variables Not Loaded
```bash
# Error symptom
echo $CLERK_SECRET_KEY
# Output is empty

# Solution
# 1. Check if .env file exists
ls -la .env

# 2. Check file content format
cat .env

# 3. Reload environment variables
export $(cat .env | grep -v '^#' | xargs)
```

#### Issue 2: Docker Build Fails
```bash
# Error symptom
Error: Cannot find module 'next'

# Solution
# 1. Check if package.json exists
ls -la package.json

# 2. Clear Docker cache
docker system prune -f

# 3. Rebuild
docker build --no-cache -t consultation-app .
```

#### Issue 3: Port Already in Use
```bash
# Error symptom
Error: Port 8000 is already in use

# Solution
# 1. Find process using the port
lsof -i :8000

# 2. Kill the process
kill -9 [process-id]

# 3. Or use different port
docker run -p 8001:8000 ...
```

---

## AWS CLI Configuration and Authentication

### AWS CLI Installation Verification

```bash
# Check if AWS CLI is installed
aws --version

# Expected output similar to:
# aws-cli/2.15.30 Python/3.11.6 Darwin/23.1.0 exe/x86_64 prompt/off
```

### AWS Configuration Command Explained

```bash
aws configure
```

**Configuration Process:**

1. **AWS Access Key ID**:
   ```
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   ```
   - This is the unique identifier for AWS user
   - Format usually starts with `AKIA` followed by 16 random characters
   - Acts like a username, not secret information

2. **AWS Secret Access Key**:
   ```
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY
   ```
   - This is the password paired with Access Key
   - Usually 40 characters long
   - **Critical**: This is sensitive information, exposure leads to account compromise

3. **Default region name**:
   ```
   Default region name [None]: us-east-1
   ```
   - Geographic location of AWS services
   - Affects network latency and data storage location
   - Common regions:
     - `us-east-1`: US East (N. Virginia) - Cheapest, most services
     - `us-west-2`: US West (Oregon) - Best for West Coast users
     - `eu-west-1`: Europe (Ireland) - Best for European users
     - `ap-southeast-1`: Asia Pacific (Singapore) - Best for Asian users

4. **Default output format**:
   ```
   Default output format [None]: json
   ```
   - Other options: `table`, `text`, `yaml`
   - `json` is best for programmatic processing

### Authentication Mechanism

After running `aws configure`, AWS CLI creates configuration files at:

```bash
# View configuration file location
ls -la ~/.aws/

# Output:
# credentials  - stores access keys
# config      - stores region and output format
```

**credentials file content:**
```
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY
```

**config file content:**
```
[default]
region = us-east-1
output = json
```

### Verify AWS Configuration

```bash
# Test AWS CLI configuration
aws sts get-caller-identity

# Expected output:
{
    "UserId": "AIDACKCEVSQ6C2EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/aiengineer"
}
```

**Output Explanation:**
- `UserId`: Unique ID of IAM user
- `Account`: 12-digit AWS account ID
- `Arn`: Amazon Resource Name, complete path of user

---

## ECR Container Image Push

### ECR Authentication Mechanism

```bash
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com
```

**Command Breakdown:**

1. **`aws ecr get-login-password --region $DEFAULT_AWS_REGION`**
   - Requests temporary login password from AWS ECR
   - Password valid for 12 hours
   - This is not your AWS password, but a temporary token specifically for Docker

2. **Pipe operation `|`**
   - Passes output of previous command as input to next command
   - Enables command chaining

3. **`docker login --username AWS --password-stdin`**
   - `--username AWS`: Fixed username for ECR is "AWS"
   - `--password-stdin`: Read password from standard input (output of previous command)
   - Avoids displaying password in command line

4. **ECR Registry URL Format**
   ```
   $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com
   ```
   - `$AWS_ACCOUNT_ID`: Your 12-digit AWS account ID
   - `dkr.ecr`: ECR service identifier
   - `$DEFAULT_AWS_REGION`: AWS region
   - `amazonaws.com`: AWS domain name

### Platform-Specific Build

```bash
docker build \
  --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**Why `--platform linux/amd64` is needed?**

1. **Apple Silicon (M1/M2/M3) Mac users**:
   - Local CPU architecture: ARM64
   - AWS runtime environment: x86_64 (AMD64)
   - Without specifying platform causes architecture mismatch errors

2. **Intel Mac and Windows users**:
   - Local CPU architecture: x86_64
   - AWS runtime environment: x86_64
   - Can omit but specifying is safer

3. **Architecture mismatch error symptoms**:
   ```
   standard_init_linux.go:228: exec user process caused: exec format error
   ```

### Image Tagging

```bash
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**Tagging Principles:**
- Docker images can have multiple tags
- Local tag: `consultation-app:latest`
- Remote tag: `123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:latest`
- Same image, different tags point to same image ID

**Tag Naming Convention:**
```
[registry-url]/[repository-name]:[tag]

Examples:
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:v1.0.0
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:latest
123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app:dev
```

### Image Push Process

```bash
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**Push Process Explained:**

1. **Layer Upload**:
   ```
   The push refers to repository [123456789012.dkr.ecr.us-east-1.amazonaws.com/consultation-app]
   a8c6b3421... Pushing [==================================================>]  5.632kB/5.632kB
   b9d7e4567... Pushing [===================>                               ]  45.23MB/108.5MB
   ```

2. **Layer Reuse**:
   - Docker images consist of multiple layers
   - Same layers are shared across multiple images
   - Only changed layers need re-upload

3. **Push Time Estimation**:
   - First push: 3-10 minutes (depends on network speed)
   - Subsequent pushes: 30 seconds - 2 minutes (only changed layers)

### ECR Push Troubleshooting

#### Issue 1: Authentication Failed
```bash
# Error message
Error response from daemon: login attempt to https://123456789012.dkr.ecr.us-east-1.amazonaws.com/v2/ failed with status: 401 Unauthorized

# Solution
# 1. Re-authenticate
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com

# 2. Check IAM permissions
aws iam list-attached-user-policies --user-name aiengineer
```

#### Issue 2: Repository Not Found
```bash
# Error message
Error response from daemon: repository consultation-app not found

# Solution
# Create repository named 'consultation-app' in ECR console
aws ecr create-repository --repository-name consultation-app --region $DEFAULT_AWS_REGION
```

#### Issue 3: Network Timeout
```bash
# Error message
Error response from daemon: net/http: TLS handshake timeout

# Solution
# 1. Check network connection
ping amazonaws.com

# 2. Retry push
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# 3. If persists, check firewall settings
```

---

## App Runner Service Deployment

### App Runner Configuration Details

#### Compute Resource Configuration

**vCPU and Memory Selection:**
```
vCPU: 0.25 vCPU
Memory: 0.5 GB
```

**Configuration Meaning:**
- `0.25 vCPU`: 1/4 of a virtual CPU core
  - Suitable for low-traffic applications
  - Lowest cost configuration
  - Can handle light concurrent requests

- `0.5 GB Memory`: 512MB memory
  - Python FastAPI applications typically need 200-300MB
  - Next.js static files are small
  - Remaining memory for system cache

**Other Available Configurations:**
```
1 vCPU, 2 GB   - Medium traffic (~100 concurrent)
2 vCPU, 4 GB   - High traffic (~500 concurrent)
4 vCPU, 12 GB  - Enterprise level (~2000 concurrent)
```

#### Environment Variable Configuration Mechanism

Environment variables set in App Runner:
```
CLERK_SECRET_KEY=sk_test_...
CLERK_JWKS_URL=https://...
OPENAI_API_KEY=sk-...
```

**Why no need for `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`?**
- This variable was compiled into static files during Docker build
- Build-time parameters vs runtime environment variables:
  - **Build-time**: Frontend code compilation, public key embedded in JavaScript
  - **Runtime**: Backend service startup, secrets used for API verification

#### Auto Scaling Configuration

```
Minimum size: 1
Maximum size: 1
```

**Why choose this configuration?**

1. **Minimum size: 1**
   - App Runner requires at least 1 instance
   - Ensures continuous service availability
   - Avoids cold start delays

2. **Maximum size: 1**
   - Cost control
   - For learning projects, 1 instance is sufficient
   - Production typically sets 2-10

**Production Recommended Configuration:**
```
Minimum size: 2  (high availability)
Maximum size: 10 (handle traffic spikes)
```

#### Health Check Detailed Configuration

```
Protocol: HTTP
Path: /health
Interval: 20 seconds
Timeout: 5 seconds
Healthy threshold: 2
Unhealthy threshold: 5
```

**Parameter Explanation:**

1. **Protocol: HTTP**
   - App Runner sends HTTP requests to your application
   - Path is `/health`
   - Expects HTTP 200 status code response

2. **Interval: 20 seconds**
   - Check every 20 seconds
   - Range: 5-300 seconds
   - 20 seconds balances timely problem detection and avoiding excessive checks

3. **Timeout: 5 seconds**
   - Maximum time to wait for response
   - Over 5 seconds considered failed check
   - Should be less than Interval time

4. **Healthy threshold: 2**
   - 2 consecutive successes to consider service healthy
   - Avoids false positives from occasional success

5. **Unhealthy threshold: 5**
   - 5 consecutive failures to consider service unhealthy
   - Avoids false negatives from occasional failures

**Health Check Endpoint Code:**
```python
@app.get("/health")
def health_check():
    """Health check endpoint for AWS App Runner"""
    return {"status": "healthy"}
```

### Deployment Process Monitoring

#### Deployment Status Changes
```
Creating → Building → Deploying → Running
```

**Stage Meanings:**

1. **Creating (1-2 minutes)**
   - App Runner creates service instances
   - Allocates compute resources
   - Configures networking

2. **Building (2-3 minutes)**
   - Pulls Docker image from ECR
   - Starts container
   - Application initialization

3. **Deploying (1-2 minutes)**
   - Health checks
   - Traffic routing configuration
   - SSL certificate setup

4. **Running**
   - Service running normally
   - Accepting external traffic

#### Deployment Log Interpretation

**Normal Deployment Logs:**
```
[INFO] Pulling image from ECR
[INFO] Starting container on port 8000
[INFO] Health check passed: GET /health returned 200
[INFO] Service is ready to receive traffic
```

**Abnormal Deployment Logs:**
```
[ERROR] Health check failed: GET /health returned 500
[ERROR] Container exited with code 1
[WARN] Service deployment failed, rolling back
```

---

## Monitoring, Debugging and Troubleshooting

### CloudWatch Log Navigation

#### Ways to Access Logs

1. **Through App Runner Console**:
   - Service page → Logs tab
   - Separated into Application logs and Service logs

2. **Direct CloudWatch Access**:
   - Search "CloudWatch" → Logs → Log groups
   - Find `/aws/apprunner/consultation-app-service/...`

#### Log Type Analysis

**Application Logs**:
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "message": "Starting FastAPI server on port 8000"
}
```

**Service Logs**:
```json
{
  "timestamp": "2024-01-15T10:29:55.000Z",
  "level": "INFO",
  "message": "Container started successfully"
}
```

### Common Error Diagnosis

#### Error 1: Health Check Failed

**Error Symptom:**
```
Health check failed: timeout after 5 seconds
```

**Diagnosis Steps:**
1. Check if application starts on port 8000
2. Verify `/health` endpoint exists
3. View application startup logs

**Solution:**
```python
# Ensure health endpoint exists
@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now()}

# Check application startup
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### Error 2: Environment Variables Not Loaded

**Error Symptom:**
```
KeyError: 'CLERK_SECRET_KEY'
```

**Diagnosis Steps:**
1. Check App Runner environment variable configuration
2. Verify variable name spelling
3. Validate variable value format

**Solution:**
```python
import os

# Safely get environment variables
clerk_secret = os.getenv("CLERK_SECRET_KEY")
if not clerk_secret:
    raise ValueError("CLERK_SECRET_KEY environment variable is required")
```

#### Error 3: Clerk Authentication Failed

**Error Symptom:**
```
Clerk authentication failed: Invalid JWKS URL
```

**Diagnosis Steps:**
1. Verify JWKS URL format
2. Check Clerk application configuration
3. Confirm public key matches

**JWKS URL Format Check:**
```
Correct format: https://[clerk-app-id].clerk.accounts.dev/.well-known/jwks.json
Wrong format: https://clerk.com/[clerk-app-id]/jwks
```

### Performance Monitoring Metrics

#### CPU and Memory Monitoring

**View through CloudWatch:**
1. CloudWatch → Metrics → AWS/AppRunner
2. Select service name
3. View key metrics:
   - `CPUUtilization`: CPU usage percentage
   - `MemoryUtilization`: Memory usage percentage
   - `RequestCount`: Number of requests
   - `ResponseTime`: Response time

**Normal Ranges:**
```
CPU Usage: 10-70% (continuous 90%+ needs scaling)
Memory Usage: 40-80% (continuous 95%+ needs more memory)
Response Time: <2 seconds (>5 seconds needs optimization)
```

#### Application Performance Optimization

**Python Application Optimization:**
```python
# 1. Use connection pooling
from openai import OpenAI
client = OpenAI()  # Reuse connections

# 2. Cache static content
from fastapi.staticfiles import StaticFiles
app.mount("/static", StaticFiles(directory="static"), name="static")

# 3. Enable GZIP compression
from fastapi.middleware.gzip import GZipMiddleware
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

---

## Update Deployment Process

### Complete Update Workflow

When you modify code, you need to redeploy to AWS:

#### Step 1: Rebuild Image

```bash
# Ensure environment variables are loaded
export $(cat .env | grep -v '^#' | xargs)

# Build new image (Important: include platform parameter)
docker build \
  --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

**Why rebuild is needed?**
- Docker images are immutable
- Code changes require creating new image layers
- Ensures all dependencies are up-to-date

#### Step 2: Re-push to ECR

```bash
# Re-authenticate (if over 12 hours)
aws ecr get-login-password --region $DEFAULT_AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com

# Re-tag image
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# Push update
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**Push Optimization:**
- Docker only pushes changed layers
- Unchanged layers are skipped
- Usually much faster than initial push

#### Step 3: Trigger App Runner Deployment

**In App Runner Console:**
1. Go to your service
2. Click "Deploy" button
3. Wait for deployment completion (3-5 minutes)

**Deployment Process Monitoring:**
```
Status: Deploying
↓
Status: Running (new version)
```

### Version Management Strategy

#### Use Semantic Versioning Tags

**Instead of only using `latest`:**
```bash
# Tag specific version
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:v1.1.0

# Also keep latest
docker tag consultation-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest

# Push both tags
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:v1.1.0
docker push $AWS_ACCOUNT_ID.dkr.ecr.$DEFAULT_AWS_REGION.amazonaws.com/consultation-app:latest
```

**Version Number Rules:**
```
v1.0.0 - Initial version
v1.0.1 - Bug fixes
v1.1.0 - New features
v2.0.0 - Breaking changes
```

#### Rollback Strategy

**If new version has issues:**

1. **Quick rollback to old version in ECR:**
   ```bash
   # View all versions in ECR
   aws ecr list-images --repository-name consultation-app

   # Re-tag old version as latest
   aws ecr batch-get-image --repository-name consultation-app --image-ids imageTag=v1.0.0 --query 'images[0].imageManifest' --output text | aws ecr put-image --repository-name consultation-app --image-tag latest --image-manifest file:///dev/stdin
   ```

2. **Redeploy in App Runner:**
   - Click "Deploy" to trigger deployment using latest tag

### Zero-Downtime Deployment Considerations

**App Runner Rolling Updates:**
- New instance starts and passes health checks
- Old instance terminates only after new one is ready
- Theoretically achieves zero downtime

**Potential Interruptions:**
- Health check failures causing rollback
- New code taking too long to start
- Database migrations requiring downtime

**Best Practices:**
```python
# 1. Graceful shutdown
import signal
import sys

def signal_handler(sig, frame):
    print('Gracefully shutting down...')
    # Complete ongoing requests
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)

# 2. Health check includes dependency checks
@app.get("/health")
def health_check():
    # Check database connections
    # Check external API availability
    return {"status": "healthy"}
```

---

## AWS Fundamentals

### Deep Dive into IAM Permission Model

#### IAM Core Components

**Users**:
- Represents people or applications
- Has permanent access credentials
- Our `aiengineer` is a user

**Groups**:
- Collection of users
- Convenient for bulk permission management
- Our `BroadAIEngineerAccess` is a group

**Policies**:
- JSON documents defining permissions
- Can be attached to users, groups, or roles
- Follow "principle of least privilege"

**Roles**:
- Temporary permission collections
- Can be "assumed" by users or services
- ECR access role used by App Runner is an example

#### Permission Policy Examples

**AWSAppRunnerFullAccess policy (simplified)**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "apprunner:*"
            ],
            "Resource": "*"
        }
    ]
}
```

**Granular Permission Control:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "apprunner:CreateService",
                "apprunner:UpdateService"
            ],
            "Resource": "arn:aws:apprunner:us-east-1:123456789012:service/consultation-app-service/*"
        }
    ]
}
```

### AWS Service Comparison

#### Container Deployment Options Comparison

**AWS App Runner**:
```
Pros:
- Fully managed, zero configuration
- Automatic HTTPS and load balancing
- Auto-scaling on demand
- Simple pricing

Cons:
- Less customization options
- Limited network control
- Potentially higher costs

Use cases: Quick prototypes, small to medium applications
```

**Amazon ECS + Fargate**:
```
Pros:
- More control options
- Better cost control
- Complex network configuration support
- Deep integration with other AWS services

Cons:
- Complex configuration
- Requires more AWS knowledge
- Steep learning curve

Use cases: Enterprise applications, complex architectures
```

**AWS Lambda**:
```
Pros:
- True serverless
- Ultimate cost efficiency (pay per request)
- Auto-scaling to zero
- No infrastructure management

Cons:
- Cold start latency
- Execution time limits (15 minutes)
- Complex local debugging

Use cases: Event-driven, short tasks, API gateways
```

#### Storage Service Comparison

**Amazon ECR vs Docker Hub**:

**ECR Advantages:**
```
- Deep integration with AWS services
- Better security and permission control
- Faster transfers within AWS network
- Vulnerability scanning support
```

**Docker Hub Advantages:**
```
- Free public repositories
- Larger community
- Simpler usage
- Better documentation and tutorials
```

### Regions and Availability Zones

#### Region Selection Strategy

**Based on User Location:**
```
Chinese users → ap-southeast-1 (Singapore)
US users → us-east-1 (Virginia) or us-west-2 (Oregon)
European users → eu-west-1 (Ireland)
```

**Based on Cost:**
```
Cheapest: us-east-1 (N. Virginia)
Cheaper: us-west-2 (Oregon)
Slightly more expensive: Other US regions
Most expensive: Asia Pacific and some European regions
```

**Based on Service Availability:**
```
New services first: us-east-1
Most complete services: us-east-1, us-west-2, eu-west-1
Some services missing: Other regions
```

### Cost Optimization Deep Dive

#### Billing Model Understanding

**App Runner Billing:**
```
Base cost: $0.0066/hour (0.25 vCPU, 0.5GB memory)
Monthly cost: $0.0066 × 24 × 30 = $4.75

Real scenarios:
- Continuous running: $4.75/month
- Business hours only (8 hours/day): $1.58/month
- Paused service: $0/month (but requires manual start/stop)
```

**ECR Storage Costs:**
```
Storage: $0.10/GB/month
Example: 500MB image = $0.05/month

Optimization strategies:
- Delete old image versions
- Use multi-stage builds to reduce image size
- Regularly clean unused images
```

#### Cost Monitoring Automation

**Set up detailed budget alerts:**

1. **Daily monitoring:**
   ```
   Daily budget: $0.33 ($10/30 days)
   Alert threshold: 80% ($0.26)
   ```

2. **Service-level monitoring:**
   ```
   App Runner: $5/month budget
   ECR: $1/month budget
   Other services: $4/month budget
   ```

3. **Automated responses:**
   ```python
   # Use AWS Lambda to monitor costs
   import boto3

   def lambda_handler(event, context):
       if event['cost'] > 8.0:  # $8 threshold
           # Send urgent notification
           # Optional: Auto-pause service
           pass
   ```

### Security Best Practices

#### Access Key Management

**Key Rotation Strategy:**
```bash
# Rotate access keys every 90 days
# 1. Create new key
aws iam create-access-key --user-name aiengineer

# 2. Update local configuration
aws configure

# 3. Test new key
aws sts get-caller-identity

# 4. Delete old key
aws iam delete-access-key --access-key-id AKIA... --user-name aiengineer
```

**Secure Key Storage:**
```bash
# Set appropriate file permissions
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config

# Check file permissions
ls -la ~/.aws/
# -rw------- (only owner can read/write)
```

#### Network Security

**HTTPS Enforcement:**
- App Runner automatically provides HTTPS
- Automatically redirects HTTP to HTTPS
- Free SSL/TLS certificates

**Environment Variable Security:**
```python
# Avoid logging sensitive information in application
import logging
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ❌ Wrong: might leak keys
logger.info(f"Using API key: {os.getenv('OPENAI_API_KEY')}")

# ✅ Correct: only log non-sensitive information
api_key = os.getenv('OPENAI_API_KEY')
if api_key:
    logger.info("OpenAI API key loaded successfully")
else:
    logger.error("OpenAI API key not found")
```

---

## Conclusion

This tutorial covers the complete process from Docker build to AWS production deployment, emphasizing:

1. **Practical Operations**: Detailed explanation of every command
2. **Troubleshooting**: Common issues and solutions
3. **Best Practices**: Security, cost control, performance optimization
4. **Deep Understanding**: Not just how to do it, but why to do it this way

Recommendations for continuing AWS learning:
- Practice with other AWS services (RDS, S3, CloudFront)
- Learn Infrastructure as Code (Terraform, CloudFormation)
- Understand CI/CD pipelines (GitHub Actions + AWS)
- Explore monitoring and log analysis (CloudWatch, X-Ray)

Remember: AWS is a vast platform. Mastering fundamental concepts is more important than memorizing specific operations. Learning through real projects and gradually deepening knowledge is the most effective approach.

---

**Other Language Versions:** [中文版本](./aws-docker-deployment-guide-zh.md)