# GitHub Actions CI/CD Installation & Self-Hosted Runner Guide (Linux)

This document provides the complete, tested, step-by-step guide for installing, configuring, and executing a **GitHub Actions Self-Hosted Runner** on **Ubuntu Linux (22.04 / 24.04 LTS)** for the COBOL training repository using SSH authentication.

---

## 🏗️ Architecture Overview

```text
┌─────────────────────────────────────────────────────────────┐
│                       GitHub Cloud                          │
│                                                             │
│   1. Git Push / PR (via SSH) ──► Repository Remote          │
│   2. Event Trigger           ──► .github/workflows/*.yml    │
│   3. Job Dispatch            ──► Polled via Outbound HTTPS  │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS Long-Poll (Port 443)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│            Local Linux Machine (/home/trainees)             │
│                                                             │
│   actions-runner Daemon (systemd service)                   │
│   ├── Checks out repository                                 │
│   ├── Validates GnuCOBOL syntax (cobc -fsyntax-only)        │
│   ├── Builds native binary & C-bridge (cobc + gcc)          │
│   ├── Executes automated test harness & SQLite validation   │
│   └── Uploads compiled binary artifact to GitHub            │
└─────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> **Network & Protocol Separation**:
> - **Git operations (push/pull/fetch)** use **SSH** (`git@github.com:dev-paplix/cobol-training-day2.git`) authenticated with your local SSH key pair (`~/.ssh/id_rsa`).
> - **Runner communication with GitHub Actions** uses **Outbound HTTPS (port 443)**. No incoming firewall ports need to be opened on your machine.

---

## 📋 Prerequisites & Toolchain Verification

Before installing the runner, verify that the required build tools, compilers, and utilities are present.

### Step 1: Install Required Packages

```bash
sudo apt update && sudo apt install -y \
  curl \
  tar \
  jq \
  git \
  build-essential \
  gcc \
  make \
  gnucobol \
  libcob4-dev \
  sqlite3 \
  libsqlite3-dev \
  gh
```

### Step 2: Diagnostic Check

Verify all components with a single check:

```bash
echo "=== Toolchain Verification ==="
cobc --version | head -n 2
gcc --version | head -n 1
sqlite3 --version
make --version | head -n 1
git --version
```

*Expected Output:*
```text
=== Toolchain Verification ===
cobc (GnuCOBOL) 3.1.2.0
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
3.37.2 2022-01-06 13:25:41 ...
GNU Make 4.3
git version 2.34.1
```

---

## 🔑 Step 1: Verify SSH Connectivity with GitHub

Verify that your Linux machine is authenticated with GitHub using SSH:

```bash
ssh -T -o StrictHostKeyChecking=accept-new git@github.com
```

*Expected Output:*
```text
Hi dev-paplix! You've successfully authenticated, but GitHub does not provide shell access.
```

If you need to generate or view your SSH public key:
```bash
# Generate key (if not already done)
ssh-keygen -t rsa -b 4096 -C "trainees@cobol-training" -N ""

# Display public key to add in GitHub -> Settings -> SSH and GPG keys
cat /home/trainees/.ssh/id_rsa.pub
```

Verify that your repository remote is configured to use SSH:
```bash
cd /home/trainees/codes/cobol-training
git remote -v
```

If the remote is HTTPS, switch it to SSH:
```bash
git remote set-url origin git@github.com:dev-paplix/cobol-training-day2.git
```

---

## 📦 Step 2: Download & Extract GitHub Actions Runner

Create a dedicated runner directory and download the official GitHub Actions runner package:

```bash
# 1. Create runner directory in user home
mkdir -p /home/trainees/actions-runner
cd /home/trainees/actions-runner

# 2. Download the runner package (v2.322.0 for Linux x64)
curl -o actions-runner-linux-x64-2.322.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-linux-x64-2.322.0.tar.gz

# 3. Extract the archive
tar xzf ./actions-runner-linux-x64-2.322.0.tar.gz

# 4. Install runner system dependencies
sudo ./bin/installdependencies.sh
```

---

## 🎫 Step 3: Obtain Runner Registration Token from GitHub

To connect your local runner to the repository, you need a one-time registration token from GitHub:

1. Open your browser and navigate to your repository:
   `https://github.com/dev-paplix/cobol-training-day2`
2. Click **Settings** (top navigation bar).
3. In the left sidebar under **Code and automation**, click **Actions** ➔ **Runners**.
4. Click the green **New self-hosted runner** button.
5. Select **Runner image: Linux** and **Architecture: x64**.
6. Under **Configure**, copy the token shown after `--token`:
   ```bash
   ./config.sh --url https://github.com/dev-paplix/cobol-training-day2 --token <YOUR_REGISTRATION_TOKEN>
   ```

> [!IMPORTANT]
> Registration tokens expire after **1 hour** if unused. If it expires before you run `config.sh`, refresh the GitHub page to generate a fresh token.

---

## ⚙️ Step 4: Configure and Register the Runner

Run the configuration script from the runner directory:

### Option A: Interactive Configuration
```bash
cd /home/trainees/actions-runner
./config.sh --url https://github.com/dev-paplix/cobol-training-day2 --token <YOUR_REGISTRATION_TOKEN>
```

When prompted:
1. **Runner name**: Press Enter for default (`linux-runner`) or type a custom name.
2. **Runner group**: Press Enter for default (`Default`).
3. **Labels**: Enter `self-hosted,linux,x64,ubuntu-latest`
4. **Work folder**: Press Enter for default (`_work`).

### Option B: Automated Non-Interactive (Unattended)
```bash
cd /home/trainees/actions-runner
./config.sh --unattended \
  --url https://github.com/dev-paplix/cobol-training-day2 \
  --token <YOUR_REGISTRATION_TOKEN> \
  --name "linux-cobol-runner" \
  --labels "self-hosted,linux,x64,ubuntu-latest" \
  --work "_work" \
  --replace
```

> [!TIP]
> Adding the `ubuntu-latest` label allows your self-hosted runner to automatically run jobs that declare `runs-on: ubuntu-latest` as well as jobs that declare `runs-on: [self-hosted, linux]`.

---

## 🚀 Step 5: Start the Runner

### Mode A: Interactive Foreground Execution (For Immediate Testing)
```bash
cd /home/trainees/actions-runner
./run.sh
```

*Expected Terminal Output:*
```text
√ Connected to GitHub
Current runner version: '2.322.0'
2026-09-09 12:55:00Z: Listening for Jobs
```
*(Press `Ctrl+C` when you want to stop testing).*

### Mode B: Systemd Background Service (Recommended for Continuous Operation)
Install the runner as a background systemd service so it runs automatically and restarts on reboot:

```bash
cd /home/trainees/actions-runner

# Install as systemd service for user 'trainees'
sudo ./svc.sh install trainees

# Start the service
sudo ./svc.sh start

# Check service status
sudo ./svc.sh status
```

*Useful service control commands:*
```bash
# Stop service
sudo ./svc.sh stop

# Restart service
sudo ./svc.sh restart

# Uninstall service
sudo ./svc.sh uninstall
```

---

## 📝 Step 6: Activate the CI/CD Workflow in Repository

Ensure the GitHub Actions workflow file is created in `.github/workflows/cobol-ci.yml`:

```bash
cd /home/trainees/codes/cobol-training
mkdir -p .github/workflows
```

The workflow file [`.github/workflows/cobol-ci.yml`](file:///home/trainees/codes/cobol-training/.github/workflows/cobol-ci.yml):

```yaml
name: COBOL & SQLite CI/CD Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build, Lint & Test COBOL Application
    runs-on: [self-hosted, linux]

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Verify Toolchain Dependencies
        run: |
          echo "=== Checking Toolchain Installation ==="
          if ! command -v cobc &>/dev/null || ! command -v sqlite3 &>/dev/null; then
            echo "Installing missing toolchain dependencies..."
            sudo apt-get update
            sudo apt-get install -y gnucobol libcob4-dev sqlite3 libsqlite3-dev build-essential gcc make
          else
            echo "GnuCOBOL and SQLite3 are already installed."
          fi

      - name: Check Toolchain Versions
        run: |
          echo "=== Compiler Diagnostic Info ==="
          cobc --version | head -n 2
          gcc --version | head -n 1
          sqlite3 --version

      - name: Syntax Linting & Check
        run: |
          echo "Running GnuCOBOL syntax validation..."
          make -C module_8 check

      - name: Build Native Binary
        run: |
          echo "Compiling COBOL application with SQLite C-bridge..."
          make -C module_8 build

      - name: Run Automated Test Harness
        run: |
          echo "Executing automated test suite..."
          make -C module_8 test

      - name: Archive Compiled Binary Artifact
        uses: actions/upload-artifact@v4
        if: success()
        with:
          name: inventory-app-linux-x64
          path: module_8/bin/inventory_app
          retention-days: 7
```

Commit and push using SSH:
```bash
cd /home/trainees/codes/cobol-training
git add .github/workflows/cobol-ci.yml
git commit -m "ci: activate GitHub Actions CI/CD pipeline for COBOL & SQLite"
git push -u origin main
```

---

## 🔍 Step 7: Verify CI/CD Pipeline Execution

1. Open your repository on GitHub: `https://github.com/dev-paplix/cobol-training-day2/actions`
2. You will see the workflow **`COBOL & SQLite CI/CD Pipeline`** triggered by your push.
3. Click on the active workflow run.
4. Your self-hosted runner will pick up the job and execute the steps:
   - `Checkout Repository`
   - `Verify Toolchain Dependencies`
   - `Check Toolchain Versions`
   - `Syntax Linting & Check` (`make check`)
   - `Build Native Binary` (`make build`)
   - `Run Automated Test Harness` (`make test`)
   - `Archive Compiled Binary Artifact` (`actions/upload-artifact@v4`)
5. Upon completion, a green checkmark will appear and the compiled binary artifact `inventory-app-linux-x64` will be available for download on the summary page.

---

## 🛠️ Maintenance & Troubleshooting

| Issue | Cause | Resolution |
| :--- | :--- | :--- |
| `Cannot find runner token` | Token expired after 1 hr | Go to repository Settings ➔ Actions ➔ Runners ➔ New self-hosted runner and obtain a new token. |
| `Permission denied (publickey)` | SSH key not added to GitHub account | Run `cat ~/.ssh/id_rsa.pub` and add it to your GitHub account under **Settings** ➔ **SSH and GPG Keys**. |
| Runner shows `Offline` on GitHub | Service stopped or process killed | Run `sudo ./svc.sh start` or launch `./run.sh` from `/home/trainees/actions-runner`. |
| Job stays in `Queued` state | `runs-on` labels do not match runner labels | Ensure `.github/workflows/*.yml` specifies `runs-on: [self-hosted, linux]` or register the runner with `--labels ubuntu-latest`. |
| Remove/Deregister runner | Decommissioning or re-registering | Run `./config.sh remove --token <TOKEN_FROM_GITHUB>`. |

