---
name: leaked-key-finder
description: Find API keys, passwords, tokens, and private keys in code, config, logs, and git history that would leak once deployed or pushed. Use before every deploy, before pushing to a shared or public repo, and after AI-generated code adds configuration.
---

# Leaked Key Finder

Secrets that reach a public repo, a client-side bundle, or a container image get scraped within minutes. This skill finds them before they leave your machine.

## When to use

Run before every push to a shared or public repo, before every deploy, and whenever the AI added configuration, API clients, or example code.

## Steps

1. **Scan the working tree.** Use ripgrep or grep. Skip `node_modules`, `vendor`, lockfiles, and build output. Search for these patterns:

   | Secret type | Pattern to search |
   |---|---|
   | AWS access key | `AKIA[0-9A-Z]{16}`, `ASIA[0-9A-Z]{16}` |
   | AWS secret | `aws_secret_access_key`, 40-character base64 next to an AWS key name |
   | GitHub token | `ghp_[A-Za-z0-9]{36}`, `github_pat_`, `gho_`, `ghs_` |
   | Stripe | `sk_live_`, `rk_live_`, `whsec_` |
   | OpenAI | `sk-[A-Za-z0-9_-]{20,}`, `sk-proj-` |
   | Anthropic | `sk-ant-` |
   | Google API | `AIza[0-9A-Za-z_-]{35}` |
   | Slack | `xox[baprs]-` |
   | Private keys | `-----BEGIN (RSA |EC |OPENSSH |DSA |PGP )?PRIVATE KEY-----` |
   | JWT | `eyJ[A-Za-z0-9_-]{10,}\.eyJ` |
   | Connection strings with a password | `://[^/\s:]+:[^@\s]+@` (for example `postgres://user:pass@host`) |
   | Generic assignments | `(password|passwd|secret|token|api[_-]?key|auth)\s*[:=]\s*["'][^"']{8,}["']` |

   Also check the names of files: `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa`, `credentials.json`, `service-account*.json`, `terraform.tfstate`, `*.sqlite` backups.

2. **Check whether secrets are ignored by git.** Confirm `.gitignore` covers env files and key files. Run `git ls-files` to see whether any such file is already tracked. A file that is ignored now but was committed earlier is still in history.

3. **Check git history.** Run `git log -p --all` or use a scanner if one is installed (`gitleaks`, `trufflehog`, `detect-secrets`). If none is installed, say so and suggest installing one. Do not install software without asking.

4. **Check where secrets can leak even if the repo is private.**
   - Client-side code: any variable with a public prefix (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `PUBLIC_`, `EXPO_PUBLIC_`) that holds a secret is shipped to every browser.
   - Docker: `ENV` or `ARG` lines with secrets, or `COPY . .` that copies `.env` into the image.
   - CI and config files: hardcoded values in workflow YAML, Helm values, Kubernetes manifests, or Terraform.
   - Logs and error messages: code that logs full headers, request bodies, or config objects.
   - Test fixtures, example files, and docs: real keys pasted as examples. Example files should use placeholders.

5. **Report, with values redacted.** Never print a full secret. Show only the first 4 and last 4 characters, or the length.

   | Severity | Location | Type | Value (redacted) | Action |
   |---|---|---|---|---|
   | Critical | `src/config.js:7` | Stripe live key | `sk_l...x9Qa` | Rotate, remove from code, load from env |
   | High | `.env` tracked in git | Database URL with password | `postgres://u:...` | Add to `.gitignore`, remove from history |

   Severity: **Critical** for live production credentials; **High** for any other real credential; **Medium** for placeholders that look real or for risky patterns with no confirmed value.

## If a real secret is found

1. **Treat it as compromised.** Removing it from the code is not enough. Once pushed, assume it was copied.
2. **Rotate it** at the provider (revoke the old key and create a new one). This is the user's action in their provider console. Tell the user what to rotate.
3. **Remove it from the code** and load it from an environment variable or a secret manager.
4. **Remove it from git history** if it was committed. Options include `git filter-repo` or BFG. This rewrites history, so confirm with the user first and warn that collaborators must re-clone.
5. **Never test a found key** against the live service to see if it works.

## Rules

- Do not print full secret values, in chat or in files you write.
- Do not commit, push, rotate, or rewrite history without the user's approval.
- If nothing is found, say which patterns and paths were checked, so the result can be trusted.
