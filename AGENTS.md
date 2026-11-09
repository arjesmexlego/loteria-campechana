# AGENTS.md — Generador de Cartillas Lotería Campechana

Instrucciones operativas para agentes de IA (y notas de contexto para cualquier
colaborador) que trabajen en este repositorio.

> **Este archivo es la memoria del proyecto.** Las herramientas de IA no
> recuerdan nada entre sesiones: lo único que sobrevive es lo que está escrito
> aquí y en `ANALISIS_PROYECTO_LOTERIA_CAMPECHANA.md`. Si algo no está
> documentado, no está guardado.

---

## 0. Estado actual y cómo retomar

| Dato | Valor |
|---|---|
| Rama | `main` (única) |
| Remoto | `https://github.com/arjesmexlego/loteria-campechana.git` |
| Deploy | GitHub Pages, serviendo `main` sin build step |
| Versión actual de `index.html` | `main`, **1178 líneas** (CSS 7–365, HTML 367–418, JS 420–1176) |
| Referencia histórica limpia | `c395fe1:index.html` (660 líneas) |
| Último commit funcional | `0a4ad29` — esquinas redondeadas y separación de 2 px entre cartillas |
| Estado del árbol | limpio, `main` == `origin/main` |

**Protocolo al iniciar una sesión nueva:**

1. Leer este `AGENTS.md` completo y el §5.1 de `ANALISIS_...md`.
2. `git fetch origin && git status` → confirmar árbol limpio y sin pendientes.
3. `git pull origin main` → sincronizar (ver §6.1, **obligatorio**: el proyecto
   también se edita desde la web de GitHub).
4. Antes de terminar cualquier cambio: `git grep -n -E '^(<<<<<<<|>>>>>>>|=======$)' -- .`
   debe dar cero (§8).

---

## 1. Qué es el proyecto

Aplicación web **estática** (HTML + CSS + JS vanilla, cero dependencias) que
genera **cartillas imprimibles** para el juego de Lotería Campechana.

- Formatos: **5x5** (tradicional, 25 casillas) y **3x3** (rápido, 9 casillas).
- Mazo completo de **90 fichas** numeradas del 1 al 90.
- Modos de generación: **Al Azar** (aleatorio) y **Juego Completo** (reparto
  matemático que cubre el mazo completo).
- Color global del **marco del centro**, con opción **"Sin color"** (fondo
  blanco y sin marco).
- Salida principal: **impresión/PDF con medidas reales en centímetros**
  (tamaño carta, margen 0.5 cm, separación de 2 px entre cartillas y esquinas
  redondeadas de 6 px, escala 100%).
- Salida alternativa: **ZIP con un PNG por cartilla** (300 DPI, sin cabecera ni
  selector, esquinas redondeadas), generado en el navegador sin dependencias
  externas.

Todo el código, comentarios y textos de UI están en **español (MX)**. Mantener
ese idioma en cualquier código o documentación nueva.

---

## 2. Estructura del repositorio

```text
Loteria-campechana/
├── index.html            # TODO el código: CSS + HTML + JS en un solo archivo (1178 líneas)
├── AGENTS.md             # Este archivo (contexto + reglas de trabajo)
├── ANALISIS_PROYECTO_LOTERIA_CAMPECHANA.md   # Análisis técnico detallado
├── .gitignore
├── fichas/               # 90 imágenes .webp (formato principal)
│   └── cartilla-Cg6x0W1r.js.descarga   # Sin uso. NO referenciado. Candidado a borrar.
└── fichaspng/            # 90 imágenes .png (respaldo / fallback)
```

**No ignorar con `.gitignore`**: `fichas/` ni `fichaspng/` (son assets
necesarios para que la app funcione en producción).

### 2.1 Mapa de `index.html`

| Rango | Contenido |
|---|---|
| 1–6 | `<head>`, metadatos |
| 7–365 | `<style>` — variables CSS, UI, grids, estados, color del centro, `@media print` |
| 367–418 | `<body>` — título, panel de control (formato, lado, color, generación), `#contenedor-cartillas`, botones de descarga e impresión |
| 420–1176 | `<script>` — toda la lógica (generación, render, swap, exportación PNG/ZIP) |

Anclas del DOM usadas por JS (no renombrar sin actualizar el JS):

- `#input-lado` (number, 5–19, step 0.5)
- `#input-cantidad` (number, 1–100)
- `#input-color-centro` (color), `#chk-sin-color-centro` (checkbox)
- `#btn-azar`, `#btn-completo`, `#btn-planilla`
- `#btn-descargar-imagenes` (botón PNG/ZIP)
- `#contenedor-cartillas`
- `body[data-cant]` — cantidad de cartillas objetivo según formato
- `body.sin-color-centro` — activa el modo "Sin color" del marco central

---

## 3. Lógica de dominio (no romper estos invariantes)

### 3.1 Estado global

```js
const CARPETA_FICHAS = 'fichas';
const EXTENSION_PRINCIPAL = '.webp';
const EXTENSION_RESPALDO = '.png';
const MAZO_LOTERIA = Array.from({ length: 90 }, (_, i) => ({ id, nombre, img }));

let cartillasGuardadas = [];   // array de arrays de fichas
let currentFormato = 5;        // 5 | 3
let currentCentroIdx = 12;     // 12 en 5x5, 4 en 3x3
let slotsPorCartilla = 25;

// Color global del marco del centro
let colorCentroActual = '#ffa000';
let centroSinColor = false;

// Exportación PNG/ZIP
let exportandoImagenes = false;
const cacheImagenesExportacion = new Map();
const FACTOR_CSS_A_300DPI = 300 / 96;
```

### 3.2 "Juego Completo" — matemática

**3x3 (partición exacta):** 10 cartillas × 9 = **90 casillas = 90 fichas**, sin
repeticiones. Se baraja el mazo y se corta en 10 bloques consecutivos
(`slice(i*9, (i+1)*9)`).

**5x5 (reparto controlado):** 4 cartillas × 25 = 100 casillas para 90 fichas →
**10 fichas repetidas exactamente 2 veces**.
1. `repetidas = mazo90.slice(0,10)`, `unicas = mazo90.slice(10,90)`
2. Cada repetida se asigna a 2 de las 4 cartillas (`barajar([0,1,2,3]).slice(0,2)`)
3. Se rellenan las casillas libres con `unicas`
4. Cada cartilla se mezcla internamente
5. **Garantía de centros únicos** entre las 4 cartillas: si dos cartillas
   comparten ficha central, se intercambia el centro con otra celda no-central
   cuyo ID aún no esté usado como centro.

### 3.3 Invariantes que NUNCA deben romperse

1. **Una misma ficha no puede ser centro de dos cartillas** a la vez.
   Si ya se usa como centro, aparece deshabilitada en el selector con `🔒 (Usado)`.
2. **El comodín** (`⭐ COMODÍN`) no cuenta como ficha para el bloqueo de centros.
3. **El swap universal preserva el reparto global**: al pedir la ficha X como
   centro, se busca X en cualquier cartilla/celda y se truequea con el centro
   actual. No se debe "simplemente overwrite" una casilla.
4. En 3x3 "Juego Completo" la cobertura del mazo es total y sin repeticiones.
5. **El color del centro se aplica solo al marco** (no al fondo). "Sin color"
   deja el centro con fondo blanco y sin marco. La exportación PNG debe
   reflejar el mismo centro y color que se ven en pantalla.
6. **El ZIP debe ser válido**: método STORE + CRC32, un PNG por cartilla, sin
   librerías externas. No sustituir por `html2canvas` salvo decisión explícita
   (sería la primera dependencia externa).
7. **La Planilla 90 no se exporta** con el botón de cartillas: al entrar en ese
   modo se limpia `cartillasGuardadas` y el botón queda deshabilitado.
8. **La exportación PNG replica el diseño**: cuadrícula negra, borde exterior
   más grueso que las líneas internas y esquinas redondeadas de 6 px (radio
   proporcional en el canvas a 300 DPI).

### 3.4 Fallback de imágenes (2 niveles)

En `crearCelda()`: se intenta `.webp`; en `onerror` se cambia a `.png`; en el
segundo error se elimina el `img` y se inyecta un fallback textual
(`<div class="fallback">` con número + nombre). La lógica depende de
`this.dataset` para no entrar en bucle.

La exportación PNG replica el mismo fallback en canvas: webp → png → texto
(número + nombre) dibujado con `ctx.fillText`.

---

## 4. Reglas de impresión (crítico)

Todo el layout de cartilla usa la variable CSS `--cartilla-lado`
(`index.html:16`, sincronizada por `actualizarMedidas()` en `index.html:448`).

- `@media print` (`index.html:335`) oculta panel de control, título, subtítulo,
  acciones y cabeceras de cartilla.
- `size: letter portrait; margin: 0.5cm`
- `#contenedor-cartillas` con `gap: 2px` y `.cartilla-box` con
  `border-radius: 6px`: las cartillas quedan **casi pegadas** (2 px de
  separación, que sirve de línea de corte) y con **esquinas redondeadas**.
- La cuadrícula sigue siendo negra: el grid usa fondo negro y `gap: 1px`
  (líneas internas de 1 px), mientras que el borde exterior de la cartilla es
  de 2 px (el doble de grosor).
- `page-break-inside: avoid` / `break-inside: avoid`
- `-webkit-print-color-adjust: exact` y `print-color-adjust: exact`
  para preservar el color del marco del centro y del comodín.

**Capacidad por hoja (letter, margen 0.5 cm):**

| Formato | Lado | Cartillas por hoja |
|---|---|---|
| 5×5 | 13.5 cm | 1 |
| 5×5 | ≤ 10 cm | 4 (2×2) |
| 3×3 | 9 cm | 4 (2×2) |

**Al modificar cualquier tamaño de celda o grid, verificar que las reglas de
impresión siguen mandando** (`!important` en `.cartilla-box`, líneas 351–359).
Si se rompe, la cartilla sale del papel.

**Salida PNG (independiente del PDF):** 300 DPI, `lado_cm / 2.54 × 300` px,
sin cabecera ni selector, con el mismo centro, color, cuadrícula negra y
esquinas redondeadas que se ven en pantalla. El ZIP se arma en el navegador
(STORE + CRC32); no añadir dependencias para esto.

---

## 5. Convenciones de código

- Vanilla JS ES6+. **Sin frameworks, bundlers, npm ni librerías externas.**
- Nombres de funciones y variables en `español` (camelCase).
- Comentarios en español.
- Render vía DOM (`createElement` / `appendChild`), nunca `innerHTML` con datos
  de usuario. El único `innerHTML` permitido es para markup estático propio
  (fallback, plantillas de etiqueta).
- Barajado siempre con `barajar()` (Fisher–Yates, `index.html:468`).
- Exportación de imágenes: **canvas 2D + ZIP propio** (STORE/CRC32). No usar
  `html2canvas` ni librerías de ZIP; mantener la app sin dependencias.
- Preferir añadir lógica nueva al final de la sección `<script>` respetando el
  orden: constantes → estado → helpers → generación → render → exportación → eventos.

---

## 6. Git y deploy

- Rama única: `main`. Remoto: `origin` → `https://github.com/arjesmexlego/loteria-campechana.git`
- **Deploy = GitHub Pages serviendo `main` automáticamente.** No hay build step.
  Publicar = hacer push a `main`.

### 6.1 ⚠️ Regla crítica: no subir archivos por la web de GitHub

**Este proyecto se edita con frecuencia desde la interfaz web de GitHub
("Add files via upload").** Eso crea commits en `origin/main` que el clon local
no tiene. Al hacer `git pull`, Git **no puede fusionar un archivo binario/
grande no trackeado** y se produce un conflicto.

**Procedimiento obligatorio antes de tocar nada:**

```bash
git fetch origin
git status                 # confirmar árvore limpia
git pull --no-rebase origin main   # o: git pull origin main
```

Si aparecen marcadores de conflicto (`<<<<<<< HEAD`, `=======`,
`>>>>>>> origin/main`) **dentro de un archivo versionado**, ese archivo está
ROTO en el repo y en producción. Procedimiento:

1. `git checkout --theirs <archivo>` (o `git checkout <commit-bueno> -- <archivo>`)
   para tomar una versión limpia completa.
2. Verificar que no queden marcadores:
   ```bash
   git grep -n -E '^(<<<<<<<|>>>>>>>|=======$)' -- .
   ```
   Debe devolver **cero** coincidencias.
3. `git add <archivo>` y continuar.

**Nunca hacer commit de un archivo que contenga marcadores de conflicto**, aunque
"parezca funcionar": en este repo quedó `index.html` completo con el archivo
duplicado dos veces (líneas 1–1322 en lugar de 660).

### 6.2 Historial

| Commit | Contenido |
|---|---|
| `1d03f04` | `chore: inicializar repositorio` — analysis + `fichas/` |
| `ed36201` | `chore: añadir .gitignore estándar` |
| `c395fe1` | `Add files via upload` — versión web de `index.html` (660 líneas, la buena) |
| `87b6d49` | `merge: resolver conflicto…` — **contiene `index.html` con marcadores** |
| `8f3cd79` | `fix: eliminar marcadores…` — restaura `index.html` limpio |
| `9b2b10c` | `docs: anadir seccion 0…` — estado del repo y protocolo de reanudación |
| `7e6cdc6` | `feat: exportar cartillas PNG/ZIP…` — descarga PNG (ZIP), PDF compacto y color del centro |
| `58305b8` | `docs: registrar el commit 7e6cdc6…` — estado e historial |
| `0a4ad29` | `feat: esquinas redondeadas y separacion de 2 px…` — gap de 2 px, radio de 6 px en pantalla/PDF/PNG y tip de impresión |

Referencias limpias de `index.html`: **`c395fe1:index.html`** (660 líneas),
**`7e6cdc6:index.html`** (1146 líneas) y **`0a4ad29:index.html`** (1178 líneas,
versión actual).

---

## 7. Política de documentación (obligatoria)

**Documentar todo lo que sea documentable.** En cada cambio:

1. **Actualizar `ANALISIS_PROYECTO_LOTERIA_CAMPECHANA.md`** si cambia:
   - estructura de archivos
   - funcionalidades de UI
   - lógica/mathemática de generación
   - variables de estado o firmas de funciones
   - reglas de impresión
   - flujo de Git/deploy
2. **Actualizar este `AGENTS.md`** si cambian:
   - el mapa de rangos de `index.html`
   - los IDs del DOM
   - las invariantes de dominio
   - el flujo de deploy
3. **Actualizar `ANALISIS_...md` §10** (tabla de archivos) al añadir/eliminar assets.
4. **Describir en el cuerpo del commit** qué cambió y por qué (no sólo *qué*).
5. Si un bug se resolvió de forma no obvia, **documentar la causa raíz** para
   que no se repita.

Preferir tablas y listas. Este proyecto se lee mejor estructurado que en prosa.

---

## 8. Verificación antes de cerrar un cambio

No hay tests automatizados (no hay test runner). Verificar **manualmente**:

```bash
# 1. Sin marcadores de conflicto en ningún archivo
git grep -n -E '^(<<<<<<<|>>>>>>>|=======$)' -- .

# 2. Árbol limpio / solo lo esperado
git status --short

# 3. El HTML empieza y termina bien
Get-Content index.html -TotalCount 1   # <!DOCTYPE html>
Get-Content index.html -Tail 1        # </html>

# 4. Ítem funcional, abriendo index.html en el navegador:
#    - "Al Azar" genera N cartillas
#    - "Juego Completo" 3x3 -> 10 cartillas, mazo cubierto sin repetidos
#    - "Juego Completo" 5x5 -> 4 cartillas, centros distintos
#    - Cambiar el centro de una cartilla: bloquea los ya usados (🔒),
#      hace swap universal, y "Comodín" no se bloquea
#    - "Planilla 90" muestra la grilla 9x10 y deshabilita "Descargar PNG"
#    - Cambiar el lado (cm) redimensiona y el PDF sale a escala real
#    - Color del centro: el marco cambia en todas las cartillas; "Sin color"
#      deja el centro blanco y sin marco
#    - "Descargar PNG (ZIP)":
#        5x5 con 4 cartillas -> 4 PNG de 1594x1594 px (lado 13.5 cm)
#        3x3 con 10 cartillas -> 10 PNG de 1063x1063 px (lado 9 cm)
#        El ZIP debe abrir sin errores y contener un PNG por cartilla
#        Las esquinas del PNG deben ser redondeadas (fondo blanco fuera de la curva)
#    - PDF: cuadrícula negra, gap de 2 px, esquinas redondeadas de 6 px y margen 0.5 cm
#      - 3x3 a 9 cm: 4 cartillas por hoja
#      - 5x5 a 13.5 cm: 1 cartilla por hoja
```

---

## 9. Pendientes / ideas (no implementadas)

- [ ] Tooltip o texto que explique la diferencia matemática entre "Al Azar" y
      "Juego Completo" (3x3 sin repetidos vs 5x5 con 10 repetidas).
- [ ] Deshabilitar botones de **generación** durante la regeneración masiva
      (la exportación PNG ya se bloquea con `exportandoImagenes`).
- [ ] Persistir formato/cantidad/lado/color del centro en `localStorage`.
- [ ] Accesibilidad: `aria-label`, `aria-pressed` en botones activos, `focus-visible`.
- [ ] Exportar la **Planilla 90** como PNG único (hoy el botón se deshabilita en
      ese modo).
- [ ] Evaluar compresión DEFLATE para el ZIP (los PNG ya vienen comprimidos).
- [ ] Decidir qué hacer con `fichas/cartilla-Cg6x0W1r.js.descarga` (borrar o
      mover a `docs/`).
- [ ] Posible modularización: `index.html` monolítico de **1178 líneas** →
      `styles.css` + `app.js`. Solo si el archivo sigue creciendo.
- [ ] Añadir `.gitattributes` para normalizar finales de línea (LF) y evitar
      conflictos de "todo el archivo cambió" en Windows (`core.autocrlf=true`).