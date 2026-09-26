# boc-radar — instrucciones para Claude Code

## Qué es
Sistema que vigila el Boletín Oficial de Cantabria (BOC) y avisa de novedades de oposiciones.
Tiene dos objetivos al mismo nivel:
1. **Producto real**: una usuaria real depende de él para no perderse plazos ni fechas de examen.
2. **Portfolio QA/SDET**: tiene que demostrar criterio de testing, observabilidad, CI/CD y seguridad.

El roadmap completo está en `docs/ROADMAP.md`. Trabaja siempre por fases y no adelantes fases futuras.

## Principios no negociables
- **Nada de lógica sin test.** Si no se puede testear, se rediseña.
- **Fallar alto y claro.** Un scraper que devuelve 0 resultados sin error es el peor bug posible del sistema. Cualquier anomalía se convierte en fallo visible.
- **Núcleo puro y bordes finos.** La lógica de dominio (parsing, clasificación, dedupe, reglas) va en funciones puras. Red, disco, reloj y notificaciones se inyectan.
- **No inventes el BOC.** Si la estructura real no encaja con lo que asume el roadmap, para y dímelo.
- **Datos reales como fixtures.** Los tests del parser usan HTML/XML real guardado, nunca HTML escrito a mano, salvo en casos límite sintéticos que estén etiquetados como tales.
- **Secretos jamás en el repo.** Emails, teléfonos, tokens y chat IDs van en GitHub Secrets o en `.env` local (ignorado por git).

## Stack
- Node 20 LTS, TypeScript `strict` + `noUncheckedIndexedAccess`, pnpm workspaces.
- Validación: zod en todas las fronteras (HTML parseado, JSON de `data/`, config y respuestas de APIs externas).
- Tests: vitest (unit, contrato, integración), fast-check (property-based), Stryker (mutation), Playwright (E2E, visual y synthetic), axe-core (accesibilidad), Lighthouse CI.
- HTTP mocking: msw o undici MockAgent. Nada de llamadas reales a red en tests salvo en la suite `live` marcada explícitamente.
- Lint/format: ESLint (typescript-eslint strict) + Prettier.
- Web: Astro estático en GitHub Pages.

## Estructura del monorepo
```
packages/
  core/        dominio puro: tipos, schemas zod, clasificador, dedupe, reglas
  sources/     fetchers + parsers por fuente (BOC y portal de empleo público ahora, ayuntamientos después)
  notifier/    canales (telegram, email, whatsapp) + outbox
  storage/     lectura/escritura de data/*.json con schemas versionados
apps/
  runner/      CLI que orquesta: fetch → parse → classify → dedupe → notify → persist
  web/         Astro: web pública + /status
fixtures/      HTML/XML real del BOC, organizado por fecha y caso
data/          estado versionado (state.json, runs.json, stats/)
docs/          ROADMAP, ADRs, TEST_STRATEGY, THREAT_MODEL, RUNBOOK
```

## Convenciones
- Conventional Commits (enforced con commitlint). Commits pequeños y atómicos.
- Una rama y una PR por bloque de trabajo. La PR incluye qué cambia, por qué y cómo se ha probado.
- Decisiones de arquitectura relevantes → ADR en `docs/adr/NNNN-titulo.md` (contexto, decisión, alternativas, consecuencias).
- Fechas: se guardan en ISO 8601 con zona. La lógica de negocio razona en `Europe/Madrid`. El cron de GitHub va en UTC: documenta la conversión.
- Errores tipados (clases o uniones discriminadas), nunca `throw "string"`.
- Logs estructurados en JSON (nivel, evento, contexto) para que el dashboard pueda consumirlos.

## Definition of Done (toda tarea)
- [ ] Lint, typecheck y tests en verde en local y en CI.
- [ ] La cobertura no baja del umbral de su paquete.
- [ ] Tests para el camino feliz, los bordes y al menos un modo de fallo.
- [ ] README o docs actualizados si cambia el comportamiento.
- [ ] Sin secretos ni datos personales en el diff.

## Forma de trabajar conmigo
- Antes de cada fase: plan breve y espera mi OK.
- Al terminar cada bloque: resumen de qué hiciste, qué decidiste y qué queda abierto.
- Si dudas entre dos opciones razonables, elige una, justifícala en una línea y sigue. Para solo si la decisión es cara de revertir.
