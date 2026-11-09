# Análisis del Proyecto: Generador de Cartillas - Lotería Campechana

## 1. Descripción General

El proyecto **Lotería Campechana** es una aplicación web estática diseñada para generar cartillas imprimibles para el juego de Lotería. Permite crear cartillas en formatos 5x5 (tradicional) y 3x3 (rápido), con opciones de generación aleatoria o con reparto matemático optimizado para "Juego Completo".

**Tipo de aplicación:** Web estática (HTML + CSS + JavaScript Vanilla)  
**Idioma:** Español (MX)  
**Propósito:** Generación e impresión de cartillas con medidas exactas en centímetros para impresión/PDF profesional.

---

## 2. Estructura del Proyecto

```text
Loteria-campechana/
├── index.html                        # Aplicación completa (UI, estilos y lógica JS)
├── AGENTS.md                         # Contexto operativo + reglas para agentes de IA
├── .gitignore
├── fichas/                           # Imágenes de fichas en formato .webp (principal)
│   ├── 1.webp ... 90.webp
│   └── cartilla-Cg6x0W1r.js.descarga # Archivo .js descargado (posible backup/export)
└── fichaspng/                        # Imágenes de fichas en formato .png (respaldo)
    └── 1.png ... 90.png
```

**Notas sobre assets:**
- Mazo completo de **90 fichas** (numeradas del 1 al 90).
- Sistema de fallback: intenta cargar `.webp` → si falla `.png` → si falla, muestra fallback textual (número + nombre).
- `cartilla-Cg6x0W1r.js.descarga` no está referenciado en `index.html`. Podría ser un backup o archivo generado externamente.

---

## 3. Stack Tecnológico

| Tecnología | Uso |
|---|---|
| **HTML5** | Estructura semántica, viewport responsive, metadatos. |
| **CSS3 Vanilla** | Variables CSS, Grid Layout, `aspect-ratio`, media queries para impresión. |
| **JavaScript ES6+** | Lógica de generación, barajado Fisher–Yates, render dinámico, manipulación del DOM. |
| **Imágenes WebP/PNG** | Assets de fichas con fallback automático y `loading="lazy"`. |
| **Sin dependencias** | 100% vanilla, sin frameworks, bundlers o librerías externas. |

---

## 4. Funcionalidades Principales

### 4.1 Formatos de Cartilla

| Formato | Casillas | Lado sugerido (cm) | Uso |
|---|---|---|---|
| **5x5 Tradicional** | 25 | 13.5 cm | Formato clásico de Lotería. |
| **3x3 Rápido** | 9 | 9.0 cm | Versión reducida, ideal para partidas rápidas. |

### 4.2 Modos de Generación

| Modo | Descripción |
|---|---|
| **Aleatorio (`generar(false)`)** | Genera N cartillas independientes. Cada cartilla recibe 25 o 9 fichas seleccionadas aleatoriamente del mazo (sin reparto global). |
| **Juego Completo (`generar(true)`)** | Genera cartillas con un reparto matemático optimizado para cubrir el mazo completo, minimizando o controlando repeticiones. |

#### 4.2.1 Matemática del Modo "Juego Completo"

**Formato 3x3 (Óptimo):**
- **10 cartillas × 9 casillas = 90 casillas exactas**
- Utiliza las **90 fichas completas** del mazo, **sin repeticiones** entre todas las cartillas.
- Implementación: `mazo90` barajado → dividido en 10 bloques consecutivos (`slice(i*9, (i+1)*9)`), partición disjunta y cobertura total.

**Formato 5x5 (Controlado):**
- **4 cartillas × 25 casillas = 100 casillas** para **90 fichas** disponibles.
- Requiere **10 fichas repetidas** (cada una aparece exactamente 2 veces).
- Algoritmo:
  1. `repetidas = mazo90.slice(0,10)`, `unicas = mazo90.slice(10,90)`
  2. Cada ficha repetida se asigna aleatoriamente a **2 de las 4 cartillas**
  3. Se rellenan las casillas restantes con fichas únicas
  4. Cada cartilla se mezcla internamente (`barajar`)
  5. **Garantía de centros únicos**: asegura que la posición central (`índice 12`) sea única entre las 4 cartillas. Si hay conflicto, intercambia el centro con otra celda no-central cuyo ID aún no esté usado como centro.

### 4.3 Funcionalidades Avanzadas

| Funcionalidad | Descripción |
|---|---|
| **Planilla 90** | Renderiza una grilla 9×10 con todas las 90 fichas del mazo (vista completa de referencia). Accesible vía botón "Planilla 90". |
| **Centro independiente por cartilla** | Cada cartilla tiene su propio selector en la cabecera para modificar la ficha central. Opciones: *Auto (aleatorio)*, *⭐ Comodín*, o seleccionar cualquier ficha del mazo. |
| **Bloqueo de centros** | Previene que una misma ficha sea centro en dos cartillas distintas. Fichas ya usadas como centro aparecen **deshabilitadas con "🔒 (Usado)"**. |
| **Intercambio Universal (Swap)** | Al asignar una ficha específica X como centro de una cartilla, el sistema **busca X en cualquier cartilla y celda** y realiza un **trueque** con el centro actual. Esto preserva la distribución/matemática global del reparto. |
| **Comodín** | Opción para establecer el centro como comodín (`⭐ COMODÍN`). Conserva fondo crema (`#fff8e1`) y usa el color de marco global del centro. |
| **Intercambio aleatorio de centro** | Opción "Centro: Auto" intercambia el centro actual con otra celda aleatoria **dentro de la misma cartilla**. |
| **Color global del centro** | Control `#input-color-centro` + casilla "Sin color". Aplica el color **solo al marco** de la casilla central de todas las cartillas. "Sin color" deja el centro con fondo blanco y sin marco. |
| **Descarga de imágenes PNG** | Botón que genera un ZIP con un PNG por cartilla, a 300 DPI y sin encabezado ni selector. Ver §4.5. |

### 4.4 Impresión y Exportación PDF

El CSS incluye reglas `@media print` estrictas para garantizar medidas exactas:

- Oculta panel de control, título, subtítulo, acciones y cabeceras.
- Usa `size: letter portrait; margin: 0.5cm`.
- `#contenedor-cartillas` usa `gap: 0` y `.cartilla-box` usa `border-radius: 0`: las cartillas quedan **pegadas entre sí** (sin espacios blancos) para cortar una sola vez.
- `page-break-inside: avoid` / `break-inside: avoid` para evitar cortes entre cartillas.
- `-webkit-print-color-adjust: exact` y `print-color-adjust: exact` para preservar colores.
- Medidas basadas en `var(--cartilla-lado)` (cm). Recomendación: **Escala 100% / Predeterminado** en ventana de impresión.

**Aprovechamiento de la hoja (letter, margen 0.5 cm):**

| Formato | Lado | Cartillas por hoja | Nota |
|---|---|---|---|
| 5×5 | 13.5 cm (predeterminado) | 1 | 2×13.5 = 27 cm > 26.94 cm útiles de alto. |
| 5×5 | ≤ 10 cm | 4 (2×2) | 2×10 = 20 cm de ancho y alto. |
| 3×3 | 9 cm (predeterminado) | 4 (2×2) | 2×9 = 18 cm de ancho y alto. |

### 4.5 Descarga de imágenes (PNG/ZIP)

- Botón **"Descargar PNG (ZIP)"** (`#btn-descargar-imagenes`).
- Exporta **un PNG por cartilla**, sin cabecera ni selector de centro, con el centro y el color de marco actuales.
- Resolución fija: **300 DPI**. Tamaño en píxeles = `lado_cm / 2.54 × 300`.
  - 13.5 cm → **1594×1594 px**.
  - 9 cm → **1063×1063 px**.
- El ZIP se construye en el navegador con método **STORE** (sin compresión) y **CRC32**, sin librerías externas.
- Nombres: `cartilla-01-5x5.png`; ZIP: `cartillas-loteria-5x5-4-AAAA-MM-DD.zip`.
- Si hay más de 30 cartillas, se pide confirmación por peso/tiempo (cada PNG puede pesar 1–3 MB).
- En **Planilla 90** el botón queda deshabilitado porque no hay cartillas generadas.
- Fallback de imagen: webp → png → texto (número + nombre) dibujado en canvas.
- El botón se deshabilita durante la exportación para evitar dobles clics o exportaciones simultáneas.

---

## 5. Arquitectura y Código

### 5.1 Componentes Clave (index.html)

| Sección | Descripción |
|---|---|
| **CSS (líneas 7–365)** | Variables globales, estilos UI, grids dinámicos (5×5, 3×3, 9×10), estados (centro especial, fallback), color del centro, reglas de impresión. |
| **HTML (367–418)** | Estructura UI: título, panel de control (formato, lado, color del centro, generación), contenedor, botones de descarga e impresión. |
| **JavaScript (420–1144)** | Lógica completa: configuración, generación, render, intercambio de centros, fallback de imágenes, exportación PNG/ZIP. |

### 5.2 Variables y Estado

```js
const CARPETA_FICHAS = 'fichas';
const EXTENSION_PRINCIPAL = '.webp';
const EXTENSION_RESPALDO = '.png';

const MAZO_LOTERIA = Array.from({ length: 90 }, (_, i) => ({
  id: i + 1,
  nombre: `Ficha ${i + 1}`,
  img: `${CARPETA_FICHAS}/${i + 1}${EXTENSION_PRINCIPAL}`
}));

let cartillasGuardadas = [];  // Array de arrays con fichas por cartilla
let currentFormato = 5;       // 5 o 3
let currentCentroIdx = 12;    // 12 (5x5), 4 (3x3)
let slotsPorCartilla = 25;

// Color global del marco del centro
let colorCentroActual = '#ffa000';
let centroSinColor = false;

// Control de exportación
let exportandoImagenes = false;
const cacheImagenesExportacion = new Map();
const FACTOR_CSS_A_300DPI = 300 / 96;
```

### 5.3 Funciones Principales

| Función | Propósito |
|---|---|
| `barajar(array)` | Fisher–Yates shuffle (in-place seguro, copia array). |
| `generar(esCompleto)` | Orquesta generación (aleatoria vs juego completo). Actualiza UI y estado. |
| `generarModoJuegoCompleto(contenedor, totalCartillas)` | Implementa reparto matemático 3×3 y 5×5 con corrección de centros. |
| `renderCartilla(contenedor, listaCartas, indexCartilla)` | Renderiza una cartilla individual con su selector de centro. |
| `crearCelda(carta, esCentro)` | Crea celda con imagen + fallback robusto (webp→png→texto). |
| `cambiarCentroIndependiente(indexCartilla, nuevoVal)` | Gestiona cambio de centro: aleatorio, comodín o ficha específica con swap universal + validación de bloqueo. |
| `actualizarMedidas()` | Sincroniza CSS variable `--cartilla-lado` con input. |
| `cambiarFormato()` | Cambia entre 5×5/3×3, ajusta medidas sugeridas y regenera. |
| `actualizarColorCentro()` | Lee color/checkbox, actualiza `--centro-color`, la clase `body.sin-color-centro` y el estado del botón de descarga. |
| `obtenerColorCentro()` | Devuelve el color hex o `null` si "Sin color" está activo. |
| `actualizarBotonDescarga()` | Habilita/deshabilita `#btn-descargar-imagenes` según haya cartillas y no haya exportación en curso. |
| `cargarImagenParaExportar(ficha)` | Precarga una ficha para canvas con fallback webp → png → `null` (texto). Cachea la promesa por ID. |
| `descargarCartillasPNG()` | Orquesta exportación 300 DPI, dibuja cada cartilla, convierte a PNG, arma el ZIP y dispara la descarga. |
| `dibujarCartillaEnCanvas(...)` | Dibuja borde exterior, líneas internas, celdas, imágenes, comodín y marco del centro en el canvas. |
| `dibujarComodinEnCanvas(...)` | Dibuja el comodín (fondo crema, estrella, texto) y su marco según color/centro. |
| `dibujarMarcoInterior(...)` | Dibuja el marco interior del centro con el color elegido. |
| `dibujarImagenContenida(...)` | Escala y centra la imagen de la ficha dentro de su celda (equivalente a `object-fit: contain`). |
| `dibujarFallbackEnCanvas(...)` | Dibuja número + nombre cuando no hay imagen disponible. |
| `crearZip(archivos)` | Construye un ZIP válido (método STORE + CRC32) sin dependencias. |
| `crc32(datos)` | Calcula CRC32 con tabla precomputada. |
| `descargarBlob(blob, nombre)` | Crea un enlace temporal y dispara la descarga del ZIP. |
| `fechaParaArchivo()` | Devuelve `AAAA-MM-DD` para el nombre del ZIP. |

### 5.4 Robustez y Manejo de Errores

- **Fallback de imágenes en 2 niveles:** primer error → intenta PNG, segundo error → elimina img y crea fallback DOM.
- **Mismo fallback en exportación:** webp → png → texto (número + nombre) dibujado en canvas.
- **Validación de inputs:** min/max en cantidad y lado (1–100, 5–19 cm).
- **Bloqueo preventivo:** evita romper invariantes (centros únicos) con alert informativo.
- **Lógica defensiva en swap:** bucles buscan coincidencias antes de asignar, evita sobrescrituras no deseadas.
- **Exportación atómica:** `exportandoImagenes` evita dobles clics y exportaciones simultáneas; las cartillas se copian al inicio para que cambios de formato/centro durante el proceso no corrompan el ZIP.
- **ZIP íntegro:** CRC32 por archivo y estructura ZIP estándar (local headers + central directory + EOCD).
- **Aviso por volumen:** más de 30 cartillas pide confirmación (cada PNG a 300 DPI puede pesar 1–3 MB).
- **Manejo de error de canvas:** si el navegador no puede leer las imágenes (p. ej. `file://` con restricciones), se muestra un mensaje sugiriendo el sitio publicado o un servidor local.

---

## 6. Evaluación UI/UX

### Fortalezas

- **Interfaz limpia y directa.** Panel de control compacto, jerarquía clara.
- **Estados visuales bien definidos.** Botón activo, centro con marco de color configurable, comodín diferenciado.
- **Feedback útil al usuario.** Nota de impresión, indicadores 🔒 para centros usados, textos descriptivos en selects (`#id. Nombre`), progreso durante la exportación de imágenes.
- **Diseño orientado a impresión.** PDF con medidas cm exactas y cartillas pegadas para cortar una sola vez.
- **Doble salida.** Impresión/PDF y descarga de PNG individuales en ZIP, sin dependencias externas.
- **Responsive.** Usa flexbox + wrap, se adapta a viewport.
- **Acciones claras.** Botones "Descargar PNG (ZIP)" e "Imprimir PDF".

### Oportunidades de Mejora (Sugeridas)

| Mejora | Prioridad | Justificación |
|---|---|---|
| **Tooltips/explicación "Juego Completo"** | Media | Explicar la diferencia matemática (3×3 sin repeticiones vs 5×5 con 10 repetidas) mejora comprensión UX. |
| **Prevenir doble click en generación** | Media | La exportación ya se bloquea; falta deshabilitar los botones de generación durante renders masivos. |
| **Persistencia de ajustes** | Baja-Media | Guardar formato, cantidad, lado y color del centro en `localStorage` entre recargas. |
| **Accesibilidad (a11y)** | Media-Alta | Añadir `aria-label`, `aria-pressed` (botones activos), `focus-visible`, mejorar contraste si necesario. |
| **Validación en tiempo real** | Baja | Feedback visual al superar límites (1–100, 5–19cm). |
| **Skeleton/loader ligero** | Baja | Útil si se aumentara cantidad o se cargaran muchas imágenes. |
| **Exportar Planilla 90 como imagen** | Media | Hoy el botón de descarga se deshabilita en modo planilla; se podría exportar la grilla 9×10 como PNG único. |
| **Compresión ZIP (DEFLATE)** | Baja | El ZIP actual usa STORE; los PNG ya vienen comprimidos, pero un ZIP con deflate podría reducir algo el peso. |

---

## 7. Estado del Proyecto y Mantenibilidad

| Aspecto | Estado | Comentario |
|---|---|---|
| **Mantenibilidad** | **Bueno** | Código monolítico pero legible, bien estructurado y comentado en español. |
| **Escalabilidad** | **Limitada** | Todo en un archivo. Para crecer convendría modularizar (JS/CSS separados). |
| **Testing** | **No existe** | No hay tests unitarios/e2e en el repo. Verificación manual documentada en `AGENTS.md` §8; la exportación PNG/ZIP se validó con Chrome headless + CDP. |
| **Calidad código** | **Buena** | Nombres claros, funciones cohesivas, lógica sólida, evita side effects innecesarios. |
| **Deploy** | **Óptimo** | Estático puro. Compatible con GitHub Pages, Netlify, Vercel, StaticHost, etc. |
| **Rendimiento** | **Bueno** | `loading="lazy"`, imágenes optimizadas (.webp), renders DOM controlados. La exportación a 300 DPI es pesada por diseño (1–3 MB por PNG); se advierte al superar 30 cartillas. |

---

## 8. Recomendación: Git y Control de Versiones

**Sí, es muy recomendable pasar este proyecto a Git.**

### Razones

1. **Evolución controlada.** Tiene lógica no trivial (reparto matemático, swap universal, bloqueo de centros). Conviene versionar cambios.
2. **Seguridad de código.** Fácil revertir experimentos (nuevos formatos, ajustes UI/UX).
3. **Colaboración futura.** Facilita trabajo en equipo.
4. **Historial claro.** Útil para documentar decisiones (matemática 5×5/3×3).
5. **Despliegue moderno.** Git + plataformas estáticas (Pages/Netlify) fluido.

### .gitignore Sugerido

Crear `.gitignore` en la raíz:

```gitignore
# Sistema operativo
.DS_Store
Thumbs.db
desktop.ini

# Archivos temporales
*.log
*.tmp
*.swp
*~

# Exportaciones generadas (no versionar PDFs/ZIPs)
*.pdf
*.zip

# Entornos/builds
node_modules/
dist/
build/
.cache/
.vite/
.parcel-cache/

# Editores/IDE
.vscode/
.idea/
*.sublime-*
```

**Importante:** **NO** ignorar `fichas/` ni `fichaspng/` (son assets del proyecto y deben versionarse).

### Sugerencia de primer commit

Mensaje sugerido:
```text
chore: inicializar repositorio Lotería Campechana

- Generador de cartillas 5x5 y 3x3
- Modos aleatorio y juego completo (reparto matemático)
- Planilla 90, centro independiente con bloqueo y swap universal
- Soporte webp/png con fallback
- Reglas de impresión PDF optimizadas (medidas cm)
```

---

## 9. Conclusión

**El proyecto está en buen estado, funcional y bien pensado.** Destacan especialmente:

- **Solución matemática sólida** para el reparto "Juego Completo" (partición perfecta 3×3, control inteligente de repeticiones y centros únicos en 5×5).
- **UX orientada a uso real** (impresión profesional con cm exactos, cartillas pegadas para cortar una sola vez y exportación PNG/ZIP).
- **Código vanilla robusto** con fallbacks bien resueltos y ZIP propio sin dependencias.
- **Fácil de desplegar y extender.**

**Recomendación:** El proyecto está listo para continuar desarrollo (modularización, mejoras UX/a11y, exportación de Planilla 90, PWA, etc.).

---

## 10. Archivos a Revisar (Detalle)

| Archivo | Estado | Acción sugerida |
|---|---|---|
| `index.html` | Principal | Mantener. Versión actual: **1146 líneas** (CSS 7–365, HTML 367–418, JS 420–1144). Referencia histórica limpia: `c395fe1:index.html` (660 líneas). Candidato a dividir en `styles.css` + `app.js` si sigue creciendo. |
| `AGENTS.md` | Contexto operativo | Mantener y actualizar junto con cada cambio (ver §11). |
| `fichas/*.webp` | OK | Versionar (assets). |
| `fichaspng/*.png` | OK (respaldo) | Versionar o evaluar si necesario mantener ambos. |
| `fichas/cartilla-Cg6x0W1r.js.descarga` | Sin uso | Investigar origen. Si es backup obsoleto, se puede eliminar o mover a `docs/`. |

---

## 11. Historial de Incidentes y Reglas de Git

### 11.1 Incidente: `index.html` commiteado con marcadores de conflicto

**Qué pasó.** El commit `87b6d49` ("merge: resolver conflicto con archivos
subidos desde web") se creó **sin resolver realmente** el conflicto. El blob
resultante de `index.html` conserva los marcadores:

| Línea | Contenido |
|---|---|
| 1 | `<<<<<<< HEAD` |
| 661 | `=======` |
| 1321 | `>>>>>>> origin/main` |

Como el bloque de conflicto contenía el archivo completo, `index.html` quedó
con **1322 líneas** (el documento duplicado dos veces) en lugar de 660. El
archivo estaba roto **tanto en el repo como en producción** (GitHub Pages
servía `main`).

**Causa raíz.** El proyecto se edita también desde la interfaz web de GitHub
("Add files via upload"). Eso genera commits en `origin/main` que el clon local
no tiene; al hacer `git pull` Git no puede fusionar un archivo grande no
trackeado y deja el conflicto sin resolver. El merge se selló con `git commit`
sin editar el archivo.

**Resolución aplicada.** Ambos lados del conflicto eran **idénticos byte a byte**
(659 líneas cada uno), por lo que la resolución fue trivial: quedarse con una
sola copia.

```bash
git checkout c395fe1 -- index.html
```

**Resultado:** `index.html` = 660 líneas, sin marcadores, idéntico a la versión
subida desde la web.

**Prevención (ver `AGENTS.md` §6.1 y §8).**

1. Siempre `git fetch origin && git pull` **antes** de editar.
2. Nunca hacer commit de un archivo con marcadores de conflicto.
3. Verificar con
   `git grep -n -E '^(<<<<<<<|>>>>>>>|=======$)' -- .` → debe dar **cero**.
4. Añadir `.gitattributes` con `* text=auto eol=lf` para normalizar finales de
   línea (pendiente; `core.autocrlf=true` en Windows causa diffs de
   "todo el archivo cambió").