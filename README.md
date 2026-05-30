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

## Configuración requerida en GitLab UI (por seed)

### Protected Branch `main` (Settings → Repository → Protected Branches)

| Campo | Valor |
|-------|-------|
| Allowed to merge | Developers + Maintainers |
| Allowed to push | No one (fuerza MR) |
| **Require pipeline to succeed** | ✅ activado |

Con esto el flujo queda:
- **MR sin archivos protegidos** → guard pasa → el autor mergea solo, sin aprobación
- **MR toca `AGENTS.md` o `.gitlab-ci.yml`** → guard falla → merge bloqueado automáticamente

### Para modificar un archivo protegido (Owner)

1. Ir a Settings → CI/CD → Variables del seed → agregar `BYPASS_PROTECTED_FILES_CHECK=1`
2. Pushear el cambio en una MR → guard pasa
3. Revisar y mergear
4. Borrar la variable

### Límite conocido (GitLab Free)

Push Rules por path requieren Premium. Sin ellos, alguien con acceso podría editar `.gitlab-ci.yml` para remover el `include:` y desactivar el guard. La mitigación es "Require pipeline to succeed" + revisión manual de diffs en MRs sospechosas. Para protección total a nivel servidor se necesita GitLab Premium.
