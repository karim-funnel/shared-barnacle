# Codex on evroc Think (GLM 5.3)

Run OpenAI Codex CLI against evroc Think models via a `codex-evroc` shell function, leaving plain `codex` untouched.

## 1. Install Codex

```bash
brew install codex
```

## 2. Install the evroc CLI and log in

Skip if already done (see `claude-code-on-evroc.md`).

```bash
curl -fsSL https://docs.evroc.com/install.sh | bash
evroc login
```

Optionally `export EVROC_API_KEY="..."` in your shell config. Without it, the CLI creates a key from your login session.

## 3. Add a function to `~/.zshrc`

```zsh
codex-evroc() {
  evroc think code codex --model zai-org/GLM-5.3 -- "$@"
}
```

Then `source ~/.zshrc` and run `codex-evroc`. Anything after the command goes to Codex, e.g. `codex-evroc exec "refactor this"`.

Codex only speaks the Responses API, which evroc's public endpoint doesn't expose. `evroc think code codex` starts a local proxy that translates, and sets the environment variables for that session only. No config files are written, so plain `codex` is unaffected.

## Gotchas

- **Use the default `~/.codex`.** A separate `CODEX_HOME` with no login makes Codex ignore the proxy and go to `api.openai.com` (401).
- **Use the alias name for `--model`.** `zai-org/GLM-5.3`, not the ID `nvidia-glm-5-3-nvfp4`.
- **Don't point Codex at `https://models.think.evroc.com/v1` yourself.** There is no `/v1/responses` endpoint (404), and current Codex dropped `wire_api = "chat"`.
- **Skip `evroc think code init codex`.** It writes a permanent `~/.codex/config.toml`, which would overwrite existing profiles, and per the docs it doesn't run the proxy.
- **"Model metadata not found" on startup is harmless.**
- **Protect your API key.** Keep `~/.zshrc` out of public dotfiles repos.
