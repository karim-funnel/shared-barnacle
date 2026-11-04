# Claude Code on evroc Think (GLM 5.3)

Run the Claude Code harness against evroc Think models via a `claude-evroc` shell function, leaving plain `claude` untouched.

## 1. Install the evroc CLI and log in

```bash
curl -fsSL https://docs.evroc.com/install.sh | bash
evroc login            # opens the browser; pick your org
```

If you have the legacy evroc CLI, uninstall it first. Both versions share `~/.evroc/config.yaml`.

## 2. Get an API key (optional)

Create a key for the Think shared models in the evroc console and put it in your shell config:

```bash
export EVROC_API_KEY="..."
```

Without it, the CLI auto-creates a key from your login session. The key's name in the console is only a label.

## 3. Add a function to `~/.zshrc`

```zsh
claude-evroc() {
  # In $HOME, ~/.claude/ counts as project settings, so ignore project/local sources there
  local extra=()
  [[ "$PWD" == "$HOME" ]] && extra=(-- --setting-sources user)
  CLAUDE_CONFIG_DIR="$HOME/.claude-evroc" \
    evroc think code claude --model zai-org/GLM-5.3 "${extra[@]}" "$@"
}
```

Then `source ~/.zshrc` and run `claude-evroc`.

`evroc think code claude` sets `ANTHROPIC_BASE_URL` and the API key for Claude Code and starts a local Anthropic-to-OpenAI translation proxy. You don't configure an endpoint yourself.

## Gotchas

- **Use the alias name for `--model`, not the listed ID.** `evroc think sharedmodel list` shows `nvidia-glm-5-3-nvfp4`, but the proxy rejects it. Use `zai-org/GLM-5.3`. `evroc think sharedmodel list -o json` shows each model's aliases.
- **Use a separate `CLAUDE_CONFIG_DIR` per backend** (here `~/.claude-evroc`). Otherwise your normal `~/.claude/settings.json` leaks in. In our case that was `CLAUDE_CODE_USE_BEDROCK=1`, which sent requests to Bedrock and produced a 400 "The provided model identifier is invalid" error that looks like a bad model name.
- **Launching from `$HOME` leaks settings anyway.** There, `~/.claude/` is read as project settings, hence `--setting-sources user` in the function.
- **Don't switch models with `/model` in a leaking session.** It can persist `"model": "zai-org/GLM-5.3"` into the normal `~/.claude/settings.json` and break plain `claude`. Remove that line if it happens.
- **Protect your API key.** Keep `~/.zshrc` out of public dotfiles repos, or move the key to an untracked file.
