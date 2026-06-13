# Laptop Setup — Install Everything You Need

This guide installs every tool the projects in this repo use, with steps for **Linux**,
**macOS**, and **Windows**. Do this once, then any project's steps will "just work".

> **You don't need everything for every project.** Use the checklist below to see what each
> tool is for, then jump to its section. At minimum, *every* project needs an **AWS account**,
> the **AWS CLI**, **Python**, and **Git**.

---

## What You Need — At a Glance

| Tool | Required for | Verify it's installed |
|------|--------------|------------------------|
| **AWS account + credentials** | Every project (it all runs in AWS) | `aws sts get-caller-identity` |
| **AWS CLI v2** | Every project (the `aws ...` commands) | `aws --version` → `aws-cli/2.x` |
| **Python 3.12+ & pip** | App code, `test_app.py`, Boto3 automation | `python3 --version`, `pip3 --version` |
| **Git** | Cloning this repo; GitHub Actions deploy steps | `git --version` |
| **A code editor** (VS Code) | Reading/editing files (recommended, optional) | opens `.md` and `.py` files |
| **curl** | Testing endpoints in many steps | `curl --version` |
| **Docker + Docker Compose** | **ECS/Fargate projects only** (containers) | `docker --version`, `docker compose version` |
| **zip** | Serverless Lambda packaging (deploy steps) | `zip --version` |
| **AWS SAM CLI** | *Optional* — one serverless challenge only | `sam --version` |

Python libraries (`boto3`, `flask`, `gunicorn`) install **per project** with
`pip install -r requirements.txt` — covered in the [Python Libraries](#7-python-libraries-per-project)
section, not installed globally.

> ⚠️ **Windows + gunicorn:** `gunicorn` does **not** run natively on Windows (it's Unix-only).
> This is fine — gunicorn only runs on the **EC2 Linux server**, never on your laptop. To run
> the web app *locally* on Windows, use the Flask dev server (`python app.py`) or
> [WSL2](#windows-strongly-recommended-install-wsl2). See [details below](#7-python-libraries-per-project).

---

## 1. AWS Account + Credentials

1. **Create an account:** https://aws.amazon.com → *Create an AWS Account* (needs a card; the
   projects are designed to stay within free tier or cost cents — see each project's `costs.md`).
2. **Create an access key for the CLI:** AWS Console → **IAM** → *Users* → your user →
   *Security credentials* → **Create access key** → *Command Line Interface (CLI)*. Copy the
   **Access key ID** and **Secret access key** (you only see the secret once).
3. **Configure the CLI** (after installing it in step 2 below):

   ```bash
   aws configure
   # AWS Access Key ID:     <paste>
   # AWS Secret Access Key: <paste>
   # Default region name:   us-east-1     # every project uses us-east-1
   # Default output format: json
   ```

4. **Verify:**

   ```bash
   aws sts get-caller-identity
   ```

> 🔒 **Never commit credentials.** The repo's `.gitignore` already excludes `.aws/`,
> `credentials`, `*.pem`, and `.env`. Keep it that way.

---

## 2. AWS CLI v2

The command-line tool for everything (`aws ec2 ...`, `aws lambda ...`, etc.).

### Linux
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### macOS
```bash
# Option A — Homebrew (simplest if you have brew)
brew install awscli

# Option B — official installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

### Windows
```powershell
# Option A — winget (Windows 10/11)
winget install Amazon.AWSCLI

# Option B — download and run the MSI:
#   https://awscli.amazonaws.com/AWSCLIV2.msi
aws --version
```

> Must say `aws-cli/2.x`. If you have an old `1.x`, upgrade — some commands differ.

---

## 3. Python 3.12+ and pip

Runs the app code, the `test_app.py` validators, and the Boto3 automation scripts.

### Linux (Debian/Ubuntu)
```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
python3 --version
```
*(Fedora/RHEL: `sudo dnf install -y python3 python3-pip`)*

### macOS
```bash
# The system Python can be old; install a current one via Homebrew:
brew install python@3.12
python3 --version
```

### Windows
```powershell
# Option A — winget
winget install Python.Python.3.12

# Option B — installer from https://www.python.org/downloads/
#   ✅ IMPORTANT: check "Add python.exe to PATH" during install
python --version
pip --version
```

> On Windows the command is usually `python` (not `python3`). On macOS/Linux use `python3`
> and `pip3`.

---

## 4. Git

To clone this repo and to follow the GitHub Actions deploy steps.

### Linux
```bash
sudo apt install -y git        # Debian/Ubuntu
# sudo dnf install -y git      # Fedora/RHEL
git --version
```

### macOS
```bash
brew install git
# or simply run `git --version` and accept the Xcode Command Line Tools prompt
```

### Windows
```powershell
winget install Git.Git
# or download from https://git-scm.com/download/win
# This also installs "Git Bash" — a Unix-like shell that makes the repo's
# bash/curl/zip snippets work on Windows.
git --version
```

Then clone the repo:
```bash
git clone <your-repo-url> cloud-projects
cd cloud-projects
```

---

## 5. A Code Editor (recommended)

Not strictly required, but **VS Code** renders the Markdown steps and Mermaid diagrams nicely.

- **All platforms:** download from https://code.visualstudio.com
- macOS: `brew install --cask visual-studio-code` · Windows: `winget install Microsoft.VisualStudioCode`
- Helpful extensions: *Python*, *Markdown Preview Mermaid Support*, *AWS Toolkit*.

---

## 6. Docker + Docker Compose — *ECS/Fargate projects only*

Skip this unless you're doing `ecs-fargate-basics` or `ecs-fargate-advanced`.

### Linux (Ubuntu) — Docker Engine
```bash
# Official convenience script (Docker Engine + Compose plugin)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER        # run docker without sudo (log out/in after)
docker --version
docker compose version
```

### macOS — Docker Desktop
```bash
brew install --cask docker
# then launch Docker Desktop once from Applications so the engine starts
docker --version
```

### Windows — Docker Desktop
```powershell
winget install Docker.DockerDesktop
# Requires WSL2 (see the Windows note below). Launch Docker Desktop once after install.
docker --version
```

> Docker Desktop must be **running** (whale icon in the menu/task bar) before `docker` commands work.

---

## 7. Python Libraries (per project)

Don't install these globally — each project lists what it needs in a `requirements.txt`.
Use a **virtual environment** so projects don't interfere with each other:

```bash
# from inside a project folder, e.g. ec2-vpc-monitored-webapp/
python3 -m venv .venv                  # create an isolated environment
source .venv/bin/activate              # macOS/Linux
# .venv\Scripts\activate               # Windows PowerShell

pip install -r src/requirements.txt    # flask, gunicorn (EC2 project)
pip install boto3                       # for scripts/setup_monitoring.py
```

| Library | Used by | Notes |
|---------|---------|-------|
| `flask` | EC2 web app (`src/app.py`) | The web framework |
| `gunicorn` | EC2 web app on the **server** | ⚠️ Unix-only — won't `pip install`/run on native Windows. The EC2 instance (Linux) runs it; on a Windows laptop just run `python app.py` to test locally, or use WSL2. |
| `boto3` | `setup_monitoring.py` in both web app projects | AWS SDK for Python |

> The **serverless** project's handler needs **no libraries at all** to run its
> `test_app.py` — it's pure standard-library Python. Just `python test_app.py`.

---

## 8. Optional Extras

| Tool | When you'd want it | Install |
|------|--------------------|---------|
| **zip** | Packaging Lambda code in serverless deploy steps | Linux: `sudo apt install zip` · macOS: preinstalled · Windows: use **Git Bash**, or PowerShell's `Compress-Archive` |
| **jq** | Prettier JSON on the command line (a few steps) | Linux: `sudo apt install jq` · macOS: `brew install jq` · Windows: `winget install jqlang.jq` |
| **AWS SAM CLI** | One serverless *challenge* (infra-as-code) — not core | https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html |

`curl` is preinstalled on macOS, modern Windows 10/11, and most Linux distros (Ubuntu:
`sudo apt install curl` if missing).

---

## Platform Notes

### Windows (strongly recommended): install WSL2
Many steps use Unix shell snippets (`bash`, `curl`, `zip`, here-docs). The smoothest
experience on Windows is **WSL2** — a real Linux environment inside Windows:

```powershell
wsl --install            # installs Ubuntu; reboot when prompted
```
Then open **Ubuntu** from the Start menu and follow the **Linux** instructions above inside
it. This also makes `gunicorn` and Docker work normally. (Without WSL2, use **Git Bash** for
the shell snippets.)

### macOS: install Homebrew first
[Homebrew](https://brew.sh) makes every install above a one-liner:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
On **Apple Silicon (M1/M2/M3)**, Docker images in the ECS projects are built for `linux/amd64`
in AWS — Docker Desktop handles the emulation; no extra steps needed for these projects.

### Linux: pick your package manager
Examples use `apt` (Debian/Ubuntu). For Fedora/RHEL/Amazon Linux, substitute `dnf`
(e.g. `sudo dnf install -y git python3`).

---

## Verify Your Whole Setup

Run these — every line should print a version (Docker only if you'll do ECS projects):

```bash
aws --version
aws sts get-caller-identity        # proves credentials work
python3 --version                  # 'python --version' on Windows
pip3 --version
git --version
curl --version
docker --version                   # ECS projects only
docker compose version             # ECS projects only
```

If all of those succeed, you're ready. Head to the [root README](README.md) and pick a
project — each one's `README.md` lists its specific prerequisites and a step-by-step path.
