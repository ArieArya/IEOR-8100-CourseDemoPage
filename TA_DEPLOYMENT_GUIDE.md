# TA Deployment Guide: Publishing Student Streamlit Apps to AWS ECS

This guide walks you through deploying a student's Streamlit project to AWS ECS with ALB and CloudFront.

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Docker installed
- Access to AWS Console
- ECR repository created: `student-streamlit-apps`
- ALB and CloudFront distribution already set up

---

## Overview

For each student project, you'll:
1. Review and validate the student's submission
2. Build Docker image for AMD64 architecture
3. Push image to ECR
4. Create ECS Task Definition
5. Create Target Group
6. Create ECS Service
7. Add ALB routing rule
8. Add CloudFront behavior
9. Test and verify deployment

**Time per project:** ~15-20 minutes (after first setup)

---

## Step 1: Receive and Validate Student Submission

### Check student provided:
- [ ] Repository link or zip file
- [ ] Project slug (lowercase, no special chars)
- [ ] `Dockerfile`
- [ ] `requirements.txt`
- [ ] Main app file (`app.py` or similar)
- [ ] `.dockerignore`
- [ ] Environment variables list (if needed)
- [ ] `DEPLOYMENT_INFO.txt`

### Quick validation:
```bash
# Clone or extract student repo
git clone [student-repo-url] student-projects/[project-slug]
# OR
unzip [student-name]-project.zip -d student-projects/[project-slug]

cd student-projects/[project-slug]

# Verify required files
ls Dockerfile .dockerignore requirements.txt app.py

# Check Dockerfile has correct baseUrlPath
grep "baseUrlPath" Dockerfile
# Should show: --server.baseUrlPath=/[project-slug]
```

### ✅ Verification:
- [ ] All required files present
- [ ] Project slug matches in Dockerfile
- [ ] No sensitive data committed (API keys, etc.)

---

## Step 2: Build Docker Image for AMD64

### Set variables:
```bash
# Student info
export PROJECT_SLUG="student-project-slug"
export STUDENT_NAME="john-doe"
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID="your-account-id"
export ECR_REPO="student-streamlit-apps"

# Verify
echo "Deploying: $PROJECT_SLUG for $STUDENT_NAME"
```

### Build image:
```bash
cd student-projects/$PROJECT_SLUG

# Build for linux/amd64 (ECS Fargate architecture)
docker build --platform linux/amd64 -t $PROJECT_SLUG:latest .
```

### ✅ Verification:
```bash
# Check build succeeded
docker images | grep $PROJECT_SLUG

# Expected output:
# student-project-slug  latest  abc123def456  1 minute ago  1.5GB
```

### Optional: Test locally before pushing:
```bash
# Run container
docker run -p 8501:8501 $PROJECT_SLUG:latest

# Test in browser
open http://localhost:8501/$PROJECT_SLUG

# Stop container
docker stop $(docker ps -q --filter ancestor=$PROJECT_SLUG:latest)
```

---

## Step 3: Push Image to ECR

### Authenticate with ECR:
```bash
aws ecr get-login-password --region $AWS_REGION | \
    docker login --username AWS --password-stdin \
    $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

### Tag and push image:
```bash
# Tag image
docker tag $PROJECT_SLUG:latest \
    $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$PROJECT_SLUG

# Push to ECR
docker push \
    $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$PROJECT_SLUG
```

### ✅ Verification:
```bash
# List images in ECR
aws ecr describe-images \
    --repository-name $ECR_REPO \
    --region $AWS_REGION \
    --query 'imageDetails[*].imageTags' \
    --output table

# Should see your project slug in the list
```

---

## Step 4: Create ECS Task Definition

### Via AWS Console

1. Go to **ECS Console** → **Task Definitions** → **Create new Task Definition**

2. **Configure task definition:**
   - **Family name:** `$PROJECT_SLUG` (e.g., `portfolio-analyzer`)
   - **Launch type:** Fargate
   - **Operating system:** Linux/X86_64
   - **CPU:** 512 (0.5 vCPU) or 1024 (1 vCPU) based on needs
   - **Memory:** 2048 MB (2 GB) or more if needed

3. **Container definition:**
   - **Container name:** `${PROJECT_SLUG}-app`
   - **Image URI:** `${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/$ECR_REPO:$PROJECT_SLUG`
   - **Port mappings:**
     - Container port: `8501`
     - Protocol: TCP
   - **Environment variables:** (if needed)
     - Add from student's list
     - Or use AWS Secrets Manager

4. **Create**

---

## Step 5: Create Target Group

### Via AWS Console:

1. Go to **EC2** → **Target Groups** → **Create target group**

2. **Basic configuration:**
   - **Target type:** IP addresses ⚠️ (NOT Instances)
   - **Target group name:** `${PROJECT_SLUG}-tg`
   - **Protocol:** HTTP
   - **Port:** 8501
   - **VPC:** Select your VPC (same as ECS cluster)

3. **Health check:**
   - **Protocol:** HTTP
   - **Path:** `/${PROJECT_SLUG}/_stcore/health` ⚠️ CRITICAL!
   - **Port:** Traffic port
   - **Healthy threshold:** 2
   - **Unhealthy threshold:** 3
   - **Timeout:** 5 seconds
   - **Interval:** 30 seconds
   - **Success codes:** 200-299

4. **Register targets:**
   - **Leave empty** - ECS will register automatically

5. **Create**

---

## Step 6: Create ECS Service

### Via AWS Console:

1. Go to **ECS Console** → **Clusters** → Select your cluster → **Services** → **Create**

2. **Environment:**
   - **Launch type:** Fargate
   - **Platform version:** Latest

3. **Deployment configuration:**
   - **Service name:** `${PROJECT_SLUG}-service`
   - **Task definition:** Select `$PROJECT_SLUG:latest`
   - **Desired tasks:** 1

4. **Networking:**
   - **VPC:** Select your VPC
   - **Subnets:** Select at least 2 subnets in different AZs
   - **Security group:** Create new or use existing
     - Allow inbound: Port 8501 from ALB security group
   - **Auto-assign public IP:** ENABLED

5. **Load balancing:**
   - **Load balancer type:** Application Load Balancer
   - **Load balancer:** Select existing ALB
   - **Container to load balance:** `${PROJECT_SLUG}-app:8501`
   - **Listener:** Use existing listener (80:HTTP or 443:HTTPS)
   - **Target group:** Select `${PROJECT_SLUG}-tg`

---

## Step 7: Add ALB Routing Rule

### Via AWS Console:

1. Go to **EC2** → **Load Balancers** → Select your ALB

2. **Listeners** tab → Click **HTTP:80** (or HTTPS:443)

3. Click **View/edit rules**

4. Click **Add rules** or **Insert Rule** (+ icon)

5. **Add condition:**
   - **Field:** Path
   - **Value:** `/${PROJECT_SLUG}*` ⚠️ Include asterisk!

6. **Add action:**
   - **Type:** Forward to
   - **Target group:** `${PROJECT_SLUG}-tg`
   - **Weight:** 1

7. **Set priority:** Lower number = higher priority (e.g., next available number)

8. **Save**

---

## Step 8: Add CloudFront Behavior

### Via AWS Console:

1. Go to **CloudFront** → **Distributions** → Select your distribution

2. **Behaviors** tab → **Create behavior**

3. **Path pattern:** `/${PROJECT_SLUG}*` ⚠️ Include asterisk!

4. **Origin and origin groups:** Select your ALB origin

5. **Viewer protocol policy:** HTTP and HTTPS (or Redirect HTTP to HTTPS)

6. **Allowed HTTP methods:** GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
   - ⚠️ **MUST include all methods** for Streamlit WebSockets!

7. **Cache policy:** CachingDisabled
   - ⚠️ **CRITICAL:** Streamlit requires no caching!

8. **Origin request policy:** AllViewer

9. **Response headers policy:** None (or CORS if needed)

10. **Compress objects automatically:** No

11. **Create behavior**

### ⏳ Wait for deployment:
CloudFront distribution must deploy (5-15 minutes)

### ✅ Verification:
```bash
# Check distribution status
aws cloudfront get-distribution \
    --id $DISTRIBUTION_ID \
    --query 'Distribution.Status'

# Wait for: "Deployed"
```

---

## Step 9: Verify Deployment

### Check ECS Service:
```bash
# Check service is running
aws ecs describe-services \
    --cluster streamlit-apps-cluster \
    --services ${PROJECT_SLUG}-service \
    --region $AWS_REGION \
    --query 'services[0].[status,runningCount,desiredCount]'

# Should show: ["ACTIVE", 1, 1]
```

### Check Target Health:
```bash
# Check target group health
aws elbv2 describe-target-health \
    --target-group-arn $TG_ARN \
    --region $AWS_REGION \
    --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]'

# Should show: ["10.x.x.x", "healthy"]
```

### Test via ALB:
```bash
# Get ALB DNS
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $ALB_ARN \
    --region $AWS_REGION \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

# Test endpoint
curl -I http://$ALB_DNS/${PROJECT_SLUG}

# Should return: HTTP/1.1 200 OK
```

### Test via CloudFront:
```bash
# Get CloudFront domain
CF_DOMAIN=$(aws cloudfront get-distribution \
    --id $DISTRIBUTION_ID \
    --query 'Distribution.DomainName' \
    --output text)

# Test endpoint
curl -I https://$CF_DOMAIN/${PROJECT_SLUG}

# Open in browser
echo "Test at: https://$CF_DOMAIN/${PROJECT_SLUG}"
```

### ✅ Full Verification Checklist:
- [ ] ECS service running (1/1 tasks)
- [ ] Target group shows "healthy"
- [ ] ALB endpoint returns 200 OK
- [ ] CloudFront endpoint returns 200 OK
- [ ] App loads in browser
- [ ] All app features work
- [ ] No console errors
- [ ] WebSocket connections work (interactive features)

---

## Step 10: Update Index Page

Add the new project to your index.html:

```html
<li class="project-card">
	<div class="project-title">
		<a href="/{project-slug}" target="_blank">{project-title}</a>
	</div>
	<div class="project-authors">
		<span>{project-authors}</span>
	</div>
	<p class="project-description">
		{project-description}
	</p>
	<a href="/{project-slug}" target="_blank" class="project-link">View Demo</a>
</li>
```

Update the index.html page in S3 (bucket: course-demos-index/index.html)

---