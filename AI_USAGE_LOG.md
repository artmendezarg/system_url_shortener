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

## 2026-09-04 — [Feature] PR — V1 Legacy Monolith mínimo (crear/redirigir, sin auth)

- **Tarea:** Día 1, Tarea #2 del plan (ARCHITECTURE.md §7) — construir el "sistema legacy" mínimo que en Día 2 será intervenido con el escenario Brownfield.
- **Prompt:** "Si" (confirmación para avanzar con "el V1 Legacy Monolith mínimo (Liquibase + crear/redirigir, sin auth)").
- **Generado por IA:** módulo Maven `v1-legacy-monolith` completo — `pom.xml` (Spring Boot 3.3.4 vía BOM importado, ya que el padre no es `spring-boot-starter-parent`), entidad JPA `UrlRecord`, `UrlRecordRepository`, `UrlShortenerService` (generador de código con `SecureRandom` + alfabeto alfanumérico + reintento hasta 5 veces ante colisión — deliberadamente simple, no el Base62 con manejo formal de colisiones reservado para V2), `UrlController` (`POST /api/v1/urls` → 201, `GET /{shortCode}` → 301 o 404, sin autenticación), changelogs de Liquibase (`changelog-master.xml` + `changelog-v1.0-init.xml`, tabla `urls` **sin** `expires_at` a propósito), `application.yml` con datasource parametrizado por variables de entorno, y tests: unitarios con Mockito para `UrlShortenerService` (incluye caso de colisión y caso de agotar reintentos) e integración end-to-end con Testcontainers (Postgres real) + MockMvc para `UrlController`.
- **Nota de riesgo declarada:** este VM de trabajo no tiene acceso a Maven Central ni puede instalar JDK17/Maven localmente (red restringida solo a GitHub/npm), por lo que el código no fue compilado ni probado localmente antes de este commit. La verificación real de compilación y tests ocurre en el pipeline de CI (`build-and-test` en GitHub Actions, que sí corre en runners sin esa restricción). Si CI revela errores, se corrigen con un commit adicional sobre esta misma rama — mismo patrón ya usado para el linter de Markdown.
- **Decisión:** pendiente de revisión del ingeniero (ver PR).

## 2026-09-04 — [Fix] PR #5 — Falta dependencia de validation (CI + build local)

- **Prompt:** el usuario corrió `mvn verify` en su Codespace (con acceso completo a Maven Central, a diferencia de este entorno de trabajo) y pegó el error real de compilación.
- **Error real:** `package jakarta.validation.constraints does not exist` / `cannot find symbol: class NotBlank` en `UrlController.java` — se usó la anotación `@NotBlank` sin declarar la dependencia `spring-boot-starter-validation` en el `pom.xml` del módulo.
- **Fix aplicado:** se agregó `org.springframework.boot:spring-boot-starter-validation` a `v1-legacy-monolith/pom.xml`.
- **Decisión:** Ajustado — corregido directamente en la misma rama (`feature/v1-legacy-monolith`), sin abrir un PR nuevo.
- **Razón:** confirma que el flujo previsto (no poder compilar localmente en este VM, dejar la primera verificación real en manos de un entorno con red completa) funcionó como estaba planeado — el error se detectó rápido corriendo `mvn verify` en el Codespace del usuario, sin necesidad de depender de los logs de CI (que además resultaron inaccesibles por la misma restricción de red que afectó a los túneles de Codespaces).

## 2026-09-04 — [Fix] PR #5 — Tests de integración fallaban: falta el flag `-parameters` del compilador

- **Prompt:** el usuario corrió `mvn verify` de nuevo tras el fix anterior; compiló, pero 2 de 6 tests fallaron (`UrlControllerIntegrationTest`) con `IllegalArgumentException: Name for argument of type [java.lang.String] not specified... Ensure that the compiler uses the '-parameters' flag`.
- **Diagnóstico:** `@PathVariable String shortCode` depende de que el compilador conserve los nombres de parámetros vía reflexión (flag `-parameters` de `javac`). `spring-boot-starter-parent` activa ese flag por defecto, pero este proyecto usa un padre propio (`url-shortener-parent`), así que nunca se configuró.
- **Fix aplicado:** (1) se agregó `<maven.compiler.parameters>true</maven.compiler.parameters>` a las properties del `pom.xml` raíz, para que todos los módulos futuros lo hereden sin tener que repetirlo; (2) se nombró explícitamente `@PathVariable("shortCode")` en `UrlController` como refuerzo, para no depender únicamente del flag del compilador.
- **Decisión:** Ajustado — corregido en la misma rama (`feature/v1-legacy-monolith`).
- **Razón:** el ciclo local (Codespace con Maven real) siguió funcionando como red de verificación rápida; cada corrida de `mvn verify` reveló un problema real distinto, resuelto con un commit incremental.

## 2026-09-04 — [Feature] PR — API Gateway básico (Spring Cloud Gateway)

- **Tarea:** Día 1, Tarea #3 del plan (ARCHITECTURE.md §7) — punto de entrada único, patrón Strangler Fig.
- **Prompt:** "si" (confirmación para avanzar con "API Gateway básico" tras mergear el V1 Legacy Monolith).
- **Generado por IA:** módulo Maven `api-gateway` (Spring Cloud Gateway 2023.0.3 sobre Spring Boot 3.3.4). Dos rutas activas: `/api/v1/**` → Monolito V1 tal cual, y `GET /{shortCode}` (redirect público sin prefijo) → Monolito V1. `/api/v2/**` responde `501 Not Implemented` vía un stub explícito (`V2StubController`) en lugar de un 404 genérico, porque el servicio V2 todavía no existe (arranca en la Tarea #4). Test de integración con un servidor HTTP fake (`com.sun.net.httpserver.HttpServer`, del JDK, para no sumar una dependencia de mocking solo para esto) que verifica que ambas rutas reales llegan al backend correcto y que `/api/v2/**` devuelve 501.
- **Decisión de diseño explícita:** la ruta de `GET /{shortCode}` hoy delega directo a V1 sin consultar Redis, porque el índice de códigos V2 en Redis (descrito en ARCHITECTURE.md §3.1) no tiene sentido hasta que exista el servicio V2 que lo llene. Se documentó en un comentario en `GatewayRoutesConfig` para que quede explícito que es una simplificación temporal, no el diseño final.
- **Riesgo declarado:** mismo patrón que el PR anterior — no se pudo compilar localmente en este entorno (sin acceso a Maven Central), por lo que la primera verificación real es `mvn verify` corrido por el ingeniero en su Codespace, seguido de CI.
- **Decisión:** pendiente de revisión del ingeniero (ver PR).

## 2026-09-04 — [Fix] PR #6 — Import incorrecto de WebTestClient (CI falló, local no)

- **Prompt:** el usuario reportó que `mvn verify` le dio BUILD SUCCESS en su Codespace corriendo solo el módulo `api-gateway`, pero el CI del PR #6 falló en compilación de tests con `cannot find symbol: class WebTestClient` en el paquete `org.springframework.boot.test.web.reactive.server`.
- **Error real:** el import generado (`org.springframework.boot.test.web.reactive.server.WebTestClient`) no existe — es una mezcla incorrecta de dos paquetes reales (`org.springframework.boot.test.web.reactive.server`, que sí existe pero solo contiene clases de auto-configuración, y `org.springframework.test.web.reactive.server`, que es donde realmente vive `WebTestClient`). El BUILD SUCCESS local fue engañoso — probablemente Maven no recompiló el test desde cero o resolvió una clase distinta por caché del repositorio local; el runner de CI, con un `.m2` limpio, expuso el error real de inmediato.
- **Fix aplicado:** corregido el import a `org.springframework.test.web.reactive.server.WebTestClient`.
- **Decisión:** Ajustado — corregido en la misma rama (`feature/api-gateway`).
- **Lección:** un "pasa en mi máquina" con caché de Maven no es prueba suficiente por sí sola; CI con entorno limpio sigue siendo la verificación de referencia, incluso cuando el build local también está disponible.

## 2026-09-04 — [Feature] PR — Contrato OpenAPI V2 + generador Base62 (arranque Escenario A / Greenfield)

- **Tarea:** Día 1, Tarea #4 del plan (ARCHITECTURE.md §7) — arranque formal del Escenario A (Greenfield). Alcance explícito de este PR: solo los puntos 1 y 3 de la descomposición de §6 (contrato REST + generador Base62); Redis, Circuit Breaker y la implementación completa del servicio quedan para el Día 2.
- **Prompt:** "si" (confirmación para avanzar con la Tarea #4 tras mergear el API Gateway).
- **Generado por IA:**
  - Módulo `v2-shortener-contract`: contrato OpenAPI 3.0.3 (`src/main/resources/openapi/shortener-v2.yaml`) que documenta los 6 endpoints de la API V2 ya resumidos en ARCHITECTURE.md §4.1 (crear, listar, bulk crear, estado de bulk job, revocar, analytics), con sus schemas de request/response, `bearerAuth` (JWT vía Keycloak) y `RedirectRule` (redirección condicional por tipo de dispositivo, según la desambiguación del Escenario C §6). Se documentó explícitamente que `GET /{shortCode}` (la redirección pública real) queda fuera de este contrato porque no vive bajo `/api/v2` — la resuelve el Gateway.
  - `Base62CodeGenerator`: generador de códigos con manejo de colisiones más robusto que el de V1 — reintenta hasta 5 veces por longitud (7 a 12 caracteres) y hace fallback a una longitud mayor si se agotan los intentos, en vez de reintentar indefinidamente. Recibe la verificación de colisión como `Predicate<String>` para poder probarse sin base de datos ni Redis (se conecta a la persistencia real en el Día 2).
  - Tests unitarios (`Base62CodeGeneratorTest`): código sin colisión, reintento antes de crecer de longitud, fallback real a longitud mayor cuando se agota la longitud inicial, y excepción cuando se agotan todas las longitudes.
- **Riesgo declarado:** mismo patrón que los PRs anteriores — no se pudo compilar/validar localmente en este entorno (sin acceso a Maven Central). El YAML del contrato sí se validó localmente con un parser de YAML (sintácticamente válido), pero no contra un validador de OpenAPI real. La verificación de compilación queda en manos del ingeniero (Codespace) + CI.
- **Decisión:** pendiente de revisión del ingeniero (ver PR).
