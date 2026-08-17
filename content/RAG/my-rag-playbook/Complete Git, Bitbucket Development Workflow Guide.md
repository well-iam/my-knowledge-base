This guide covers the end-to-end setup and daily development workflow for collaborating on a software project using Git, Bitbucket, and `uv`.

## Phase 1: SSH Key Authentication Setup
Using SSH keys provides a secure, passwordless connection between your development machine (local PC or remote server) and Bitbucket.
### 1. Generating an SSH Key
Open your terminal (PowerShell) and run:

```Bash
ssh-keygen -t ed25519 -C "your_email@company.com"
```

- **`-t ed25519`**: Specifies the Ed25519 Elliptic Curve algorithm (modern, fast, and highly secure).    
- **`-C "email"`**: Adds a recognizable comment tag to the key.

When prompted for a file path, press **Enter** to accept the default location (`~/.ssh/id_ed25519`). Passphrases are optional.

### 2. Adding the Public Key to Bitbucket
A public key file consists of **three parts** separated by spaces: the algorithm (`ssh-ed25519`), the Base64 key string, and your email comment. **You must copy the entire line.**
1. Log in to Bitbucket.
2. Go to **Personal Settings** $\rightarrow$ **SSH Keys** $\rightarrow$ **Add Key**.
3. Paste the entire contents of your `.pub` file and click **Add key**.

### 3. Testing the SSH Connection
Verify that Bitbucket recognizes your key by running:
```Bash
ssh -T git@bitbucket.org
```

**Expected Output:**
```
authenticated via ssh key.
You can use git to connect to Bitbucket. Shell access is disabled.
```
> _Note:_ The "Shell access is disabled" notice is normal. Bitbucket uses SSH strictly as an encrypted transport layer for Git operations, not for shell access.

### 4. Multi-Machine Strategy (Local PC vs Remote Server)
If you work both on a local machine and on a remote server (e.g., via VS Code Remote SSH):
- **Recommended Practice:** Generate a separate SSH key pair on **each** machine and add both public keys to your Bitbucket account (labeling them clearly, e.g., _"Laptop"_ and _"Dev Server"_). This allows you to revoke access for one machine independently if needed.

## Phase 2: Repository Initial Setup & Protection

### 1. Project `.gitignore` Configuration
To avoid uploading massive vector databases, virtual environments, or sensitive credentials, create a `.gitignore` file in your project root:

```
# === Virtual Environments ===
.venv/
venv/

# === Python Cache ===
__pycache__/
*.py[cod]
.pytest_cache/

# === RAG & Data Directories ===
# Ignore data contents but keep directory structure via .gitkeep
data/raw/*
data/processed/*
data/vector_store/*

!data/raw/.gitkeep
!data/processed/.gitkeep
!data/vector_store/.gitkeep

# === Secrets & Environment Variables ===
.env

# === IDE Settings ===
.vscode/
.idea/
```

> **Tip:** Create empty `.gitkeep` files inside `data/raw/` and `data/vector_store/` so Git tracks the directory structure without committing heavy files.

### 2. Linking Local Code to Bitbucket Remote
Run the following commands inside your local project directory:
```Bash
# 1. Add the remote URL (replace with your Bitbucket SSH link)
git remote add origin git@bitbucket.org:workspace-name/rag_system.git

# 2. Ensure your primary branch is named 'main'
git branch -M main

# 3. Save your files locally
git add .
git commit -m "chore: initial project setup"

# 4. Push and set the upstream tracking branch
git push -u origin main
```

- **`-u` (`--set-upstream`)**: Links your local `main` branch directly to `origin/main`. After running this once, you can simply type `git push` or `git pull`.

### 3. Setting Up Branch Protection Rules (Bitbucket Web UI)
To prevent accidental direct pushes to production code:
1. Go to **Repository Settings** $\rightarrow$ **Branch permissions** (or **Branch restrictions**).
2. Click **Add permission**.
3. **Select branches:** Choose _By branch name or pattern_ and enter **`main`**.
4. **Write access:** Select _Only specific people or groups have write access_ and leave the field **empty**. This blocks direct `git push` commands to `main`.
5. **Merge access:** Select _Everyone with access to the repository has merge access_.
6. **Merge settings (Tab):** Set **Minimum approvals** to **`1`**.
7. Click **Save**.

## Phase 3: Daily Feature Lifecycle & Team Workflow
### Step 1: Start from an Updated `main` Branch
Before writing any code, switch to `main` and pull the latest changes:
```Bash
git checkout main
git pull
uv sync
```

- **`uv sync`**: Ensures your local virtual environment (`.venv`) stays in sync with any dependency changes added by your teammates in `pyproject.toml` or `uv.lock`.

### Step 2: Create a Feature Branch
Isolate your changes by creating a dedicated branch:
```Bash
git checkout -b feature/docling-tables
```

### Step 3: Local Development & Micro-Commits
Work on your feature. As you reach micro-milestones or add dependencies:

```Bash
# If you add a new Python library using uv:
uv add docling

# Stage configuration and code files
git add pyproject.toml uv.lock src/ingestion.py

# Commit with a clear message
git commit -m "feat: implement table extraction via Docling"
```

### Step 4: Push the Feature Branch to Bitbucket

Publish your branch to the remote repository:
```Bash
git push -u origin feature/docling-tables
```

### Step 5: Open a Pull Request (PR)
1. Navigate to your repository in Bitbucket.
2. Click **Create Pull Request** on the pop-up banner (or go to **Pull Requests** $\rightarrow$ **Create pull request**).
3. Set **Source branch** to `feature/docling-tables` and **Destination branch** to `main`.
4. Assign your teammate as a **Reviewer**.
5. Write a brief summary of your changes and click **Create pull request**.

### Step 6: Code Review & Merging
1. **Review:** The assigned reviewer inspects the code diff, leaves comments if needed, and clicks **Approve**.
2. **Merge Strategy Selection:**
    - **Merge commit (`git merge --no-ff`):** Preserves all individual commits and adds a explicit merge commit on `main`.
    - **Squash (`git merge --squash`):** Combines all feature commits into a single clean commit on `main` (ideal for keeping history tidy).
3. Click **Merge** and check the box **"Delete source branch"** to keep the repository clean.

### Step 7: How Teammates Update Their Local Workspaces
Once a PR is merged into `main`, all team members should run the following sequence to incorporate the changes into their active workspace:

```Bash
# 1. Switch to main and pull latest changes
git checkout main
git pull

# 2. Update Python virtual environment dependencies
uv sync

# 3. Switch back to your working feature branch and merge main
git checkout feature/your-other-feature
git merge main
```

## Summary Cheat Sheet

|**Task**|**Command / Action**|
|---|---|
|**Check SSH Keys (Windows)**|`Get-Content ~/.ssh/id_ed25519.pub`|
|**Test SSH Connection**|`ssh -T git@bitbucket.org`|
|**Start New Feature**|`git checkout main` $\rightarrow$ `git pull` $\rightarrow$ `git checkout -b feature/name`|
|**Install Library (`uv`)**|`uv add <package-name>`|
|**Save Progress**|`git add .` $\rightarrow$ `git commit -m "message"`|
|**Publish Branch**|`git push -u origin feature/name`|
|**Submit Code**|Create Pull Request on Bitbucket Web UI|
|**Sync Local Environment**|`git pull` $\rightarrow$ `uv sync`|