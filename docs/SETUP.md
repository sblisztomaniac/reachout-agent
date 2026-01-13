# Reachout Agent - Local Development Setup

This guide walks you through setting up the Reachout Agent backend for local development.

---

## Prerequisites

### Required Software

- **Python 3.11+** (recommended: 3.11 or 3.12)
- **pip** (Python package manager, included with Python)
- **Git** (for version control)
- **curl** or **Postman** (for API testing)

### Required Accounts & API Keys

Before starting development, obtain these credentials:

1. **ZeroDB Account**
   - Sign up at https://zerodb.ai (or use AI-Native Studio dashboard)
   - You'll create a project and get API credentials in Step 4 below

2. **AI-Native Studio API Key**
   - Access the AI-Native Studio dashboard
   - Navigate to API Keys section
   - Generate a new API key for agent orchestration, embeddings, and agent bridge

3. **Email Provider** (choose ONE):
   - **SendGrid**: Sign up at https://sendgrid.com and get API key
   - **Resend**: Sign up at https://resend.com and get API key
   - **SMTP**: Use your existing email provider (Gmail, Outlook, etc.)

4. **Optional External Tools**:
   - **Firecrawl API**: Get key from https://firecrawl.dev (for web scraping)
   - **Web Search MCP**: Set up local MCP server or use Brave Search API

---

## Step 1: Clone the Repository

```bash
git clone <repository-url>
cd reachout-agent
```

---

## Step 2: Create Python Virtual Environment

### On macOS/Linux:

```bash
# Create virtual environment
python3.11 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Verify Python version
python --version  # Should show Python 3.11.x
```

### On Windows:

```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
venv\Scripts\activate

# Verify Python version
python --version  # Should show Python 3.11.x
```

---

## Step 3: Install Python Dependencies

```bash
# Upgrade pip to latest version
pip install --upgrade pip

# Install all project dependencies
pip install -r backend/requirements.txt

# Verify installation
pip list
```

Expected packages:
- fastapi
- uvicorn
- pydantic
- python-jose (for JWT)
- passlib (for password hashing)
- python-dotenv (for environment variables)
- httpx (for HTTP client to external APIs)
- pytest (for testing)
- pytest-asyncio (for async tests)

---

## Step 4: Configure Environment Variables

### 4.1 Create .env File

```bash
# Copy the example file
cp .env.example .env

# Open .env in your editor
nano .env  # or use your preferred editor
```

### 4.2 Generate JWT Secret

```bash
# Generate a secure random JWT secret
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Copy the output and paste it into `.env` as `JWT_SECRET=<generated_secret>`

### 4.3 Configure ZeroDB (see docs/ZeroDB-INIT.md for details)

For now, leave these as placeholders (you'll fill them in Step 5):
```
ZERODB_PROJECT_ID=your_project_id_here
ZERODB_API_KEY=your_zerodb_api_key_here
```

### 4.4 Configure AI-Native APIs

```
AI_NATIVE_API_KEY=<your_api_key_from_dashboard>
AI_NATIVE_BASE_URL=https://api.ai-native.io/v1/public
```

### 4.5 Configure Email Provider

Choose ONE option:

**Option 1: SendGrid**
```
SENDGRID_API_KEY=<your_sendgrid_key>
SENDGRID_FROM_EMAIL=noreply@yourdomain.com
SENDGRID_FROM_NAME=Your Company
```

**Option 2: Resend**
```
RESEND_API_KEY=<your_resend_key>
RESEND_FROM_EMAIL=noreply@yourdomain.com
```

**Option 3: SMTP (Gmail example)**
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM_EMAIL=your_email@gmail.com
```

### 4.6 Set Environment Mode

```
ENVIRONMENT=development
LOG_LEVEL=INFO
DEBUG=true
```

---

## Step 5: Initialize ZeroDB

**IMPORTANT**: You must create a ZeroDB project and tables before running the application.

Follow the detailed guide: [ZeroDB-INIT.md](./ZeroDB-INIT.md)

Summary:
1. Create ZeroDB project via API or dashboard
2. Get your `project_id` and `api_key`
3. Update `.env` with these credentials
4. Run table creation script to create all 13 tables
5. Verify tables were created successfully

Once complete, your `.env` should have:
```
ZERODB_PROJECT_ID=<actual_project_id>
ZERODB_API_KEY=<actual_api_key>
```

---

## Step 6: Verify Configuration

Create a simple test script to verify your environment is configured correctly:

```bash
# Create test script
cat > test_config.py << 'EOF'
import os
from dotenv import load_dotenv

load_dotenv()

required_vars = [
    "ZERODB_PROJECT_ID",
    "ZERODB_API_KEY",
    "JWT_SECRET",
    "AI_NATIVE_API_KEY"
]

print("Checking environment variables...")
for var in required_vars:
    value = os.getenv(var)
    if value and value != f"your_{var.lower()}_here" and "generate" not in value:
        print(f"✓ {var} is set")
    else:
        print(f"✗ {var} is NOT set or still has placeholder value")

print("\nIf all variables show ✓, your configuration is ready!")
EOF

# Run the test
python test_config.py
```

All variables should show ✓ before proceeding.

---

## Step 7: Run the FastAPI Backend

### Start the development server:

```bash
# Navigate to backend directory
cd backend

# Run with uvicorn (auto-reload enabled)
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

Expected output:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
INFO:     Started server process
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

### Verify the server is running:

Open your browser to: http://localhost:8000

You should see:
```json
{
  "message": "Reachout Agent API",
  "version": "1.0.0",
  "status": "healthy"
}
```

### Access API documentation:

- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

---

## Step 8: Run Tests

```bash
# Navigate to backend directory (if not already there)
cd backend

# Run all tests
pytest

# Run with verbose output
pytest -v

# Run with coverage report
pytest --cov=. --cov-report=html

# Open coverage report (macOS)
open htmlcov/index.html
```

Expected output:
```
============================= test session starts ==============================
collected 15 items

tests/test_auth.py ....                                                  [ 26%]
tests/test_users.py .....                                                [ 60%]
tests/test_campaigns.py ......                                           [100%]

============================== 15 passed in 2.34s ===============================
```

---

## Step 9: Test API Endpoints Manually

### Register a new user:

```bash
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePassword123!"
  }'
```

Expected response:
```json
{
  "user_id": "uuid-here",
  "email": "test@example.com",
  "access_token": "jwt-token-here"
}
```

### Login:

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePassword123!"
  }'
```

### Test authenticated endpoint:

```bash
# Save token from login response
TOKEN="<access_token_from_login>"

# Get user profile
curl -X GET http://localhost:8000/api/v1/users/me \
  -H "Authorization: Bearer $TOKEN"
```

---

## Step 10: Development Workflow

### Typical development cycle:

1. **Make code changes** in `backend/` directory
2. **Uvicorn auto-reloads** (if running with `--reload`)
3. **Run tests** to verify changes: `pytest`
4. **Test manually** using curl or Swagger UI
5. **Commit changes** when tests pass

### Useful commands:

```bash
# Stop the server: CTRL+C

# Restart the server:
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Run specific test file:
pytest tests/test_auth.py

# Run tests matching pattern:
pytest -k "test_login"

# Run tests with print statements visible:
pytest -s

# Check code formatting (if using black):
black backend/

# Check for linting issues (if using ruff):
ruff check backend/
```

---

## Common Issues & Troubleshooting

### Issue 1: "ModuleNotFoundError: No module named 'fastapi'"

**Solution**: Make sure virtual environment is activated and dependencies are installed:
```bash
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r backend/requirements.txt
```

### Issue 2: "KeyError: 'ZERODB_PROJECT_ID'"

**Solution**: Your `.env` file is missing or not loaded correctly:
```bash
# Verify .env exists
ls -la .env

# Verify variables are set
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.getenv('ZERODB_PROJECT_ID'))"
```

### Issue 3: "401 Unauthorized" when calling ZeroDB API

**Solution**: Check your `ZERODB_API_KEY` is correct:
```bash
# Test ZeroDB authentication
curl -X GET https://api.zerodb.ai/v1/public/projects/<project_id> \
  -H "Authorization: Bearer <your_api_key>"
```

See [API-REFERENCE.md](./API-REFERENCE.md) for more details on authentication.

### Issue 4: Tests failing with "Connection refused"

**Solution**: Make sure you're using mock clients for tests (not hitting real APIs):
```python
# In tests/conftest.py
@pytest.fixture
def mock_zerodb_client():
    # Return mock client instead of real one
    return MockZeroDBClient()
```

### Issue 5: Port 8000 already in use

**Solution**: Either stop the existing process or use a different port:
```bash
# Find process using port 8000
lsof -i :8000

# Kill the process
kill -9 <PID>

# Or use a different port
uvicorn main:app --reload --port 8001
```

---

## Next Steps

Once your local environment is running:

1. **Explore the API**: Use Swagger UI at http://localhost:8000/docs
2. **Read the PRD**: See [prd.md](./prd.md) to understand the product vision
3. **Review data model**: See [datamodel.md](./datamodel.md) for table schemas
4. **Start implementing stories**: Run `/ralph-story` to begin autonomous development

---

## Additional Resources

- **API Authentication**: [API-REFERENCE.md](./API-REFERENCE.md)
- **ZeroDB Setup**: [ZeroDB-INIT.md](./ZeroDB-INIT.md)
- **Testing Guide**: [TESTING.md](./TESTING.md) (coming soon)
- **External Services**: [EXTERNAL-SERVICES.md](./EXTERNAL-SERVICES.md) (coming soon)
- **Project Documentation**: All docs in `docs/` directory

---

## Getting Help

If you encounter issues not covered here:

1. Check the [AGENTS.md](../AGENTS.md) file for project-specific patterns
2. Review error logs in `backend/logs/` (if logging is configured)
3. Search existing GitHub issues (if repository is on GitHub)
4. Ask in the project's communication channel (Slack, Discord, etc.)

---

## Environment Cleanup

When you're done developing:

```bash
# Deactivate virtual environment
deactivate

# Stop all running servers
# (CTRL+C in terminals running uvicorn)

# Optionally remove virtual environment
rm -rf venv
```

To start again later:
```bash
source venv/bin/activate  # Reactivate environment
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000  # Restart server
```
