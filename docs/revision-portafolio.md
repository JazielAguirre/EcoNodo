# Revisión del repositorio para presentación como portafolio

**Fecha:** 2026-06-06
**Objetivo:** dejar constancia de una revisión manual del repositorio `EcoNodo` antes de
compartirlo como pieza de portafolio profesional — qué está en buen estado, qué conviene
ajustar y qué requiere atención antes de mostrarlo a terceros.

Esta revisión complementa el análisis de arquitectura en
[`analisis-graphify.md`](analisis-graphify.md); aquí el foco es la **presentación del
repositorio** (higiene de Git, seguridad superficial, recursos visuales y despliegue),
no el diseño del sistema.

> No se muestra ninguna clave, token ni credencial real en este documento.

---

## Estado general

El proyecto está funcionalmente completo y bien documentado: el firmware, la Edge
Function, el frontend y la carpeta `docs/` reflejan un proceso de desarrollo real, con
decisiones registradas y corregidas sobre la marcha. Antes de compartirlo como portafolio,
quedan pendientes tareas de **higiene de repositorio** (dependencias versionadas de más),
**recursos visuales** (capturas, fotos del nodo) y **metadatos del proyecto** (licencia,
activación de GitHub Pages) — ninguna de ellas afecta al funcionamiento del sistema.

## Aspectos correctos

- **Separación de secretos**: `secrets.h`, los `.env` de las Edge Functions, `.mcp.json`
  y el archivo de contraseña de Supabase están correctamente listados en `.gitignore` y
  no aparecen en el historial de archivos rastreados.
- **Patrón `*.example.*`**: tanto `js/config.example.js` como
  `firmware/econodo_esp32/secrets.example.h` documentan la forma de las credenciales sin
  exponer valores reales — una práctica que cualquier persona que clone el repo agradece.
- **`build/css/app.css` versionado intencionalmente**: el CSS compilado está incluido en
  el repositorio y **debe permanecer así**. GitHub Pages sirve los archivos estáticos
  directamente, sin ejecutar ningún paso de build (no hay Sass disponible en el
  despliegue), y las páginas HTML enlazan ese archivo de forma directa
  (`build/css/app.css`). Quitarlo del control de versiones rompería el sitio publicado.
  Solo los mapas de origen (`build/css/*.map`) están — correctamente — ignorados.
- **Documentación del proceso**: la carpeta `docs/` registra decisiones técnicas,
  errores corregidos (por ejemplo, el cambio de un módulo LCD I2C a uno paralelo HD44780)
  y cambios de diseño con su razonamiento — algo poco común y valioso en un portafolio.
- **Capa de *fallback* a datos simulados**: tanto `obtenerUltimaLectura()` como
  `obtenerHistorial()` (en `js/api.js`) recurren a datos de ejemplo si Supabase no está
  configurado, la petición falla o no hay filas — el sitio es siempre navegable, incluso
  sin nodo físico activo.

## Aspectos que conviene corregir

- **`node_modules/` rastreado por Git** (ver sección siguiente): infla el historial y el
  tamaño del repositorio sin aportar valor — las dependencias se reinstalan con
  `npm install`.
- **Documentación desactualizada respecto al firmware actual**: `docs/lcd-carrusel-datos.md`
  describía una iteración del carrusel LCD con `TOTAL_PAGINAS_LCD = 9`, mientras que el
  firmware vigente usa `7` tras una refactorización posterior (marquee continuo). Ya se
  añadió una nota de advertencia al inicio de ese documento enlazando a la versión
  vigente (`lcd-marquee-carrusel.md`) — se conservó el contenido original como registro
  histórico, sin borrarlo. De forma similar, `CLAUDE.md` describía `js/historial.js`
  como datos "hardcodeados"; ya se corrigió esa descripción para reflejar que el
  historial se obtiene dinámicamente vía `obtenerHistorial()` con *fallback* simulado.
- **Sin capturas ni recursos visuales** (ver sección dedicada más abajo).
- **GitHub Pages sin activar** en el repositorio personal (ver sección dedicada).
- **Sin archivo `LICENSE`** (ver sección dedicada).

## Archivos rastreados innecesariamente

`node_modules/` está incluido en el control de versiones:

```
$ git ls-files node_modules | wc -l
1369
```

**1369 archivos** de dependencias de Node.js están actualmente rastreados por Git, pese a
que `node_modules/` ya figura en `.gitignore`. Esto ocurre porque `.gitignore` solo afecta
a archivos *no rastreados todavía*: una vez que un archivo se añadió al índice de Git,
seguir ignorándolo en `.gitignore` no lo "destraquea" automáticamente.

**Recomendación** (no ejecutada en esta revisión, queda documentada para aplicarse de
forma consciente):

```bash
git rm -r --cached node_modules
git add .gitignore
git commit -m "Dejar de versionar dependencias de Node.js"
```

Notas sobre este comando:
- `git rm -r --cached` elimina los archivos **únicamente del índice de Git** (deja de
  rastrearlos); no borra nada del disco — `node_modules/` permanece instalado localmente
  y el proyecto sigue funcionando con `npm install` / `npm run sass` / `npm run dev`.
- `git add .gitignore` confirma que la regla `node_modules/` queda activa para el futuro.
- Tras el commit, cualquier persona que clone el repositorio simplemente ejecuta
  `npm install` para regenerar la carpeta — el flujo de trabajo no cambia.

> Esta limpieza no se ejecutó como parte de esta revisión (se pidió explícitamente no
> hacer commits ni cambios destructivos). Queda como una acción recomendada, a realizar
> de forma deliberada y revisando el resultado de `git status` antes de confirmar.

## Seguridad

Se revisó el contenido de `js/config.js` (sin exponer su valor) decodificando únicamente
los campos `role` e `iss` del JWT que contiene. El resultado confirma que se trata de una
clave **`anon`** (pública) de Supabase — **no** una clave `service_role` (administrativa).

Esto es relevante porque:

- Las claves `anon` están **diseñadas para exponerse** en aplicaciones cliente: cualquier
  persona que abra las herramientas de desarrollador del navegador puede verla, y eso es
  esperado por el propio modelo de seguridad de Supabase.
- **Su exposición solo es aceptable si las políticas de Row Level Security (RLS) de la
  base de datos restringen correctamente el acceso** — es decir, si la tabla `lecturas`
  (y cualquier otra expuesta vía PostgREST) tiene políticas que limitan las operaciones
  permitidas con esa clave (por ejemplo, solo lectura, sin acceso a otras tablas).
- Esta revisión **no verificó las políticas RLS configuradas en el proyecto de Supabase**
  — esa comprobación requiere acceso al panel del proyecto y queda fuera del alcance de
  una revisión de archivos locales. Es la pieza que falta para confirmar que la
  exposición de la clave `anon` es efectivamente segura.

No se detectaron secretos privados versionados: el token del nodo ESP32, las
credenciales WiFi y las claves de servicio de las Edge Functions están correctamente
excluidos del repositorio mediante `.gitignore`, sin rastros de ellos en el historial de
archivos rastreados. El frontend contiene una `anon key` pública de Supabase, cuyo uso
es esperado en una aplicación cliente y cuya seguridad depende de políticas RLS
correctamente configuradas (ver detalle arriba).

## Capturas y recursos visuales pendientes

El `README.md` ya incluye una sección "Capturas del proyecto" con marcadores `TODO`
indicando exactamente qué imágenes faltan:

- Captura del modal introductorio del dashboard
- Captura del dashboard con las tarjetas de sensores
- Captura del historial con las gráficas de Chart.js
- Captura del panel de alertas
- Captura de la página de configuración
- Fotografía del nodo ESP32 con los sensores y la pantalla LCD
- Fotografía o video corto de la pantalla LCD mostrando el carrusel de datos

**Recomendación**: antes de compartir el repositorio como pieza de portafolio, capturar
estas imágenes (con datos simulados es suficiente — no es necesario tener el nodo físico
encendido) y sustituir los comentarios `TODO` por las imágenes reales con rutas relativas
(por ejemplo, `docs/img/dashboard.png`). Las fotografías del nodo físico son las que más
valor aportan: son las que diferencian a EcoNodo de un proyecto puramente de software.

## GitHub Pages

Se verificó el estado del despliegue:

- `https://jazielaguirre.github.io/EcoNodo/` → responde **404** (Pages no está activado
  para este repositorio personal todavía).
- `https://neyralopez.github.io/econodo/` → responde correctamente (despliegue del
  repositorio original/upstream).

El `README.md` ya refleja este estado en su sección "Demostración", indicando que la
activación está pendiente y sin enlazar al despliegue ajeno como si fuera el propio.

**Recomendación**: activar GitHub Pages desde la configuración del repositorio
(`Settings → Pages`, rama `main`, carpeta raíz) antes de compartir el enlace en el
portafolio — el sitio ya está preparado como estático y no requiere ningún paso de build
adicional para HTML/JS.

## Licencia

No existe un archivo `LICENSE` en la raíz del repositorio.

**Recomendación**: agregar uno antes de presentar el proyecto públicamente, para dejar
explícito qué pueden hacer terceros con el código (por ejemplo, MIT es una opción común
y permisiva para proyectos de portafolio). Esta revisión no crea el archivo de licencia
—es una decisión que corresponde al autor del repositorio—, pero el `README.md` ya
incluye una sección "Licencia" que señala su ausencia de forma transparente.

## Recomendaciones antes de compartir el proyecto

En orden sugerido de prioridad:

1. **Tomar y agregar las capturas/fotos pendientes** — es lo que más impacto visual tiene
   para quien revise el portafolio por primera vez.
2. **Decidir y agregar una licencia** (por ejemplo, MIT) — clarifica el uso del código.
3. **Activar GitHub Pages** y actualizar el enlace de demostración en el `README.md`
   una vez que el sitio esté disponible en el dominio propio.
4. **Dejar de versionar `node_modules/`** ejecutando — de forma consciente y revisando
   el resultado — el comando documentado en la sección correspondiente de este archivo.
5. **Confirmar las políticas RLS en el panel de Supabase** para validar que la exposición
   de la clave `anon` en `js/config.js` está efectivamente acotada.

Ninguna de estas tareas requiere modificar la lógica funcional del firmware, las Edge
Functions o el frontend — son ajustes de presentación, higiene de repositorio y
configuración externa.
