# GitHub Actions CI/CD Installation & Self-Hosted Runner Reference Guide (Windows)

This document serves as a complete reference guide for installing, configuring, and operating a **GitHub Actions Self-Hosted Runner** on a **Windows (Windows 10 / 11 / Windows Server)** machine for the COBOL training repository using SSH authentication.

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
│          Local Windows Machine (C:\Users\trainees)          │
│                                                             │
│   actions-runner Windows Service (svc.cmd)                  │
│   ├── Git for Windows / PowerShell Shell Host               │
│   ├── Checks out repository                                 │
│   ├── Validates GnuCOBOL syntax (cobc.exe)                  │
│   ├── Builds native Windows PE binaries (.exe / .dll)       │
│   └── Executes automated test harnesses                     │
└─────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> **Port & Security Protocol**:
> - Like Linux, the Windows runner connects to GitHub via **Outbound HTTPS (Port 443)** using TLS. No incoming firewall ports or router forwarding are needed.
> - Git pushes and pulls authenticate using **OpenSSH** (`git@github.com:dev-paplix/cobol-training-day2.git`).

---

## 📋 Prerequisites on Windows

Before configuring the runner, ensure the following Windows software packages are installed:

1. **PowerShell 5.1+ or PowerShell 7+** (Built-in or installed via Winget).
2. **Git for Windows** (includes Git Bash and OpenSSH client).
   ```powershell
   winget install --id Git.Git -e --source winget
   ```
3. **GnuCOBOL for Windows** (via MSYS2/MinGW or pre-built Windows GnuCOBOL binaries):
   - Compiler executable: `cobc.exe` added to Windows `PATH`.
4. **SQLite3 for Windows**:
   - SQLite CLI tools and `sqlite3.dll` added to Windows `PATH`.
5. **.NET 8 SDK for Windows** (for Module 5 interop):
   ```powershell
   winget install Microsoft.DotNet.SDK.8
   ```

---

## 🔑 Step 1: Set Up & Verify SSH on Windows

Windows 10/11 comes with the native OpenSSH client built-in.

### 1. Generate SSH Key Pair (PowerShell)
Open PowerShell as the trainee user (`C:\Users\trainees`):
```powershell
ssh-keygen -t rsa -b 4096 -C "trainees@cobol-training" -N '""'
```

### 2. View Public Key to Add to GitHub
```powershell
Get-Content C:\Users\trainees\.ssh\id_rsa.pub
```
Copy the output and add it under your GitHub account:
**GitHub ➔ Settings ➔ SSH and GPG keys ➔ New SSH key**.

### 3. Test SSH Authentication
```powershell
ssh -T -o StrictHostKeyChecking=accept-new git@github.com
```
*Expected Output:*
```text
Hi dev-paplix! You've successfully authenticated, but GitHub does not provide shell access.
```

### 4. Verify Git Remote URL
```powershell
cd C:\Users\trainees\codes\cobol-training
git remote -v
```
If using HTTPS, switch to SSH:
```powershell
git remote set-url origin git@github.com:dev-paplix/cobol-training-day2.git
```

---

## 📦 Step 2: Download & Extract the Windows Runner

Run PowerShell as **Administrator** (or as your local trainee user):

```powershell
# 1. Create a dedicated runner folder
New-Item -ItemType Directory -Path "C:\actions-runner" -Force
Set-Location "C:\actions-runner"

# 2. Download the latest Windows x64 runner package (v2.322.0)
Invoke-WebRequest -Uri "https://github.com/actions/runner/releases/download/v2.322.0/actions-runner-win-x64-2.322.0.zip" -OutFile "actions-runner-win-x64-2.322.0.zip"

# 3. Extract the ZIP package
Expand-Archive -Path ".\actions-runner-win-x64-2.322.0.zip" -DestinationPath "." -Force

# 4. Optional: Remove archive to save space
Remove-Item ".\actions-runner-win-x64-2.322.0.zip"
```

---

## 🎫 Step 3: Obtain Runner Token from GitHub

1. Open your repository: `https://github.com/dev-paplix/cobol-training-day2`
2. Go to **Settings** ➔ **Actions** ➔ **Runners**.
3. Click **New self-hosted runner**.
4. Select **OS: Windows**, **Architecture: x64**.
5. Copy the registration token string.

---

## ⚙️ Step 4: Configure and Register the Runner

From `C:\actions-runner` in PowerShell:

### Option A: Interactive Registration
```powershell
Set-Location "C:\actions-runner"
.\config.cmd --url https://github.com/dev-paplix/cobol-training-day2 --token <YOUR_REGISTRATION_TOKEN>
```

When prompted:
1. **Runner name**: Enter `windows-cobol-runner` (or press Enter for computer name).
2. **Runner group**: Press Enter for `Default`.
3. **Labels**: Enter `self-hosted,windows,x64,windows-latest`.
4. **Work folder**: Press Enter for `_work`.

### Option B: Automated Non-Interactive (Unattended)
```powershell
Set-Location "C:\actions-runner"
.\config.cmd --unattended `
  --url https://github.com/dev-paplix/cobol-training-day2 `
  --token <YOUR_REGISTRATION_TOKEN> `
  --name "windows-cobol-runner" `
  --labels "self-hosted,windows,x64,windows-latest" `
  --work "_work" `
  --replace
```

---

## 🚀 Step 5: Run the Runner

### Mode A: Interactive Console (Immediate Testing)
```powershell
Set-Location "C:\actions-runner"
.\run.cmd
```
*Expected Output:*
```text
√ Connected to GitHub
Current runner version: '2.322.0'
2026-09-09 13:00:00Z: Listening for Jobs
```

### Mode B: Windows Service Daemon (Recommended for Production)
Run PowerShell as **Administrator**:

```powershell
Set-Location "C:\actions-runner"

# Install runner as a Windows background service
.\svc.cmd install

# Start the service
.\svc.cmd start

# Check service status
.\svc.cmd status
```

*Managing the Windows Service:*
```powershell
# Stop service
.\svc.cmd stop

# Uninstall service
.\svc.cmd uninstall
```

*(You can also manage the service from Windows Services GUI: press `Win + R`, type `services.msc`, and locate "GitHub Actions Runner").*

---

## 📝 Step 6: Sample Windows Workflow Reference

Below is a reference GitHub Actions workflow targeting a Windows self-hosted runner (`.github/workflows/cobol-windows-ci.yml`):

```yaml
name: COBOL Windows CI/CD Pipeline (Reference)

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test on Windows Runner
    runs-on: [self-hosted, windows]

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Diagnostic Toolchain Check
        shell: powershell
        run: |
          Write-Host "=== Diagnostic Toolchain Check ==="
          git --version
          if (Get-Command cobc -ErrorAction SilentlyContinue) {
              cobc --version
          } else {
              Write-Warning "GnuCOBOL (cobc) not found in PATH"
          }
          if (Get-Command dotnet -ErrorAction SilentlyContinue) {
              dotnet --version
          }

      - name: Build & Test Application
        shell: powershell
        run: |
          Write-Host "Running automated build steps on Windows..."
          # Example Windows build invocation:
          # cobc -x -free -I module_8/src module_8/src/app.cob -o module_8/bin/inventory_app.exe
```

---

## 🛠️ Windows Troubleshooting & Tips

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| `File ... cannot be loaded because running scripts is disabled` | PowerShell ExecutionPolicy restriction | Run as Administrator: `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`. |
| `cobc is not recognized` | GnuCOBOL binary directory not in system `PATH` | Add GnuCOBOL `bin` folder (e.g., `C:\msys64\mingw64\bin`) to Windows System Environment Variables `PATH`. |
| `Access is denied` during `.\svc.cmd install` | Missing elevated Administrator rights | Right-click PowerShell and choose **"Run as Administrator"**. |
| `Permission denied (publickey)` in Git | SSH agent not running or key not loaded | Start ssh-agent: `Get-Service ssh-agent | Set-Service -StartupType Automatic ; Start-Service ssh-agent`, then `ssh-add C:\Users\trainees\.ssh\id_rsa`. |
| Deregistering Windows Runner | Removing runner registration | Run `.\config.cmd remove --token <TOKEN_FROM_GITHUB>`. |

