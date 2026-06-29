# adc-ci

CI de seguridad **centralizada** para todos los repos de `A-Digital-Cafe` (root
`ADC-platform` + presets). La lógica vive una sola vez aquí, como
[reusable workflow](.github/workflows/security-suite.yml); cada repo solo lleva
un caller fino ([`templates/security.yml`](templates/security.yml)).

> **Este repo debe ser PÚBLICO.** Un reusable workflow en repo privado no puede
> ser invocado por repos públicos. El workflow no contiene secrets; se inyectan
> en runtime (el caller pasa solo los secrets SMTP requeridos en el job weekly).

## Qué corre

| Herramienta | Qué hace | Cuándo |
| --- | --- | --- |
| **Semgrep** (`--config auto`) | SAST estático | PR aprobado · semanal |
| **OSV-Scanner** | Vulns de dependencias + **licencias** (alternativa libre a FOSSA) | PR aprobado · semanal |
| **Trivy** (`config`) | Misconfig de Dockerfiles / docker-compose (solo si el repo tiene) | PR aprobado · semanal |
| **OSSF Scorecard** | Prácticas de seguridad del repo | push a main · semanal |
| **Email** | Resumen semanal por mail | semanal |

### Modos / triggers (definidos en el caller)

- **PR aprobado** (`pull_request_review` = approved) → modo `pr`: Semgrep + OSV +
  Trivy como **gate** (el check falla si hay hallazgos). Corre en contexto del
  repo base, así que es seguro ante PRs de la comunidad (forks).
- **Periódico** (cron cada 4 días, 06:00 UTC) → modo `weekly`: suite completa +
  OSSF Scorecard, report-only + email.
- **Manual** (`workflow_dispatch` desde la pestaña Actions → "Run workflow") →
  modo elegible (`weekly`/`pr`/`main`, default `weekly`) + input `only`
  (`all`/`semgrep`/`osv`/`trivy`/`scorecard`) para correr **un solo job**
  on-demand (ej. solo Scorecard).

Todos los modos escriben al **Job Summary** del run y, en PR, cada job aparece
como **check**.

> **Repos privados:** la query GraphQL `ListCommits` de Scorecard y su detección
> de SAST necesitan scopes de lectura extra (`actions`, `checks`, `issues`,
> `pull-requests`) además de `contents`+`id-token`. Como el caller solo puede
> acotar el token del reusable, esos scopes se conceden en **ambos** lados (job
> `scorecard` del reusable y jobs `scheduled`/`manual` del caller).
> Ref: <https://github.com/ossf/scorecard-action/issues/1248>.

## Setup (una sola vez)

1. **Crear este repo público** en la org: `A-Digital-Cafe/adc-ci`, y `git push`.
2. **Permitir acceso desde la org** (necesario para que los privados lo invoquen):
   `adc-ci` → Settings → Actions → General → *Access* →
   "Accessible from repositories in the A-Digital-Cafe organization".
3. **Política de Actions de la org**: Settings (org) → Actions → General → permitir
   actions de terceros usadas aquí, o pin por SHA (ya están pineadas).
4. **Secrets de org** (para el email semanal; el caller los pasa explícitos en weekly):
   `SMTP_SERVER`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SECURITY_REPORT_TO`.
   Sin estos, el semanal corre igual pero omite el email.
5. **En cada repo** (root + presets): copiar `templates/security.yml` a
   `.github/workflows/security.yml` (ver `scripts/distribute.sh` en el monorepo).

## SonarQube

SonarQube ya está configurado desde su web y postea sus propios checks en los PR
vía la GitHub App / análisis automático. **No** se añade ningún `sonar-scanner`
aquí para no duplicar ni pisar esa integración. Esta suite es complementaria.

## Licencias permitidas

La allowlist (env `ALLOWED_LICENSES` del reusable) es report-only. Ajustala ahí
si algún preset necesita otra licencia.
