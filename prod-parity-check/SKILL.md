---
name: prod-parity-check
description: Find code that works on a laptop but will break once deployed to production. Use before every deploy, release, or push to a production branch, in any language or hosting setup.
---

# Prod Parity Check

Local development hides many problems. This skill checks the code and deploy config for the differences that break production.

## When to use

Run before every deploy or release. It is most useful after the AI added new config, new environment variables, new file writes, new external URLs, or new dependencies.

## Steps

1. **Identify the target environment.** Find how the app is deployed: Dockerfiles, `docker-compose*.yml`, Kubernetes manifests (`k8s/`, `helm/`), `serverless.yml`, `vercel.json`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`, `app.yaml`, Terraform (`*.tf`), CI deploy jobs, or platform config. Note what runs in production: which command starts the app, which OS image, which port.

2. **Run the checks below.** Use search tools (grep or ripgrep) across the whole repo. Skip `node_modules`, `vendor`, build output, and virtualenvs.

   | Check | What to look for | Why it breaks |
   |---|---|---|
   | Environment variables | Every env var read in code (`process.env`, `os.environ`, `getenv`, `System.getenv`, `os.Getenv`, config loaders) must exist in the deploy config, with a documented default or a clear startup error | Missing var crashes or silently uses a dev value |
   | Hardcoded local addresses | `localhost`, `127.0.0.1`, `0.0.0.0`, `:3000`, `:8080` in code or config outside tests | Production cannot reach them |
   | Debug mode | `DEBUG=true`, `debug: true`, `NODE_ENV=development`, `APP_ENV=dev`, verbose logging, stack traces returned to users | Leaks data, slows the app |
   | Local file writes | Writing to relative paths, `./uploads`, `./data`, local SQLite files, or logs to disk | Containers and serverless have read-only or ephemeral disks |
   | Database target | Connection strings pointing at dev, staging, or a local DB; missing SSL/TLS setting | Wrong data, or connection refused |
   | Migrations | Schema changes with no migration file, or migrations not run by the deploy step | App errors on missing columns |
   | Build vs dev command | Production runs the dev server (`npm run dev`, `flask run`, `rails s` in development mode) | Slow, insecure, or missing build output |
   | Lockfiles | Dependency manifest changed but lockfile not updated (`package-lock.json`, `poetry.lock`, `go.sum`, `Cargo.lock`, `Pipfile.lock`) | Different versions in production |
   | Case-sensitive paths | Imports or file names whose case does not match on disk | Works on Windows or macOS, fails on Linux |
   | OS-specific code | Hardcoded `\` path separators, `C:\` paths, shell commands that exist only on one OS | Fails on Linux containers |
   | CORS and origins | Allowed origins set to `*` or to localhost only | Browser requests fail, or are open to any site |
   | Cookies and HTTPS | Cookies without `Secure` or `SameSite`; hardcoded `http://` links | Login breaks behind HTTPS |
   | Port and host binding | App binds to `localhost` only, or ignores the platform's `PORT` variable | Platform cannot route traffic |
   | Health check | Deploy config expects an endpoint or probe that the app does not serve | Deploy rolls back |
   | Timezones and locale | Code assumes the server's local timezone or a fixed locale | Wrong dates in production |
   | Static assets | Asset URLs or paths that only resolve in dev; missing build step for assets | Broken CSS, JS, or images |
   | Third-party sandboxes | Test keys, sandbox URLs, or test webhook secrets in production config | Payments or emails go nowhere |

3. **Check the deploy path itself.** Confirm the Dockerfile or build script copies the files the app needs and does not copy secrets or `.env` files into the image. Confirm the start command matches what the app expects.

4. **Report.** Use a table sorted by severity:

   | Severity | Location | Problem | Fix |
   |---|---|---|---|
   | Blocker | `config/db.js:12` | Database URL hardcoded to `localhost` | Read from `DATABASE_URL` env var |
   | Warning | `Dockerfile:8` | Runs `npm run dev` | Use `npm start` after `npm run build` |

   - **Blocker**: production will fail or leak data. Do not deploy.
   - **Warning**: likely a problem or a risk. Fix before or right after deploy.
   - **Note**: a difference that is probably fine. State why.

## Rules

- Quote file and line for every finding.
- Do not change production config without asking. Propose the diff.
- If no deploy config exists, say so, and list the assumptions you made about the production environment.
