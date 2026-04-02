# Git Home Server Migration Plan (Detailed)

## Current State
- **22 project directories** in `~/projects/` — all currently **non-git**
- **Backup completed**: `~/backup/projects-*.tar.gz` archives exist
- **Target**: Each subdirectory becomes its own git repository

## Goals
Convert each project folder into its own git repo, ready for remote push when needed.

## Plan Steps

### 1. Initialize Git in Each Subdirectory
```bash
# For each project directory
cd ~/projects && for dir in */; do
  cd "$dir" && git init -b main
  cd ~/projects
  echo "Initialized git in $dir"
done
```

### 2. Configure Git in Each Repo
```bash
# Run in each subdirectory
cd ~/projects/
for dir in */; do
  cd "$dir"
  git config user.name "Bill"
  git config user.email "adminbill@fedora"
  git branch -m main
  cd ~/projects
  echo "Configured git in $dir"
done
```

### 3. Add `.gitignore` to Each Project (Optional)
```bash
# Create a shared .gitignore template in ~/projects/
cd ~/projects

# For each project, add minimal .gitignore
echo '# Common build artifacts\nnode_modules\ndist\n__pycache__\n*.log' > .gitignore.template

# Copy to each subdirectory
cp .gitignore.template ~/projects/CMMS/.gitignore
cp .gitignore.template ~/projects/docs/.gitignore
# ... etc
```

### 4. Initial Commits in Each Repo
```bash
# In each subdirectory
cd ~/projects/
for dir in */; do
  cd "$dir"
  git add .
  git commit -m "Initial structure"
  cd ~/projects
  echo "Committed in $dir"
done
```

### 5. Verify All Repos
```bash
# Check each repo has a main branch
cd ~/projects/
for dir in */; do
  cd "$dir"
  echo "$dir: $(git branch -v 2&& echo 'main' || echo 'no repo')"
  cd ~/projects
ndone
```

## Notes
- Each project now has its own git repo
- Subdirectories can be pushed to separate remotes independently
- `~/projects/` root remains a shared workspace, not a git repo itself
- No remote required yet — can add later per project
