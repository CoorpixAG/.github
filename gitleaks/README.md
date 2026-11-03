# Gitleaks at Coorpix

Every pull request in CoorpixAG is scanned for secrets (API keys, passwords, private keys,
connection strings). The check is called **gitleaks** and is required before merge on every
repository. It looks only at the commits your pull request adds.

## When the check fails

1. Open the job log. It shows the file, the line and the rule. The value itself is redacted.
2. **A real secret:** remove it from the code, and **rotate it** at the provider (OpenAI, Azure,
   AWS, ...). Deleting it from the branch is not enough, because it is already in the pushed
   commits. Read it from the environment or Key Vault instead. Rewrite the branch
   (`git rebase -i`, then force-push the branch, never `main`) so the value does not reach `main`.
3. **Not a secret** (a test value, an example, a hash):
   - add `# gitleaks:allow` (or `// gitleaks:allow`) at the end of that line, or
   - add the finding's fingerprint to `.gitleaksignore` in the repository root.
   If a whole kind of false positive keeps coming back across repositories, change
   `gitleaks/gitleaks.toml` here instead.

## Catch it before you commit (recommended)

Install gitleaks once:

```bash
brew install gitleaks
```

Then, in each repository, add a pre-commit hook:

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
cfg=.gitleaks.toml
[ -f "$cfg" ] || { cfg=$(mktemp); curl -sSfL https://raw.githubusercontent.com/CoorpixAG/.github/main/gitleaks/gitleaks.toml -o "$cfg"; }
exec gitleaks git --pre-commit --staged --redact --no-banner --config "$cfg"
EOF
chmod +x .git/hooks/pre-commit
```

Repositories that already use the [pre-commit](https://pre-commit.com) framework can add this to
`.pre-commit-config.yaml` instead:

```yaml
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.30.1
  hooks:
    - id: gitleaks
```

## What the shared config allows

`gitleaks.toml` uses gitleaks' built-in rules and only takes out known false positives:

| Allowed | Why it is safe |
|---|---|
| `*-sealed.yaml` | SealedSecrets hold only ciphertext |
| Azurite emulator key | Published by Microsoft, opens nothing |
| Generic "api key" values in tests, docs (`.md`, `.mdx`) and `.env.*.example` | Only the catch-all rule; a real OpenAI, AWS, Azure, GitHub or private key is still reported by its own rule |
