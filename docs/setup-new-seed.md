# Aplicar guards a un nuevo seed

Pasos para replicar la protección completa en cualquier seed nuevo.

---

## Seeds Node.js (React, Next.js, etc.)

### 1. CI workflow

Crear `.github/workflows/ci.yml`. Si el seed ya tiene un job `ci` de lint/build, agregar el job `protected-files` encima:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  protected-files:
    uses: blanck1945/boogiepop-platform-guards/.github/workflows/protected-files.yml@main
    with:
      owner_username: blanck1945

  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run build
```

### 2. Husky

```bash
npm install --save-dev husky
npx husky
```

Agregar a `package.json`:

```json
"scripts": {
  "prepare": "husky"
}
```

Crear `.husky/pre-commit`:

```sh
#!/bin/sh
STAGED=$(git diff --cached --name-only)
FOUND=""

for f in AGENTS.md; do
  echo "$STAGED" | grep -qx "$f" && FOUND="$FOUND $f"
done
for d in .github/workflows; do
  echo "$STAGED" | grep -q "^$d/" && FOUND="$FOUND $d/"
done

if [ -n "$FOUND" ]; then
  echo ""
  echo "WARNING: archivos protegidos en este commit:"
  for f in $FOUND; do echo "  - $f"; done
  echo "  El PR va a requerir aprobacion del Owner para mergear."
  echo ""
fi
```

Crear `.husky/pre-push`:

```sh
#!/bin/sh
echo "lint..."
npm run lint || exit 1
echo "build..."
npm run build || exit 1

FOUND=""
while read local_ref local_sha remote_ref remote_sha; do
  if [ "$remote_sha" = "0000000000000000000000000000000000000000" ]; then
    BASE=$(git merge-base "$local_sha" github/main 2>/dev/null || git merge-base "$local_sha" origin/main 2>/dev/null || echo "")
    RANGE="${BASE:+$BASE..}$local_sha"
  else
    RANGE="$remote_sha..$local_sha"
  fi

  CHANGED=$(git diff --name-only $RANGE 2>/dev/null)

  for f in AGENTS.md; do
    echo "$CHANGED" | grep -qx "$f" && FOUND="$FOUND $f"
  done
  for d in .github/workflows; do
    echo "$CHANGED" | grep -q "^$d/" && FOUND="$FOUND $d/"
  done
done

if [ -n "$FOUND" ]; then
  echo ""
  echo "WARNING: archivos protegidos en este push:"
  for f in $FOUND; do echo "  - $f"; done
  echo "  El PR va a requerir aprobacion del Owner para mergear."
  echo ""
fi
```

### 3. Branch protection (vía API)

```powershell
$token = "<GITHUB_TOKEN>"
$repo  = "<repo-name>"   # ej: boogiepop-streamlit-seed
$headers = @{ Authorization = "Bearer $token"; Accept = "application/vnd.github+json"; "Content-Type" = "application/json" }
$body = @{
  required_status_checks = @{
    strict   = $true
    contexts = @("protected-files / check-protected-files")
  }
  enforce_admins = $true
  required_pull_request_reviews = @{
    dismiss_stale_reviews           = $true
    require_last_push_approval      = $false
    required_approving_review_count = 0
  }
  restrictions = $null
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Method PUT -Uri "https://api.github.com/repos/blanck1945/$repo/branches/main/protection" -Headers $headers -Body $body
```

---

## Seeds Python (Streamlit, etc.)

### 1. CI workflow

Idéntico al de Node.js — el job `protected-files` es agnóstico al lenguaje. Solo cambia el job `ci`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  protected-files:
    uses: blanck1945/boogiepop-platform-guards/.github/workflows/protected-files.yml@main
    with:
      owner_username: blanck1945

  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: ruff check .        # o flake8, pylint, etc.
      - run: python -m pytest    # si hay tests
```

### 2. Hooks locales (equivalente a Husky para Python)

Instalar `pre-commit`:

```bash
pip install pre-commit
```

Crear `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local
    hooks:
      - id: warn-protected-files
        name: warn protected files
        language: script
        entry: .hooks/warn-protected-files.sh
        always_run: true
        pass_filenames: false

      - id: pre-push-checks
        name: lint + test
        language: script
        entry: .hooks/pre-push-checks.sh
        stages: [push]
        always_run: true
        pass_filenames: false
```

Crear `.hooks/warn-protected-files.sh` (mismo contenido que el pre-commit de Node).

Crear `.hooks/pre-push-checks.sh` con los comandos de lint/test del proyecto Python.

Activar los hooks:

```bash
pre-commit install
pre-commit install --hook-type pre-push
```

### 3. Branch protection

Idéntico al de Node.js — misma API, mismo script PowerShell.

---

## Checklist por seed

- [ ] `.github/workflows/ci.yml` con job `protected-files`
- [ ] Hooks locales instalados y funcionando
- [ ] `package.json` con `"prepare": "husky"` (Node) o `.pre-commit-config.yaml` (Python)
- [ ] Branch protection configurada: required check + enforce_admins + dismiss_stale_reviews
- [ ] Verificar nombre exacto del check: `GET /repos/{owner}/{repo}/commits/{sha}/check-runs`
