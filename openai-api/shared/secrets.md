# API Keys & Secrets — Handling Pattern

**Rule:** the OpenAI API key (`sk-...`) is a production secret and must be treated like a password in every language. See SKILL.md "Secrets & API Keys" for the binding rules. This file shows the concrete pattern in each supported language plus the `.env` / `.gitignore` setup.

## The Setup (do this once per project)

### `.gitignore`

Add these lines if they aren't already present (create the file if missing):

```
# Environment files — never commit
.env
.env.local
.env.*.local
.env.development
.env.production
```

### `.env` (local development only — never committed)

```
OPENAI_API_KEY=sk-...
# Optional:
# OPENAI_ORG_ID=org_...
# OPENAI_PROJECT_ID=proj_...
# OPENAI_BASE_URL=https://api.openai.com/v1
```

### `.env.example` (committed — documents what to set without values)

```
OPENAI_API_KEY=
# OPENAI_ORG_ID=
# OPENAI_PROJECT_ID=
```

### Verification before committing

Before any commit that adds OpenAI code:

1. `git status` — confirm `.env` is NOT listed as a tracked / staged file.
2. `git grep -i "sk-"` — confirm no key-shaped string is in tracked files.
3. The code instantiates the client with no `api_key=` argument — relying on the env var.

---

## Python

```python
# install: pip install openai python-dotenv
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()              # loads .env into os.environ (no-op in production)

client = OpenAI()          # reads OPENAI_API_KEY from env automatically
```

**Don't:**

```python
# ✗ hardcoded
client = OpenAI(api_key="sk-...")

# ✗ printing
print(f"Key loaded: {os.environ['OPENAI_API_KEY']}")

# ✗ committing .env
# (verify .gitignore before adding any OpenAI code)
```

**If you absolutely need to confirm the key is set** (e.g., in a CLI tool):

```python
import os
key = os.environ.get("OPENAI_API_KEY")
if not key:
    raise SystemExit("OPENAI_API_KEY is not set. Add it to .env or your environment.")
print(f"OPENAI_API_KEY: {key[:7]}…<redacted>")    # redacted display only
```

---

## TypeScript / Node

```ts
// install: npm install openai dotenv
import 'dotenv/config';                   // loads .env into process.env
import OpenAI from 'openai';

const client = new OpenAI();              // reads OPENAI_API_KEY from env
```

For ESM `dotenv/config` import-side-effect; for CommonJS use `require('dotenv').config()`. In Next.js / Vite, the framework loads `.env` automatically — don't add `dotenv`.

**Don't:**

```ts
// ✗ hardcoded
const client = new OpenAI({ apiKey: 'sk-...' });

// ✗ printing
console.log('Key:', process.env.OPENAI_API_KEY);

// ✗ exposing the key to the browser
// (env vars prefixed NEXT_PUBLIC_ / VITE_ are inlined into client JS —
//  never put OPENAI_API_KEY behind those prefixes)
```

**Browser / SPA note:** Never call OpenAI directly from a browser-side script. The key would be embedded in the bundle and visible to every visitor. Always proxy through your server (Next.js Route Handler, Express endpoint, Cloudflare Worker, etc.).

---

## C# / .NET

```csharp
// install: dotnet add package OpenAI
//          dotnet add package DotNetEnv  (for local .env loading)
using DotNetEnv;
using OpenAI.Responses;

Env.Load();                                                // local dev only

ResponsesClient client = new(
    apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));
```

For ASP.NET / production, use `Microsoft.Extensions.Configuration` with the user-secrets provider (`dotnet user-secrets`) in development and a secret store (Azure Key Vault, AWS Secrets Manager) in production. The key is then read via `IConfiguration["OpenAI:ApiKey"]`.

**Don't:**

```csharp
// ✗ hardcoded
ResponsesClient client = new(apiKey: "sk-...");

// ✗ in appsettings.json (committed!)
// {"OpenAI": {"ApiKey": "sk-..."}}        ← NO. Use user-secrets / Key Vault.

// ✗ printing
Console.WriteLine($"Key: {Environment.GetEnvironmentVariable("OPENAI_API_KEY")}");
```

---

## Java

```java
// Maven: io.github.cdimascio:dotenv-java for local .env loading
import io.github.cdimascio.dotenv.Dotenv;
import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;

Dotenv dotenv = Dotenv.configure().ignoreIfMissing().load();   // local dev only

// OkHttp clients read OPENAI_API_KEY from System.getenv automatically:
OpenAIClient client = OpenAIOkHttpClient.fromEnv();

// Or explicitly (still reading from env, not hardcoded):
OpenAIClient client = OpenAIOkHttpClient.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .build();
```

For Spring Boot, use `@Value("${OPENAI_API_KEY}")` and bind via `application-local.yml` (gitignored) or Spring Cloud Config / Vault in production.

**Don't:**

```java
// ✗ hardcoded
OpenAIClient client = OpenAIOkHttpClient.builder().apiKey("sk-...").build();

// ✗ committed application.properties / yml with the key value
// ✗ logging:
System.out.println("Key: " + System.getenv("OPENAI_API_KEY"));
```

---

## cURL / raw HTTP

For shell scripts:

```bash
# Use the env var — don't paste the key on the command line
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.2", "input": "Hello"}'
```

The value comes from `$OPENAI_API_KEY` — set in `~/.zshrc` / `~/.bashrc` (for personal dev), in a sourced `.env` (for project dev — `set -a; source .env; set +a`), or via CI/CD env injection.

**Don't:**

```bash
# ✗ key in shell history forever
curl ... -H "Authorization: Bearer sk-..."

# ✗ echoing
echo "Using $OPENAI_API_KEY"
```

---

## CI / GitHub Actions

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

Set the secret in **Settings → Secrets and variables → Actions** in the GitHub repo UI. The value is encrypted; the workflow log shows `***` instead of the actual value.

**Never** put the key into `.github/workflows/*.yml` directly.

---

## Verifying No Key Was Committed

If you suspect a key may have been committed:

```bash
git log --all -p | grep -E "sk-[A-Za-z0-9]{20,}"
```

If you find one:

1. **Treat the key as compromised immediately** — rotate it in the OpenAI dashboard.
2. Remove it from history (`git filter-repo` or BFG Repo-Cleaner).
3. Force-push the rewritten history (coordinate with collaborators).

A leaked key on a public repo is typically scraped and abused within minutes — rotating is non-negotiable.

---

## Detecting Accidental Disclosure in Code Review

When reviewing any code change before commit, look for:

- A `sk-` prefixed string anywhere in source.
- A `.env`, `.env.local`, etc. in `git status`.
- `print(...)`, `console.log(...)`, `Console.WriteLine(...)`, `System.out.println(...)`, `log.info(...)` calls that include `OPENAI_API_KEY` or `apiKey` or `Authorization` headers.
- `appsettings.json`, `application.yml`, `config.toml`, or similar committed config files containing the key value.
- Frontend env vars (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`) holding the OpenAI key.

If any of these are present, fix before continuing.
