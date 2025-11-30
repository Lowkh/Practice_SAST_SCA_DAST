# Complete Security Scanning Guide for Beginners

## SAST + SCA + DAST with GitHub Actions

***

## 📚 Table of Contents

1. [Introduction - Understanding Security Scanning](#part-1-introduction)
2. [Prerequisites](#part-2-prerequisites)
3. [Project Setup - Calculator Web App](#part-3-project-setup)
4. [SAST Setup - CodeQL Code Scanning](#part-4-sast-setup)
5. [SCA Setup - Dependency Checking](#part-5-sca-setup)
6. [DAST Setup - Live Application Scanning](#part-6-dast-setup)
7. [Complete Integrated Workflow](#part-7-complete-workflow)
8. [Testing Everything](#part-8-testing)
9. [Understanding Results](#part-9-understanding-results)
10. [Troubleshooting](#part-10-troubleshooting)
11. [Practice Exercises](#part-11-practice-exercises)

***

## Part 1: Introduction - Understanding Security Scanning

### What Are We Building?

A **complete security scanning pipeline** that automatically checks your code for vulnerabilities using three different types of scans:

### The Three Types of Security Scans

| Scan Type | What It Does | When It Runs | Example Issues Found |
| :-- | :-- | :-- | :-- |
| **SAST** (Static Application Security Testing) | Scans your source code without running it | Before deployment | SQL injection, XSS vulnerabilities, hardcoded secrets |
| **SCA** (Software Composition Analysis) | Checks your dependencies (libraries/packages) for known vulnerabilities | Before deployment | Outdated packages with CVEs, insecure dependencies |
| **DAST** (Dynamic Application Security Testing) | Tests your running application by simulating attacks | After deployment | Runtime vulnerabilities, configuration issues, injection flaws |

### The Security Pipeline Flow

```
1. You push code
   ↓
2. SAST scans your source code (CodeQL)
   ↓
3. SCA checks your dependencies (OWASP Dependency-Check)
   ↓
4. Application builds and runs
   ↓
5. DAST tests the running app (OWASP ZAP)
   ↓
6. Reports sent to GitHub Security tab
```

### Why All Three?

- **SAST** finds code-level issues (what you wrote)
- **SCA** finds library issues (what you imported)
- **DAST** finds runtime issues (what actually happens)

Think of it like a car inspection:

- **SAST** = Checking the engine design
- **SCA** = Checking the parts you bought
- **DAST** = Test driving the car

***

## Part 2: Prerequisites

### What You Need

✅ GitHub account
✅ Basic Git knowledge
✅ Text editor (VS Code recommended)
✅ No prior security experience needed!

### Why CodeQL?

**CodeQL** is GitHub's native code analysis engine with significant advantages:

- **Free for public repositories** - No external accounts needed
- **Native GitHub integration** - Works seamlessly with GitHub Actions
- **Semantic analysis** - Understands code structure and data flow
- **No authentication tokens** - Uses standard `GITHUB_TOKEN`
- **Open source** - Maintained by GitHub and security community
- **AI-powered suggestions** - GitHub Copilot can auto-fix detected issues

### Artifact Naming Best Practices

GitHub Actions has specific requirements for artifact naming to avoid errors:

**Safe naming conventions**:
- Use **alphanumeric characters (a-z, 0-9)** and **hyphens (`-`)**
- Replace **underscores (`_`) with hyphens (`-`)** - underscores can cause API validation errors
- Avoid special characters: `/`, `\`, `:`, `|`, `?`, `*`, `"`, `<`, `>`
- Make each artifact name **unique** within a workflow (no duplicate names)
- Keep names concise and descriptive

**Examples**:
- ✅ `sast-results`, `sca-reports`, `dast-findings`
- ❌ `sast_results`, `sca_reports`, `dast/findings`

### Time Required

- Initial setup: 30-45 minutes
- Each practice exercise: 15-20 minutes

***

## Part 3: Project Setup - Calculator Web App

We'll create a simple calculator web application that we can scan for security issues.

### Step 3.1: Create Project Structure

Create a new folder with this structure:

```
calculator-security-demo/
├── app.py                    # Flask web application
├── requirements.txt          # Python dependencies
├── templates/
│   └── index.html           # Web interface
├── Dockerfile               # Container configuration
└── .github/
    └── workflows/
        ├── 1-sast-only.yml         # SAST scan only
        ├── 2-sca-only.yml          # SCA scan only
        ├── 3-dast-only.yml         # DAST scan only
        └── 4-complete-security.yml # All three scans
```

### Step 3.2: Create `app.py`

```python
# Simple Flask Calculator Application
from flask import Flask, render_template, request, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    """Render the calculator homepage"""
    return render_template('index.html')

@app.route('/calculate', methods=['POST'])
def calculate():
    """Perform calculation based on user input"""
    try:
        data = request.get_json()
        num1 = float(data['num1'])
        num2 = float(data['num2'])
        operation = data['operation']
        
        if operation == 'add':
            result = num1 + num2
        elif operation == 'subtract':
            result = num1 - num2
        elif operation == 'multiply':
            result = num1 * num2
        elif operation == 'divide':
            if num2 == 0:
                return jsonify({'error': 'Cannot divide by zero'}), 400
            result = num1 / num2
        else:
            return jsonify({'error': 'Invalid operation'}), 400
        
        return jsonify({'result': result})
    
    except (ValueError, KeyError) as e:
        return jsonify({'error': 'Invalid input'}), 400

@app.after_request
def add_security_headers(response):
    """Add security headers to all responses"""
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    return response

if __name__ == '__main__':
    # Note: debug=False in production
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### Step 3.3: Create `templates/index.html`

Create a `templates` folder and add `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simple Calculator</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 400px;
            margin: 50px auto;
            padding: 20px;
            background-color: #f0f0f0;
        }
        .calculator {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        input {
            width: 100%;
            padding: 10px;
            margin: 10px 0;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        button {
            width: 48%;
            padding: 10px;
            margin: 5px 1%;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            background-color: #4CAF50;
            color: white;
            font-size: 16px;
        }
        button:hover {
            background-color: #45a049;
        }
        #result {
            margin-top: 20px;
            padding: 15px;
            background-color: #e8f5e9;
            border-radius: 4px;
            font-size: 20px;
            text-align: center;
        }
        .error {
            background-color: #ffebee;
            color: #c62828;
        }
    </style>
</head>
<body>
    <div class="calculator">
        <h1>Simple Calculator</h1>
        <input type="number" id="num1" placeholder="Enter first number" step="any">
        <input type="number" id="num2" placeholder="Enter second number" step="any">
        
        <button onclick="calculate('add')">Add (+)</button>
        <button onclick="calculate('subtract')">Subtract (-)</button>
        <button onclick="calculate('multiply')">Multiply (×)</button>
        <button onclick="calculate('divide')">Divide (÷)</button>
        
        <div id="result"></div>
    </div>

    <script>
        async function calculate(operation) {
            const num1 = document.getElementById('num1').value;
            const num2 = document.getElementById('num2').value;
            const resultDiv = document.getElementById('result');
            
            if (!num1 || !num2) {
                resultDiv.className = 'error';
                resultDiv.textContent = 'Please enter both numbers';
                return;
            }
            
            try {
                const response = await fetch('/calculate', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        num1: parseFloat(num1),
                        num2: parseFloat(num2),
                        operation: operation
                    })
                });
                
                const data = await response.json();
                
                if (response.ok) {
                    resultDiv.className = '';
                    resultDiv.textContent = `Result: ${data.result}`;
                } else {
                    resultDiv.className = 'error';
                    resultDiv.textContent = `Error: ${data.error}`;
                }
            } catch (error) {
                resultDiv.className = 'error';
                resultDiv.textContent = 'Connection error';
            }
        }
    </script>
</body>
</html>
```

### Step 3.4: Create `requirements.txt`

```txt
flask==2.0.1
werkzeug==2.0.1
```

**Note:** These are intentionally older versions with known vulnerabilities so we can see the scanners find them!

### Step 3.5: Create `Dockerfile`

```dockerfile
# Use Python 3.9 slim image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY app.py .
COPY templates/ templates/

# Expose port 5000
EXPOSE 5000

# Run the application
CMD ["python", "app.py"]
```

### Step 3.6: Initialize Git Repository

```bash
# Navigate to your project folder
cd calculator-security-demo

# Initialize git
git init

# Create .gitignore
echo "__pycache__/" > .gitignore
echo "*.pyc" >> .gitignore
echo ".pytest_cache/" >> .gitignore

# Add all files
git add .

# First commit
git commit -m "Initial commit: Calculator web app"
```

***

## Part 4: SAST Setup - CodeQL Code Scanning

### What Is SAST?

**SAST (Static Application Security Testing)** analyzes your source code to find security vulnerabilities without running the application.

**CodeQL** will scan for:

- Code injection vulnerabilities
- Hardcoded secrets and sensitive data
- Insecure configurations
- Insecure data flow patterns
- OWASP Top 10 vulnerabilities

### Why CodeQL Instead of External Tools?

| Feature | CodeQL | External SAST Tools |
| :-- | :-- | :-- |
| **Cost** | Free for public repos | Often requires paid subscription |
| **Setup** | No external tokens needed | Requires account setup and API keys |
| **Integration** | Native GitHub | Third-party service |
| **Speed** | 2-5 minutes | Variable, often slower |
| **Maintenance** | GitHub-maintained | Vendor-maintained |
| **Analysis Type** | Semantic (code structure) | Rules-based or ML-based |

### Step 4.1: Enable Code Scanning

1. Go to your GitHub repository
2. Click **Security** tab → **Code scanning alerts**
3. Click **Set up code scanning** → **Set up with CodeQL**
4. Select **Advanced** (for customization)

### Step 4.2: Create SAST Workflow with CodeQL

Create `.github/workflows/1-sast-only.yml`:

```yaml
# SAST (Static Application Security Testing) with CodeQL
name: 1. SAST - Code Security Scan

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

permissions:
  contents: read
  security-events: write  # Required for SARIF upload

jobs:
  analyze:
    name: CodeQL Code Scan (SAST)
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        language: [ 'python' ]
    
    steps:
      # Step 1: Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # Step 2: Initialize CodeQL
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended
      
      # Step 3: Set up Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      # Step 4: Install dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
      
      # Step 5: Perform CodeQL Analysis
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: sast
```

### Understanding the SAST Workflow

**Trigger (`on:`)**: Runs on push and pull requests to main/master

**Key Steps**:

1. **Checkout code**: Downloads your repository
2. **Initialize CodeQL**: Sets up the analysis engine
3. **Set up Python**: Installs Python environment
4. **Install dependencies**: Installs packages from requirements.txt
5. **Perform Analysis**: Analyzes code for vulnerabilities

**Important Configurations**:

- `languages: [ 'python' ]` → Specify languages to scan
- `queries: security-extended` → Uses comprehensive security rules
- `category: sast` → Categorizes results in GitHub Security tab

### CodeQL Query Suites Available

- `security-and-quality` → Security and code quality checks
- `security-extended` → Extended security queries (recommended)
- `security-minimal` → Fast, minimal security checks

***

## Part 5: SCA Setup - Dependency Checking

### What Is SCA?

**SCA (Software Composition Analysis)** checks your project dependencies (libraries and packages) for known security vulnerabilities.

**OWASP Dependency-Check** will scan for:

- Outdated dependencies with CVEs
- Known vulnerable packages
- License compliance issues

### Step 5.1: No Additional Accounts Needed!

OWASP Dependency-Check is completely free and doesn't require an account or token.

### Step 5.2: Create SCA Workflow

Create `.github/workflows/2-sca-only.yml`:

```yaml
# SCA (Software Composition Analysis) with OWASP Dependency-Check
name: 2. SCA - Dependency Security Scan

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

permissions:
  contents: read
  security-events: write

jobs:
  sca:
    name: OWASP Dependency Check (SCA)
    runs-on: ubuntu-latest
    
    steps:
      # Step 1: Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # Step 2: Set up Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      # Step 3: Install dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      # Step 4: Run OWASP Dependency-Check
      - name: Run OWASP Dependency-Check
        uses: dependency-check/Dependency-Check_Action@main
        id: depcheck
        with:
          project: 'calculator-app'
          path: '.'
          format: 'ALL'  # Generate HTML, JSON, CSV, XML
          out: 'reports'
          args: >
            --failOnCVSS 7
            --enableRetired
      
      # Step 5: Upload Dependency-Check reports
      # ✅ FIXED: Using proper artifact naming (hyphens, not underscores)
      - name: Upload Dependency-Check reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sca-reports
          path: reports/
      
      # Step 6: Upload to GitHub Security (if SARIF available)
      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always() && hashFiles('reports/dependency-check-report.sarif') != ''
        with:
          sarif_file: reports/dependency-check-report.sarif
          category: sca
```

### Understanding the SCA Workflow

**Key Configurations**:

- `project: 'calculator-app'` → Your project name
- `path: '.'` → Scan current directory
- `format: 'ALL'` → Generate multiple report formats
- `--failOnCVSS 7` → Fail if vulnerabilities have severity ≥7 (HIGH)
- `--enableRetired` → Include retired vulnerabilities
- `name: sca-reports` → Safe artifact name (hyphens, not underscores)

**What Gets Scanned**:

- Python packages in `requirements.txt`
- Any JAR files, JavaScript, Ruby gems, etc.
- Transitive dependencies (dependencies of dependencies)

***

## Part 6: DAST Setup - Live Application Scanning

### What Is DAST?

**DAST (Dynamic Application Security Testing)** tests your **running application** by simulating real attacks.

**OWASP ZAP (Zed Attack Proxy)** will test for:

- SQL injection
- Cross-site scripting (XSS)
- Security misconfigurations
- Broken authentication

### Step 6.1: No Account Needed!

OWASP ZAP is completely free and open-source.

### Step 6.2: Create DAST Workflow

Create `.github/workflows/3-dast-only.yml`:

```yaml
# DAST (Dynamic Application Security Testing) with OWASP ZAP
name: 3. DAST - Live Application Scan

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

permissions:
  contents: read
  security-events: write
  issues: write  # For ZAP to create issues

jobs:
  dast:
    name: OWASP ZAP Scan (DAST)
    runs-on: ubuntu-latest
    
    steps:
      # Step 1: Checkout code
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # Step 2: Set up Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      # Step 3: Install dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      # Step 4: Start the application in background
      - name: Start Flask application
        run: |
          python app.py &
          sleep 10  # Wait for app to start
          echo "Application started on http://localhost:5000"
      
      # Step 5: Verify application is running
      - name: Verify application is running
        run: |
          curl -f http://localhost:5000 || exit 1
          echo "Application is responding"
      
      # Step 6: Run OWASP ZAP Baseline Scan
      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'http://localhost:5000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'  # Include alpha rules
          allow_issue_writing: false
      
      # Step 7: Run OWASP ZAP Full Scan (more thorough)
      - name: ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.10.0
        with:
          target: 'http://localhost:5000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a -j'  # Include AJAX spider
          allow_issue_writing: false
      
      # ✅ FIXED: Proper artifact naming (hyphens, not underscores)
      # ✅ FIXED: Unique artifact names to avoid conflicts
      - name: Upload ZAP baseline reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-baseline-reports
          path: report_html.html
      
      - name: Upload ZAP fullscan reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-fullscan-reports
          path: |
            report_json.json
            report_md.md
```

### Artifact Naming Fixes Explained

The original error occurred due to improper artifact naming:

- ❌ **Original**: `name: zap_scan` - Contains underscore that triggers API validation errors
- ✅ **Fixed**: `name: zap-baseline-reports` - Uses hyphens and is descriptive

**Key fixes applied**:

1. **Replaced underscores with hyphens**: `zap_scan` → `zap-baseline-reports`
2. **Created unique names**: `zap-baseline-reports` and `zap-fullscan-reports` instead of duplicate names
3. **Made names descriptive**: Clearly indicates which scan generated the reports

### Step 6.3: Create ZAP Rules File (Optional)

Create `.zap/rules.tsv` to customize ZAP scanning:

```tsv
10035	IGNORE	(Strict-Transport-Security Header)
10063	IGNORE	(Feature Policy Header)
```

This tells ZAP to ignore certain warnings that may not be relevant for a demo app.

### Understanding the DAST Workflow

**The Process**:

1. **Start the app**: Runs Flask application in background
2. **Wait**: Gives app time to fully start (10 seconds)
3. **Verify**: Checks app is accessible
4. **Baseline scan**: Quick passive scan (5-10 minutes)
5. **Full scan**: Deep active scan (15-30 minutes)
6. **Upload reports**: Saves scan results with proper naming

**Scan Types**:

- **Baseline**: Passive scan, no attacks, fast
- **Full**: Active attacks, thorough, slower

***

## Part 7: Complete Integrated Workflow

Now let's combine all three scans into one comprehensive security pipeline with proper artifact naming!

### Step 7.1: Create Complete Workflow

Create `.github/workflows/4-complete-security.yml`:

```yaml
# Complete Security Pipeline: SAST + SCA + DAST
name: 4. Complete Security Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

permissions:
  contents: read
  security-events: write
  issues: write

jobs:
  # JOB 1: SAST - Static Code Analysis
  sast:
    name: "1️⃣ SAST - Code Analysis"
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        language: [ 'python' ]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Perform CodeQL Analysis (SAST)
        uses: github/codeql-action/analyze@v3
        with:
          category: sast

  # JOB 2: SCA - Dependency Analysis
  sca:
    name: "2️⃣ SCA - Dependency Check"
    runs-on: ubuntu-latest
    needs: sast  # Run after SAST
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Run OWASP Dependency-Check (SCA)
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'calculator-app'
          path: '.'
          format: 'ALL'
          out: 'reports'
          args: >
            --failOnCVSS 7
            --enableRetired
      
      # ✅ FIXED: Proper artifact naming
      - name: Upload SCA reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sca-reports
          path: reports/

  # JOB 3: Build Application
  build:
    name: "3️⃣ Build Application"
    runs-on: ubuntu-latest
    needs: [sast, sca]  # Run after SAST and SCA pass
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Test application starts
        run: |
          python app.py &
          sleep 5
          curl -f http://localhost:5000 || exit 1
          echo "✅ Application builds and runs successfully"

  # JOB 4: DAST - Dynamic Application Testing
  dast:
    name: "4️⃣ DAST - Live Security Scan"
    runs-on: ubuntu-latest
    needs: build  # Run after successful build
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Start application
        run: |
          python app.py &
          sleep 10
          echo "Application running on http://localhost:5000"
      
      - name: Verify application
        run: curl -f http://localhost:5000
      
      - name: Run ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a'
          allow_issue_writing: false
      
      - name: Run ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.10.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a -j'
          allow_issue_writing: false
      
      # ✅ FIXED: Proper artifact naming with hyphens and unique names
      - name: Upload ZAP baseline reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-baseline-reports
          path: report_html.html
      
      - name: Upload ZAP fullscan reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-fullscan-reports
          path: |
            report_json.json
            report_md.md

  # JOB 5: Security Summary
  summary:
    name: "📊 Security Summary"
    runs-on: ubuntu-latest
    needs: [sast, sca, dast]
    if: always()
    
    steps:
      - name: Generate Security Summary
        run: |
          echo "# 🔒 Security Scan Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "## Scan Results" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ SAST (CodeQL Analysis): Completed" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ SCA (Dependency Check): Completed" >> $GITHUB_STEP_SUMMARY
          echo "- ✅ DAST (Dynamic Testing): Completed" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "## Artifacts Generated" >> $GITHUB_STEP_SUMMARY
          echo "- CodeQL results in Security tab" >> $GITHUB_STEP_SUMMARY
          echo "- SCA reports: sca-reports" >> $GITHUB_STEP_SUMMARY
          echo "- DAST baseline: zap-baseline-reports" >> $GITHUB_STEP_SUMMARY
          echo "- DAST fullscan: zap-fullscan-reports" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "📁 Check the **Actions** tab for detailed reports" >> $GITHUB_STEP_SUMMARY
          echo "🔍 Check the **Security** tab for vulnerability details" >> $GITHUB_STEP_SUMMARY
```

### Understanding the Complete Workflow

**Job Execution Order**:

```
1. SAST (CodeQL) → Runs first
         ↓
2. SCA (Dependency scan) → Runs after SAST
         ↓
3. Build (Verify app works) → Runs after both pass
         ↓
4. DAST (Live scan) → Runs after successful build
         ↓
5. Summary → Generates report with artifact names
```

**Artifact Naming Summary**:

- `sca-reports` - Dependency check findings
- `zap-baseline-reports` - ZAP passive scan results
- `zap-fullscan-reports` - ZAP active scan results

All names follow the safe naming convention: alphanumeric + hyphens, no underscores or special characters.

***

## Part 8: Testing Everything

### Step 8.1: Push to GitHub

```bash
# Create repository on GitHub first
# Then connect your local repo

# Add remote (replace with your repo URL)
git remote add origin https://github.com/YOUR_USERNAME/calculator-security-demo.git

# Push all files
git add .
git commit -m "Add complete security scanning pipeline with CodeQL and fixed artifact naming"
git push -u origin main
```

### Step 8.2: Watch Workflows Run

1. Go to your repository on GitHub
2. Click **Actions** tab
3. You should see workflows running:
    - `1. SAST - Code Security Scan`
    - `2. SCA - Dependency Security Scan`
    - `3. DAST - Live Application Scan`
    - `4. Complete Security Pipeline`

### Step 8.3: Check Workflow Progress

Click on any running workflow to see:

- Real-time logs
- Each step's progress
- Green checkmarks ✅ or red Xs ❌
- Artifact uploads with proper naming

**Expected Timeline**:

- SAST (CodeQL): 2-5 minutes
- SCA: 3-5 minutes
- DAST Baseline: 5-10 minutes
- DAST Full: 15-30 minutes
- Complete Pipeline: 25-45 minutes total

### Step 8.4: Verify Artifact Names

1. Go to completed workflow run
2. Scroll to **Artifacts** section
3. Verify artifact names show:
    - ✅ `sca-reports`
    - ✅ `zap-baseline-reports`
    - ✅ `zap-fullscan-reports`
    - ❌ No underscores or invalid characters

***

## Part 9: Understanding Results

### Where to Find Results

#### 1. Actions Tab

**Path**: Repository → Actions → Select workflow run

**What you see**:

- Workflow execution logs
- Step-by-step progress
- Error messages and warnings
- Downloaded artifacts with proper naming

**How to download reports**:

1. Go to completed workflow run
2. Scroll to bottom "Artifacts" section
3. Click artifact names to download (e.g., `sca-reports`, `zap-baseline-reports`)

#### 2. Security Tab

**Path**: Repository → Security → Code scanning alerts

**What you see**:

- Organized vulnerability list
- Severity ratings
- Affected files and lines
- Remediation suggestions

**Filter by**:

- Severity (Critical, High, Medium, Low)
- Status (Open, Fixed, Dismissed)
- Tool (CodeQL, Dependency-Check, ZAP)

### Reading Vulnerability Reports

#### Severity Levels

| Severity | CVSS Score | Action Required | Example |
| :-- | :-- | :-- | :-- |
| 🔴 **Critical** | 9.0-10.0 | Fix immediately | Remote code execution |
| 🟠 **High** | 7.0-8.9 | Fix within days | SQL injection |
| 🟡 **Medium** | 4.0-6.9 | Fix within weeks | Information disclosure |
| 🔵 **Low** | 0.1-3.9 | Monitor, fix when possible | Minor config issue |

#### CodeQL Report Example

```
❌ High Severity: SQL Injection Detected
File: app.py
Line: 42
Issue: User input concatenated directly into SQL query

Recommendation: Use parameterized queries
Fix: Replace string concatenation with prepared statements
```

#### SCA Report Example

```
❌ Critical: Known Vulnerability in flask
Package: flask
Current Version: 2.0.1
Vulnerable: Yes (CVE-2023-XXXXX)
Fixed Version: 2.3.0

Recommendation: Upgrade to flask==2.3.0
Impact: Remote code execution possible
```

#### DAST Report Example

```
❌ Medium: Missing Security Header
URL: http://localhost:5000
Issue: X-Content-Type-Options header not set

Recommendation: Add security headers to responses
Fix: Header already added in app.py!
```

***

## Part 10: Troubleshooting

### Common Issues and Solutions

#### Issue 1: Artifact Name Not Valid Error

**Symptoms**:

```
Error: Create Artifact Container failed: The artifact name zap_scan is not valid
```

**Solution**:

1. Replace underscores (`_`) with hyphens (`-`)
2. Remove special characters: `/`, `\`, `:`, `|`, `?`, `*`, `"`, `<`, `>`
3. Make sure each artifact has a unique name (no duplicates)
4. Use descriptive alphanumeric names

**Example fix**:

```yaml
# ❌ Wrong
- name: Upload ZAP reports
  with:
    name: zap_scan

# ✅ Correct
- name: Upload ZAP baseline reports
  with:
    name: zap-baseline-reports

- name: Upload ZAP fullscan reports
  with:
    name: zap-fullscan-reports
```

#### Issue 2: CodeQL Analysis Takes Too Long

**Symptoms**:

- CodeQL step runs for 10+ minutes
- May timeout on free GitHub Actions runners

**Solution**:

1. CodeQL builds a database for each scan - this is normal
2. Subsequent scans reuse the database (faster)
3. Consider adjusting query suite if time is critical:

```yaml
with:
  queries: security-minimal  # Faster, less comprehensive
```

#### Issue 3: SCA Takes Too Long

**Symptoms**:

- Workflow runs for 30+ minutes
- "Downloading NVD database" step is slow

**Solution**:

This is normal for first run! The database is large (~500MB).

- Subsequent runs use cached database (much faster)
- Consider running SCA on schedule instead of every push:

```yaml
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
```

#### Issue 4: DAST Can't Connect to Application

**Symptoms**:

```
Error: Connection refused to http://localhost:5000
```

**Solution**:

1. Increase sleep time after starting app:

```yaml
- name: Start application
  run: |
    python app.py &
    sleep 15  # Increase from 10 to 15
```

2. Add better health check:

```yaml
- name: Wait for application
  run: |
    for i in {1..30}; do
      curl -f http://localhost:5000 && break
      sleep 1
    done
```

#### Issue 5: Workflow Shows "Queued" Forever

**Symptoms**:

- Workflow stuck in "Queued" state
- No logs appearing

**Solution**:

- Check GitHub Actions status page
- Free accounts have limited concurrent jobs (wait for others to finish)
- Consider upgrading GitHub plan for more runners

#### Issue 6: Too Many False Positives

**Symptoms**:

- Hundreds of low-severity issues
- Many irrelevant warnings

**Solution**:

1. Adjust severity thresholds:

```yaml
# SCA - only fail on critical
args: --failOnCVSS 9
```

2. Create suppression files for known false positives

#### Issue 7: Workflow Permissions Error

**Symptoms**:

```
Error: Resource not accessible by integration
```

**Solution**:

Add required permissions to workflow:

```yaml
permissions:
  contents: read
  security-events: write
  issues: write
```

***

## Part 11: Practice Exercises (Optional)

### Exercise 1: Basic Setup (30 minutes)

**Goal**: Get all three scans running with proper artifact naming

**Steps**:

1. ✅ Create calculator project with all files
2. ✅ Create all four workflow files with proper naming
3. ✅ Push to GitHub
4. ✅ Verify all workflows run successfully
5. ✅ Check that artifacts appear with safe names

**Success Criteria**:

- All workflows appear in Actions tab
- Artifacts section shows: `sca-reports`, `zap-baseline-reports`, `zap-fullscan-reports`
- No "artifact name not valid" errors

### Exercise 2: Fix Vulnerabilities (45 minutes)

**Goal**: Remediate found vulnerabilities

**Steps**:

1. Run complete security pipeline
2. Review Security tab for vulnerabilities
3. Fix at least 3 issues:
    - Update Flask to latest version
    - Update Werkzeug to latest version
    - Verify security headers are present

**Fix for security headers** (already in `app.py`):

```python
@app.after_request
def add_security_headers(response):
    """Add security headers to all responses"""
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    return response
```

**Success Criteria**:

- Fewer vulnerabilities in next scan
- Understanding how to read and fix issues

### Exercise 3: Customize Scan Settings (30 minutes)

**Goal**: Optimize CodeQL scanning for your needs

**Tasks**:

1. Modify SAST to use different query suites:

```yaml
with:
  queries: security-minimal  # Fast scan
  # or: security-extended   # Comprehensive scan
```

2. Modify SCA to fail on CVSS ≥9:

```yaml
with:
  args: --failOnCVSS 9
```

3. Add CodeQL schedule to run daily:

```yaml
on:
  schedule:
    - cron: '0 3 * * *'  # 3 AM daily
  workflow_dispatch:  # Manual trigger
```

**Success Criteria**:

- Workflows adjust behavior based on configuration
- Understanding of CodeQL options

### Exercise 4: Rename Artifacts (20 minutes)

**Goal**: Practice safe artifact naming

**Tasks**:

1. Identify any artifacts using underscores
2. Create new versions with proper naming:
   - Replace `_` with `-`
   - Ensure unique names
   - Test the workflow

**Examples to rename**:

```yaml
# ❌ Old naming
- name: sast_results
- name: sca_findings
- name: dast_scan

# ✅ New naming
- name: sast-results
- name: sca-findings
- name: dast-scan
```

**Success Criteria**:

- All artifacts upload successfully
- No "artifact name not valid" errors
- Clear naming convention applied

### Exercise 5: Create Security Report (30 minutes)

**Goal**: Document findings and remediations

**Steps**:

1. Download all scan reports from Actions artifacts
2. Create a `SECURITY_FINDINGS.md` document with:
    - Summary of vulnerabilities found
    - Actions taken to fix them
    - Remaining issues and justification
3. Commit and push to repository

**Template**:

```markdown
# Security Scan Results - [Date]

## Summary
- SAST (CodeQL): X issues found
- SCA: Y vulnerabilities detected
- DAST: Z security concerns identified

## Critical Findings
1. [Issue name] - Status: Fixed/In Progress/Accepted Risk

## Remediation Actions
1. Updated Flask from 2.0.1 → 2.3.2
2. Added security headers (X-Content-Type-Options, etc.)
3. [Additional fixes]

## Artifacts Used
- CodeQL results (from Security tab)
- sca-reports
- zap-baseline-reports
- zap-fullscan-reports
```

***

## Summary and Next Steps

### What You've Learned

✅ **SAST with CodeQL**: Scanning source code for vulnerabilities
✅ **SCA**: Checking dependencies for known issues
✅ **DAST with ZAP**: Testing running applications for security flaws
✅ **Proper artifact naming**: Avoiding GitHub Actions validation errors
✅ **Security Reporting**: Reading and understanding scan results

### Key Takeaways on Artifact Naming

- **Use hyphens, not underscores**: `zap-reports` not `zap_reports`
- **Make names unique**: No duplicate artifact names in same workflow
- **Keep it simple**: Alphanumeric + hyphens only
- **Be descriptive**: `zap-baseline-reports` is clearer than `reports`

### Best Practices Checklist

- [ ] Run all three scan types (SAST + SCA + DAST)
- [ ] Use proper artifact naming (hyphens, no underscores)
- [ ] Scan on every push and pull request
- [ ] Review Security tab weekly
- [ ] Fix critical/high issues within days
- [ ] Keep dependencies updated
- [ ] Don't disable scans to "make builds pass"
- [ ] Document security decisions
- [ ] Educate team on findings

### Next Steps

**Beginner → Intermediate**:

1. Add container scanning (Docker image scan with Trivy)
2. Implement security policies (branch protection rules)
3. Set up scheduled scans for off-peak hours
4. Integrate with Slack for notifications

**Intermediate → Advanced**:

1. Custom CodeQL queries for domain-specific issues
2. Integration with JIRA for ticket creation
3. Security testing in staging environments
4. Compliance reporting (SOC2, PCI-DSS)

### Additional Resources

**Tools**:

- [GitHub CodeQL Documentation](https://codeql.github.com/docs/)
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [GitHub Actions Upload Artifact](https://github.com/actions/upload-artifact)
- [GitHub Security Features](https://docs.github.com/en/code-security)

**Learning**:

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [GitHub Security Lab](https://securitylab.github.com/)
- [CodeQL Learning](https://codeql.github.com/learn/)

***

## Quick Reference Card

### Scan Type Comparison

| Feature | SAST (CodeQL) | SCA (Dep-Check) | DAST (ZAP) |
| :-- | :-- | :-- | :-- |
| **What it scans** | Source code | Dependencies | Running app |
| **Requires token** | No | No | No |
| **Speed** | 2-5 min | 3-5 min | 15-30 min |
| **When to run** | Every push | Daily/weekly | After deployment |
| **Best for finding** | Code flaws | Outdated libs | Runtime issues |
| **Cost** | Free | Free | Free |
| **Setup time** | 5 minutes | 5 minutes | 10 minutes |

### Safe Artifact Naming Rules

| Rule | Safe | Unsafe |
| :-- | :-- | :-- |
| **Characters** | a-z, 0-9, `-` | `_`, `/`, `\`, `:`, `\|`, `?`, `*`, `"`, `<`, `>` |
| **Underscore** | `zap-reports` | `zap_reports` |
| **Multiple files** | Each unique name | Same name twice |
| **Length** | `sca-reports` | `my-security-scan-action-reports-from-job-123` |
| **Clarity** | `sast-codeql-results` | `artifact1`, `data`, `results` |

### CodeQL vs Other SAST Tools

| Aspect | CodeQL | Other SAST |
| :-- | :-- | :-- |
| **Cost** | Free (public repos) | Often paid |
| **Setup** | No external token | Requires account + API key |
| **Analysis** | Semantic/structural | Rules-based or ML |
| **Integration** | Native GitHub | External service |
| **Speed** | 2-5 minutes | Variable |
| **False Positives** | 5-8% | 8-15% |
| **Best for** | Data flow bugs | Quick inline scanning |

### Workflow Trigger Options

```yaml
# On every push
on: push

# On pull requests only
on: pull_request

# Daily at 2 AM UTC
on:
  schedule:
    - cron: '0 2 * * *'

# Manual trigger
on: workflow_dispatch

# Multiple triggers
on:
  push:
    branches: [ main ]
  pull_request:
  schedule:
    - cron: '0 2 * * *'
```

***

**Remember**: Security is a journey, not a destination. Start with these basics and gradually improve your security posture over time! Always use safe artifact naming to avoid unnecessary workflow failures.

<div align="center">⁂</div>
