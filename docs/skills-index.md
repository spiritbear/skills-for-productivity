# Skills Index

## homebrew

Sets up Homebrew on macOS if it isn't already installed, configures shell PATH integration, and installs a default set of CLI tools (`gh`, `mise`, `uv`, `ty`). Also wires up GitHub authentication for the `gh` CLI via the 1Password shell plugin, storing the PAT in 1Password and injecting `GITHUB_TOKEN` automatically. Accepts optional package names as arguments to install additional formulae beyond the defaults.

## mise

Installs a language runtime (Python, Ruby, or Elixir) via `mise` and sets it as the global default. Installs `mise` via Homebrew if missing and ensures shell activation is configured in `~/.zshrc`. Supports a version specifier (e.g. `python@3.12`) or defaults to `@latest`; for Elixir it also installs the required Erlang runtime.

## garmin-activity

Pulls a single Garmin Connect activity by ID using the `garmin` CLI and appends a formatted summary to the matching daily note in the user's Obsidian vault. Trigger phrases include "log my run", "log my hike", "fetch this activity", or providing a Garmin activity ID directly. Requires the user to already be authenticated with `garmin auth login`.

## tonal-workout

Pulls recent Tonal workout data via the `toneget` fetch script and writes detailed summaries (exercises, sets, reps, volume) into the matching daily notes. Accepts an optional argument: a number of days (`--days N`, defaults to 7) or a specific date (`YYYY-MM-DD`). Credentials are injected at runtime via the 1Password CLI from `op://Private/Tonal`.
