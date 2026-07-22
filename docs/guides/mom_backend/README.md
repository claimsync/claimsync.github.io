# Git Submodule Management

This project uses the `medication-optimization-alternatives-search-modules` repository as a Git submodule to maintain clean separation and easy updates.

## Initial Setup

### For New Clones
When cloning this repository for the first time:

```bash
# Clone with submodules
git clone --recursive https://github.com/your-org/mom-backend.git

# OR if you already cloned without --recursive
git clone https://github.com/your-org/mom-backend.git
cd mom-backend
git submodule update --init --recursive
```

### Adding the Submodule (One-time setup - already done)
```bash
# This was already done, documented for reference
git submodule add https://github.com/your-org/medication-optimization-alternatives-search-modules.git medication-optimization-alternatives-search-modules
git submodule update --init
git add .gitmodules medication-optimization-alternatives-search-modules
git commit -m "Add medication optimization module as submodule"
```

## Working with the Submodule

### Updating to Latest Version
To update the submodule to the latest version:

```bash
cd medication-optimization-alternatives-search-modules
git pull origin main  # or the appropriate branch
cd ..
git add medication-optimization-alternatives-search-modules
git commit -m "Update medication optimization module to latest version"
git push
```

### Checking Submodule Status
```bash
# Check submodule status
git submodule status

# See which commit the submodule is pointing to
git ls-tree HEAD medication-optimization-alternatives-search-modules
```

### Working on Submodule Code
If you need to make changes to the submodule:

```bash
cd medication-optimization-alternatives-search-modules
git checkout -b feature/your-feature-name
# Make your changes
git add .
git commit -m "Your changes"
git push origin feature/your-feature-name
# Create PR in the submodule repository

# After PR is merged, update main project
git checkout main
git pull origin main
cd ..
git add medication-optimization-alternatives-search-modules
git commit -m "Update submodule to include new feature"
```

## Deployment Considerations

### GitHub Actions
The deployment workflow automatically handles submodule updates:

```yaml
script: |
  cd ${{ secrets.APP_DIR }}
  git fetch --all
  git checkout ${{ github.ref_name }}
  git reset --hard origin/${{ github.ref_name }}
  git pull origin ${{ github.ref_name }} --rebase
  
  # Update submodules to match the commit referenced in main repo
  git submodule update --init --recursive
```

### Manual Deployment
If deploying manually:

```bash
cd /opt/flask-apps/mom-backend
git pull
git submodule update --init --recursive
# Continue with your normal deployment steps
```

## Important Notes

- **Never modify files directly** in the `medication-optimization-alternatives-search-modules` directory. Always make changes in the original repository.
- The submodule points to a **specific commit**, not a branch. This ensures consistency across deployments.
- When team members pull updates, they should run `git submodule update` to get the correct submodule version.
- The `.gitmodules` file tracks the submodule configuration - don't modify it manually.

## Troubleshooting

### Submodule Directory is Empty
```bash
git submodule update --init --recursive
```

### Submodule Shows Modified Files
This usually means someone modified files directly in the submodule directory:
```bash
cd medication-optimization-alternatives-search-modules
git status
git checkout .  # Discard local changes
cd ..
```

### Updating All Team Members
When you update the submodule, notify team members to run:
```bash
git pull
git submodule update --recursive
```

## File Structure
```
mom-backend/
├── api/
├── config.py
├── medication-optimization-alternatives-search-modules/  # ← Submodule
│   ├── (submodule contents)
│   └── ...
├── requirements.txt
└── .gitmodules  # ← Submodule configuration
```