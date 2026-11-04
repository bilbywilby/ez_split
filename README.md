# Multi-Repository Splitting & Initialization Guide

This guide provides automated and manual procedures for packaging independent subdirectories from a monolithic parent workspace into dedicated GitHub repositories from terminal environments (including Termux, Linux, and macOS).

---

## 1. Termux Error Troubleshooting & Quick Resolution

### Problem 1: `fatal: not a git repository`
You ran `git status` or `git add` inside `~/downloads`. Downloads is an unversioned folder. Your Git repositories are located in your home directory (`~`).

**Fix:** Navigate back to home before running repository tasks:
```bash
cd ~
```

### Problem 2: `bash: syntax error near unexpected token` / `import: command not found`
Python code was pasted directly into a Bash interactive shell instead of running inside the Python interpreter.

**Fix:** Create a `.py` file first, or run Python inline:
```bash
# Option A: Save and execute Python file
cat << 'EOF' > verify_env.py
import os, sys, shutil, subprocess

print(f"Working Directory: {os.getcwd()}")
print(f"Python Runtime: {sys.version.split()[0]}")
EOF
python3 verify_env.py

# Option B: Run quick inline python check
python3 -c "import shutil; print(shutil.disk_usage('.'))"
```

---

## 2. Automated Multi-Repository Script

Save this script as `split_and_push.sh` in your home directory (`~`). It targets the repositories found in your home folder.

### Prerequisites
* **Git**: Installed (`pkg install git`)
* **GitHub CLI (`gh`)**: Installed (`pkg install gh`) and authenticated (`gh auth login`)

### Script (`~/split_and_push.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# Configuration
# ==============================================================================
GITHUB_USER="bilbywilby"
VISIBILITY="public" # Options: "public" or "private"

# Target directories located in ~/
DIRECTORIES=(
    "netprobe"
    "tech-ref-portal"
    "Keystone_Health"
    "capyreader"
)

# Ensure script runs relative to Home directory
cd ~

# ==============================================================================
# Execution Loop
# ==============================================================================
for dir in "${DIRECTORIES[@]}"; do
    if [ -d "$dir" ]; then
        echo "=========================================="
        echo "Processing directory: $dir"
        echo "=========================================="
        
        cd "$dir"
        
        # 1. Clean temporary artifacts, swap files, and build outputs
        find . -type f \( -name "*.swp" -o -name "*.swo" -o -name "*~" -o -name ".DS_Store" \) -delete
        find . -type d -name "dist" -exec rm -rf {} + 2>/dev/null || true
        find . -type d -name "build" -exec rm -rf {} + 2>/dev/null || true
        
        # 2. Initialize Git if not already a repository
        if [ ! -d ".git" ]; then
            git init
            git branch -M main
        else
            echo "Git repository already initialized in $dir."
        fi
        
        # 3. Create a standard .gitignore if none exists
        if [ ! -f ".gitignore" ]; then
            cat << 'EOF' > .gitignore
__pycache__/
*.py[cod]
*$py.class
dist/
build/
*.swp
*.swo
.DS_Store
.env
node_modules/
EOF
            echo "Created default .gitignore"
        fi
        
        # 4. Stage and commit files
        git add .
        if git diff --cached --quiet; then
            echo "No changes to commit for $dir."
        else
            git commit -m "Initial structure commit for $dir"
            echo "Committed changes successfully."
        fi
        
        # 5. Create remote GitHub repository and push
        if command -v gh &> /dev/null; then
            if ! git remote get-url origin &> /dev/null; then
                echo "Creating remote GitHub repository: $GITHUB_USER/$dir"
                gh repo create "$dir" --"$VISIBILITY" --source=. --remote=origin --push || {
                    echo "[WARNING] Could not create remote via GitHub CLI. Link manually."
                }
            else
                echo "Remote 'origin' already exists. Pushing current branch..."
                git push -u origin main || git push -u origin master
            fi
        else
            echo "[INFO] GitHub CLI ('gh') not found. Link remote manually using:"
            echo "  git remote add origin https://github.com/$GITHUB_USER/$dir.git"
            echo "  git push -u origin main"
        fi
        
        cd ~ > /dev/null
        echo ""
    else
        echo "[WARNING] Directory '$dir' not found in ~. Skipping."
    fi
done

echo "Processing complete."
```

### Execution

```bash
cd ~
chmod +x split_and_push.sh
./split_and_push.sh
```

---

## 3. Manual Execution Procedure (Example: `ez_split`)

To manually connect and push an individual repository (e.g., `ez_split` or any target folder):

### Step 1: Navigate to Target Folder
```bash
cd ~/your-target-folder
```

### Step 2: Clean Build Artifacts
```bash
find . -type f \( -name "*.swp" -o -name "*.swo" -o -name "*~" -o -name ".DS_Store" \) -delete
rm -rf dist/ build/ node_modules/
```

### Step 3: Initialize Git and Stage Files
```bash
git init
git branch -M main
git add .
git commit -m "Initial commit for repository"
```

### Step 4: Link Remote & Push to GitHub

#### Method A: Using GitHub CLI
```bash
gh repo create ez_split --public --source=. --remote=origin --push
```

#### Method B: Manual Git Remote Link
```bash
git remote add origin https://github.com/bilbywilby/ez_split.git
git push -u origin main
```
