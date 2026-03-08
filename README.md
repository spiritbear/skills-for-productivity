# Skills for Productivity

A collection of Claude Code skills for setting up and managing a development environment.

## 1Password Setup

### Overview

1Password is used as the single source of truth for all secrets and credentials. The setup has two layers:

1. **Global environment variables** — injected into every shell session from a 1Password Environment
2. **CLI tool credentials** — injected per-command via 1Password Shell Plugins

### Prerequisites

- [1Password desktop app](https://1password.com/downloads/)
- 1Password CLI beta (see below)
- Enable the CLI integration in the 1Password desktop app: **Settings → Developer → Command-Line Interface**

### Installing 1Password CLI (Beta Required)

The stable 1Password CLI (available via `brew install --cask 1password-cli`) does **not** include the `op environment` command, which is needed to inject global environment variables into the shell. The `op environment` command was introduced in **v2.33.0-beta.02** and is currently only available in the beta release.

Install the beta via Homebrew:

```bash
brew install --cask 1password-cli@beta
```

If you already have the stable version installed, uninstall it first:

```bash
brew uninstall 1password-cli
brew install --cask 1password-cli@beta
```

Verify the version is 2.33.0-beta.02 or later:

```bash
op --version
```

> Once `op environment` lands in the stable release, the `@beta` cask can be replaced with the standard `1password-cli` cask.

### Global Environment Variables

Secrets are stored in a **1Password Environment** (Private vault → Developer → Environments). On every new shell session, they are automatically exported by adding the following to `~/.zshrc`:

```bash
source <(op environment read <ENVIRONMENT_ID> 2>/dev/null)
```

This means secrets like `OPENAI_API_KEY`, `TODOIST_API_TOKEN`, etc. are never hardcoded in shell config files.

To add a new global secret:
1. Open the 1Password desktop app
2. Go to **Developer → View Environments** → select your environment
3. Add the key-value pair
4. It will be available in all new shell sessions automatically

### GitHub CLI Authentication

The `gh` CLI is authenticated via a 1Password Shell Plugin. This means:

- `gh` is aliased to `op plugin run -- gh` in `~/.config/op/plugins.sh`
- A GitHub Personal Access Token (PAT) is stored in 1Password (Private vault → Github → Access Token)
- 1Password injects the token as `GITHUB_TOKEN` at runtime whenever `gh` is called

To set this up on a new machine, run:

```bash
op plugin init gh
```

This will prompt you to create a new PAT on GitHub and store it in 1Password automatically.

To verify authentication is working:

```bash
gh auth status
```

### Adding New CLI Tool Credentials

1Password Shell Plugins support many CLI tools beyond `gh`. To add a new one:

```bash
op plugin init <tool-name>
```

This stores the credential in 1Password and adds an alias to `~/.config/op/plugins.sh`.

## Runtime Version Management (mise)

[mise](https://mise.jdx.dev/) is used to manage Python, Node, and Ruby versions. It replaces `asdf` and is a faster, drop-in alternative.

### Installation

```bash
brew install mise
```

Activate mise in your shell by adding this to `~/.zshrc`:

```bash
eval "$(mise activate zsh)"
```

### Usage

Install and set a global runtime version:

```bash
mise use --global node@lts
mise use --global python@3.12
mise use --global ruby@3.3
```

Set a version for a specific project (creates a `.mise.toml` in the project root):

```bash
mise use node@20
```

List installed runtimes:

```bash
mise list
```

### Migrating from asdf

mise reads `.tool-versions` files, so existing asdf configurations work without changes. To fully migrate, remove the asdf plugin from `~/.zshrc` and replace the asdf activation line with `eval "$(mise activate zsh)"`.

## Skills

### `/homebrew`

Sets up Homebrew, installs default packages (`gh`), and configures GitHub authentication via 1Password.

```
/homebrew [package1] [package2] ...
```
