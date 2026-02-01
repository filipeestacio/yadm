# System Installation Guide

This guide will help you (or an AI agent) set up a fresh Linux system with all the development tools and CLI utilities I use daily.

**System:** Linux (distro-agnostic)  
**Package Managers:** Detects apt, dnf, pacman, zypper automatically  
**Shell:** zsh with starship.rs  
**Terminal Multiplexer:** tmux + sesh  
**Shell History:** atuin  
**Container Runtime:** Podman (Docker-compatible)

---

## 🎯 Overview

This installation guide is organized into sections that can be executed sequentially or in parallel (where noted).

**Estimated Total Time:** 30-45 minutes

---

## 📋 Prerequisites & Package Manager Detection

This script will detect your distribution and use the appropriate package manager:

```bash
# Detect package manager
detect_package_manager() {
  if command -v apt-get &> /dev/null; then
    PKG_MANAGER="apt"
    PKG_INSTALL="sudo apt-get install -y"
    PKG_UPDATE="sudo apt-get update && sudo apt-get upgrade -y"
  elif command -v dnf &> /dev/null; then
    PKG_MANAGER="dnf"
    PKG_INSTALL="sudo dnf install -y"
    PKG_UPDATE="sudo dnf upgrade -y"
  elif command -v pacman &> /dev/null; then
    PKG_MANAGER="pacman"
    PKG_INSTALL="sudo pacman -S --noconfirm"
    PKG_UPDATE="sudo pacman -Syu --noconfirm"
  elif command -v zypper &> /dev/null; then
    PKG_MANAGER="zypper"
    PKG_INSTALL="sudo zypper install -y"
    PKG_UPDATE="sudo zypper update -y"
  else
    echo "Error: No supported package manager found!"
    exit 1
  fi
  echo "Detected package manager: $PKG_MANAGER"
}

# Run detection
detect_package_manager

# Update system
$PKG_UPDATE

# Install base development tools
if [ "$PKG_MANAGER" = "apt" ]; then
  sudo apt-get install -y build-essential git curl wget
elif [ "$PKG_MANAGER" = "dnf" ]; then
  sudo dnf groupinstall -y "Development Tools"
  sudo dnf install -y git curl wget
elif [ "$PKG_MANAGER" = "pacman" ]; then
  sudo pacman -S --noconfirm base-devel git curl wget
elif [ "$PKG_MANAGER" = "zypper" ]; then
  sudo zypper install -y -t pattern devel_basis
  sudo zypper install -y git curl wget
fi
```

---

## 1. Package Manager Setup

### Arch Linux - Install yay (AUR Helper)

```bash
# Only for Arch-based systems
if [ "$PKG_MANAGER" = "pacman" ]; then
  if ! command -v yay &> /dev/null; then
    cd /tmp
    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si --noconfirm
    cd ..
    rm -rf yay
  fi
  yay --version
fi
```

---

## 2. Core System Utilities

Install essential CLI tools used daily:

```bash
# Helper function to install packages across distros
install_pkg() {
  local pkg=$1
  case "$PKG_MANAGER" in
    apt)
      # Some packages have different names on Debian/Ubuntu
      case "$pkg" in
        fd) pkg="fd-find" ;;
        ripgrep) pkg="ripgrep" ;;
        bat) pkg="bat" ;;
      esac
      ;;
    dnf)
      # Fedora package names
      case "$pkg" in
        fd) pkg="fd-find" ;;
      esac
      ;;
  esac
  $PKG_INSTALL "$pkg"
}

# Modern replacements for classic Unix tools
# Note: Some tools may need cargo installation if not in repos
for pkg in eza bat fd ripgrep fzf jq btop tmux zoxide tree ncdu unzip p7zip; do
  case "$PKG_MANAGER" in
    pacman)
      yay -S --noconfirm "$pkg"
      ;;
    *)
      install_pkg "$pkg" || echo "Warning: $pkg not found in repos, may need cargo install"
      ;;
  esac
done

# Install cargo for Rust-based tools if needed
if ! command -v cargo &> /dev/null; then
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
  source "$HOME/.cargo/env"
fi

# Install tools that might not be in package repos via cargo
cargo install --locked \
  eza \
  bat \
  fd-find \
  ripgrep \
  du-dust \
  procs \
  hyperfine \
  tokei \
  yq

# Note: duf, btop may still need distro-specific installation
```

---

## 3. Shell Setup

### Install Zsh and Starship

```bash
# Install zsh
$PKG_INSTALL zsh

# Set zsh as default shell
chsh -s $(which zsh)

# Install Starship prompt
curl -sS https://starship.rs/install.sh | sh

# Create starship config directory
mkdir -p ~/.config

# Add starship init to zshrc (will be in your dotfiles, but for reference):
# echo 'eval "$(starship init zsh)"' >> ~/.zshrc

# Optional: Install zsh plugins
# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions

# zsh-syntax-highlighting  
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.zsh/zsh-syntax-highlighting

# Add to ~/.zshrc:
# source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
# source ~/.zsh/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

---

## 4. Shell History with Atuin

```bash
# Install atuin (try package manager first, then cargo)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm atuin
    ;;
  apt)
    # Debian/Ubuntu - use cargo or download binary
    cargo install atuin
    ;;
  dnf)
    # Fedora
    $PKG_INSTALL atuin
    ;;
  *)
    cargo install atuin
    ;;
esac

# Initialize atuin for zsh
atuin init zsh > ~/.atuin-init.zsh

# Add to ~/.zshrc (will be restored from dotfiles, but for reference):
# eval "$(atuin init zsh)"

# Sign up for atuin sync (optional but recommended)
# atuin register -u <your-username> -e <your-email>
# atuin login

# Or import existing atuin config after yadm restore
```

---

## 5. Terminal Multiplexer Setup

### Tmux + Sesh

```bash
# Install tmux
$PKG_INSTALL tmux

# Install sesh (session manager)
# Try package manager first, then cargo
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm sesh
    ;;
  *)
    # Install via cargo if not in repos
    cargo install sesh
    ;;
esac

# Install TPM (Tmux Plugin Manager)
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# Note: Your tmux.conf will be restored from dotfiles
# After restore, start tmux and press: Ctrl+b I (capital i) to install plugins
```

---

## 6. Git Tools

```bash
# Install git
$PKG_INSTALL git

# Install git-delta (better git diff)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm git-delta
    ;;
  apt)
    cargo install git-delta
    ;;
  dnf)
    $PKG_INSTALL git-delta
    ;;
  *)
    cargo install git-delta
    ;;
esac

# Install lazygit (terminal UI)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm lazygit
    ;;
  apt)
    # Ubuntu/Debian - use PPA or download binary
    LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
    curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
    tar xf lazygit.tar.gz lazygit
    sudo install lazygit /usr/local/bin
    rm lazygit lazygit.tar.gz
    ;;
  dnf)
    sudo dnf copr enable atim/lazygit -y
    $PKG_INSTALL lazygit
    ;;
  *)
    cargo install lazygit
    ;;
esac

# Install GitHub CLI
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm github-cli
    ;;
  apt)
    curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
    sudo apt update
    $PKG_INSTALL gh
    ;;
  dnf)
    $PKG_INSTALL gh
    ;;
  *)
    $PKG_INSTALL gh
    ;;
esac

# Install tig (if desired)
$PKG_INSTALL tig

# Configure git (will be restored from dotfiles, but for reference)
# git config --global user.name "Your Name"
# git config --global user.email "your.email@example.com"
# git config --global core.pager delta
# git config --global interactive.diffFilter "delta --color-only"
```

---

## 7. Development Tools

### Node.js (via NVM)

```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Load NVM (or restart shell)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Install Node.js LTS
nvm install 24
nvm use 24
nvm alias default 24

# Install global packages
npm install -g \
  pnpm \          # Fast package manager
  yarn \          # Alternative package manager
  npm-check-updates  # Update package.json dependencies
```

### Bun

```bash
# Install bun (fast JavaScript runtime and toolkit)
curl -fsSL https://bun.sh/install | bash

# Verify installation
bun --version
```

### Python Tools

```bash
# Install Python and tools
yay -S --noconfirm \
  python \
  python-pip \
  python-pipx \
  uv              # Fast Python package installer

# Install pipx packages
pipx install poetry
pipx install black
pipx install ruff
pipx install httpie
```

---

## 8. Cloud & Infrastructure Tools

### AWS CLI

```bash
# Install AWS CLI v2 (distro-agnostic)
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# Verify installation
aws --version

# Install aws-vault (optional, for credential management)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm aws-vault
    ;;
  apt|dnf)
    # Install from GitHub releases
    AWS_VAULT_VERSION=$(curl -s https://api.github.com/repos/99designs/aws-vault/releases/latest | grep -Po '"tag_name": "v\K[^"]*')
    curl -L "https://github.com/99designs/aws-vault/releases/download/v${AWS_VAULT_VERSION}/aws-vault-linux-amd64" -o aws-vault
    chmod +x aws-vault
    sudo mv aws-vault /usr/local/bin/
    ;;
esac
```

### Podman (Docker Alternative)

```bash
# Install Podman - a daemonless Docker alternative
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm podman podman-compose podman-docker
    ;;
  apt)
    $PKG_INSTALL podman podman-compose
    # Install docker CLI compatibility package
    sudo apt-get install -y podman-docker
    ;;
  dnf)
    $PKG_INSTALL podman podman-compose podman-docker
    ;;
  zypper)
    $PKG_INSTALL podman podman-compose
    ;;
esac

# Enable and start podman socket (for docker-compose compatibility)
systemctl --user enable --now podman.socket

# Create docker alias for podman (if not using podman-docker package)
# echo 'alias docker=podman' >> ~/.zshrc

# Set up rootless mode
podman system migrate

# Verify installation
podman --version
podman run hello-world

# Note: podman-docker package provides 'docker' command that calls podman
# No need to add user to docker group (rootless by default)

# Optional: Install lazydocker for TUI management
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm lazydocker
    ;;
  *)
    # Install via Go
    go install github.com/jesseduffield/lazydocker@latest
    # Or download binary
    curl -Lo lazydocker.tar.gz https://github.com/jesseduffield/lazydocker/releases/latest/download/lazydocker_*_Linux_x86_64.tar.gz
    tar xf lazydocker.tar.gz lazydocker
    sudo install lazydocker /usr/local/bin
    rm lazydocker lazydocker.tar.gz
    ;;
esac

# Note: lazydocker works with podman when DOCKER_HOST is set:
# export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
```

### Tailscale

```bash
# Install Tailscale VPN (distro-agnostic)
curl -fsSL https://tailscale.com/install.sh | sh

# Start and enable service
sudo systemctl start tailscaled
sudo systemctl enable tailscaled

# Authenticate (will open browser)
sudo tailscale up
```

---

## 9. Code Editors & IDEs

```bash
# Install Zed editor (distro-agnostic)
curl -f https://zed.dev/install.sh | sh

# Install Obsidian (note-taking)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm obsidian
    ;;
  apt)
    # Download from website
    wget "https://github.com/obsidianmd/obsidian-releases/releases/latest/download/obsidian_*_amd64.deb"
    sudo dpkg -i obsidian_*_amd64.deb
    sudo apt-get install -f -y
    rm obsidian_*_amd64.deb
    ;;
  dnf)
    # Download AppImage or Flatpak
    flatpak install -y flathub md.obsidian.Obsidian
    ;;
esac
```

---

## 10. AI Development Tools

### OpenCode

```bash
# Install OpenCode AI coding assistant
# Follow instructions at: https://opencode.ai/docs/installation

# The installation script will:
# - Install OpenCode CLI
# - Set up configuration directory at ~/.config/opencode/

# After installation, restore your config from yadm dotfiles
```

### MCP Servers (for OpenCode)

```bash
# Install Node.js MCP servers (via npx/uvx as needed)
# These are defined in your opencode.jsonc and loaded on demand

# Chrome DevTools MCP (already in your config)
# npx -y chrome-devtools-mcp@latest

# Install uv for Python-based MCP servers
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## 11. 1Password

```bash
# Install 1Password (distro-specific)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm 1password 1password-cli
    ;;
  apt)
    # Add 1Password repository
    curl -sS https://downloads.1password.com/linux/keys/1password.asc | sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/$(dpkg --print-architecture) stable main" | sudo tee /etc/apt/sources.list.d/1password.list
    sudo mkdir -p /etc/debsig/policies/AC2D62742012EA22/
    curl -sS https://downloads.1password.com/linux/debian/debsig/1password.pol | sudo tee /etc/debsig/policies/AC2D62742012EA22/1password.pol
    sudo mkdir -p /usr/share/debsig/keyrings/AC2D62742012EA22
    curl -sS https://downloads.1password.com/linux/keys/1password.asc | sudo gpg --dearmor --output /usr/share/debsig/keyrings/AC2D62742012EA22/debsig.gpg
    sudo apt update
    $PKG_INSTALL 1password 1password-cli
    ;;
  dnf)
    # Add 1Password repository
    sudo rpm --import https://downloads.1password.com/linux/keys/1password.asc
    sudo sh -c 'echo -e "[1password]\nname=1Password Stable Channel\nbaseurl=https://downloads.1password.com/linux/rpm/stable/\$basearch\nenabled=1\ngpgcheck=1\nrepo_gpgcheck=1\ngpgkey=\"https://downloads.1password.com/linux/keys/1password.asc\"" > /etc/yum.repos.d/1password.list'
    $PKG_INSTALL 1password 1password-cli
    ;;
esac

# Enable 1Password SSH agent (after installing desktop app):
# 1. Open 1Password
# 2. Settings → Developer
# 3. Enable "Use the SSH agent"

# Verify op CLI
op --version
```

---

## 12. Dotfiles & Configuration Management

```bash
# Install yadm (distro-agnostic)
case "$PKG_MANAGER" in
  pacman)
    yay -S --noconfirm yadm
    ;;
  apt)
    $PKG_INSTALL yadm
    ;;
  dnf)
    $PKG_INSTALL yadm
    ;;
  *)
    # Install manually
    curl -fLo /usr/local/bin/yadm https://github.com/TheLocehiliosan/yadm/raw/master/yadm
    chmod a+x /usr/local/bin/yadm
    ;;
esac

# Clone your dotfiles (do this AFTER 1Password SSH agent is set up)
yadm clone git@github.com:filipeestacio/dotfiles.git

# If there are conflicts, backup and overwrite:
# yadm ls-files | xargs -I{} cp --parents {} ~/dotfiles-backup/
# yadm reset --hard origin/main

# Reload shell configuration
source ~/.zshrc
```

---

## 13. Additional Tools from Atuin History

Based on your command history, you also use these tools:

```bash
# Backlog.md MCP server (task management)
# This should be configured via your OpenCode config

# Additional utilities
$PKG_INSTALL \
  vim \
  neovim \
  man-db \
  htop \
  rsync

# Note: which command should be available by default on most systems
```

---

## 14. Custom Scripts and Tools

Based on your atuin history, you have custom scripts in `~/LocalTools/Scripts/`:

```bash
# These should be restored from your dotfiles/backup
# Expected scripts:
# - ask.sh
# - process.sh
# - talk.sh
# - todo.sh

# If not in dotfiles, you may need to restore from backup
# They are aliased in your shell config as: ask, process, talk, todo
```

---

## 15. Post-Installation Verification

Run these commands to verify everything is installed:

```bash
# Core tools
command -v eza fd rg fzf jq bat btop tmux

# Development
command -v node npm pnpm bun python3 pipx uv

# Git tools
command -v git delta lazygit gh

# Cloud tools
command -v aws podman op

# Editors
command -v zed opencode

# Shell tools
command -v zsh starship atuin sesh yadm

# Print versions
echo "Node: $(node --version)"
echo "pnpm: $(pnpm --version)"
echo "Python: $(python3 --version)"
echo "AWS CLI: $(aws --version)"
echo "Podman: $(podman --version)"
echo "Git: $(git --version)"
echo "Atuin: $(atuin --version)"
echo "Starship: $(starship --version)"
```

---

## 16. Restore Your Configurations

After installing all tools, restore your configurations:

```bash
# 1. SSH keys via 1Password (already done in step 11)

# 2. Dotfiles via yadm (already done in step 12)

# 3. Superpowers
git clone git@github.com:filipeestacio/superpowers.git ~/.config/opencode/superpowers
cd ~/.config/opencode/superpowers
git checkout filipe/opencode-update-plan

# 4. Workspace
git clone git@github.com:MyUtilityConnection/workspace.git ~/code/myutilityconnection
cd ~/code/myutilityconnection
git submodule update --init --recursive

# 5. Install OpenCode dependencies
cd ~/.config/opencode
npm install  # or bun install

# 6. Atuin sync (if you use it)
# atuin login
# atuin sync
```

---

## 🤖 AI Agent Instructions

If you're an AI agent setting up this system:

### Execution Order

1. **Run Prerequisites** (Package manager detection) - REQUIRED FIRST
2. **Run sections 1-2** (Package manager setup + Core utilities) - Can run in parallel
3. **Run sections 3-5** (Shell + Atuin + Tmux) - Sequential
4. **Run sections 6-10** (Dev tools) - Can run in parallel
5. **Run section 11** (1Password) - **REQUIRED before section 12**
6. **Run section 12** (Dotfiles) - **MUST run after 1Password**
7. **Run sections 13-14** (Additional tools) - Can run in parallel
8. **Run sections 15-16** (Verification + Restore) - Sequential

### Error Handling

- If a package fails to install, try the cargo/manual installation method
- If NVM installation fails, manual install: `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash`
- If Podman socket isn't working: `systemctl --user restart podman.socket`
- If 1Password SSH agent isn't working: Verify `~/.1password/agent.sock` exists and `SSH_AUTH_SOCK` is set
- For Arch Linux, if yay fails to build: Install manually with `makepkg -si`

### Distro-Specific Notes

**Debian/Ubuntu (apt):**
- Some packages may need `sudo add-apt-repository universe` first
- fd-find is called `fdfind` instead of `fd` (create alias or symlink)
- bat might be called `batcat` (create alias)

**Fedora/RHEL (dnf):**
- Enable EPEL repository: `sudo dnf install epel-release`
- Some tools may need COPR repos

**Arch Linux (pacman):**
- Use `yay` for AUR packages
- Most modern CLI tools available in AUR

### User Interaction Required

These steps require user interaction and cannot be fully automated:

1. **1Password desktop app**: User must sign in manually
2. **1Password SSH agent**: User must enable in Settings → Developer
3. **Tailscale**: User must authenticate via browser
4. **Atuin sync** (optional): User must register/login
5. **AWS credentials**: User must configure via `aws configure` or environment variables

### Verification Commands

After installation, run the verification commands in section 15 and report any missing tools.

---

## 📊 Tool Categories Summary

### Package Management
- Distro package managers (apt/dnf/pacman/zypper)
- cargo (Rust tools)
- yay (Arch AUR helper, Arch only)

### Modern CLI Replacements
- `eza` → ls
- `bat` → cat
- `fd` → find
- `rg` (ripgrep) → grep
- `zoxide` (z) → cd
- `delta` → git diff
- `btop` → top/htop
- `duf` → df
- `dust` → du

### Development
- `node` + `npm` + `pnpm` (via nvm)
- `bun`
- `python3` + `pipx` + `uv`

### Git Tools
- `git` + `delta` + `lazygit` + `gh`

### Cloud & Infrastructure
- `aws` + `podman` + `tailscale`

### Shell Environment
- `zsh` + `starship`
- `atuin` (shell history)
- `tmux` + `sesh` (terminal multiplexer)
- `fzf` (fuzzy finder)

### Editors & IDEs
- `zed` + `opencode`
- `obsidian` (notes)

### Security & Secrets
- `1password` + `1password-cli` (op)

### Configuration Management
- `yadm` (dotfiles)

---

## 🔗 Additional Resources

- **Starship Prompt**: https://starship.rs/
- **Modern Unix Tools**: https://github.com/ibraheemdev/modern-unix
- **Podman Documentation**: https://docs.podman.io/
- **Command Line Tools**: https://github.com/alebcay/awesome-shell

---

**Last Updated:** February 2026  
**System:** Linux (distro-agnostic)  
**Maintained by:** Filipe Estacio
