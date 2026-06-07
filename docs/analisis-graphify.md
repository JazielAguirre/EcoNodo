# Análisis del repositorio EcoNodo con Graphify

> **Nota metodológica:** Graphify ya había sido ejecutado previamente sobre este repositorio
> (los resultados se encuentran en `graphify-out/GRAPH_REPORT.md` y `graphify-out/graph.html`).
> Esta tarea **no volvió a ejecutar la herramienta**; consistió en **revisar, interpretar y
> validar manualmente** esos resultados contra el estado actual del código fuente, para luego
> usarlos como base de un README profesional.

## Objetivo del análisis

Entender la arquitectura real de EcoNodo, identificar sus componentes centrales, sus
relaciones y su estado de madurez, y validar que la información generada por Graphify
coincide con el código actual — corrigiendo cualquier dato obsoleto antes de documentar
el proyecto públicamente.

## Archivos de Graphify revisados

- `graphify-out/GRAPH_REPORT.md` — reporte de auditoría (174 nodos, 244 aristas,
  18 comunidades, 87% de relaciones `EXTRACTED`, 13% `INFERRED`, 0% `AMBIGUOUS`).
- `graphify-out/graph.html` — visualización interactiva del grafo (inspeccionada como
  fuente de datos; confirma los mismos nodos y comunidades que el reporte).
- `graphify-out/graph.json` — datos crudos del grafo (nodos, aristas, comunidades).
- `graphify-out/.graphify_labels.json` — etiquetas de comunidades.

## Alcance

El grafo cubre el frontend (HTML/SCSS/JS), la capa de datos (`js/api.js`), la
documentación interna (`docs/*.md`, `CLAUDE.md`), la Edge Function de Supabase
(`supabase/functions/ingest-lectura/index.ts`) y, de forma indirecta —a través de la
documentación—, el firmware del ESP32. El propio reporte advierte que el corpus
(~14 000 palabras) es pequeño y "cabe en una sola ventana de contexto", por lo que el
valor del grafo aquí es más de **mapa de relaciones** que de compresión de información.

## Arquitectura identificada

Graphify reconoció correctamente la arquitectura de extremo a extremo descrita en la
documentación interna y replicada en varios nodos ("hyperedge" de arquitectura IoT):

```
Sensores (BME680 + SDS011) → ESP32 → Edge Function (Supabase) → PostgreSQL → Frontend web
```

Y dentro del frontend, cuatro páginas (`index.html`, `historial.html`, `alertas.html`,
`configuracion.html`) que comparten un mismo `<header>` + `<nav>` y se apoyan en módulos
JS especializados (`api.js`, `app.js`, `historial.js`, `alertas` vía `app.js`,
`configuracion.js`, `modal-intro.js`).

## Flujo de datos

El flujo identificado por el grafo, **validado leyendo el código fuente**, es:

1. **Sensores → ESP32**: el firmware (`firmware/econodo_esp32/econodo_esp32.ino`) lee
   el BME680 (temperatura, humedad, presión y resistencia de gas, usada como indicador
   relativo de compuestos orgánicos volátiles — no como medición certificada de
   concentración química) por SPI y el SDS011 (PM2.5, PM10) por UART en cada ciclo de
   `loop()`.
2. **ESP32 → Edge Function**: cada `INTERVALO_ENVIO_MS` (60 segundos), `enviarLectura()`
   arma un payload JSON y lo envía por HTTPS (`HTTPClient.POST`) a la Edge Function
   `ingest-lectura`, autenticándose con un token de nodo en el header `Authorization: Bearer`.
3. **Edge Function → PostgreSQL**: `supabase/functions/ingest-lectura/index.ts` valida el
   método HTTP, el token (`NODO_SECRET_TOKEN`), el cuerpo JSON, los campos requeridos y el
   valor de `estado`; luego inserta el registro en la tabla `lecturas` usando un cliente de
   Supabase con la `service_role key` (inyectada como variable de entorno, nunca expuesta
   al frontend).
4. **PostgreSQL → Frontend**: `js/api.js` consulta la tabla `lecturas` vía la API REST de
   Supabase (PostgREST) usando la `anon key` pública, con `obtenerUltimaLectura()` (última
   lectura) y `obtenerHistorial()` (últimas 200 lecturas).
5. **Frontend → Usuario**: `app.js`, `historial.js`, `alertas` (en `app.js`) y
   `configuracion.js` consumen esos datos y actualizan el DOM, las gráficas (Chart.js) y
   las alertas en pantalla.

Un detalle relevante confirmado en el código: **todo el flujo tiene un mecanismo de
respaldo (fallback) a datos simulados**. Si `USE_SUPABASE` es `false`, si la petición
falla (red, CORS, error HTTP) o si la tabla devuelve cero filas, `js/api.js` recurre a
`generarDatosFake()` / `generarDatosFakeHistorial()`. Esto permite que el sitio funcione
de forma autocontenida (por ejemplo, en GitHub Pages) incluso sin un nodo físico activo.

## Componentes principales

- **Firmware ESP32** (`firmware/econodo_esp32/econodo_esp32.ino`): lectura de sensores,
  cálculo del estado de calidad del aire, control de una pantalla LCD 1602A paralela
  (HD44780, modo 4 bits) mediante un carrusel de páginas con efecto *marquee* no
  bloqueante, y envío periódico de lecturas a la nube con reintentos.
- **Edge Function `ingest-lectura`** (`supabase/functions/ingest-lectura/index.ts`):
  punto de entrada único y autenticado para insertar lecturas en la base de datos,
  con validaciones de método, token, payload y estado.
- **Capa de datos del frontend** (`js/api.js`): abstrae el origen de los datos
  (Supabase real o datos simulados) detrás de dos funciones (`obtenerUltimaLectura`,
  `obtenerHistorial`).
- **Dashboard** (`index.html` + `js/app.js`): tarjetas en vivo de temperatura, humedad,
  presión y calidad del aire (con PM2.5, PM10 y un indicador de resistencia de gas
  mostrado como `voc`, derivado del BME680 — no una concentración química medida),
  actualizadas cada 3 segundos, más un modal introductorio.
- **Historial** (`historial.html` + `js/historial.js`): cuatro gráficas de Chart.js
  actualizadas in-place, con filtros por período (Hoy / 7 días / 30 días / Todo).
- **Alertas** (`alertas.html`, lógica en `js/app.js`): generación de alertas según
  umbrales configurables, con estado por tipo de sensor para evitar duplicados, sonido
  opcional (Web Audio API) y autodesaparición.
- **Configuración** (`configuracion.html` + `js/configuracion.js`): formulario que lee
  y escribe preferencias en `localStorage` (umbrales de alerta, sonido, duración,
  tipos de alerta activos).
- **Build de estilos** (`gulpfile.js` + `src/scss/`): SCSS modular compilado a
  `build/css/app.css` mediante Gulp + Dart Sass.

## Archivos centrales

Los "god nodes" reportados por Graphify se confirmaron como puntos de integración reales:

- `index.html` — la página con más conexiones; concentra navegación, tarjetas del
  dashboard y el modal introductorio.
- `js/app.js` (`actualizarSistema`, `generarAlertas`, `alertasActivas`) — el módulo que
  conecta la capa de datos, el estado de alertas y el renderizado del dashboard y la
  página de alertas.
- `js/historial.js` (`actualizarGraficasCustom`) — patrón de actualización in-place de
  las cuatro gráficas de Chart.js.
- `supabase/functions/ingest-lectura/index.ts` — único punto de entrada autenticado
  para escribir en la base de datos.
- Documentación del firmware LCD (`docs/lcd-*.md`, `docs/modal-introduccion-dashboard.md`)
  — concentran la mayor densidad de decisiones de diseño documentadas.

## Relaciones importantes

- **`generarDatosFake` ↔ `obtenerUltimaLectura`/`obtenerHistorial`**: Graphify identificó
  correctamente que los datos simulados son el *fallback* de la capa real de datos, no un
  sustituto permanente — el código en `js/api.js` lo confirma explícitamente.
- **`alertas.html` → `actualizarSistema`**: la página de alertas reutiliza la misma
  lógica de generación/actualización que el dashboard (ambas cargan `js/app.js`).
- **Patrón arquitectónico replicado** (hyperedge detectado): la cadena
  ESP32 → Supabase → Dashboard aparece documentada de forma consistente en el modal
  introductorio, en `CLAUDE.md`, en el plan de la Edge Function y en el propio firmware.
- **Evolución documentada del firmware LCD** (hyperedge detectado): los documentos en
  `docs/` narran, en orden, el intento inicial con módulo I2C, su corrección a una LCD
  paralela HD44780, el carrusel de páginas, el efecto marquee continuo y el ajuste de
  fluidez frente a una llamada HTTP bloqueante — una traza útil de cómo evolucionó el
  diseño y por qué.

## Tecnologías detectadas

Confirmadas por inspección directa del código y la configuración:

- **Hardware/firmware**: ESP32, sensor BME680 (SPI), sensor SDS011 (UART), pantalla
  LCD 1602A paralela (HD44780, librería `LiquidCrystal`), Arduino (`.ino`,
  `WiFi.h`, `HTTPClient.h`, `WiFiClientSecure.h`).
- **Backend / nube**: Supabase (Auth REST / PostgREST, Edge Functions sobre Deno,
  base de datos PostgreSQL, variables de entorno para secretos).
- **Frontend**: HTML5, JavaScript (ES2017+, `async/await`, módulos sin bundler),
  SCSS/Sass compilado con Gulp, Chart.js (vía CDN), Web Audio API, `localStorage`.
- **Despliegue**: GitHub Pages (sitio estático, sin paso de build para HTML/JS).
- **Herramientas de desarrollo**: npm, Gulp 5, `gulp-sass`, Dart Sass.

## Fortalezas del proyecto

- Separación clara de responsabilidades: capa de datos (`api.js`), lógica de negocio
  (`app.js`, `historial.js`, `configuracion.js`), presentación (HTML/SCSS) y firmware
  (`.ino`) están desacoplados y documentados.
- **Diseño resiliente**: el frontend funciona con o sin backend real gracias al
  mecanismo de *fallback* a datos simulados, lo cual también facilita las demostraciones.
- **Manejo cuidadoso de secretos**: `secrets.h`, `js/config.js`, `.mcp.json` y los
  archivos `.env` de las Edge Functions están en `.gitignore`; existen plantillas
  (`secrets.example.h`, `js/config.example.js`) para que cualquiera pueda configurar su
  propia instancia sin exponer credenciales.
- **Documentación interna abundante**: ocho documentos en `docs/` narran decisiones de
  diseño, correcciones de hardware y cambios de UI con su razonamiento (el "por qué"),
  lo cual Graphify pudo aprovechar para construir nodos de tipo `rationale`.
- **Comunicación firmware ↔ nube robusta**: reintentos con backoff simple, *timeouts*
  cortos para no bloquear la actualización de la LCD, y validación estricta en el
  servidor (Edge Function) de token, payload y estado.

## Riesgos o deuda técnica detectada

- **`node_modules/` está rastreado por Git** (1369 archivos según `git ls-files`,
  pese a estar listado en `.gitignore`). Esto infla el repositorio y puede causar
  conflictos; se recomienda una limpieza con `git rm -r --cached node_modules` en una
  rama/commit dedicado — **no se realizó ninguna acción automática**, solo se reporta.
- **`build/css/app.css` (CSS compilado) está versionado**, mientras que `.gitignore`
  solo excluye los `*.map`. Esto puede generar diffs ruidosos cada vez que se recompila
  el SCSS; conviene decidir conscientemente si se versiona el build o se genera en
  despliegue.
- **`js/config.js` está versionado** con una URL de proyecto y una clave pública
  (`anon key`) de Supabase reales. Esto **no es una fuga de credenciales** — se validó
  que el rol embebido en el JWT es `anon` (clave pública diseñada para exponerse en
  clientes, protegida por políticas de Row Level Security en PostgreSQL) y el mensaje
  del commit (`Preparar configuración pública para GitHub Pages`) confirma que fue
  intencional para el despliegue estático. Sin embargo, el comentario dentro de
  `js/config.example.js` ("…NUNCA debe subirse al repo") queda **desactualizado /
  contradictorio** respecto al estado real y conviene aclararlo para evitar confusión.
- **51 nodos aislados** reportados por Graphify (p. ej. `filtroDia`, `option`, `labels`,
  variables de gráficas) — en su mayoría son variables de UI o identificadores de bajo
  nivel sin relaciones semánticas relevantes; no representan deuda real, pero confirman
  que el grafo es más útil para mapear módulos que para auditar variables sueltas.
- **Cohesión baja en algunas comunidades** (p. ej. "Historial Charts & Filters": 0.10;
  "Dashboard Layout & Navigation": 0.14): son agrupaciones naturales de una SPA pequeña
  (UI + lógica + datos en el mismo archivo), no necesariamente señal de que deban
  dividirse — pero es un punto a vigilar si el proyecto crece.

## Recomendaciones futuras

- Decidir conscientemente si `node_modules/` y `build/css/app.css` deben seguir
  versionados, y limpiar el historial si no es así (en una rama separada, sin afectar
  `main` directamente).
- Actualizar el comentario de `js/config.example.js` para reflejar con precisión que la
  `anon key` de Supabase es pública por diseño (a diferencia de `secrets.h`, que sí debe
  permanecer fuera del repositorio).
- Mantener actualizada la documentación técnica (`docs/lcd-carrusel-datos.md` describe
  `TOTAL_PAGINAS_LCD = 9`, mientras que el firmware actual usa `7`); ver sección de
  validación manual.
- Considerar mover la lógica de alertas (actualmente dentro de `app.js`) a su propio
  módulo si el dashboard sigue creciendo, dado que ya concentra varias responsabilidades
  (estado, render, sonido, iconografía).

## Archivos excluidos por seguridad

Los siguientes archivos **no fueron leídos en su contenido sensible, ni se incluyó
ningún valor de ellos en este análisis ni en el README**:

- `firmware/econodo_esp32/secrets.h` (credenciales WiFi y token del nodo — gitignored)
- `js/config.js` (se verificó únicamente que la clave embebida es de tipo `anon`,
  pública por diseño de Supabase; no se transcribió su valor)
- `.mcp.json`
- `Supabase Econodo Password`
- `firmware/econodo_esp32_old/` (firmware antiguo con credenciales hardcodeadas,
  conservado solo como referencia local y excluido de Git)

## Validación manual de los hallazgos

Se contrastaron los hallazgos más relevantes del grafo contra el código real:

| Hallazgo de Graphify | Validación manual | Resultado |
|---|---|---|
| Flujo ESP32 → Edge Function → Supabase | Lectura de `enviarLectura()` en el `.ino` y de `index.ts` | **Confirmado**: autenticación por token, validación de payload, inserción con `service_role key` |
| `js/historial.js` descrito como "hardcoded historial array" (heredado de `CLAUDE.md`) | Lectura completa de `historial.js` y `api.js` | **Desactualizado**: el array `historial` ahora se llena dinámicamente desde `obtenerHistorial()` (Supabase con fallback simulado), no está hardcodeado |
| `TOTAL_PAGINAS_LCD = 9` (nodo extraído de `docs/lcd-carrusel-datos.md`) | `grep` en `econodo_esp32.ino` línea 136 | **Desactualizado**: el firmware actual define `TOTAL_PAGINAS_LCD = 7`; el documento describe una iteración anterior del carrusel |
| `js/config.js` con credenciales reales versionado | Decodificación del JWT (solo el claim `role`, sin imprimir la clave) | **Confirmado pero no es un riesgo**: rol `anon`, clave pública de Supabase, commit explícito "Preparar configuración pública para GitHub Pages" |
| Despliegue en GitHub Pages | `WebFetch` a `https://jazielaguirre.github.io/EcoNodo/` | **Pendiente**: la URL responde `404 Not Found`; no hay rama `gh-pages` ni workflow de despliegue configurado en este repositorio |
| Uso de `localStorage` para configuración | Lectura de `configuracion.js` y `app.js` (`obtenerUmbrales`, `obtenerConfigAlerta`) | **Confirmado**: las preferencias (umbrales, sonido, duración, tipos de alerta) se guardan y leen bajo la clave `econodo_config` |
| Manejo de secretos | Revisión de `.gitignore` y `git ls-files` | **Confirmado**: `secrets.h`, `.mcp.json`, `.env` y la contraseña de Supabase están excluidos; `secrets.example.h` y `config.example.js` sirven de plantilla |
| `node_modules` rastreado por Git | `git ls-files | grep node_modules` | **Confirmado**: 1369 archivos versionados pese a estar en `.gitignore` (probablemente quedaron de antes de agregarlo) |

## Conclusión

El análisis de Graphify resultó, en términos generales, **preciso y útil**: identificó
correctamente la arquitectura de extremo a extremo, los módulos centrales, el patrón de
*fallback* a datos simulados y la evolución documentada del firmware de la LCD. Las
únicas discrepancias encontradas (`TOTAL_PAGINAS_LCD` y la descripción de
`historial.js` como "hardcoded") provienen de **documentación interna desactualizada**
(`docs/lcd-carrusel-datos.md` y `CLAUDE.md`), no de errores de extracción del propio
Graphify — el grafo simplemente reflejó fielmente lo que esos documentos decían en el
momento de la extracción. Se usó el código fuente actual como fuente de verdad para
resolver ambas discrepancias antes de redactar el README.

No se detectaron secretos privados versionados: `secrets.h`, `.mcp.json` y las
contraseñas están correctamente excluidos de Git. El frontend contiene una `anon key`
pública de Supabase (`js/config.js`), cuyo uso es esperado en una aplicación cliente y
cuya seguridad depende de que las políticas de Row Level Security de PostgreSQL estén
correctamente configuradas — no de ocultar la propia clave.
