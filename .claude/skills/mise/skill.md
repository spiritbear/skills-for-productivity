---
name: mise
description: Install Python, Ruby, or Elixir runtimes using mise
argument-hint: python|ruby|elixir
---

Install a language runtime using mise. Supported languages: `python`, `ruby`, `elixir`.

## Step 1: Check mise is installed

```bash
which mise
```

If not found, install it:
```bash
brew install mise
```

Then verify mise is activated in `~/.zshrc`. If the following line is missing, add it:
```bash
eval "$(mise activate zsh)"
```

## Step 2: Validate the argument

$ARGUMENTS must be one of: `python`, `ruby`, or `elixir`.

If $ARGUMENTS is empty or not one of those three, stop and ask the user which language they want to install.

## Step 3: Check if the language is already installed

```bash
mise list $ARGUMENTS
```

If a version is already installed and set as the global default, inform the user and ask if they'd like to install a different version or upgrade.

## Step 4: Determine the version to install

If the user did not specify a version, install the latest stable version:
- Python: `mise use --global python@latest`
- Ruby: `mise use --global ruby@latest`
- Elixir: `mise use --global elixir@latest`

If the user specified a version (e.g. `python@3.12`), use that instead.

For Elixir, also install Erlang as it is required:
```bash
mise use --global erlang@latest
mise use --global elixir@latest
```

## Step 5: Install the runtime

If $ARGUMENTS already contains a version specifier (e.g. `python@3.12` or `python@latest`), use it directly:
```bash
mise use --global $ARGUMENTS
```

If $ARGUMENTS is just a language name with no version (e.g. `python`), append `@latest`:
```bash
mise use --global $ARGUMENTS@latest
```

This installs the runtime and sets it as the global default.

## Step 6: Verify the installation

Run the appropriate version command to confirm the install succeeded:

- Python:
  ```bash
  python --version
  ```
- Ruby:
  ```bash
  ruby --version
  ```
- Elixir:
  ```bash
  elixir --version
  ```

If the version command fails, remind the user to open a new terminal or run `eval "$(mise activate zsh)"` in the current session.
