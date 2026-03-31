# Documentación técnica — Analizador de Prosa v2

> Referencia exhaustiva del código. Describe cada función, estructura de datos,
> variable global y bloque CSS relevante.
> Para el *por qué* de las decisiones, ver `DECISIONES.md`.

---

## Estructura del fichero

```
analizador-texto.html
├── <style>          CSS (variables, layout, componentes, clases de análisis)
├── <body>
│   ├── <header>     Título y subtítulo
│   └── .layout      Grid principal (editor + sidebar)
│       ├── .editor-area
│       │   ├── .toolbar   Botones de vista + toggle proposiciones
│       │   ├── .tabs      Editar / Analizado
│       │   ├── #pane-input    <textarea>
│       │   └── #pane-rendered <div> con HTML anotado
│       └── .sidebar
│           ├── Estadísticas
│           ├── Distribución longitud (solo en vista "frases")
│           ├── Verbos por forma (solo en vista "verbos")
│           └── Ortografía y gramática (botón + resultados API)
└── <script>
    ├── Estado global
    ├── sylCount() / segSyl() / sylClass()
    ├── splitSentences() / splitPropositions()
    ├── VT / IRR / SW / MIN_SYL_LEN
    ├── classifyToken()
    ├── tokenize()
    ├── analyze()
    ├── esc()
    ├── updateStats() / updateFrPanel() / updateVbPanel()
    ├── renderView()
    ├── setView() / switchTab() / onTextChange()
    ├── runOrtografia()
    └── Inicialización
```

---

## CSS — variables y temas

### Variables de color (`:root`)

```css
--bg        /* fondo principal (crema claro en light, casi negro en dark) */
--bg2       /* fondo secundario (header, toolbar, tabs) */
--ink       /* texto principal */
--ink2      /* texto secundario (labels) */
--ink3      /* texto terciario (placeholders, metadatos) */
--accent    /* color de acento principal (terracota) */
--accent2   /* acento hover (naranja más vivo) */
--border    /* borde sutil (12% opacidad) */
--border2   /* borde más visible (22% opacidad) */
--panel     /* fondo del sidebar (ligeramente más claro que --bg) */
```

El dark mode se activa mediante `@media (prefers-color-scheme: dark)` que sobreescribe las mismas variables. No hay JavaScript involucrado.

### Clases de longitud de frase

Aplicadas como `<span class="fl-N">` en la vista "frases":

| Clase | Rango | Color |
|-------|-------|-------|
| `.fl-1` | 1–3 sílabas | Verde `rgba(0,200,100,0.32)` |
| `.fl-2` | 4–8 sílabas | Lima/amarillo `rgba(200,210,0,0.38)` |
| `.fl-3` | 9–15 sílabas | Naranja `rgba(255,120,0,0.30)` |
| `.fl-4` | 16–25 sílabas | Magenta `rgba(255,0,140,0.26)` |
| `.fl-5` | 26+ sílabas | Rojo `rgba(200,0,30,0.28)` |

### Clases de marcado verbal

Los verbos se marcan con `<span class="vm">` más un estilo inline. La clase `.vm` proporciona `position:relative` para el tooltip. El tooltip `.tt` (hijo directo) se muestra con `.vm:hover .tt { display:block }`.

El color y fondo de cada tipo verbal se aplica como estilo inline (no como clase CSS), permitiendo que los colores estén centralizados en el array `VT` en JavaScript.

---

## Estado global

```javascript
let currentView = 'plain';
// Valores posibles: 'plain' | 'frases' | 'verbos'
// Controla qué renderiza renderView() y qué paneles del sidebar son visibles.

let AD = null;
// Objeto de análisis. null si el textarea está vacío.
// Se reasigna en cada llamada a analyze().
// Estructura: ver sección "Objeto AD" más abajo.
```

---

## Funciones de sílabas

### `sylCount(word) → number`

Cuenta las sílabas de una sola palabra.

**Parámetros:**
- `word` — string. Puede contener mayúsculas, signos de puntuación pegados; se normaliza internamente.

**Proceso:**
1. Normalización: `toLowerCase()` + eliminar todo lo que no sea `[a-záéíóúüñ]`.
2. Si el resultado está vacío, devuelve `0`.
3. Recorre el string carácter a carácter con índice `i`:
   - Si el carácter actual es vocal:
     - **Triptongo:** comprueba si `word[i..i+2]` está en `TRIPH`. Si sí, `count++`, `i+=3`.
     - **Dos vocales seguidas:** comprueba si es hiato o diptongo:
       - **Hiato:** dos vocales fuertes (`aeoáéó`) contiguas, o dos iguales, o la segunda es vocal débil acentuada (`íú`), o la primera es vocal débil acentuada. En hiato: `count++`, `i++` (avanza solo una vocal).
       - **Diptongo:** el par está en `DIPH`. `count++`, `i+=2`.
       - **Caso residual** (dos vocales que no forman diptongo ni hiato según las listas): `count++`, `i++`.
     - **Vocal sola:** `count++`, `i++`.
   - Si el carácter es consonante: `i++` (sin contar).
4. Devuelve `count`, con mínimo de 1 (toda palabra tiene al menos una sílaba).

**Constantes internas:**
```javascript
V        = 'aeiouáéíóúü'           // todas las vocales
STRONG   = 'aeoáéó'                // vocales fuertes
ACC_WEAK = 'íú'                    // débiles con tilde (forman hiato)
DIPH     = Set de 22 pares         // diptongos reconocidos
TRIPH    = Set de 6 triples        // triptongos reconocidos
```

---

### `segSyl(seg) → number`

Suma las sílabas de todas las palabras de un segmento de texto.

```javascript
segSyl(seg) {
  return seg.trim().split(/\s+/).filter(Boolean)
            .reduce((s,w) => s + sylCount(w), 0);
}
```

**Uso:** se llama sobre frases completas o proposiciones.

---

### `sylClass(n) → 1|2|3|4|5`

Devuelve el cluster de longitud dado un número de sílabas.

```javascript
n <= 3  → 1   (muy corta)
n <= 8  → 2   (corta)
n <= 15 → 3   (media)
n <= 25 → 4   (larga)
else    → 5   (muy larga)
```

---

## Funciones de segmentación

### `splitSentences(text) → string[]`

Divide el texto en frases usando como separadores `.`, `!`, `?`, `…` seguidos de espacio.

```javascript
text.split(/(?<=[.!?…]+)\s+/).map(s => s.trim()).filter(Boolean)
```

Usa lookbehind positivo para que el delimitador quede incluido en la frase a la izquierda, no perdido.

**Limitación:** no distingue puntos de abreviatura (`Sr.`, `p. ej.`). En prosa literaria esto es un caso marginal.

---

### `splitPropositions(sent) → string[]`

Divide una frase en proposiciones separando por `,`, `;`, `:`, `—` (`\u2014`), `–` (`\u2013`).

**Proceso:**
1. `sent.split(/([,;:\u2014\u2013])/)` — split capturando el delimitador como grupo.
2. Recorre los fragmentos acumulando en `cur`. Cuando encuentra un delimitador, lo añade a `cur` y vuelca `cur` en `segs` (si no está vacío tras eliminar delimitadores y espacios).
3. El fragmento final (sin delimitador) también se vuelca.

**Resultado:** el delimitador queda pegado al segmento izquierdo. Ejemplo:
```
"Arrastraba los pies, sin apartar los ojos"
→ ["Arrastraba los pies,", " sin apartar los ojos"]
```

Si no hay delimitadores internos, devuelve `[sent]`.

---

## Detector de verbos

### `VT` — Array de tipos verbales

Array de 10 objetos, ordenados de más específico a más genérico. El orden importa porque `classifyToken` usa el primer match.

```javascript
{
  key:   string,   // identificador interno ('indefinido', 'imperfecto1', etc.)
  label: string,   // etiqueta legible para el UI
  color: string,   // color CSS para borde y texto de chip
  bg:    string,   // color CSS para fondo (con transparencia)
  suf:   string[]  // sufijos, ordenados de más largo a más corto dentro del tipo
}
```

**Orden de los tipos (de más a menos específico):**

1. `indefinido3s` — 3ª persona del singular exclusivamente. Sufijo único `-ó`. Captura antes que `indefinido` para que las formas en `-ó` reciban su categoría propia. Irregulares: *fue*, *hizo*, *vino*, *entró*, *detuvo*…
2. `indefinido` — resto de personas del pretérito indefinido: `-é`, `-aste`, `-iste`, `-asteis`, `-aron`, `-ieron`, `-imos`, `-isteis`. Irregulares: *fui*, *fuiste*, *fueron*, *hice*, *vine*…
3. `imperfecto1` — sufijos con `-ab-` en la terminación, ordenados del más largo al más corto para evitar que `-aba` tape a `-ábamos`.
4. `imperfecto23` — sufijos con `-í-` acentuada: `-íamos`, `-íais`, `-ían`, `-ías`, `-ía`. La tilde evita la mayoría de falsos positivos.
5. `futuro` — sufijos con vocal temática + desinencia de futuro (siempre llevan tilde): `-aré`, `-erá`, etc.
6. `condicional` — sufijos `-aría`, `-ería`, `-iría` y sus variantes.
7. `subjuntivo` — sufijos de imperfecto de subjuntivo (`-ara`, `-era`, `-ase`, `-ese`) y sus variantes. Ambiguos con sustantivos; el filtro de stem ayuda.
8. `participio` — sufijos `-ado/-ada/-ados/-adas`, `-ido/-ida/-idos/-idas`, más irregulares frecuentes (`-uerto`, `-ierto`, `-cho`, etc.).
9. `gerundio` — sufijos `-ando`, `-iendo`, `-yendo` y sus variantes reflexivas.
10. `infinitivo` — sufijos `-ar`, `-er`, `-ir` y sus variantes reflexivas. Los más cortos, se aplican con mínimo de longitud más alto.
11. `presente` — solo formas plurales inequívocas: `-amos`, `-emos`, `-imos`, `-áis`, `-éis`, `-ís`. Los singulares son demasiado ambiguos.

---

### `IRR` — Irregulares explícitos

```javascript
{
  indefinido3s: Set<string>,  // fue, hizo, vio, entró, detuvo... (3ª singular)
  indefinido:   Set<string>,  // fui, fuiste, fueron, hice, viniste... (resto)
  presente:     Set<string>,  // es, son, hay, ha, van, tiene, hace...
  imperfecto1:  Set<string>,  // estaba, llevaba, arrastraba...
  imperfecto23: Set<string>,  // era, tenía, había, podía...
}
```

Comprobados **antes** que los sufijos en `classifyToken`. Un token que esté aquí nunca llega a la comprobación de sufijos.

**Política de mantenimiento:** cuando un verbo irregular frecuente no se detecta, añadirlo aquí. Es preferible ampliar `IRR` que relajar los criterios de sufijo.

---

### `SW` — Stopwords (lista negra)

`Set<string>` con todas las palabras que **nunca** deben clasificarse como verbos. Incluye:

1. Palabras gramaticales: artículos, pronombres, preposiciones, conjunciones, adverbios comunes.
2. Sustantivos y adjetivos que comparten sufijo con formas verbales frecuentes.
3. Nombres propios frecuentes en el contexto de uso (personajes de la novela de Álvaro).

Todos los tokens se normalizan a minúsculas antes de comparar con `SW`.

**Política de mantenimiento:** cuando el analizador marque un sustantivo/adjetivo como verbo, añadirlo a `SW`. Es la forma esperada y sostenible de mejorar la precisión sin alterar la arquitectura.

---

### `MIN_SYL_LEN` — Longitud mínima por tipo

```javascript
{
  indefinido3s: 2,  // sufijo '-ó' es muy corto; el filtro de stem hace el resto
  indefinido: 3,    // 'fui' tiene 3 chars, es el más corto válido
  imperfecto1: 4,
  imperfecto23: 3,
  futuro: 5,
  condicional: 6,
  subjuntivo: 4,
  participio: 3,
  gerundio: 5,
  infinitivo: 3,    // 'dar', 'ver', 'ir' son válidos
  presente: 4
}
```

En `classifyToken`, si `t.length < MIN_SYL_LEN[vt.key] + 1`, se salta ese tipo completamente.

---

### `classifyToken(raw) → string|null`

Clasifica un token de texto como tipo verbal o devuelve `null`.

**Parámetros:**
- `raw` — string. Token tal como salió del tokenizador (con posibles mayúsculas).

**Proceso:**
1. Normaliza: `raw.toLowerCase().replace(/[^a-záéíóúüñ]/g,'')`.
2. Si longitud < 2: `null`.
3. Si está en `SW`: `null`.
4. Comprueba `IRR`: recorre las entradas de `Object.entries(IRR)` y devuelve la clave si el token está en algún `Set`.
5. Comprueba sufijos en orden de `VT`:
   - Para cada tipo `vt`:
     - Si `t.length < MIN_SYL_LEN[vt.key] + 1`: continúa al siguiente tipo.
     - Para cada sufijo `suf` de `vt.suf`:
       - Si `t.endsWith(suf)` y `t.length > suf.length + 1`:
         - Calcula `stem = t.slice(0, t.length - suf.length)`.
         - Si `stem.length >= 2` y `stem` contiene vocal: devuelve `vt.key`.
6. Si ningún tipo coincide: `null`.

**Complejidad:** O(|VT| × max(|suf|)) por token. Con los tamaños actuales (~10 tipos × ~10 sufijos máximo) es negligible.

---

### `tokenize(text) → {text: string, isWord: boolean}[]`

Divide el texto en tokens alternos: palabras y no-palabras.

**Regex:**
```javascript
/([a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+(?:[-'][a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+)*)|([^a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+)/g
```

- Grupo 1: palabra española. Incluye guiones y apóstrofes internos para manejar contracciones y palabras compuestas.
- Grupo 2: todo lo demás (espacios, puntuación, números, emojis, etc.).

**Garantía:** la concatenación de todos los tokens (en orden) reconstituye el texto original exacto. Esto es crítico para que el HTML renderizado no pierda ni gane caracteres.

---

## Función principal de análisis

### `analyze() → object|null`

Ejecuta todo el análisis local sobre el texto del textarea.

**Devuelve `null`** si el textarea está vacío.

**Devuelve el objeto `AD`:**

```javascript
{
  sents:  string[],         // frases del texto (splitSentences)
  syls:   number[],         // sílabas por frase (paralelo a sents)
  dist:   number[5],        // distribución: [#fl1, #fl2, #fl3, #fl4, #fl5]
  totSyl: number,           // total de sílabas del texto
  avg:    string,           // media de sílabas por frase (toFixed(1))
  nw:     number,           // número de palabras
  flesch: number,           // índice Flesch-Szigriszt (0–100)
  vbt:    {                 // verbos por tipo
    indefinido3s: string[],
    indefinido:   string[],
    imperfecto1:  string[],
    imperfecto23: string[],
    futuro:       string[],
    condicional:  string[],
    subjuntivo:   string[],
    participio:   string[],
    gerundio:     string[],
    infinitivo:   string[],
    presente:     string[]
  }
}
```

**Fórmula Flesch-Szigriszt:**
```
flesch = 206.84 - 0.60 × (totSyl/nw × 10) - 1.02 × (sents.length/nw × 100)
```
Acotado a [0, 100] con `Math.max(0, Math.min(100, ...))`.

---

## Función auxiliar

### `esc(s) → string`

Escapa HTML básico para inserción segura en `innerHTML`.

```javascript
s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;')
```

**Uso obligatorio** en todo valor que provenga del texto del usuario antes de insertarlo en HTML. El incumplimiento de esta regla es el vector del bug de v1.

---

## Funciones de actualización del sidebar

### `updateStats()`

Lee `AD` y actualiza los 5 elementos con id `st-w`, `st-s`, `st-sy`, `st-av`, `st-fl`. Añade etiqueta de dificultad al Flesch (`fácil`/`normal`/`difícil`).

---

### `updateFrPanel()`

Actualiza el panel de distribución de longitud (solo visible en vista "frases"):
- Barra sparkbar: 5 columnas con altura proporcional a `dist[i] / max(dist)`, altura mínima 2px.
- Leyenda: 5 items con dot de color y contador.

Constantes de presentación:
```javascript
F_COLORS = ['#00b850','#99a800','#ff6a00','#e6007a','#c0001e']
F_BGS    = ['rgba(0,200,100,0.32)','rgba(200,210,0,0.38)','rgba(255,120,0,0.30)','rgba(255,0,140,0.26)','rgba(200,0,30,0.28)']
F_LABELS = ['1–3 síl','4–8 síl','9–15 síl','16–25 síl','26+ síl']
```

---

### `updateVbPanel()`

Actualiza el panel de verbos (solo visible en vista "verbos"):
- Por cada tipo en `VT`, si `AD.vbt[vt.key]` tiene elementos:
  - Título con color del tipo.
  - Chips con los verbos detectados (deduplicados con `Set`).
- Si no hay ningún verbo detectado: mensaje "No se detectaron verbos".

---

## Función de renderizado

### `renderView()`

Genera el HTML del panel "Analizado" según `currentView`.

**Vista `plain`:** `el.textContent = text`. No hay innerHTML, no hay riesgo.

**Vista `frases`:**
1. Lee `useProp = chk-seg.checked`.
2. Para cada frase en `AD.sents`:
   - Si `useProp`: divide en proposiciones con `splitPropositions`.
   - Para cada segmento: calcula sílabas, obtiene clase, genera `<span class="fl-N" title="N sílabas">`.
3. El resultado se asigna a `el.innerHTML`.

**Vista `verbos`:**
1. Tokeniza el texto original con `tokenize()`.
2. Para cada token:
   - Si no es palabra: `html += esc(tok.text)`.
   - Si es palabra pero sin clasificación: `html += esc(tok.text)`.
   - Si es palabra con clasificación: genera `<span class="vm" style="..."><span class="tt">LABEL</span>PALABRA</span>`.
3. El resultado se asigna a `el.innerHTML`.

**Invariante crítico:** `esc()` se aplica siempre a `tok.text` antes de insertarlo. Nunca se hace `.replace()` sobre `html` una vez iniciada la construcción.

---

## Control de vista y pestañas

### `setView(v)`

Cambia la vista activa.

1. Actualiza `currentView`.
2. Actualiza la clase `.on` de los tres botones de vista.
3. Muestra/oculta `sec-fr`, `sec-vb`, `lbl-seg`.
4. Si `v !== 'plain'`: recalcula `AD`, actualiza stats, actualiza el panel correspondiente, renderiza, cambia a pestaña "Analizado".
5. Si `v === 'plain'`: cambia a pestaña "Editar".

---

### `switchTab(tab)`

Cambia entre las pestañas "Editar" y "Analizado".

- Actualiza la clase `.active` de los `<div class="tab">`.
- Muestra/oculta `pane-input` y `pane-rendered`.
- Si se cambia a "Analizado": recalcula `AD`, actualiza stats y renderiza (para reflejar cambios del textarea).

---

### `onTextChange()`

Handler del evento `oninput` del textarea.

1. Recalcula `AD`.
2. Actualiza stats.
3. Si `currentView !== 'plain'`: renderiza (actualización en tiempo real).

---

## Análisis ortográfico

### `runOrtografia()` (async)

**Flujo:**
1. Lee el texto del textarea. Si vacío, sale.
2. Desactiva el botón y muestra spinner.
3. Hace `fetch` a `https://api.anthropic.com/v1/messages` con:
   - `model: 'claude-sonnet-4-20250514'`
   - `max_tokens: 1000`
   - System prompt que instruye al modelo a devolver JSON puro (sin markdown).
   - Un mensaje de usuario con el texto completo.
4. Recibe la respuesta, extrae el texto de `data.content` (puede haber varios bloques).
5. Limpia posibles backticks con `.replace(/```json|```/g,'')`.
6. `JSON.parse()` en `try/catch` — si falla, muestra mensaje de error.
7. Si `parsed.errores.length === 0`: muestra el resumen en verde.
8. Si hay errores: renderiza una lista de `div.error-item` con tipo, fragmento, sugerencia y nota.
9. En el `finally`: rehabilita el botón.

**Estructura del JSON esperado:**
```json
{
  "errores": [
    {
      "tipo": "ortografía|gramática|puntuación|concordancia|repetición",
      "fragmento": "texto original tal como aparece",
      "sugerencia": "cómo debería quedar",
      "nota": "breve explicación lingüística"
    }
  ],
  "resumen": "valoración general del texto en una frase"
}
```

---

## Inicialización

```javascript
AD = analyze();
updateStats();
```

Al cargar la página, el textarea tiene el texto de `SAMPLE` (hardcodeado). Se analiza inmediatamente para que las estadísticas estén disponibles desde el primer momento, sin necesidad de que el usuario pulse nada.

---

## Dependencias externas

Ninguna. El fichero funciona completamente offline excepto para:
- La llamada a la API de Claude (`runOrtografia`), que requiere red y que el entorno tenga las credenciales de la API disponibles (el cliente de Claude.ai las inyecta automáticamente).

---

## Glosario rápido

| Símbolo/nombre | Descripción |
|---|---|
| `AD` | Objeto de análisis actual (Analysis Data) |
| `VT` | Array de definiciones de tipos verbales (Verb Types) |
| `IRR` | Mapa de verbos irregulares explícitos (Irregulars) |
| `SW` | Set de palabras excluidas del análisis verbal (StopWords) |
| `MIN_SYL_LEN` | Longitud mínima de token por tipo verbal |
| `F_COLORS` | Colores de los 5 clusters de longitud de frase |
| `fl-N` | Clase CSS para el N-ésimo cluster de longitud |
| `vm` | Clase CSS base para marcas verbales (Verb Mark) |
| `tt` | Clase CSS para el tooltip de una marca verbal (ToolTip) |
| `esc()` | Función de escape HTML (imprescindible antes de innerHTML) |
| `sylCount()` | Contador de sílabas por palabra |
| `segSyl()` | Suma de sílabas de un segmento |
| `sylClass()` | Cluster (1–5) dado un número de sílabas |
