# boc-radar — Roadmap completo

Cada fase termina con un entregable funcionando, su CI en verde y su criterio de salida cumplido. No se empieza una fase sin cerrar la anterior.

---

## Fase 0 — Investigación y cimientos

**Objetivo:** entender la fuente real y dejar el repo listo para trabajar con calidad desde el primer commit.

### 0.1 Investigación del BOC (sin código de producto)
- Cómo se publica el BOC: calendario (días hábiles o no), hora aproximada de publicación, formato del sumario diario, existencia de RSS/XML/API, estructura de URLs y paginación.
- Dónde aparecen las convocatorias y resoluciones de empleo público (sección, subsección, organismo emisor).
- Qué campos se pueden extraer de forma fiable: número de anuncio, fecha, organismo, título, enlace, PDF.
- `robots.txt`, condiciones de reutilización de la información y límites razonables de peticiones.
- Casos raros: boletines extraordinarios, correcciones de errores, anuncios que modifican otros anteriores.
- **Entregable:** `docs/adr/0001-estrategia-de-obtencion.md` con la estrategia elegida y las alternativas descartadas.

### 0.2 Esqueleto del repo
- pnpm workspaces con la estructura de `CLAUDE.md`.
- tsconfig base estricto compartido, ESLint, Prettier, commitlint, lint-staged + husky.
- vitest configurado por paquete con umbrales de cobertura (core ≥ 95 %, resto ≥ 85 %).
- Plantillas de PR e issues, `CODEOWNERS`, `.editorconfig`, `.nvmrc`.
- Workflow `ci.yml`: install con caché, lint, typecheck, test y cobertura publicada como artefacto.
- Renovate (o Dependabot) configurado.

**Salida:** repo vacío pero con CI verde, protección de rama `main` (PR obligatoria y checks requeridos) y ADR 0001 aprobado.

---

## Fase 1 — Núcleo: obtener, entender y recordar

**Objetivo:** cada día, el sistema sabe qué se ha publicado, qué es relevante y qué es nuevo.

### 1.1 Fetcher (`packages/sources/boc`)
- Cliente HTTP con timeout, reintentos con backoff exponencial y jitter, rate limit y User-Agent identificable con contacto.
- Soporte de "día sin boletín" como resultado válido, distinto de un error.
- Modo `--from-fixtures` para ejecutar todo el pipeline sin red.

### 1.2 Parser
- Función pura `parseSumario(html): Anuncio[]`.
- Salida validada con zod. Cualquier campo obligatorio ausente es un error tipado que indica qué selector falló.
- **Huella estructural**: hash de la estructura DOM relevante (etiquetas y clases, sin texto) que se guarda en cada ejecución. Si cambia, se alerta de posible cambio de maquetación aunque el parseo siga funcionando.

### 1.3 Fixtures
- `fixtures/boc/YYYY-MM-DD/` con HTML real y un `expected.json` (golden file).
- Casos mínimos: día normal, día con varias convocatorias relevantes, día sin nada relevante, día sin boletín, boletín extraordinario y corrección de errores.
- Script `pnpm fixtures:capture <fecha>` para capturar nuevos casos reales.

### 1.4 Clasificador (`packages/core`)
- Reglas declarativas en `config/rules.yaml`: cuerpo, tipo de hito (plazo de inscripción, lista de admitidos, fecha de examen, resultados...), patrones y organismo.
- Hitos iniciales:
  - CTS Rama Jurídica (A1) y Cuerpo de Gestión (A2): fechas de examen.
  - Cuerpo Administrativo y Cuerpo General Auxiliar: apertura de plazo y fechas de examen.
- Cada clasificación devuelve la regla que la disparó (trazabilidad para depurar falsos positivos).
- Normalización de texto: tildes, mayúsculas y variantes ("Cuerpo Técnico Superior" frente a "CTS").

### 1.5 Estado y deduplicación (`packages/storage`)
- `data/state.json` con schema versionado (`schemaVersion`) y migraciones testeadas.
- Clave de dedupe estable por anuncio. Las correcciones enlazan con el anuncio original en vez de contar como novedad independiente.
- Escritura atómica (fichero temporal y rename).

### 1.6 Registro de ejecuciones
- `data/runs.json` (ventana deslizante, por ejemplo 180 días): inicio, fin, duración, estado, nº parseados, nº relevantes, nº nuevos, huella estructural, versión del código y motivo del error si lo hay.

### 1.7 Runner y cron
- `apps/runner`: CLI `boc-radar run [--date] [--dry-run] [--from-fixtures]`.
- Workflow `scrape.yml` con cron diario (documentar la conversión UTC ↔ Europe/Madrid) y `workflow_dispatch` manual con fecha opcional.
- Commit automático de `data/` solo si hay cambios, con un bot identificado.
- `concurrency` para evitar dos ejecuciones simultáneas.
- El job falla si: error de red persistente, parseo inválido, 0 anuncios en un día con boletín o cambio de huella sin revisar.
- **Ubicación del fetch pendiente de ADR 0001 (H0):** `cantabria.es` no responde a los runners de GitHub, así que la descarga se separa del procesamiento y se ejecuta desde un origen que el BOC acepte.

### 1.8 Portal de Empleo Público (`packages/sources/empleopublico`)
- Interfaz `Source` común (fetch, parse, huella) diseñada aquí y usada también por el BOC. Se adelanta de la Fase 7 (ADR 0001, H1).
- Fichas por proceso de los cuerpos objetivo en `empleopublico.cantabria.es`.
- Cubre los hitos que no salen en el BOC: plantillas, calificaciones y fechas de los ejercicios posteriores al primero.
- Mismas garantías que el BOC: fixtures reales, huella estructural y fallo alto ante 0 resultados.
- Dedupe entre fuentes: el mismo hito publicado en el BOC y en el portal no genera dos novedades.

### Tests de la fase
- Contrato: todos los fixtures producen exactamente su `expected.json`.
- Property-based (fast-check): el clasificador es determinista, la normalización es idempotente y el dedupe nunca duplica ni pierde anuncios.
- Integración: pipeline completo con fetcher mockeado (msw), incluidos timeouts, 500, 404 y HTML truncado.
- Suite `live` (no en PR, solo manual o semanal): una petición real para detectar deriva.

**Salida:** el cron corre a diario, `data/` se actualiza sola y un cambio de HTML en el BOC rompe CI con un mensaje que dice qué se rompió.

---

## Fase 2 — Calidad avanzada del núcleo

**Objetivo:** demostrar que los tests son buenos, no solo que existen.

- **Mutation testing** con Stryker sobre `core` y el parser. Umbral de mutation score ≥ 80 % en `core`, publicado en CI.
- **Pirámide de tests documentada** en `docs/TEST_STRATEGY.md`: qué se prueba en cada nivel, por qué y qué NO se prueba.
- **Matriz de riesgos**: lista de modos de fallo (BOC caído, HTML cambiado, falso positivo, falso negativo, aviso duplicado, aviso perdido, secreto caducado, cron que no corre...) y qué test, alerta o control mitiga cada uno.
- **Revisión de falsos negativos**: script que recorre los últimos N boletines y lista anuncios de empleo público no clasificados, para revisión manual y mejora de reglas.
- Badges en el README: CI, cobertura y mutation score.

**Salida:** `TEST_STRATEGY.md` y matriz de riesgos completas, con mutation score publicado.

---

## Fase 3 — Notificaciones

**Objetivo:** que la persona se entere sin mirar nada.

### 3.1 Diseño
- **Patrón outbox**: el runner escribe notificaciones pendientes en `data/outbox.json` y un paso separado las envía y las marca como enviadas. Así una caída a mitad no duplica ni pierde avisos.
- Idempotencia por `(anuncioId, canal, destinatario)`.
- Destinatarios y preferencias en config cifrada o en secrets: quién recibe qué hitos y por qué canal.
- Plantillas por hito con enlace directo al anuncio y al PDF.
- Modos: inmediato para hitos críticos (plazo abierto, fecha de examen) y resumen semanal para el resto.

### 3.2 Canales
1. **Telegram** (bot API): el primero, el más fiable.
2. **Email**: 3-4 cuentas de Gmail, vía SMTP con contraseña de aplicación o un proveedor transaccional (decidir en ADR).
3. **WhatsApp vía CallMeBot**: marcado como best-effort, con su limitación documentada.

### 3.3 Canal de operador (para mí, no para la usuaria)
- Telegram separado para alertas técnicas: fallo de run, cambio de huella, outbox atascado, secreto a punto de caducar.

### Tests de la fase
- Contrato con cada API externa mockeada (payload exacto, cabeceras, gestión de 429 y 5xx).
- Tests de la outbox: reintentos, envío parcial y reanudación.
- Snapshot de las plantillas renderizadas.
- `--dry-run` imprime lo que se enviaría sin enviarlo.

**Salida:** un anuncio relevante real llega por Telegram y por email en menos de una hora tras la ejecución, sin duplicados.

---

## Fase 4 — Web pública

**Objetivo:** una web que se actualiza sola y que sirve de escaparate.

### 4.1 Contenido
- Portada: próximos hitos (cuenta atrás hasta el siguiente examen o cierre de plazo).
- Página por oposición: línea temporal de anuncios, estado actual y enlaces oficiales.
- **Estadísticas históricas**: inscritos por año, presentados, aprobados y ratios.
  - Estos datos probablemente NO estén estructurados en el BOC. Crear `data/stats/*.csv` curado a mano, con columna de fuente (URL y fecha) por cada cifra y un schema zod que lo valide en CI.
  - Gráficos sencillos y accesibles.
- Página `/about` con la arquitectura y el enlace al repo (parte portfolio).

### 4.2 Técnica
- Astro estático generado a partir de `data/`, desplegado a GitHub Pages en cada cambio de `data/` o de `apps/web`.
- Dominio: subdominio de pedromorago.com (por ejemplo, `oposiciones.pedromorago.com`).
- Responsive, modo oscuro y sin JavaScript obligatorio para el contenido.
- Cabeceras de seguridad y CSP (vía meta o proxy, según lo que permita Pages; documentarlo).

### Tests de la fase
- **E2E con Playwright**: navegación, contenido derivado de un `data/` de prueba, cuenta atrás con reloj falseado y enlaces rotos.
- **Regresión visual** con screenshots de Playwright en móvil y escritorio.
- **Accesibilidad**: axe-core en todas las páginas, 0 violaciones serias.
- **Lighthouse CI** con presupuestos (performance, accessibility y SEO ≥ 90).
- Build de la web en cada PR con preview si es viable.

**Salida:** web pública en el subdominio, regenerada automáticamente y con toda la suite de la web en CI.

---

## Fase 5 — Observabilidad y dashboard de salud

**Objetivo:** "monitorizo mi propio sistema en producción".

- **Página `/status`** en la web, generada desde `runs.json`:
  - Última ejecución y su estado.
  - Frescura de datos (tiempo desde el último run correcto).
  - Histórico de 90 días en forma de barras (verde, rojo o gris para día sin boletín).
  - Duración media, anuncios por día y cambios de huella.
  - Estado de la outbox y de cada canal.
- **SLOs definidos** en `docs/SLO.md`, por ejemplo:
  - Frescura: datos con menos de 26 h en días hábiles, ≥ 99 % del mes.
  - Latencia de aviso: ≤ 2 h desde la publicación en el BOC para hitos críticos.
  - Cero avisos duplicados.
- **Dead man's switch** (por ejemplo healthchecks.io): si el cron no hace ping, alerta aunque GitHub no ejecute nada.
- **Synthetic monitoring**: workflow programado que ejecuta un subconjunto de Playwright contra la web en producción.
- **Runbook** en `docs/RUNBOOK.md`: qué hacer ante cada alerta (el BOC ha cambiado el HTML, cae un canal, caduca un token...).
- Badge de estado en el README enlazando a `/status`.

**Salida:** cualquier fallo relevante me llega en minutos y `/status` lo refleja públicamente.

---

## Fase 6 — Seguridad y cadena de suministro

**Objetivo:** hilo QA + seguridad coherente con mi experiencia en threat modeling.

- **Threat model** en `docs/THREAT_MODEL.md` con STRIDE sobre el flujo de datos (BOC → runner → data → web y notificaciones): activos, amenazas, mitigaciones y riesgo residual. Incluir diagrama de flujo de datos.
- GitHub Actions endurecido:
  - `permissions` mínimos por workflow y por job.
  - Actions fijadas por SHA, no por tag.
  - Sin `pull_request_target` inseguro y sin secretos expuestos a PRs de forks.
- Escaneo: CodeQL, gitleaks (pre-commit y CI), `pnpm audit` con política definida y SBOM (CycloneDX) generado en cada release.
- **OWASP ZAP baseline** contra la web desplegada, de forma programada.
- Sanitización de todo texto del BOC antes de renderizarlo en la web o en mensajes (el BOC es una entrada externa).
- Rotación documentada de tokens y fechas de caducidad vigiladas por el canal de operador.
- Privacidad: ningún dato personal de destinatarios en el repo público. Documentarlo en el README.

**Salida:** threat model publicado, pipelines endurecidos y escáneres en verde.

---

## Fase 7 — Radar de ayuntamientos

**Objetivo:** ampliar cobertura sin reescribir.

- Reutilizar la interfaz `Source` creada en la Fase 1.8 para añadir fuentes como plugins.
- Primera ampliación: anuncios de ayuntamientos de Cantabria publicados en el propio BOC (sección de administración local).
- Reglas por ayuntamiento y categoría de puesto afín (jurídico, administrativo, auxiliar) configurables en YAML.
- Nivel de aviso "radar": solo resumen semanal, nunca aviso inmediato, salvo que se configure lo contrario.
- Evaluar sedes electrónicas municipales solo si aportan algo que el BOC no tenga (ADR).

**Salida:** resumen semanal de radar funcionando, con fixtures y tests propios.

---

## Fase 8 — Empaquetado para portfolio

**Objetivo:** que alguien entienda el valor en 2 minutos y la profundidad en 20.

- README final: problema, demo (GIF de la web y captura de un aviso), arquitectura en Mermaid, estrategia de testing resumida, badges y enlaces a `/status` y a los docs.
- `docs/adr/` completo y ordenado.
- Artículo para pedromorago.com: "Cómo monitorizo un scraper en producción" (problema real, decisiones, fallos que detectó el sistema y métricas reales de `runs.json`).
- Sección "Incidentes reales": registro de cada vez que el sistema detectó un cambio o un fallo y cómo se resolvió. Es la prueba más convincente de todo el proyecto.
- Release `v1.0.0` con changelog generado (release-please).

**Salida:** repo, web, status y artículo enlazados entre sí y desde el portfolio.

---

## Backlog / ideas fuera de alcance
- Export `.ics` con las fechas de examen para suscribirse desde el calendario.
- Extracción de fechas desde el PDF del anuncio (pdf parsing y validación manual).
- Feed RSS propio de la web.
- Multiusuario con suscripción por hitos.
