# API Hunter

Herramienta de reconocimiento para bug bounty: abre el sitio objetivo en un
navegador real (Playwright/Chromium), navega por él, intercepta todas las
llamadas XHR/fetch, y además descarga y analiza estáticamente cada archivo
`.js` cargado para extraer rutas de API que quizás no se disparan durante el
crawl (endpoints admin, flujos ocultos, feature flags, etc.).

Con todos los endpoints descubiertos, marca como **candidatos a
IDOR/BOLA** aquellos cuyo path o query params contienen algo que parece un
identificador de recurso (ID numérico secuencial, UUID, ObjectId de Mongo,
hash/token, o un nombre de parámetro típico como `user_id`, `account_id`,
`order_id`, etc.) y genera un reporte HTML priorizado por riesgo, con una
sugerencia de cómo probarlo **manualmente** (la herramienta NO envía
payloads de explotación ni intenta confirmar el IDOR por ti — eso lo
decides y ejecutas tú, con autorización).

## Uso previsto

Solo contra objetivos para los que tengas autorización explícita (programas
de bug bounty, pentests contratados, tus propios laboratorios).

## Instalación

```bash
cd api-hunter
npm install
npx playwright install chromium
```

## Uso básico

```bash
node src/index.js --url https://objetivo.com
```

## Crawl autenticado (recomendado para encontrar más IDOR)

La mayoría de los IDOR interesantes están detrás de login, porque ahí es
donde existen "mis recursos" vs "recursos de otro usuario". Pásale tu
cookie de sesión (obtenida de tu propia cuenta de prueba) para que el
crawler navegue autenticado:

```bash
node src/index.js --url https://objetivo.com/dashboard --cookie "session=TU_COOKIE_AQUI"
```

## Motor de descubrimiento: katana (real) + red de seguridad propia

El descubrimiento de URLs/JS ya **no** depende solo de un crawler casero.
El flujo real es:

1. **`crawlSite` (Playwright)** visita las páginas con un navegador real,
   captura XHR/fetch y el HTML ya renderizado de cada página.
2. **`katana`** (https://github.com/projectdiscovery/katana, el crawler de
   ProjectDiscovery) corre como motor de descubrimiento principal, si está
   instalado — detecta la instalación automáticamente
   (`isKatanaAvailable()` en `src/katanaRunner.js`) y si no está presente
   sigue solo con lo propio, avisando cómo instalarlo. Se invoca con
   `-jc` (parsing de endpoints dentro de JS), `-kf all` (robots.txt +
   sitemap.xml) y salida `-jsonl`. Con `--katana-jsluice` activa `-jsl`
   (parsing de JS a nivel AST vía jsluice — más preciso que cualquier
   regex, más lento/memoria). Con `--katana-headless` usa Chrome headless
   de katana para SPAs que solo cargan ciertas rutas/scripts al ejecutarse.
3. **`src/jsCrawler.js`** (nuestro BFS propio) corre ENCIMA de la semilla
   combinada (crawl dinámico + katana) como red de seguridad adicional:
   descarga cada `.js`, busca referencias a OTROS `.js` adentro (`import()`
   dinámico, `importScripts()`, nombres de chunk de webpack/Next/Vite) y
   sigue la cadena hasta `--js-depth` niveles o `--max-js-files` archivos.
   También intenta `//# sourceMappingURL=...` para sacar nombres de
   archivo originales de los sourcemaps.
4. Cada `.js` descubierto —por la vía que sea— se analiza **uno por uno**
   con `jsExtractor.js`: saca endpoints, el VERBO real de la llamada
   (`.post`/`.put`/`.get`...) y los nombres de campo del body
   (`{cedula:e}`, `JSON.stringify({id_usuario:...})`) para cruzarlos contra
   nombres típicos de ID/PII en `endpointAnalyzer.js`.

katana resuelve el descubrimiento (crawling real, headless si hace falta,
jsluice a nivel AST); nuestro código sigue siendo el que **puntúa**
candidatos a IDOR/BOLA mirando nombres de campo — eso katana no lo hace.

### Instalar katana (opcional, pero recomendado)

```bash
go install github.com/projectdiscovery/katana/cmd/katana@latest
# o bajar el binario precompilado:
# https://github.com/projectdiscovery/katana/releases
```

Si no está instalado, `api-hunter` lo detecta y sigue funcionando solo con
el crawler propio (`--use-katana false` para forzar eso aunque esté
instalado).

### Certificados TLS inválidos/autofirmados

Varios objetivos reales (sobre todo infraestructura interna) tienen TLS
mal configurado — por eso tantos reportes de bug bounty usan `curl -k`.
El `fetch()` nativo de Node rechaza esos certificados por defecto y antes
**lo hacía en silencio** (0 archivos encontrados, sin ninguna pista de por
qué). Ahora:

- `--insecure` hace que tanto el crawl de Playwright como las descargas de
  JS ignoren errores de certificado (equivalente a `curl -k`).
- Cualquier fetch que falle (TLS, timeout, 404, lo que sea) se reporta en
  consola con la URL y el motivo — ya no desaparece en silencio.

## Opciones

| Flag             | Default               | Descripción                                   |
|------------------|------------------------|------------------------------------------------|
| `--url`          | (requerido)            | URL inicial del crawl                           |
| `--max-pages`    | 15                     | Límite de páginas internas a visitar            |
| `--cookie`       | —                      | Header Cookie para crawl autenticado            |
| `--out`          | ./api-hunter-output    | Carpeta de salida (JSON + HTML)                 |
| `--headless`     | true                   | `--no-headless` para ver el navegador en acción |
| `--max-js-files` | 150                    | Límite total de `.js` a descubrir/descargar (BFS) |
| `--js-depth`     | 6                      | Profundidad máxima siguiendo referencias JS→JS  |
| `--insecure`     | false                  | Como `curl -k`: ignora certificados TLS inválidos/autofirmados |
| `--use-katana`   | true                   | Usar el binario real de `katana` como motor principal, si está instalado |
| `--katana-headless` | false               | katana usa Chrome headless (más lento, ve rutas que solo aparecen al ejecutar la SPA) |
| `--katana-jsluice`  | false               | katana usa jsluice (parsing de JS a nivel AST, más preciso — más memoria) |
| `--katana-depth`    | 3                   | Profundidad de crawl de katana |

## Salida

- `api-hunter-output/inventory.json` — inventario completo, por si quieres
  alimentarlo a otra herramienta (ej. tu DOMXRay o scripts propios).
- `api-hunter-output/report.html` — reporte visual, filas en rojo = candidatos
  a IDOR, con la columna "Cómo probarlo manualmente".

## Cómo interpretar el riesgo

- **IDs numéricos secuenciales en el path** (`/api/orders/4521`): riesgo
  alto, fáciles de enumerar (`4520`, `4522`...).
- **UUID / Mongo ObjectId**: riesgo medio — no enumerable por fuerza bruta,
  pero igual vale la pena probar con un ID real de otra cuenta tuya (horizontal
  IDOR) si tienes dos usuarios de prueba.
- **Métodos POST/PUT/PATCH/DELETE** con ID: suman puntos extra porque, si son
  vulnerables, el impacto es de modificación/borrado de datos ajenos, no solo
  lectura.

## Próximos pasos sugeridos (no incluidos todavía)

- Soporte multi-cuenta: loguear con dos sesiones distintas automáticamente y
  comparar respuestas línea por línea (reduce el trabajo manual a revisar
  diffs).
- Export a Burp/Caido (sitemap o `.har`) para seguir trabajando ahí.
- Detección de GraphQL (introspección) como módulo aparte.

Dime cuál de estos quieres que construyamos primero y seguimos iterando.
