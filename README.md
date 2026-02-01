# Dotfiles - Full System Restore Guide

This repository contains my dotfiles and system configuration, managed with [yadm](https://yadm.io/).

## 🎯 What's Included

- **OpenCode Configuration** (`~/.config/opencode/`)
  - Core configs: `opencode.jsonc`, `dcp.jsonc`, `pickle-thinker.jsonc`
  - Agent instructions: `AGENTS.md`
  - Custom commands and knowledge base
  - Plugin configurations
  - Devcontainer settings

- **Shell Configuration** (`~/.zshrc`)
  - Starship prompt
  - Environment variables
  - 1Password SSH agent setup
  - Aliases and functions

- **Git Configuration** (`~/.gitconfig`)

- **SSH Configuration** (`~/.ssh/config`)
  - Configured for 1Password SSH agent

---

## 📦 Prerequisites

Before starting the restore process, install these tools:

```bash
# Debian/Ubuntu (apt)
sudo apt update
sudo apt install yadm git

# Fedora/RHEL (dnf)
sudo dnf install yadm git

# Arch Linux (pacman)
sudo pacman -S yadm git

# openSUSE (zypper)
sudo zypper install yadm git
```

You'll also need:
- **1Password** desktop app ([download here](https://1password.com/downloads))
- **1Password CLI** (`op`) - usually included with desktop app
- **OpenCode** ([installation guide](https://opencode.ai/docs/installation))

---

## 🚀 Full Restore Process

### Step 1: Install 1Password and Enable SSH Agent

This must be done **first** because you need SSH access to clone repositories.

1. **Install and sign in to 1Password desktop app**
   ```bash
   # macOS: Download from https://1password.com/downloads
   # Linux: Follow instructions at https://1password.com/downloads/linux/
   ```

2. **Enable 1Password SSH agent**
   - Open 1Password
   - Go to **Settings** → **Developer**
   - Enable **"Use the SSH agent"**
   - Enable **"Display key names when authorizing connections"** (optional)

3. **Verify SSH agent is working**
   ```bash
   # Check if socket exists
   ls -la ~/.1password/agent.sock
   
   # Test authentication (you'll get a biometric prompt)
   ssh-add -l
   ```

4. **Test GitHub SSH connection**
   ```bash
   ssh -T git@github.com
   # You should see: "Hi <username>! You've successfully authenticated..."
   ```

---

### Step 2: Clone Dotfiles with yadm

```bash
# Clone this repository
yadm clone git@github.com:filipeestacio/dotfiles.git

# If you have existing files that conflict, yadm will warn you
# Back them up or remove them, then run:
yadm reset --hard origin/main
```

**What this restores:**
- `~/.config/opencode/` (OpenCode configuration)
- `~/.ssh/config` (SSH configuration for 1Password agent)
- `~/.zshrc` or `~/.bashrc` (shell configuration)
- `~/.gitconfig` (git configuration)
- Any other dotfiles tracked in this repo

---

### Step 3: Reload Shell Configuration

```bash
# For zsh
source ~/.zshrc

# For bash
source ~/.bashrc

# Verify SSH_AUTH_SOCK is set
echo $SSH_AUTH_SOCK
# Should output: /home/<username>/.1password/agent.sock
```

---

### Step 4: Restore Superpowers

Superpowers is managed as a separate git repository (not tracked in yadm).

```bash
# Clone your fork
git clone git@github.com:filipeestacio/superpowers.git ~/.config/opencode/superpowers

# Switch to your custom branch with OpenCode modifications
cd ~/.config/opencode/superpowers
git checkout filipe/opencode-update-plan

# Add upstream remote (to pull updates from original repo)
git remote add upstream https://github.com/obra/superpowers.git

# Verify remotes
git remote -v
# Should show:
#   origin    git@github.com:filipeestacio/superpowers.git
#   upstream  https://github.com/obra/superpowers.git
```

---

### Step 5: Restore Workspace

```bash
# Clone workspace repository
git clone git@github.com:MyUtilityConnection/workspace.git ~/code/myutilityconnection

# Initialize and update submodules
cd ~/code/myutilityconnection
git submodule update --init --recursive

# Verify submodules are loaded
git submodule status
```

---

### Step 6: Install Dependencies

```bash
# Install Node.js modules for OpenCode plugins (if needed)
cd ~/.config/opencode
npm install  # or: bun install

# Install any workspace dependencies
cd ~/code/myutilityconnection
# Follow project-specific setup instructions
```

---

### Step 7: Verify Everything Works

```bash
# Test SSH access
ssh -T git@github.com

# Test git operations
cd ~/code/myutilityconnection
git pull
git status

# Verify superpowers
cd ~/.config/opencode/superpowers
git status
git log --oneline -3

# Start OpenCode and verify configuration
# OpenCode should load your custom skills and settings
```

---

## 🔧 Configuration Details

### OpenCode Configuration Structure

```
~/.config/opencode/
├── opencode.jsonc          # Main OpenCode config
├── dcp.jsonc               # Dynamic Context Pruning plugin
├── pickle-thinker.jsonc    # Pickle Thinker plugin config
├── AGENTS.md               # Agent instructions
├── .gitignore              # Ignored files/directories
├── command/                # Custom commands
│   ├── devcontainer.md
│   ├── workspaces.md
│   └── worktree.md
├── devcontainers/          # Devcontainer configs
│   └── config.json
├── knowledge/              # Knowledge base
│   ├── agent-specific/
│   ├── business/
│   ├── technical/
│   └── README.md
├── plugin/                 # Plugin configurations
├── superpowers/            # Superpowers skills (separate git repo)
└── logs/                   # Runtime logs (not tracked)
```

### Environment Variables

Your OpenCode configuration uses environment variable placeholders for secrets:
- `{env:GITHUB_BEARER_TOKEN}`
- `{env:CONTEXT7_API_KEY}`
- `{env:EXA_API_KEY}`
- `{env:OBSIDIAN_API_KEY}`
- etc.

**Set these in your shell config** or use 1Password CLI:

```bash
# Option 1: In ~/.zshrc or ~/.bashrc
export GITHUB_BEARER_TOKEN="your-token-here"
export CONTEXT7_API_KEY="your-key-here"

# Option 2: Using 1Password CLI (recommended)
export GITHUB_BEARER_TOKEN=$(op read "op://Personal/GitHub/bearer-token")
export CONTEXT7_API_KEY=$(op read "op://Personal/Context7/api-key")
```

---

## 🔑 SSH Key Management

This setup uses **1Password SSH agent** for secure key management.

### How It Works

1. SSH keys are stored in 1Password vault (not on disk)
2. `~/.ssh/config` points to `~/.1password/agent.sock`
3. When SSH is used, 1Password prompts for biometric authentication
4. No plaintext private keys on your filesystem

### SSH Configuration

Your `~/.ssh/config` should contain:

```ssh-config
# Use 1Password SSH Agent
Host *
    IdentityAgent ~/.1password/agent.sock

Host github.com
    Hostname github.com
    User git
```

### Managing SSH Keys in 1Password

**View available keys:**
```bash
ssh-add -l
```

**Add a new SSH key to 1Password:**
1. Open 1Password desktop app
2. Click **+** → **SSH Key**
3. Enter a title (e.g., "GitHub Personal", "Work GitLab")
4. Paste your private key
5. Paste your public key
6. Save

**Generate a new SSH key for 1Password:**
```bash
# Generate new key
ssh-keygen -t ed25519 -C "your-email@example.com" -f /tmp/new-key

# Import to 1Password (via desktop app UI)
# Then delete the temporary files
rm /tmp/new-key /tmp/new-key.pub
```

---

## 🔄 Keeping Things Updated

### Update Dotfiles

```bash
# Pull latest dotfiles
yadm pull

# Add new dotfiles
yadm add ~/.config/some-new-config

# Commit and push
yadm commit -m "Add new configuration"
yadm push
```

### Update Superpowers

```bash
cd ~/.config/opencode/superpowers

# Pull latest changes from your fork
git pull origin filipe/opencode-update-plan

# Sync with upstream (original repo)
git fetch upstream
git merge upstream/main
git push origin filipe/opencode-update-plan
```

### Update Workspace

```bash
cd ~/code/myutilityconnection

# Pull latest changes
git pull

# Update submodules
git submodule update --remote
```

---

## 🛠️ Troubleshooting

### SSH Authentication Issues

**Problem:** `Permission denied (publickey)` when cloning repos

**Solution:**
```bash
# 1. Verify 1Password is running
pgrep -x "1Password" || open -a "1Password"

# 2. Check SSH agent socket exists
ls -la ~/.1password/agent.sock

# 3. Verify SSH_AUTH_SOCK is set
echo $SSH_AUTH_SOCK
# Should be: ~/.1password/agent.sock

# 4. List available keys
ssh-add -l

# 5. Test GitHub connection with verbose output
ssh -vT git@github.com
```

### yadm Clone Conflicts

**Problem:** `yadm clone` fails due to existing files

**Solution:**
```bash
# Backup existing conflicting files
mkdir -p ~/dotfiles-backup
yadm ls-files | xargs -I{} cp --parents {} ~/dotfiles-backup/

# Remove conflicting files
yadm ls-files | xargs rm

# Try clone again
yadm clone git@github.com:filipeestacio/dotfiles.git
```

### OpenCode Not Finding Configuration

**Problem:** OpenCode doesn't load custom configurations

**Solution:**
```bash
# Verify OpenCode config location
ls -la ~/.config/opencode/opencode.jsonc

# Check OpenCode is using correct config directory
echo $OPENCODE_CONFIG_DIR  # Should be empty or ~/.config/opencode

# Restart OpenCode completely
```

### Superpowers Skills Not Loading

**Problem:** OpenCode doesn't see superpowers skills

**Solution:**
```bash
# Verify superpowers is on correct branch
cd ~/.config/opencode/superpowers
git branch --show-current
# Should show: filipe/opencode-update-plan

# Verify skills exist
ls -la ~/.config/opencode/superpowers/skills/

# Check OpenCode loads superpowers
# In OpenCode, type: /list-skills
```

---

## 📚 Key Repositories

| Repository | Purpose | Location |
|------------|---------|----------|
| [filipeestacio/dotfiles](https://github.com/filipeestacio/dotfiles) | This repo - dotfiles and configs | `~/.local/share/yadm/repo.git` |
| [filipeestacio/superpowers](https://github.com/filipeestacio/superpowers) | Custom superpowers skills | `~/.config/opencode/superpowers` |
| [MyUtilityConnection/workspace](https://github.com/MyUtilityConnection/workspace) | Project workspace | `~/code/myutilityconnection` |

---

## 🔐 Security Notes

- **No secrets in this repo**: All sensitive values use `{env:VAR}` placeholders
- **SSH keys in 1Password**: Private keys never stored on disk
- **Environment variables**: Store in 1Password or secure password manager
- **Never commit**: `.env` files, API keys, tokens, passwords

---

## 📖 Additional Resources

- **yadm Documentation**: https://yadm.io/docs/getting_started
- **1Password SSH Agent**: https://developer.1password.com/docs/ssh/
- **OpenCode Documentation**: https://opencode.ai/docs
- **Superpowers (Original)**: https://github.com/obra/superpowers

---

## 💡 Quick Reference

```bash
# View yadm-managed files
yadm ls-files

# Check yadm status
yadm status

# Edit yadm config
yadm gitconfig --list

# Bootstrap script (if you create one)
yadm bootstrap

# Test SSH with verbose output
ssh -vT git@github.com

# List SSH keys in 1Password agent
ssh-add -l

# Sign in to 1Password CLI
op signin
```

---

## ✅ Post-Restore Checklist

After running through the restore process, verify:

- [ ] yadm dotfiles cloned and applied
- [ ] Shell configuration loaded (SSH_AUTH_SOCK set)
- [ ] 1Password SSH agent working (`ssh -T git@github.com` succeeds)
- [ ] Superpowers cloned and on correct branch
- [ ] Workspace cloned with submodules initialized
- [ ] OpenCode loads custom configuration
- [ ] Environment variables set (check MCP servers work)
- [ ] Git operations work (pull/push)
- [ ] All custom commands available in OpenCode

---

## 🤝 Contributing

This is a personal dotfiles repository. If you're setting up your own:

1. Fork this repo
2. Customize configurations for your setup
3. Update repository URLs in this README
4. Remove/modify workspace-specific sections

---

## 📝 License

This is personal configuration. Use at your own risk. No warranty provided.

---

**Last Updated:** February 2026  
**Maintained by:** Filipe Estacio  
**Contact:** Via GitHub issues on this repository
