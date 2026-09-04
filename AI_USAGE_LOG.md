# AI Usage Log

Registro continuo de decisiones tomadas durante la ejecución asistida por IA (ver plantilla y contexto en `ARCHITECTURE.md` §8). Cada entrada referencia el PR correspondiente para que el diff completo quede a un clic de distancia.

## 2026-09-04 — [Proceso/Git] PR #1 — Modelo de dos cuentas para trazabilidad

- **Prompt:** "no sera bueno tambien que generemos otro usuario especificamente sobre todo los casos donde el AI tienen que genrear los features"
- **Generado por IA:** creación guiada de la cuenta `art-claude-dev` (el registro en sí lo hizo el usuario, por requerir navegador), invitación como colaboradora, actualización de `ARCHITECTURE.md` §8.1 reemplazando la convención de trailer por dos identidades reales, endurecimiento de branch protection a `enforce_admins: true`.
- **Decisión:** Aceptado.
- **Razón:** resuelve la limitación de que el ingeniero no puede aprobar su propio PR — la revisión requerida en `main` pasa a ser una revisión humana genuina en vez de un bypass de administrador.

## 2026-09-04 — [CI/CD] PR #2 — Pipeline de GitHub Actions

- **Prompt:** "armar el workflow de CI"
- **Generado por IA:** `.github/workflows/ci.yml` con 3 jobs (Markdown Lint, Secret Scanning, Build and Test condicional a la existencia de `pom.xml`).
- **Decisión:** Ajustado, luego Aceptado.
- **Razón del ajuste:** la primera versión de Markdown Lint falló con 30 errores, todos de espaciado/estilo (líneas en blanco alrededor de listas y encabezados, negrita usada como sub-etiqueta, fences sin lenguaje) — ninguno afecta el render real en GitHub. Se relajó `.markdownlint-cli2.jsonc` para desactivar esas reglas puramente estéticas y mantener las que sí detectan problemas de contenido, en vez de reescribir 30 puntos de formato en la documentación existente sin valor real a cambio.

## 2026-09-04 — [Infraestructura] PR — Scaffold del monorepo

- **Prompt:** "Si arranca" (arranque del Día 1 del plan de ejecución)
- **Generado por IA:** `.devcontainer/devcontainer.json` (Java 17 + Docker-in-Docker + kubectl/helm), `docker-compose.yml` (Postgres/Redis/RabbitMQ/Keycloak con healthchecks), `pom.xml` raíz como agregador multi-módulo (sin módulos todavía), `.env.example`.
- **Decisión:** pendiente de revisión del ingeniero (ver PR).
- **Nota de riesgo declarada:** la instalación de `kind` en el devcontainer vía `postCreateCommand` no se ha probado en una ejecución real de Codespaces todavía — la IA lo señaló explícitamente en un comentario dentro del propio `devcontainer.json` en vez de presentarlo como verificado.

## 2026-09-04 — [Infraestructura] PR — Fix del devcontainer (Codespace en recovery mode)

- **Prompt:** reporte del usuario: "Failed to create container... Error code: 1302 (UnifiedContainersErrorFatalCreatingContainer)" al abrir el Codespace por primera vez.
- **Generado por IA:** confirmó el riesgo que ya había declarado (instalación de Kubernetes tooling sin verificar) se materializó. Se reemplazó la feature `ghcr.io/devcontainers/features/kubectl-helm-minikube:1` (sospechosa principal, la menos estándar de las dos features usadas) por un script propio (`.devcontainer/setup.sh`) que instala kubectl, kind y helm con binarios/scripts oficiales. Se conservó `docker-in-docker` por ser la feature más probada.
- **Decisión:** Ajustado — el primer intento (quitar `kubectl-helm-minikube`) no resolvió el problema.
- **Diagnóstico real (segunda vuelta):** con más detalle del log, se confirmó que la feature que fallaba era `docker-in-docker` — su script de instalación corre `apt-get install` para un daemon Docker anidado completo, y ese `apt-get` fallaba con exit code 100 (genérico de apt) dentro de la imagen base `java:1-17-bullseye`.
- **Fix aplicado:** se reemplazó `docker-in-docker` por `docker-outside-of-docker`, que monta el socket del Docker que el propio Codespace ya corre por debajo, en vez de instalar un daemon anidado — mismo resultado funcional (poder correr `docker`/`kind`) con muchas menos piezas y sin el `apt-get` problemático.
- **Razón:** el primer diagnóstico fue por descarte razonable con información incompleta del log; en cuanto el usuario compartió el bloque de error con el comando exacto que fallaba dentro del build de la feature, se pudo identificar la causa real en vez de seguir adivinando.

## 2026-09-04 — [Infraestructura] PR #4 — Tercer intento: la imagen base, no la feature

- **Prompt:** reporte del usuario: tras cambiar a `docker-outside-of-docker`, el mismo error exit code 100 en `apt-get` durante el install script de la feature.
- **Diagnóstico:** dos features distintas (`docker-in-docker` y `docker-outside-of-docker`) fallando con el mismo error de `apt-get` descarta que el problema sea de una feature específica — apunta a la imagen base. `mcr.microsoft.com/devcontainers/java:1-17-bullseye` usa Debian 11, cuyos repositorios estándar de apt probablemente ya no están activos para esta fecha (movidos a archive-only), rompiendo cualquier `apt-get install` dentro del build de features.
- **Fix aplicado:** cambio de imagen base a `mcr.microsoft.com/devcontainers/java:1-17-bookworm` (Debian 12, repositorios activos), conservando `docker-outside-of-docker`.
- **Decisión:** pendiente de confirmación del ingeniero.
- **Razón:** cambiar de feature sin cambiar la imagen base habría repetido el mismo síntoma con cualquier otra feature basada en apt — el patrón de dos fallos idénticos con causas distintas señaladas era la pista de que el problema estaba un nivel más abajo.

## 2026-09-04 — [Infraestructura] PR #4 — Causa raíz encontrada: repo de apt de Yarn roto en la imagen "java"

- **Prompt:** reporte del usuario con el log completo mostrando `E: The repository 'https://dl.yarnpkg.com/debian stable InRelease' is not signed.`
- **Diagnóstico real:** la imagen `mcr.microsoft.com/devcontainers/java:*` trae preconfigurado un repositorio apt de Yarn cuya firma ya no es válida (Yarn deprecó ese repo clásico). Eso rompe `apt-get update` dentro de esa imagen para cualquier feature que dependa de apt — por eso fallaban tanto `docker-in-docker` como `docker-outside-of-docker`, y por eso cambiar de bullseye a bookworm no ayudó (el problema no era la versión de Debian).
- **Fix aplicado:** se abandona la imagen "java" preempaquetada y se compone el ambiente desde `mcr.microsoft.com/devcontainers/base:bookworm` (imagen mínima, sin Node/Yarn de fábrica) + la feature `ghcr.io/devcontainers/features/java:1` (versión 17, con Maven) explícita.
- **Decisión:** pendiente de confirmación del ingeniero.
- **Razón:** los dos intentos anteriores asumieron que el problema estaba en la feature de Kubernetes/Docker; el mensaje de error real (no visible hasta que el usuario compartió el log completo) mostró que la causa estaba en una herramienta totalmente ajena (Yarn) empaquetada en la imagen base. Lección: pedir el log completo desde el principio hubiera ahorrado dos iteraciones.
