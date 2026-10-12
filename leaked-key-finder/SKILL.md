---
name: leaked-key-finder
description: Find API keys, passwords, tokens, and private keys in code, config, logs, and git history that would leak once deployed or pushed. Use before every deploy, before pushing to a shared or public repo, and after AI-generated code adds configuration. Works across languages, including Python, Java, Node.js, IBM ESQL and message-flow properties, front-end (HTML, CSS, JavaScript), Django, and Flask.
---

# Leaked Key Finder

Secrets that reach a public repo, a client-side bundle, or a container image get scraped within minutes. This skill finds them before they leave your machine.

## When to use

Run this before every push to a shared or public repo, before every deploy, and whenever the AI added configuration, API clients, or example code.

## Steps

1. **Scan the working tree.** Use ripgrep or grep. Skip `node_modules`, `vendor`, `target`, `build`, `dist`, lockfiles, and binary files. Run each group below.

   **Known key formats (any language):**

   | Secret type | Pattern to search |
   |---|---|
   | AWS access key | `AKIA[0-9A-Z]{16}`, `ASIA[0-9A-Z]{16}` |
   | AWS secret | `aws_secret_access_key`, 40-character base64 next to an AWS key name |
   | GitHub token | `ghp_[A-Za-z0-9]{36}`, `github_pat_`, `gho_`, `ghs_` |
   | Stripe | `sk_live_`, `rk_live_`, `whsec_` |
   | OpenAI | `sk-[A-Za-z0-9_-]{20,}`, `sk-proj-` |
   | Anthropic | `sk-ant-` |
   | Google API (including Maps in HTML or JS) | `AIza[0-9A-Za-z_-]{35}` |
   | Slack | `xox[baprs]-` |
   | Private keys | `-----BEGIN ([A-Z]+ )?PRIVATE KEY-----` |
   | JWT | `eyJ[A-Za-z0-9_-]{10,}\.eyJ` |
   | Connection string with user and password | `://[^/\s:]+:[^@\s]+@` (for example `postgres://user:pass@host`) |

   **Names that hold secrets (use case-insensitive search, and allow identifier forms):**

   | Form | Example | Pattern idea |
   |---|---|---|
   | Assignment with `=` or `:` | `SECRET_KEY = "..."`, `api_key: "..."`, `app.secret_key = "dev"` (values of 3+ characters count) | `[A-Za-z_]*(secret|password|passwd|token|api[_-]?key)[A-Za-z_]*\s*[:=]\s*["'][^"']{3,}["']` |
   | Dictionary or hash key | `'PASSWORD': 'hunter2'` (Django), `"password" => "..."` (PHP) | `["']?[A-Za-z_]*(secret|password|passwd|token|api[_-]?key)[A-Za-z_]*["']?\s*[:=]?\s*["'][^"']{3,}["']` |
   | Space-separated declaration | `DECLARE dbPassword CHARACTER 'hunter2'` (ESQL), `SET x = 'hunter2'` | `(DECLARE|SET)\s+\S*(password|secret|token|key)\S*\s+(CHARACTER\s+)?'[^']{8,}'` |
   | Properties and env files (no quotes) | `OrderFlow#dbPassword=hunter2`, `db.password=hunter2`, `DB_PASSWORD=hunter2` | `(?i)[A-Za-z_.#]*(password|passwd|secret|token|api[_-]?key)[A-Za-z_.#]*\s*=\s*\S{8,}` |
   | Credentials in a URL query string | `jdbc:postgresql://host/db?user=app&password=hunter2` | `(?i)[?&](password|passwd|pwd|secret|token|api_?key)=[^&\s"']{6,}` |

   Also check the names of files: `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`, `id_rsa`, `credentials.json`, `service-account*.json`, `terraform.tfstate`, `*.sqlite` backups, `*.properties` files under `flow/`, `overrides/`, or `config/`.

2. **Check whether secrets are ignored by git.** Confirm `.gitignore` covers env files, key files, and keystores. Run `git ls-files` to see whether any such file is already tracked. A file that is ignored now but was committed earlier is still in history.

3. **Check git history.** Run `git log -p --all` or use a scanner if one is installed (`gitleaks`, `trufflehog`, `detect-secrets`). Also search for deleted secret files: `git log --all --diff-filter=D --name-only` and `git log --all -- '*.env' '*.properties' '*.jks'`. If no scanner is installed, say so and suggest installing one. Do not install software without asking.

4. **Check where secrets can leak even if the repo is private.**

   | Where | What to look for | Stack examples |
   |---|---|---|
   | Client-side code | Secrets in HTML, CSS, or JS files; variables with public prefixes (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `PUBLIC_`, `EXPO_PUBLIC_`) that hold a secret; `console.log` or `console.error` printing a secret; secrets in `<script>` tags | HTML, JS, React, Vue, Angular |
   | Logs | Logging, printing, or slicing a secret variable (`print(api_key)`, `System.out.println(password)`, `log.info(config)`, `STRIPE_KEY.slice(0, 8)`) | Python, Java, Node.js, ESQL (`LOG`, `PROPAGATE`), Django, Flask |
   | Docker | `ENV` or `ARG` lines with secrets; `COPY . .` with no `.dockerignore` that would copy `.env` or key files into the image | Any Dockerfile |
   | Java and Spring | Secrets in `application.properties`, `application.yml`, `pom.xml`, `build.gradle`, `*.jks`; `@Value("${...:secret}")` with a real default | Java, Spring |
   | Django | `SECRET_KEY`, `DATABASES` `PASSWORD`, `EMAIL_HOST_PASSWORD`, `AWS_SECRET_ACCESS_KEY` in `settings.py` | Django |
   | Flask | `app.secret_key = "..."`, `SQLALCHEMY_DATABASE_URI` with a password, `config.from_object` with secrets in the file | Flask |
   | IBM ESQL and App Connect | Passwords in `DECLARE` or `SET` statements inside `*.esql`; passwords in `*.properties` override files or `*.msgflow` properties; secrets in `*.bar` or `*.application` files; `ibmint` or `mqsi` commands with passwords on the command line | IBM ESQL, IBM App Connect, IIB |
   | CI and config | Hardcoded values in workflow YAML, Helm values, Kubernetes manifests, or Terraform | Any |
   | Test fixtures and docs | Real keys pasted as examples. Example files should use placeholders. | Any |

5. **Report, with values redacted.** Never print a full secret. Show only the first 4 and last 4 characters, or the length.

   | Severity | Location | Type | Value (redacted) | Action |
   |---|---|---|---|---|
   | Critical | `src/config.js:7` | Stripe live key | `sk_l...x9Qa` | Rotate, remove from code, load from env |
   | High | `.env` in git history | Database URL with password | `postgres://u:...` | Add to `.gitignore`, remove from history |
   | High | `flow/overrides.properties:2` | Message flow DB password | `hunt...2222` | Move to a secure property store, remove from file |

   Severity: **Critical** for live production credentials; **High** for any other real credential; **Medium** for placeholders that look real or for risky patterns with no confirmed value.

   **Known example values** (such as the AWS documentation example key `AKIAIOSFODNN7EXAMPLE`) should be listed as "example, not a real key" and not counted as Critical.

## If a real secret is found

1. **Treat it as compromised.** Removing it from the code is not enough. Once pushed, assume it was copied.
2. **Rotate it** at the provider (revoke the old key and create a new one). This is the user's action in their provider console. Tell the user what to rotate.
3. **Remove it from the code** and load it from an environment variable, a secret manager, or the platform's secure property store.
4. **Remove it from git history** if it was committed. Options include `git filter-repo` or BFG. This rewrites history, so confirm with the user first and warn that collaborators must re-clone.
5. **Never test a found key** against the live service to see if it works.

## Rules

- Do not print full secret values, in chat or in files you write.
- Do not commit, push, rotate, or rewrite history without the user's approval.
- If nothing is found, say which patterns, groups, and paths were checked, so the result can be trusted.
