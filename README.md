# Analizador de Prosa

Herramienta de análisis estilístico para prosa literaria en castellano. Diseñada para escritores que quieren retroalimentación visual sobre ritmo, tiempos verbales y legibilidad de su texto.

## Instalación

Descarga el fichero `analizador-texto.html` y ábrelo en cualquier navegador moderno (Chrome, Firefox, Safari, Edge). No requiere instalación, servidor ni conexión a internet para el análisis local.

> En versiones futuras puede que se añadan ficheros adicionales (diccionarios, módulos). Por ahora, un solo fichero es suficiente.

## Uso

### Escribir o pegar texto

Escribe directamente en el área de texto o pega tu texto desde cualquier editor. El texto se guarda automáticamente en el navegador: si recargas la página, recuperas lo que estabas trabajando.

### Vistas de análisis

La barra superior tiene tres modos. Cambia entre ellos en cualquier momento; el texto original no se altera.

#### Texto

Vista limpia, sin marcado. Modo de edición por defecto.

#### Longitud de frases

Colorea cada frase según su número de sílabas:

| Color | Rango | Lectura |
|---|---|---|
| Verde | 1–6 sílabas | Muy corta, impacto máximo |
| Amarillo | 7–15 sílabas | Corta |
| Naranja | 16–25 sílabas | Media |
| Magenta | 26–40 sílabas | Larga, exige atención |
| Rojo | 41+ sílabas | Muy larga, riesgo de pérdida |

El panel lateral muestra la distribución con una barra y los recuentos por categoría.

**Separar proposiciones:** activa el checkbox para colorear también por cláusulas (comas, puntos y coma, rayas), no solo por frases completas. Útil para ver el ritmo interno de las frases largas.

#### Verbos

Colorea cada verbo detectado según su forma. Pasa el ratón sobre cualquier verbo para ver la etiqueta de la categoría. El panel lateral lista todos los verbos encontrados agrupados por forma.

| Color | Forma verbal |
|---|---|
| Rojo | Pret. indefinido 3ª sing. (*entró*, *miró*, *dijo*) |
| Azul oscuro | Pret. indefinido resto (*canté*, *fueron*, *viniste*) |
| Naranja | Pret. imperfecto 1ª conj. (*caminaba*, *esperaban*) |
| Verde | Pret. imperfecto 2ª/3ª conj. (*tenía*, *podían*) |
| Azul | Futuro (*llegará*, *haremos*) |
| Cian | Condicional (*querría*, *podría*) |
| Púrpura | Subjuntivo (*fuera*, *hubiera*, *hiciese*) |
| Verde azulado | Participio (*dicho*, *cantado*, *abierto*) |
| Magenta | Gerundio (*corriendo*, *habiéndose*) |
| Lima | Infinitivo (*correr*, *hablar*) |
| Verde oscuro | Presente (*somos*, *tenéis*) |

> La detección es heurística (sufijos + lista de irregulares), no un analizador morfológico completo. Puede haber falsos positivos ocasionales, especialmente con sustantivos que terminan igual que formas verbales.

### Estadísticas

El panel lateral muestra siempre, independientemente de la vista activa:

- **Palabras** — total de palabras en el texto.
- **Frases** — número de oraciones (separadas por `.`, `!`, `?`, `…`).
- **Sílabas totales** — suma de sílabas de todo el texto.
- **Síl./frase (media)** — media de sílabas por frase.
- **Flesch-Szigriszt** — índice de legibilidad adaptado al español (0–100). Por encima de 75: fácil. 50–74: normal. Por debajo de 50: difícil o denso.
- **Flow** — variabilidad rítmica de las frases (0–100). Mide cuánto varía la longitud entre frases consecutivas. Un texto con frases de tamaños muy distintos puntuará alto; un texto con frases todas similares puntuará bajo.

| Rango | Nivel |
|---|---|
| 0–25 | bajo |
| 26–60 | medio |
| 61–80 | bueno |
| 81–90 | alto |
| 91–100 | óptimo |

### Ortografía y gramática

El botón **Analizar texto** envía el texto a la API de Claude para un análisis de:

- Errores de ortografía y puntuación.
- Problemas de concordancia (género, número).
- Repeticiones léxicas cercanas.

Esta función requiere conexión a internet y una API key de Anthropic.

#### Cómo configurar la API key

1. Obtén tu API key en [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys) (requiere cuenta gratuita en Anthropic).
2. Pégala en el campo que aparece en el panel lateral, bajo el título "Ortografía y gramática".
3. La key se guarda en el navegador (localStorage) y se recupera automáticamente en recargas. No se envía a ningún servidor propio: va directamente desde tu navegador a la API de Anthropic.

> Cada llamada consume tokens de tu cuenta. Un análisis típico de un párrafo cuesta menos de 0,01 USD con el modelo Sonnet.

## Limitaciones conocidas

- El detector de verbos no cubre los presentes de 3ª persona singular (*camina*, *corre*) por su alta ambigüedad morfológica.
- La sinalefa no se implementa en el conteo de sílabas (relevante para poesía, no para prosa).
- Extranjerismos y siglas pueden dar conteos de sílabas incorrectos.

## Requisitos

- Navegador moderno con soporte de ES2020+ (Chrome 85+, Firefox 79+, Safari 14+, Edge 85+).
- Conexión a internet y API key de Anthropic solo para el análisis ortográfico.
