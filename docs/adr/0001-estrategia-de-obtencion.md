# ADR 0001 — Estrategia de obtención de datos del BOC

- **Estado:** Propuesto (borrador). **No aprobable** hasta completar el [plan de verificación](#plan-de-verificación-bloqueante).
- **Fecha:** 2026-09-26
- **Fase:** 0.1

## Nota sobre el método

Ni el entorno de investigación ni los runners de GitHub Actions han conseguido conectar con ningún host `*.cantabria.es` (ver [H0](#h0-los-servidores-de-cantabriaes-no-responden-a-ips-de-centros-de-datos-fuera-de-españa)). Todo lo de abajo, salvo H0, sale de índices de buscadores (títulos, fragmentos y URLs indexadas) y de fuentes secundarias, no de peticiones reales al BOC.

Por eso cada afirmación lleva una etiqueta de confianza:

| Etiqueta | Significado |
|---|---|
| **[CONFIRMADO]** | Texto normativo o varias fuentes independientes coincidentes. |
| **[INDICIO]** | Visto en URLs o fragmentos indexados. Plausible, pero sin haber visto el HTML real. |
| **[PENDIENTE]** | No se ha podido comprobar. |

Nada de este documento sirve todavía para escribir selectores, schemas ni fixtures. Primero hay que cerrar el plan de verificación.

---

## Contexto

### Qué es el BOC y cuándo se publica

- Lo regula el **Decreto 18/2010, de 18 de marzo**. La edición electrónica es la única oficial y auténtica desde el 1 de enero de 2010. El acceso es universal, público y gratuito. **[CONFIRMADO]**
- **Calendario:** se publica de lunes a viernes, salvo los días declarados inhábiles en todo el territorio de Cantabria. Además puede haber **boletines extraordinarios cualquier día del año**. **[CONFIRMADO]**
  - Ejemplos indexados: BOC Extraordinario n.º 8 (21-05-2025) y n.º 27 (lunes 29-12-2025). **[INDICIO]**
  - Consecuencia de diseño: un día puede tener **0, 1 o varios boletines** (ordinario más extraordinarios), también en fin de semana. "Día sin boletín" no es un booleano, es un conjunto vacío.
  - Los festivos locales no cuentan: solo los inhábiles en toda la comunidad, es decir, las fiestas laborales nacionales y autonómicas que fija cada año un decreto publicado en el propio BOC.
- **Hora de publicación:** desconocida. El único dato encontrado es de un tercero (vLex), que actualiza *su* base de datos a las 9:00, y eso no es la hora del BOC. **[PENDIENTE]** → hay que medirla.

### Estructura de URLs observada

Es una aplicación Java/Struts (rutas `*.do`) en `https://boc.cantabria.es/boces/`. La misma aplicación responde también en `https://aplicacionesweb.cantabria.es/boces/`. Se toma `boc.cantabria.es` como host canónico.

| Ruta | Qué parece ser | Confianza |
|---|---|---|
| `/boces/` | Calendario de boletines (portada) | [INDICIO] |
| `/boces/boletines.do?boton=UltimoBOCPublicado` | Último boletín publicado | [INDICIO] |
| `/boces/boletines.do?boton=VerTodos` | Búsqueda de boletines por año | [INDICIO] |
| `/boces/boletines.do?boton=anterior&id=N` / `boton=siguiente&id=N` | Navegación al boletín anterior o siguiente | [INDICIO] |
| `/boces/verBoletin.do?idBolOrd=N` | Página de un boletín concreto, con su sumario. IDs vistos entre 30242 y 44685 | [INDICIO] |
| `/boces/verAnuncioAction.do?idAnuBlob=N` | PDF firmado de un anuncio | [INDICIO] |
| `/boces/inicioBusquedaAnuncios.do` | Buscador de anuncios (formulario) | [INDICIO] |
| `/boces/inicioAvisos.do` | "Contacto, suscripciones, RSS" | [INDICIO] |
| `/boces/verDecreto.do` | Decreto 18/2010 consolidado | [INDICIO] |
| `/boces/boletines.do?boton=departamentos&id=N`, `boton=accesos&id=N` | Sin determinar | [PENDIENTE] |

No se sabe si el calendario acepta una fecha como parámetro ni cómo distingue ordinario y extraordinario. **[PENDIENTE]**

### Contenido de un boletín

- La página de un boletín (`verBoletin.do`) muestra un índice por **sección → organismo → anuncio**. También enlaza al **sumario en PDF** (unos 440 KB) y al **boletín completo en PDF** (unos 10,9 MB). **[INDICIO]**
- **Secciones:** **[INDICIO]**
  - 1 Disposiciones generales
  - 2 Autoridades y personal
    - 2.1 Nombramientos, ceses y otras situaciones
    - 2.2 Cursos, oposiciones y concursos
  - 3 Contratación administrativa
  - 4 Economía y hacienda
  - 5 Expropiación forzosa
  - 6 Subvenciones y ayudas
  - 7 Otros anuncios

  Las subsecciones fuera de la 2 y la posible existencia de más secciones siguen sin verificar.
- **Identificadores de anuncio:**
  - Código de verificación con formato `CVE-AAAA-NNNNN` (p. ej. `CVE-2010-1277`), que también aparece como "Anuncio número 2025-4976".
  - El PDF lleva una cabecera del tipo `boc.cantabria.es Pág. 1817 MARTES, 2 DE FEBRERO DE 2021 - BOC NÚM. 21 1/1`.

  **[INDICIO]** No se sabe si el HTML del sumario expone el CVE o solo el `idAnuBlob`. **[PENDIENTE]**

### Dónde aparece el empleo público

- En la subsección **2.2 Cursos, oposiciones y concursos**. **[INDICIO]** Muchos resultados indexados lo confirman.
- Las convocatorias de la Administración de la Comunidad Autónoma salen como **Órdenes PRE/NN/AAAA** de la Consejería de Presidencia, por ejemplo la Orden PRE/89/2024 de CTS Rama Jurídica. **[INDICIO]** El nombre de la consejería cambia con cada legislatura, así que las reglas no deben depender de él.
- La 2.2 mezcla muchos emisores: ayuntamientos, Universidad de Cantabria, Servicio Cántabro de Salud, Educación y Justicia. Casi todo es ruido para los hitos de la Fase 1, y eso justifica el clasificador.
- Las bases comunes de los procesos selectivos son la **Orden PRE/19/2010, de 2 de julio**, con modificaciones de 2012, 2014, 2018 y 2019. **[INDICIO]** Esas bases dicen qué se publica en el BOC y qué solo en la web (ver hallazgo H1).

### Correcciones y anuncios que modifican otros

- Patrón de título observado en dos ejemplos: *"Corrección de errores al anuncio publicado en el Boletín Oficial de Cantabria número 43, de 4 de marzo de 2025, de \<título del anuncio original\>"*. **[INDICIO]**
- La corrección identifica el original por **número de boletín, fecha y título**, no por CVE (a falta de ver el texto completo). Para enlazarla hay que resolver número y fecha a un boletín y luego emparejar por título.
- Sin investigar: modificaciones de bases, ampliaciones de plazo y anuncios que anulan otros. **[PENDIENTE]**

### Otros canales oficiales

- **RSS:** `https://www.cantabria.es/web/gobierno/enlaces_rss` lista feeds por sección del BOC, entre ellos "Autoridades y Personal: Cursos, Oposiciones y Concursos". **[INDICIO]** Faltan por conocer la URL del feed, el formato, los campos, cuántos elementos trae y la latencia. Sobre todo falta saber **qué cobertura tiene**: la sección hermana `www.cantabria.es/anuncios-boc` se titula "Anuncios BOC de las **Consejerías** del Gobierno de Cantabria", lo que sugiere que podría no incluir ayuntamientos ni otros emisores. **[PENDIENTE]**
- **AVISOSBOC** (`avisosboc.cantabria.es`): avisos por email según sección y subsección. Según su ayuda, la información recibida solo puede usarse de forma **personal o interna, sin reutilizarla con otros fines**. **[INDICIO]**
- **Datos abiertos:** no se ha encontrado ningún dataset del BOC. **[INDICIO]**
- **Portal de Empleo Público** (`empleopublico.cantabria.es`, de la Dirección General de Función Pública): tiene una ficha por proceso (Cuerpo Administrativo, Cuerpo General Auxiliar, Cuerpo de Gestión, CTS Rama Jurídica...). Cada ficha recoge listas, tribunal, fecha y lugar del primer ejercicio, **plantillas de respuestas y aprobados de cada ejercicio**. **[INDICIO]**

### Marco legal y robots

- El acceso a la edición electrónica es libre y gratuito (Decreto 18/2010). **[CONFIRMADO]**
- Los actos, acuerdos y resoluciones de organismos públicos no son objeto de propiedad intelectual (art. 13 del TRLPI). La reutilización de información del sector público se rige por la Ley 37/2007.
- Las condiciones concretas del aviso legal de `boc.cantabria.es` están sin ver. **[PENDIENTE]**
- `robots.txt` de `boc.cantabria.es`, `www.cantabria.es` y `empleopublico.cantabria.es`: **[PENDIENTE]**.

---

## Hallazgos que no encajan con el roadmap (requieren decisión)

### H0. Los servidores de `cantabria.es` no responden a IPs de centros de datos fuera de España

**[CONFIRMADO]** 2026-09-26, con dos orígenes de red independientes:

| Origen | `*.cantabria.es` | Controles |
|---|---|---|
| Entorno cloud de investigación (Full network access) | El túnel se abre y el servidor no responde al *ClientHello* TLS. Reset tras ~11 s en `boc`, `www` y `empleopublico`. | Responden: `boe.es`, `bocm.es`, `scsalud.es`, `unican.es`, `parlamento-cantabria.es`. |
| Runner de GitHub Actions `ubuntu-latest` (IP de Microsoft, AS8075, Des Moines, EE. UU.) | *Timeout* de conexión TCP (30 s) en `boc`, `aplicacionesweb`, `www` y `empleopublico`, por HTTPS y por HTTP. | `boe.es` responde en 1,2 s. |

Evidencia: [run 36207102024](https://github.com/pedromorago/boc-radar/actions/runs/36207102024) del workflow temporal `probe-boc-access`.

**Interpretación [INDICIO]:** el cortafuegos del Gobierno de Cantabria descarta el tráfico de IPs de *hosting*, de fuera de España o de ambos. Los organismos de Cantabria alojados en otras redes, como el Servicio Cántabro de Salud o la Universidad, sí responden. Falta distinguir si el filtro es **geográfico** (ES/UE) o **por proveedor** (rangos cloud). Se sabrá probando desde una IP de centro de datos en España.

**Consecuencia:** el supuesto de la Fase 1.7 (cron en runners de GitHub) **no se cumple**. El fetch tiene que salir desde una IP que el BOC acepte. El resto del pipeline (clasificar, persistir, web, notificar) no toca `cantabria.es` y puede seguir en GitHub Actions.

Opciones para el fetch:

| Opción | Coste | Pros | Contras |
|---|---|---|---|
| **(a) Runner *self-hosted* en casa** (Raspberry Pi, NAS o PC con IP residencial española), con etiqueta propia y usado solo por el workflow de fetch | ~0 € | Se mantiene todo en GitHub Actions. IP legítima de un ciudadano en España. | La máquina tiene que estar encendida y conectada, y hay que vigilarla con el *dead man's switch*. Un runner *self-hosted* en **repo público** es un riesgo conocido: nunca debe ejecutar workflows de PR (sin `pull_request`, fork PRs con aprobación obligatoria). Encaja con la Fase 6. |
| **(b) Función en la nube en región España** (AWS `eu-south-2`, GCP `europe-southwest1` o Azure Spain Central), que deja el HTML crudo donde lo recoja GitHub Actions | Capa gratuita, probablemente | Sin hardware propio y con alta disponibilidad. | **Solo sirve si el filtro es geográfico**, no por proveedor: hay que probarlo antes. Suma una nube y credenciales. |
| **(c) Proxy residencial comercial** | De pago | — | Enmascara el origen para esquivar un filtro: éticamente discutible y contrario al espíritu del proyecto. **Descartada.** |

**Recomendación:** probar primero (b), porque es barata de comprobar. Si el filtro resulta ser por proveedor, pasar a (a). En ambos casos, separar el fetch (que deja el HTML crudo versionado o como artefacto) del procesamiento. Así el núcleo sigue siendo puro y testeable sin red, y la ubicación del fetch es un detalle intercambiable.

**Decisión (2026-09-26): se prueba primero (b).** La prueba es una función en AWS Lambda en `eu-south-2` (España) que pide `robots.txt` y el calendario del BOC, con `boe.es` como control. Se elige Lambda por su capa gratuita permanente y porque se prueba desde la consola, sin herramientas de despliegue. Si falla, se pasa a (a). El resultado se anotará aquí.

### H1. No todos los hitos se publican en el BOC
Hay indicios de que el BOC recoge la convocatoria, las listas de admitidos (con fecha y lugar del **primer** ejercicio) y los nombramientos. En cambio, las plantillas, las calificaciones de cada ejercicio y, previsiblemente, las **fechas de los ejercicios siguientes** se publican solo en `empleopublico.cantabria.es`.

Ejemplos indexados:
- Cuerpo General Auxiliar 2025: plantilla del ejercicio único (19-01-2025) y aprobados (10-02-2025) en el portal.
- "Aprobados segundo ejercicio" de otras categorías, también en el portal.

Consecuencia: en procesos con varios ejercicios, como CTS Rama Jurídica, un sistema que solo mire el BOC detectaría la fecha del primer examen y **perdería las siguientes**. **[INDICIO; se confirma leyendo las bases comunes]**

Opciones:
- **(a)** Solo BOC, con la laguna documentada.
- **(b)** Añadir el portal como segunda fuente dentro de la Fase 1, como bloque 1.8.
- **(c)** Añadirlo como bloque nuevo entre la Fase 1 y la Fase 2.

**Recomendación: (b).** El objetivo del producto es no perder fechas de examen, y diseñar la interfaz `Source` desde el principio cuesta poco ahora y mucho después. Implica adelantar a la Fase 1 la interfaz que el roadmap sitúa en la Fase 7.

**Decisión (2026-09-26): aceptada (b).** Bloque 1.8 en el roadmap e interfaz `Source` desde la Fase 1.

### H2. "CTS Gestión" no aparece como tal
Existen el **Cuerpo Técnico Superior** (A1), con Rama Jurídica entre otras, y el **Cuerpo de Gestión** (A2), que es un cuerpo distinto. No se ha encontrado ninguna "Rama Gestión" del CTS. Hay que aclarar a qué cuerpo se refiere el roadmap.

**Decisión (2026-09-26):** es el **Cuerpo de Gestión (A2)**. Roadmap corregido.

### H3. La fecha del examen está dentro del PDF, no en el sumario
El sumario solo da el título: *"...se aprueba la relación definitiva de admitidos... y se fija la fecha del primer ejercicio"*. En la Fase 1 el aviso puede decir "se ha publicado la resolución que fija la fecha", con enlace, pero **no la fecha**.

La cuenta atrás de la Fase 4 necesita la fecha real. Eso exige una de dos cosas:
- extraer fechas del PDF (hoy en el *backlog*), o
- cargar la fecha a mano en `data/`, con su fuente, igual que las estadísticas.

A decidir antes de la Fase 4. No bloquea la Fase 1.

---

## Decisión propuesta (provisional)

**Fuente primaria: la web del BOC, leyendo el HTML de la página de cada boletín del día.**

Flujo para una fecha D (razonando en `Europe/Madrid`):

1. **Descubrimiento.** Obtener la lista de boletines de D (ordinario y extraordinarios) desde el calendario. Si la lista está vacía, es un **día sin boletín**: resultado válido, distinto de un error. El mecanismo exacto está **[PENDIENTE]**. Si el calendario no admite fechas, el plan B es navegar con `boton=anterior/siguiente` desde `UltimoBOCPublicado`.
2. **Sumario.** Para cada boletín, descargar `verBoletin.do?idBolOrd=N` y parsear el sumario completo, no solo la 2.2. De cada anuncio sale un registro con estos campos:
   - identificadores: CVE si existe, `idAnuBlob`
   - clasificación: sección, subsección, organismo
   - contenido: título, URL del PDF
   - boletín: número, fecha, tipo (ordinario o extraordinario)

   El sumario completo hace falta para la huella estructural y para validar que el parseo no devuelve 0 anuncios.
3. **Filtrado y clasificación.** Se hacen después, en `core`, con funciones puras. El PDF **no se descarga** en la Fase 1: solo se guarda el enlace.
4. **Conciliación con RSS (fuente secundaria).** Solo si la verificación confirma que existe un feed de la 2.2. Cada día se comprueba que todo elemento del RSS aparece en el sumario parseado, y cualquier discrepancia es un fallo visible.
   - **Criterio para invertir los papeles** (RSS primario y HTML secundario): el feed cubre el 100 % de los anuncios de la 2.2 del sumario durante 10 días hábiles, trae un identificador estable y no depende de maquetación.

### Parámetros operativos propuestos

- **Volumen:** entre 2 y 6 peticiones al día (calendario más uno o dos boletines, y RSS). Concurrencia 1 y al menos 2 s entre peticiones.
- **Identificación:** User-Agent `boc-radar/<versión> (+https://github.com/pedromorago/boc-radar)`. El contacto va en la URL del repo, sin email en el código.
- **Peticiones condicionales:** `If-Modified-Since` o `ETag`, si el servidor los soporta. **[PENDIENTE]**
- **Horario:** a fijar cuando se mida la hora real de publicación. Idea inicial: una primera ejecución temprana y reintentos hasta un corte a media mañana (hora de Madrid). Si llega el corte en un día hábil no festivo sin boletín, el job falla.
- **Días inhábiles:** fichero de configuración anual con las fiestas laborales de Cantabria (nacionales y autonómicas), cada una con la URL del decreto en el BOC. CI falla si falta el fichero del año en curso.
- **Sin navegador headless:** una aplicación Struts renderizada en servidor no debería necesitar JS. **[PENDIENTE]** Si lo necesitara, se reabre este ADR.

---

## Alternativas consideradas

| Alternativa | Veredicto | Motivo |
|---|---|---|
| **RSS como única fuente** | Aplazada | Cobertura dudosa (¿solo Consejerías?), posible límite de elementos, no permite huella del boletín ni detectar anuncios que faltan. Puede pasar a primaria si cumple el criterio de arriba. |
| **AVISOSBOC (email)** | Descartada | Sus condiciones prohíben reutilizar la información, lo que choca con la web pública. Además exige un buzón, parsear correos y asumir la latencia del email, y no permite recuperar el histórico. |
| **Parsear los PDF** (sumario o boletín completo) | Descartada como primaria | Pesados (unos 11 MB), extracción de texto frágil y los mismos datos que el HTML. El PDF del sumario podría servir como tercera comprobación (recuento de CVE) si hiciera falta. |
| **Buscador de anuncios** (`inicioBusquedaAnuncios.do`) | Descartada para el día a día | Es un formulario, probablemente con sesión. Útil para buscar casos concretos al capturar fixtures. |
| **Navegar solo por `idBolOrd`** | Plan B | No hay garantía de que los IDs sean contiguos ni ordenados por fecha. Solo como respaldo del descubrimiento. |
| **Agregadores de terceros** (dateas, govclipping, bots de X/Telegram...) | Descartada | No son oficiales, tienen latencia y términos desconocidos y pueden desaparecer. Contradice "no inventes el BOC". |
| **Datos abiertos / API del BOE** | Descartada | No hay dataset del BOC y la API del BOE no cubre el BOC. |
| **Navegador headless (Playwright)** | Descartada salvo necesidad | Más pesado y frágil que HTTP más un parser HTML si el sitio no requiere JS. |

---

## Consecuencias

**Positivas**
- Fuente oficial, con muy pocas peticiones al día.
- El sumario completo permite huella estructural, recuento y validación de "0 anuncios".
- El mismo flujo sirve para ordinarios y extraordinarios, y la conciliación con RSS (si existe) detecta anuncios perdidos.

**Negativas**
- Depende de la maquetación HTML. Se mitiga con huella estructural, fixtures reales y la suite `live`.
- Depende de que el sitio esté disponible, sin SLA conocido.
- El fetch no puede ejecutarse en runners de GitHub (H0): hace falta un origen de red aceptado por `cantabria.es`, que es una pieza más que operar y monitorizar.
- Sin fecha de examen hasta resolver H3, y sin los hitos que solo salen en el portal hasta resolver H1.
- Posible codificación antigua (ISO-8859-1). **[PENDIENTE]**

**Ajustes al roadmap que implicaría** (a confirmar contigo)
- **1.1:** el descubrimiento devuelve 0..n boletines por fecha, no un booleano.
- **1.3:** añadir dos casos de fixture: "día con ordinario y extraordinario" y "extraordinario en fin de semana".
- **1.5:** la corrección se enlaza con el original por número de boletín, fecha y título, no por un ID.
- **1.7:** configuración de días inhábiles por año, y el corte horario como condición de fallo. **El fetch se separa del procesamiento** y se ejecuta donde decida H0; `scrape.yml` procesa el HTML crudo que deja el fetch.
- **Si se acepta H1-(b):** bloque 1.8 con el portal de empleo público e interfaz `Source` desde la Fase 1.

---

## Plan de verificación (bloqueante)

Requiere un origen de red que `cantabria.es` acepte (ver H0). Mientras no exista, se puede ejecutar `probe-boc-access` en un runner *self-hosted* o hacer las comprobaciones a mano desde un navegador en España. Cada punto deja evidencia en este ADR, y los HTML de ejemplo se guardan como candidatos a fixture en la Fase 1.3.

0. **H0:** comprobar si una IP de centro de datos **en España** llega a `boc.cantabria.es`. Decide entre las opciones (a) y (b).

1. `robots.txt` de los tres hosts.
2. Aviso legal y condiciones de reutilización de `boc.cantabria.es`.
3. Calendario:
   - cómo se pasa de una fecha a sus boletines (parámetros)
   - cómo se marca un extraordinario
   - qué muestra un día sin boletín
4. HTML real de `verBoletin.do`:
   - estructura, codificación y paginación
   - qué campos expone: CVE, `idAnuBlob`, sección, subsección, organismo
5. Si el sitio requiere JS, cookies o sesión.
6. Cabeceras de respuesta (`Last-Modified`, `ETag`, `Cache-Control`) y comportamiento ante errores: 404 y página de error con 200.
7. RSS de la 2.2:
   - URL, formato, campos y número de elementos
   - cobertura comparada con el sumario durante varios días
8. Dos o tres correcciones de errores reales: ¿citan el CVE del original?
9. Hora de publicación: fijar el método de medición (muestreo manual durante 10 días hábiles o una sonda temporal tras tu OK).
10. Bases comunes (Orden PRE/19/2010 y modificaciones) y una convocatoria reciente de cada cuerpo objetivo: qué hitos van al BOC y cuáles solo al portal. Cierra H1.
11. Fechas candidatas para los fixtures de la Fase 1.3. Solo se identifican, no se capturan.

---

## Fuentes consultadas

Consultadas a través de índices de búsqueda el 2026-09-26. No se accedió a ellas directamente.

- Calendario del BOC: <https://boc.cantabria.es/boces/>
- Último boletín: <https://boc.cantabria.es/boces/boletines.do?boton=UltimoBOCPublicado>
- Ejemplo de boletín: <https://boc.cantabria.es/boces/verBoletin.do?idBolOrd=41983>
- Buscador de anuncios: <https://boc.cantabria.es/boces/inicioBusquedaAnuncios.do>
- Contacto, suscripciones y RSS: <https://boc.cantabria.es/boces/inicioAvisos.do>
- Decreto 18/2010 consolidado: <https://boc.cantabria.es/boces/verDecreto.do>
- Decreto 18/2010 en vLex: <https://vlex.es/vid/decreto-regula-boletin-oficial-cantabria-80612234>
- RSS del Gobierno de Cantabria: <https://www.cantabria.es/web/gobierno/enlaces_rss>
- Anuncios BOC de las Consejerías: <https://www.cantabria.es/anuncios-boc>
- AVISOSBOC (ayuda de usuario): <https://avisosboc.cantabria.es/avisosboc/ayuda-usuario>
- Portal de Empleo Público: <https://empleopublico.cantabria.es/>
- Decretos y bases generales: <https://empleopublico.cantabria.es/decretos-y-bases-generales>
- Ficha del Cuerpo General Auxiliar: <https://empleopublico.cantabria.es/detalle-fp/-/asset_publisher/G3ZBUVmyumfx/content/cuerpo-general-auxiliar.-2>
- Ficha de CTS Rama Jurídica: <https://empleopublico.cantabria.es/detalle-fp/-/asset_publisher/G3ZBUVmyumfx/content/cuerpo-t%C3%A9cnico-superior-rama-jur%C3%ADdica.-1>
- Ejemplo de corrección de errores: <https://boc.cantabria.es/boces/boletines.do?boton=anterior&id=39603>
- Sede electrónica, boletines oficiales: <https://sede.cantabria.es/sede/informacion/boletines-oficiales>
