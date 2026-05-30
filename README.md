# boogiepop-platform-guards

Guards de CI centralizados para los seeds de la plataforma Boogiepop.

Los seeds incluyen los guards vía `include:` — la lógica vive acá, fuera del alcance del usuario.

## Cómo incluir en un seed

```yaml
# .gitlab-ci.yml del seed
include:
  - project: '<namespace>/boogiepop-platform-guards'
    file: '/guards/protected-files.yml'
    ref: main

stages:
  - verify
  - build
  - deploy

# ... resto del pipeline
```

## Guards disponibles

### `guards/protected-files.yml`

Job `check-protected-files` en stage `verify`. Falla el pipeline si alguno de estos archivos fue modificado en el commit/MR:

- `AGENTS.md` — contrato de plataforma del seed
- `.gitlab-ci.yml` — pipeline que incluye estos guards

**Bypass de emergencia:** un Owner del proyecto puede setear `BYPASS_PROTECTED_FILES_CHECK=1` en Settings → CI/CD → Variables. Debe documentarse en el issue tracker.

## Agregar archivos protegidos

Editar la lista `PROTECTED` en `guards/protected-files.yml` en este repo. Solo mantenedores de plataforma tienen acceso de escritura acá.

## Push Rules (complementario)

Configurar en GitLab UI → Project Settings → Push Rules para los seeds:

- **"Prevent pushing secret files"** con regex: `(AGENTS\.md|\.gitlab-ci\.yml)`

Esto rechaza el push a nivel servidor antes de que llegue al pipeline.
