---
name: homebrew
description: Set up Homebrew and install packages on macOS
argument-hint: [package1] [package2] ...
---

Set up Homebrew on macOS and optionally install packages.

## Step 1: Check if Homebrew is installed

Run:
```bash
which brew
```

If it exits successfully, Homebrew is already installed — skip to Step 2.

If not found, install Homebrew using the official install script:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After installation, check if brew needs to be added to the shell PATH. On Apple Silicon Macs, the install script will output instructions to add brew to the PATH — follow them. The typical commands are:
```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

On Intel Macs, Homebrew installs to `/usr/local/bin` which is already on PATH.

Verify the installation succeeded:
```bash
brew --version
```

## Step 2: Install default packages

For each of the following packages, check if it is already installed before installing:

```bash
brew list gh &>/dev/null || brew install gh
brew list mise &>/dev/null || brew install mise
brew list uv &>/dev/null || brew install uv
brew list ty &>/dev/null || brew install ty
```

## Step 3: Set up GitHub authentication via 1Password

Check if the user is already authenticated:
```bash
gh auth status
```

If not authenticated (non-zero exit), set up the 1Password shell plugin for `gh`. This will store a GitHub PAT in 1Password and inject it automatically:
```bash
op plugin init gh
```

When prompted:
- Choose to create a new Personal Access Token (PAT) on GitHub
- 1Password will store it and inject it as `GITHUB_TOKEN` via the `gh` alias in `~/.config/op/plugins.sh`

After setup, verify authentication succeeded:
```bash
gh auth status
```

## Step 4: Install additional packages (if $ARGUMENTS provided)

If the user passed package names as $ARGUMENTS, check each one and only install if not already installed:
```bash
for pkg in $ARGUMENTS; do brew list "$pkg" &>/dev/null || brew install "$pkg"; done
```
