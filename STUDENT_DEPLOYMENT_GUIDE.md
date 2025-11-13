# Student Deployment Guide: Preparing Your Streamlit Project for AWS

This guide will help you prepare your Streamlit project for deployment on AWS ECS. Follow each step carefully and verify your setup before submitting.

---

## Prerequisites

- Your Streamlit application running locally
- Docker installed on your computer
- Python environment with all dependencies

---

## Step 1: Choose Your Project Slug

Your project slug will be part of the URL where students and instructors access your app.

### Requirements:
- Use lowercase letters, numbers, and hyphens only
- No spaces or special characters
- Keep it short and descriptive (e.g., `portfolio-analyzer`, `market-predictor`, `jane-doe-finance`)

### Example:
```
✅ Good: agentics-finance, stock-analyzer, team-alpha
❌ Bad: Agentics Finance, stock_analyzer!, my@project
```

**Your Project Slug:** `_______________________` (Write this down!)

**Your Final URL will be:** `https://[cloudfront-domain]/[your-slug]`

---

## Step 2: Organize Your Project Structure

Ensure your project has this structure:

```
your-project/
├── app.py                 # Your main Streamlit app (or main.py)
├── requirements.txt       # All Python dependencies
├── data/                  # Data files (if needed)
├── utils/                 # Helper modules
├── agents/                # Your agents/tools
└── README.md             # Project description
```

### ✅ Verification:
```bash
# Check that your main file exists
ls app.py  # or ls main.py

# Check requirements.txt exists
ls requirements.txt
```

---

## Step 3: Create `requirements.txt`

List all Python packages your app needs.

### Generate requirements.txt:

**Option A: If using venv/virtualenv:**
```bash
pip freeze > requirements.txt
```

**Option B: Manual creation:**
```txt
streamlit>=1.31.0
pandas>=2.0.0
plotly>=5.18.0
# Add all other packages you import
```

### ✅ Verification:
```bash
# Check the file exists and has content
cat requirements.txt

# Test installation in a fresh environment
python -m venv test_env
source test_env/bin/activate  # On Windows: test_env\Scripts\activate
pip install -r requirements.txt
deactivate
rm -rf test_env
```

---

## Step 4: Test Your App Locally

Make sure your app runs without errors.

### Run your app:
```bash
streamlit run app.py
```

### ✅ Verification Checklist:
- [ ] App starts without errors
- [ ] All visualizations load correctly
- [ ] All features work as expected
- [ ] No missing dependencies errors
- [ ] Data files load properly

---

## Step 5: Create `.dockerignore` File

This tells Docker which files to exclude from the build.

### Create `.dockerignore`:
```bash
# Create the file
touch .dockerignore
```

### Add this content:
```dockerignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python

# Virtual environments
venv/
env/
ENV/
.venv

# IDEs
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Git
.git/
.gitignore

# Testing
.pytest_cache/
.coverage

# Environment files
.env
.env.local

# Outputs
visualizations/
*.log
```

### ✅ Verification:
```bash
# Check file exists
ls -la .dockerignore
```

---

## Step 6: Create `Dockerfile`

This file tells Docker how to build your application.

### Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies (needed for some Python packages)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        build-essential \
        g++ \
        gcc \
        python3-dev && \
    rm -rf /var/lib/apt/lists/*

# Copy requirements first (for better caching)
COPY requirements.txt .

# Copy application files
COPY . .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Expose Streamlit port
EXPOSE 8501

# IMPORTANT: Replace 'your-project-slug' with YOUR actual slug!
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0", "--server.baseUrlPath=/your-project-slug"]
```

### 🔴 CRITICAL: Update the CMD line!

Replace `your-project-slug` with your actual project slug from Step 1.

**Example:**
If your slug is `portfolio-analyzer`, the line should be:
```dockerfile
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0", "--server.baseUrlPath=/portfolio-analyzer"]
```

### If your main file is NOT `app.py`:
Replace `app.py` with your actual filename (e.g., `main.py`)

### ✅ Verification:
```bash
# Check Dockerfile exists
ls Dockerfile

# Verify the CMD line has YOUR slug (not "your-project-slug")
grep "baseUrlPath" Dockerfile
# Should show: --server.baseUrlPath=/[YOUR-ACTUAL-SLUG]
```

---

## Step 7: Handle Environment Variables

If your app uses API keys or secrets:

### Create `.env.example`:
```bash
# Create example file
cat > .env.example << 'EOF'
# Example environment variables
# Students should create a .env file with actual values
GOOGLE_API_KEY=your_key_here
OPENAI_API_KEY=your_key_here
# Add any other variables you need
EOF
```

### Create `.env` for local testing:
```bash
cp .env.example .env
# Edit .env and add your ACTUAL keys
```

### 📧 Email TA:
List all environment variables your app needs and send to TA via email (see step 10).

### ✅ Verification:
```bash
# Make sure .env is in .gitignore
echo ".env" >> .gitignore

# Verify .env.example exists
ls .env.example
```

---

## Step 8: Test Docker Build Locally

Build and test your Docker container before submitting.

### Build the image:
```bash
# Replace 'your-project-slug' with your actual slug
docker build --platform linux/amd64 -t your-project-slug:test .
```

**Note:** This will take 5-15 minutes. Watch for errors!

### ✅ Verification During Build:
- [ ] No "file not found" errors
- [ ] All dependencies install successfully
- [ ] Build completes with "Successfully built" message

### Run the container locally:
```bash
# Replace 'your-project-slug' with your actual slug
docker run -p 8501:8501 your-project-slug:test
```

### Test in browser:
```
http://localhost:8501/your-project-slug
```

### ✅ Verification:
- [ ] Container starts without errors
- [ ] App loads in browser with the correct path
- [ ] All features work in the containerized version
- [ ] No "module not found" errors

### Stop the container:
```bash
# Press Ctrl+C in terminal, or:
docker ps  # Find container ID
docker stop [container-id]
```

---

## Step 9: Create Project README

Update your README.md with deployment information.

### Add to your README.md:
```markdown
# [Your Project Name]

## Description
[Brief description of your project]

## Deployment Information

- **Project Slug:** `your-project-slug`
- **Deployment URL:** `https://[cloudfront-domain]/your-project-slug`
- **Main File:** `app.py`

## Environment Variables Required
- `GOOGLE_API_KEY`: Google Gemini API key
- [List all other variables]

## Local Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Run app
streamlit run app.py
```

## Docker Build

```bash
docker build --platform linux/amd64 -t your-project-slug:latest .
docker run -p 8501:8501 your-project-slug:latest
```

## Features
- [Feature 1]
- [Feature 2]
- [Feature 3]
```

### ✅ Verification:
```bash
# Check README exists and is complete
cat README.md
```

---

## Step 10: Submit to TA

### Create a clean repository:

```bash
# Make sure everything is committed
git status
git add .
git commit -m "Prepare for deployment"

# Create a deployment info file
cat > DEPLOYMENT_INFO.txt << EOF
Project Slug: your-project-slug
Student Name: Your Name
Student Email: your.email@university.edu
Main File: app.py

Environment Variables Needed:
- GOOGLE_API_KEY: [Yes/No]
- OPENAI_API_KEY: [Yes/No]
- [List others]

Verified Working Locally: [Yes/No]
Verified Working in Docker: [Yes/No]

Notes:
[Any special instructions or notes for the TA]
EOF
```

### Share your repository:
- Push to GitHub/GitLab
- Or zip the entire project folder
- Email TA with the link or zip file

### 📧 Email to TA:

```
Subject: Deployment Request - [Your Name] - [Project Slug]

Hi [TA Name],

I've prepared my Streamlit project for deployment. Here are the details:

Repository: [GitHub link or attached zip]
Project Slug: your-project-slug
Main File: app.py

Environment Variables Required:
- GOOGLE_API_KEY: ...
- ...

I've verified:
✅ App runs locally
✅ Docker builds successfully
✅ Docker container runs correctly

Best regards,
[Your Name]
```

