# AI Usage Log

Registro continuo de decisiones tomadas durante la ejecución asistida por IA (ver plantilla y contexto en `ARCHITECTURE.md` §8). Cada entrada referencia el PR correspondiente para que el diff completo quede a un clic de distancia.

### 2026-09-04 — [Proceso/Git] PR #1 — Modelo de dos cuentas para trazabilidad

- **Prompt:** "no sera bueno tambien que generemos otro usuario especificamente sobre todo los casos donde el AI tienen que genrear los features"
- **Generado por IA:** creación guiada de la cuenta `art-claude-dev` (el registro en sí lo hizo el usuario, por requerir navegador), invitación como colaboradora, actualización de `ARCHITECTURE.md` §8.1 reemplazando la convención de trailer por dos identidades reales, endurecimiento de branch protection a `enforce_admins: true`.
- **Decisión:** Aceptado.
- **Razón:** resuelve la limitación de que el ingeniero no puede aprobar su propio PR — la revisión requerida en `main` pasa a ser una revisión humana genuina en vez de un bypass de administrador.

### 2026-09-04 — [CI/CD] PR #2 — Pipeline de GitHub Actions

- **Prompt:** "armar el workflow de CI"
- **Generado por IA:** `.github/workflows/ci.yml` con 3 jobs (Markdown Lint, Secret Scanning, Build and Test condicional a la existencia de `pom.xml`).
- **Decisión:** Ajustado, luego Aceptado.
- **Razón del ajuste:** la primera versión de Markdown Lint falló con 30 errores, todos de espaciado/estilo (líneas en blanco alrededor de listas y encabezados, negrita usada como sub-etiqueta, fences sin lenguaje) — ninguno afecta el render real en GitHub. Se relajó `.markdownlint-cli2.jsonc` para desactivar esas reglas puramente estéticas y mantener las que sí detectan problemas de contenido, en vez de reescribir 30 puntos de formato en la documentación existente sin valor real a cambio.

### 2026-09-04 — [Infraestructura] PR — Scaffold del monorepo

- **Prompt:** "Si arranca" (arranque del Día 1 del plan de ejecución)
- **Generado por IA:** `.devcontainer/devcontainer.json` (Java 17 + Docker-in-Docker + kubectl/helm), `docker-compose.yml` (Postgres/Redis/RabbitMQ/Keycloak con healthchecks), `pom.xml` raíz como agregador multi-módulo (sin módulos todavía), `.env.example`.
- **Decisión:** pendiente de revisión del ingeniero (ver PR).
- **Nota de riesgo declarada:** la instalación de `kind` en el devcontainer vía `postCreateCommand` no se ha probado en una ejecución real de Codespaces todavía — la IA lo señaló explícitamente en un comentario dentro del propio `devcontainer.json` en vez de presentarlo como verificado.
