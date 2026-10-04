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
| **Comodín** | Opción para establecer el centro como comodín (`⭐ COMODÍN`). Se resalta visualmente con fondo amarillo (`#fffde7`) y borde dorado. |
| **Intercambio aleatorio de centro** | Opción "Centro: Auto" intercambia el centro actual con otra celda aleatoria **dentro de la misma cartilla**. |

### 4.4 Impresión y Exportación PDF

El CSS incluye reglas `@media print` estrictas para garantizar medidas exactas:

- Oculta panel de control, título, subtítulo, acciones y cabeceras.
- Usa `size: letter portrait; margin: 1cm`.
- `page-break-inside: avoid` / `break-inside: avoid` para evitar cortes entre cartillas.
- `-webkit-print-color-adjust: exact` y `print-color-adjust: exact` para preservar colores.
- Medidas basadas en `var(--cartilla-lado)` (cm). Recomendación: **Escala 100% / Predeterminado** en ventana de impresión.

---

## 5. Arquitectura y Código

### 5.1 Componentes Clave (index.html)

| Sección | Descripción |
|---|---|
| **CSS (líneas 7–308)** | Variables globales, estilos UI, grids dinámicos (5×5, 3×3, 9×10), estados (centro especial, fallback), reglas de impresión. |
| **HTML (310–351)** | Estructura UI: título, panel de control, contenedor, botón imprimir. |
| **JavaScript (353–658)** | Lógica completa: configuración, generación, render, intercambio de centros, fallback de imágenes. |

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

### 5.4 Robustez y Manejo de Errores

- **Fallback de imágenes en 2 niveles:** primer error → intenta PNG, segundo error → elimina img y crea fallback DOM.
- **Validación de inputs:** min/max en cantidad y lado (1–100, 5–19 cm).
- **Bloqueo preventivo:** evita romper invariantes (centros únicos) con alert informativo.
- **Lógica defensiva en swap:** bucles buscan coincidencias antes de asignar, evita sobrescrituras no deseadas.

---

## 6. Evaluación UI/UX

### Fortalezas

- **Interfaz limpia y directa.** Panel de control compacto, jerarquía clara.
- **Estados visuales bien definidos.** Botón activo, centro resaltado (amarillo), comodín diferenciado.
- **Feedback útil al usuario.** Nota de impresión, indicadores 🔒 para centros usados, textos descriptivos en selects (`#id. Nombre`).
- **Diseño orientado a impresión.** Pensado desde el inicio para PDF con medidas cm exactas.
- **Responsive.** Usa flexbox + wrap, se adapta a viewport.
- **Acción primaria clara.** Botón "Imprimir PDF" prominente.

### Oportunidades de Mejora (Sugeridas)

| Mejora | Prioridad | Justificación |
|---|---|---|
| **Tooltips/explicación "Juego Completo"** | Media | Explicar la diferencia matemática (3×3 sin repeticiones vs 5×5 con 10 repetidas) mejora comprensión UX. |
| **Prevenir doble click** | Media | Deshabilitar botones durante regeneración masiva para evitar renders duplicados. |
| **Persistencia de ajustes** | Baja-Media | Guardar formato, cantidad, lado en `localStorage` entre recargas. |
| **Accesibilidad (a11y)** | Media-Alta | Añadir `aria-label`, `aria-pressed` (botones activos), `focus-visible`, mejorar contraste si necesario. |
| **Validación en tiempo real** | Baja | Feedback visual al superar límites (1–100, 5–19cm). |
| **Skeleton/loader ligero** | Baja | Útil si se aumentara cantidad o se cargaran muchas imágenes. |
| **Exportar por cartilla (PNG/JPG)** | Alta (feature) | Útil para compartir digitalmente (html2canvas). Fuera del scope actual pero natural. |

---

## 7. Estado del Proyecto y Mantenibilidad

| Aspecto | Estado | Comentario |
|---|---|---|
| **Mantenibilidad** | **Bueno** | Código monolítico pero legible, bien estructurado y comentado en español. |
| **Escalabilidad** | **Limitada** | Todo en un archivo. Para crecer convendría modularizar (JS/CSS separados). |
| **Testing** | **No existe** | No hay tests unitarios/e2e. Verificable manualmente. |
| **Calidad código** | **Buena** | Nombres claros, funciones cohesivas, lógica sólida, evita side effects innecesarios. |
| **Deploy** | **Óptimo** | Estático puro. Compatible con GitHub Pages, Netlify, Vercel, StaticHost, etc. |
| **Rendimiento** | **Bueno** | `loading="lazy"`, imágenes optimizadas (.webp), renders DOM controlados, sin loops pesados innecesarios. |

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
- **UX orientada a uso real** (impresión profesional con cm exactos).
- **Código vanilla robusto** con fallbacks bien resueltos.
- **Fácil de desplegar y extender.**

**Recomendación:** Inicializar Git inmediatamente. El código está listo para continuar desarrollo (modularización, mejoras UX/a11y, exportación de imágenes por cartilla, PWA, etc.).

---

## 10. Archivos a Revisar (Detalle)

| Archivo | Estado | Acción sugerida |
|---|---|---|
| `index.html` | Principal | Mantener. Ideal candidato para dividir en `styles.css`, `app.js` en futuro si crece. |
| `fichas/*.webp` | OK | Versionar (assets). |
| `fichaspng/*.png` | OK (respaldo) | Versionar o evaluar si necesario mantener ambos. |
| `fichas/cartilla-Cg6x0W1r.js.descarga` | Sin uso | Investigar origen. Si es backup obsoleto, se puede eliminar o mover a `docs/`. |