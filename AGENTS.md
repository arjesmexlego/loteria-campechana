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
| Última versión buena de `index.html` | `c395fe1:index.html` (660 líneas) |
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
- Salida principal: **impresión/PDF con medidas reales en centímetros**
  (tamaño carta, margen 1 cm, escala 100%).

Todo el código, comentarios y textos de UI están en **español (MX)**. Mantener
ese idioma en cualquier código o documentación nueva.

---

## 2. Estructura del repositorio

```text
Loteria-campechana/
├── index.html            # TODO el código: CSS + HTML + JS en un solo archivo (660 líneas)
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
| 7–308 | `<style>` — variables CSS, UI, grids, estados, `@media print` |
| 310–351 | `<body>` — título, panel de control, `#contenedor-cartillas`, botón imprimir |
| 353–658 | `<script>` — toda la lógica |

Anclas del DOM usadas por JS (no renombrar sin actualizar el JS):

- `#input-lado` (number, 5–19, step 0.5)
- `#input-cantidad` (number, 1–100)
- `#btn-azar`, `#btn-completo`, `#btn-planilla`
- `#contenedor-cartillas`
- `body[data-cant]` — cantidad de cartillas objetivo según formato

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

### 3.4 Fallback de imágenes (2 niveles)

En `crearCelda()`: se intenta `.webp`; en `onerror` se cambia a `.png`; en el
segundo error se elimina el `img` y se inyecta un fallback textual
(`<div class="fallback">` con número + nombre). La lógica depende de
`this.dataset` para no entrar en bucle.

---

## 4. Reglas de impresión (crítico)

Todo el layout de cartilla usa la variable CSS `--cartilla-lado`
(`index.html:16`, sincronizada por `actualizarMedidas()` en `index.html:373`).

- `@media print` (`index.html:279`) oculta panel de control, título, subtítulo,
  acciones y cabeceras de cartilla.
- `size: letter portrait; margin: 1cm`
- `page-break-inside: avoid` / `break-inside: avoid`
- `-webkit-print-color-adjust: exact` y `print-color-adjust: exact`
  para preservar el resaltado del centro y del comodín.

**Al modificar cualquier tamaño de celda o grid, verificar que las reglas de
impresión siguen mandando** (`!important` en 297–298). Si se rompe, la cartilla
sale del papel.

---

## 5. Convenciones de código

- Vanilla JS ES6+. **Sin frameworks, bundlers, npm ni librerías externas.**
- Nombres de funciones y variables en `español` (camelCase).
- Comentarios en español.
- Render vía DOM (`createElement` / `appendChild`), nunca `innerHTML` con datos
  de usuario. El único `innerHTML` permitido es para markup estático propio
  (fallback, plantillas de etiqueta).
- Barajado siempre con `barajar()` (Fisher–Yates, `index.html:393`).
- Preferir añadir lógica nueva al final de la sección `<script>` respetando el
  orden: constantes → estado → helpers → generación → render → eventos.

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

Referencias limpias de `index.html`: **`c395fe1:index.html`**.

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
#    - "Planilla 90" muestra la grilla 9x10
#    - Cambiar el lado (cm) redimensiona y el PDF sale a escala real
```

---

## 9. Pendientes / ideas (no implementadas)

- [ ] Tooltip o texto que explique la diferencia matemática entre "Al Azar" y
      "Juego Completo" (3x3 sin repetidos vs 5x5 con 10 repetidas).
- [ ] Deshabilitar botones durante la regeneración masiva (evitar doble clic).
- [ ] Persistir formato/cantidad/lado en `localStorage`.
- [ ] Accesibilidad: `aria-label`, `aria-pressed` en botones activos, `focus-visible`.
- [ ] Exportar cartillas a PNG/JPG (requeriría `html2canvas` — primera dependencia
      externa; documentar la decisión si se acepta).
- [ ] Decidir qué hacer con `fichas/cartilla-Cg6x0W1r.js.descarga` (borrar o
      mover a `docs/`).
- [ ] Posible modularización: `index.html` monolítico de 660 líneas → `styles.css`
      + `app.js`. Solo si el archivo sigue creciendo.
- [ ] Añadir `.gitattributes` para normalizar finales de línea (LF) y evitar
      conflictos de "todo el archivo cambió" en Windows (`core.autocrlf=true`).