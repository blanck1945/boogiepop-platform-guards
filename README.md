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

Job `check-protected-files` en stage `verify`. Falla el pipeline si alguno de estos archivos fue modificado en el commit/MR sin aprobación del Owner:

- `AGENTS.md` — contrato de plataforma del seed
- `.gitlab-ci.yml` — pipeline que incluye estos guards

**Flujo cuando se toca un archivo protegido en una MR:**
1. Pipeline falla con mensaje "requiere aprobación del Owner"
2. Owner abre la MR y hace click en **Approve** en GitLab
3. Re-run del pipeline → guard consulta la API → ve la aprobación → pasa → merge habilitado

## Setup requerido (una sola vez)

### 1. Crear un Group Access Token

En **GitLab → grupo `boogiepop-phatom` → Settings → Access Tokens:**

| Campo | Valor |
|-------|-------|
| Token name | `platform-guards-api` |
| Role | Reporter |
| Scopes | `read_api` |

Copiar el token generado.

### 2. Agregar variables CI/CD al grupo

En **GitLab → grupo → Settings → CI/CD → Variables** (se heredan en todos los proyectos del grupo):

| Variable | Valor | Flags |
|----------|-------|-------|
| `PLATFORM_API_TOKEN` | el token del paso anterior | Masked, Protected |
| `PLATFORM_OWNER_USERNAME` | tu username de GitLab | — |

Con variables a nivel grupo, todos los seeds las heredan automáticamente — no hay que configurarlas por proyecto.

### 3. Configuración de rama protegida `main` (por seed)

En **Settings → Repository → Protected Branches → `main`:**

| Campo | Valor |
|-------|-------|
| Allowed to merge | Developers + Maintainers |
| Allowed to push | No one (fuerza MR) |
| **Require a successful pipeline** | ✅ activado |

Con esto el flujo queda:
- **MR sin archivos protegidos** → guard pasa → el autor mergea solo, sin aprobación
- **MR toca `AGENTS.md` o `.gitlab-ci.yml`** → guard falla → merge bloqueado → Owner aprueba en UI → re-run → guard pasa → merge habilitado

## Agregar archivos protegidos

Editar la lista `PROTECTED` en `guards/protected-files.yml` en este repo. Solo mantenedores de plataforma tienen acceso de escritura acá.

## Límite conocido (GitLab Free)

Push Rules por path requieren Premium. Si alguien con acceso edita `.gitlab-ci.yml` y elimina el `include:` Y el job `check-ci-integrity`, los guards no corren. La mitigación es "Require pipeline to succeed" + revisión manual de diffs en MRs sospechosas.
