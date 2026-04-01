# Memoria de decisiones — Analizador de Prosa

> Documento vivo. Registra el *por qué* de cada decisión de diseño y arquitectura,
> los problemas encontrados y cómo se resolvieron, y las alternativas descartadas.
> Actualizar con cada iteración significativa.

---

## Versión actual: v2 (30 marzo 2026)

---

## 1. Propósito del proyecto

Herramienta de análisis estilístico para prosa literaria en castellano. El objetivo principal es dar al escritor retroalimentación visual sobre su propio texto en cuatro dimensiones:

1. **Variedad rítmica** — ¿hay frases de distintas longitudes o el texto es monótono?
2. **Tiempos verbales** — ¿qué formas dominan? ¿hay acumulación de pretéritos indefinidos que suenan igual?
3. **Ortografía y gramática** — errores básicos.
4. **Legibilidad** — índice Flesch-Szigriszt adaptado al español.

El proyecto nació de una necesidad concreta del autor (Álvaro Korta) trabajando en una novela. No es una herramienta académica ni un corrector genérico: está orientada a la escritura literaria, donde el estilo importa tanto como la corrección.

---

## 2. Arquitectura general

### 2.1 Fichero único HTML

**Decisión:** todo en un único `.html` — CSS, HTML y JavaScript en el mismo fichero.

**Razón:** máxima portabilidad. El usuario puede guardar el fichero y abrirlo en cualquier navegador sin servidor, sin npm, sin dependencias externas. Es una herramienta de escritorio disfrazada de página web.

**Alternativa descartada:** React + bundler. Innecesariamente complejo para un uso individual. Añade fricción de instalación sin aportar nada relevante en este caso.

### 2.2 Análisis local vs. API

**Decisión:** el análisis morfológico (sílabas, frases, verbos) es completamente local en JavaScript. Solo el análisis ortográfico/gramatical usa la API de Claude.

**Razón:** el análisis local es instantáneo (responde al escribir), funciona sin conexión y no tiene coste por llamada. El análisis ortográfico requiere comprensión contextual real que un sistema de sufijos no puede dar — ahí sí justifica la llamada a la API.

**Consecuencia:** el detector de verbos es heurístico, no perfecto. Esto es aceptable porque su función es orientativa (¿hay demasiados indefinidos acumulados?), no exhaustiva.

### 2.3 Estado global mínimo

**Decisión:** dos variables globales: `currentView` (string) y `AD` (objeto de análisis).

**Razón:** la herramienta no tiene sesiones, no persiste datos, no tiene usuario. El estado es trivial. Una arquitectura más elaborada (Redux, stores reactivos) sería sobreingeniería.

---

## 3. El bug crítico de v1: regex sobre HTML

### Problema

En v1, el renderizado de verbos funcionaba así:
1. Se escapaba el texto a HTML.
2. Se aplicaban regex sobre esa cadena HTML para envolver verbos en `<span>`.
3. Resultado: los regex encontraban coincidencias dentro del propio HTML generado (en atributos `data-label="Participio"`, en clases `vt-presente`, etc.) y los envolvían en nuevos `<span`, produciendo markup recursivo e inválido.

**Ejemplo del output roto:**
```
<span class="verb-mark vt-<span class="verb-mark vt-presente"...">
```

### Solución (v2)

Arquitectura completamente distinta para el renderizado:

1. `tokenize(text)` divide el texto plano en tokens `{text, isWord}`.
2. `classifyToken(token)` analiza cada token de forma independiente.
3. `renderView()` construye el HTML en una sola pasada lineal, concatenando strings de `esc(tok.text)` o `<span>...</span>`.

**Regla invariante:** los regex nunca tocan una cadena que ya contiene HTML. El HTML se genera una sola vez, al final, a partir de decisiones ya tomadas sobre texto plano.

### Por qué esto es importante para el futuro

Si se añaden nuevas vistas o anotaciones, deben seguir este mismo patrón: **analizar primero, marcar después, nunca sobre el markup**.

---

## 4. Detector de verbos

### 4.1 Estrategia: sufijos + lista blanca de irregulares + lista negra de stopwords

El español es una lengua de morfología rica. Un detector de verbos por sufijos tiene dos problemas opuestos:

- **Falsos negativos:** los verbos irregulares no terminan en los sufijos esperados (`fue`, `hice`, `puso`…).
- **Falsos positivos:** muchos sustantivos y adjetivos terminan igual que formas verbales (`espalda` → `-aba`, `bastón` → `-ón` no da problemas pero `vida` → `-ida` sí, `rato` → `-ato` no, `mirada` → `-ada` sí).

**Solución adoptada:**
1. **`IRR`** — lista blanca explícita de irregulares frecuentes, clasificados por tipo verbal. Primera comprobación, máxima prioridad.
2. **`VT` con sufijos ordenados por especificidad** — sufijos más largos primero dentro de cada tipo, tipos más específicos antes que los genéricos. El orden importa porque el primer match gana.
3. **`SW`** — lista negra de palabras que nunca son verbos en este contexto: artículos, pronombres, preposiciones, conjunciones, y una lista manual de sustantivos frecuentes en prosa literaria española que coinciden morfológicamente con formas verbales.
4. **Filtro de stem:** el stem (raíz tras quitar el sufijo) debe tener ≥2 caracteres y contener al menos una vocal. Esto elimina falsos positivos donde el sufijo es casi toda la palabra.
5. **Longitud mínima por tipo** (`MIN_SYL_LEN`): por ejemplo, el presente solo se detecta con tokens de ≥4 caracteres para evitar que palabras cortas frecuentes se marquen.

### 4.2 Decisión deliberada: el presente tiene cobertura reducida

Los sufijos de presente de indicativo (`-o`, `-as`, `-a`, `-an`, `-en`, `-es`) son tan ambiguos que habilitarlos generaría un ruido inaceptable. La decisión es detectar **solo las formas plurales inequívocas** (`-amos`, `-emos`, `-imos`, `-áis`, `-éis`, `-ís`) más los irregulares de la lista blanca.

El coste es que muchos presentes de 3ª persona singular no se marcan (`camina`, `corre`, `sabe`). Se acepta como compensación necesaria para evitar que la herramienta sea inútil por exceso de ruido.

**Alternativa futura:** usar un diccionario morfológico completo (FreeLing, Spacy con modelo español). Implicaría abandonar el modelo de fichero único o hacer llamadas a una API especializada.

### 4.3 Desdoblamiento del pretérito indefinido en 3ª singular y resto

**Decisión:** el tipo `indefinido` se divide en dos:
- `indefinido3s` — 3ª persona del singular: *cantó*, *comió*, *fue*, *hizo*…
- `indefinido` — el resto de personas: *canté*, *cantaste*, *cantaron*, *fuiste*, *fueron*…

**Razón:** la 3ª persona del singular del indefinido es la forma narrativa dominante en la prosa literaria española. Cuando se acumula en párrafos cortos produce una cadencia monótona muy reconocible ("entró, miró, cogió, salió"). Al tener su propio color, el escritor puede ver de un vistazo si hay concentración de esa forma concreta, independientemente del resto de personas del mismo tiempo.

**Implementación:**
- En `VT`, `indefinido3s` aparece **antes** que `indefinido` (primer match gana). Su único sufijo es `'-ó'`.
- En `IRR`, los irregulares de 3ª singular (*fue*, *hizo*, *vino*, *entró*…) pasan a `IRR.indefinido3s`. Los irregulares de otras personas (*fui*, *fuiste*, *fueron*…) quedan en `IRR.indefinido`.
- `MIN_SYL_LEN.indefinido3s = 2` (el más bajo posible); el filtro de stem (`stem.length >= 2`) ya previene falsos positivos con palabras de dos letras.

**Colores:** misma familia roja para mantener la relación visual, pero distinguibles:
- `indefinido3s`: rojo puro (`.vt-indefinido3s`)
- `indefinido`: rojo-naranja / azul (`.vt-indefinido`)

### 4.4 Estilos de tipos verbales: de inline a clases CSS (v4)

Hasta v3, cada entrada de `VT` tenía dos propiedades de color (`color`, `bg`) que se aplicaban como estilos inline en cada `<span>` generado. En v4 se migró a clases CSS.

**Motivación:**
- Los estilos inline mezclaban datos (JS) con presentación (CSS).
- Cambiar un color requería tocar el array `VT` y regenerar el HTML.
- No había forma limpia de tener variantes contextuales (panel vs. texto) sin lógica JS adicional (la función `bgSolid()` fue un parche temporal).

**Solución adoptada:**
1. Cada tipo verbal tiene ahora una propiedad `cls` (p.ej. `'vt-indefinido3s'`) en lugar de `color` y `bg`.
2. En CSS, cada clase `.vt-xxx` define:
   - `--vtc`: custom property con el color del tipo (usado para texto y borde inferior).
   - `background`: fondo sólido del tipo.
   - `color: var(--vtc)`: color de texto en chips del panel.
3. La variante `.rendered-text .vm` añade `border-bottom: 1.5px solid var(--vtc)` una sola vez para todos los tipos.
4. Las variantes `.rendered-text .vt-xxx` pueden sobreescribir el fondo si el panel y el texto necesitan valores distintos.

**Consecuencia:** cambiar colores es ahora puramente CSS. El array `VT` en JavaScript no tiene ninguna propiedad de presentación.

### 4.5 Orden de prioridad entre tipos

El orden en `VT` no es arbitrario. Casos conflictivos:

- `-era` puede ser subjuntivo (`tuviera`) o sustantivo (`primera`, `carrera`). Está en la lista negra de SW.
- `-ado/-ada` puede ser participio (`cantado`) o sustantivo (`estado`, `mirada`). Los más frecuentes van a SW.
- `-ando` puede ser gerundio (`cantando`) o nombre propio / sustantivo (`Fernando`, `mando`). El filtro de stem ayuda (`mand-` tiene vocal, `fernanánd-` también — esto es un límite del sistema).
- `-ía` puede ser imperfecto (`tenía`) o sustantivo (`energía`, `alegría`). Los sufijos acentuados son más fiables que los sin acento porque los sustantivos rara vez llevan esa acentuación en esa posición.

### 4.6 Stopwords: política de expansión y orden

La lista `SW` tiene dos tipos de entradas:

1. **Palabras gramaticales** (artículos, pronombres, preposiciones…) — lista cerrada y estable.
2. **Sustantivos/adjetivos con sufijos verbales** — lista abierta, debe crecer con el uso.
3. **Nombres propios** — personajes frecuentes en el texto de trabajo.

Cuando el analizador marque un sustantivo incorrecto, la solución es añadirlo a `SW`. Esta es la forma esperada de evolución del detector.

**Orden (v4):** dentro de cada categoría comentada (`// false positives`, `// proper names`), las entradas se mantienen en orden alfabético. Esto facilita buscar duplicados y localizar palabras al revisar la lista.

---

## 5. Contador de sílabas

### 5.1 Algoritmo

El contador implementa las reglas ortográficas del español para diptongos, hiatos y triptongos:

- **Diptongo:** vocal fuerte + débil (o débil + fuerte) sin tilde en la débil → misma sílaba.
- **Hiato:** dos vocales fuertes contiguas, o débil acentuada + fuerte, o dos vocales iguales → sílabas separadas.
- **Triptongo:** débil + fuerte + débil → misma sílaba.

La precedencia es: triptongo > hiato > diptongo. El hiato se detecta antes que el diptongo porque es la excepción que rompe la regla general.

### 5.2 Limitaciones conocidas

- No implementa **sinalefa** (fusión de sílabas entre palabras). La sinalefa es relevante para poesía, menos para prosa donde el objetivo es medir la "densidad" de la frase, no su escansión métrica exacta.
- Los **números**, **siglas** y **extranjerismos** se tratan como secuencias de letras españolas. Esto puede dar conteos incorrectos para palabras como "streaming" o "club".
- La **tilde diacrítica** no afecta al conteo silábico (el algoritmo ya la trata igual que la vocal acentuada por ser parte del hiato).

### 5.3 Referencia

Las reglas siguen la normativa de la RAE (Ortografía de la lengua española, 2010, cap. 4) y son consistentes con la implementación de referencia de Fernández Huerta para el índice Szigriszt-Pazos.

---

## 6. Clusters de longitud de frase

### 6.1 Decisión sobre los rangos

Los rangos elegidos (1–3 / 4–8 / 9–15 / 16–25 / 26+) se basan en:

- La adaptación al español del índice de Flesch por Fernández Huerta (1959) y su desarrollo por Szigriszt-Pazos (1992), que toman como referencia la media de sílabas por frase.
- Estudios de legibilidad en prosa narrativa española que identifican la frase de 9–15 sílabas como la "zona de confort" para lectura fluida.
- La intuición del propio escritor: una frase de 3 sílabas tiene un impacto diferente a una de 25, y esa diferencia debe ser visible de un vistazo.

**Colores asignados:**
- Verde (1–3): máximo impacto, máxima brevedad. El verde frío contrasta sin connotar "peligro".
- Lima/amarillo (4–8): corta, ágil.
- Naranja (9–15): zona media — el naranja llama la atención pero sin alarmar.
- Magenta (16–25): larga, empieza a ser llamativa.
- Rojo (26+): muy larga, zona de riesgo de pérdida del lector.

**Decisión de paleta (v3):** los colores originales (rojo→naranja→verde→azul→morado) tenían opacidades muy bajas (0.14–0.16) que hacían los fondos casi imperceptibles, especialmente en pantallas brillantes. Se subió la opacidad (0.26–0.38) y se usaron colores más saturados, fuera de la paleta general de la UI, para que el marcado sea útil de un vistazo. La progresión ahora va de verde/frío (frases cortas) a rojo/cálido (frases largas), invirtiendo la convención de semáforo anterior pero ganando en visibilidad.

### 6.2 Toggle de proposiciones

**Decisión:** añadir una opción para dividir también por proposiciones (comas, puntos y coma, dos puntos, rayas).

**Razón:** una frase larga con varias subordinadas tiene pausas internas. El escritor puede querer ver no solo la longitud de la frase completa sino la de cada cláusula, que corresponde mejor a las unidades de respiración al leer en voz alta.

**Implementación:** `splitPropositions()` divide en los signos de puntuación mencionados, preservando el delimitador al final del segmento izquierdo (para no perder el contexto visual al leer el texto coloreado).

---

## 7. Índice Flesch-Szigriszt

### 7.1 Fórmula utilizada

```
Szigriszt = 206.84 - 0.60 × (sílabas/palabra × 10) - 1.02 × (frases/palabra × 100)
```

Esta es la adaptación al español de Szigriszt-Pazos (1992), también llamada "Fórmula de perspicuidad". Es la versión más utilizada en lingüística computacional española. La fórmula original de Flesch en inglés usa coeficientes distintos y no es directamente aplicable al español por la diferencia en longitud media de palabras.

**Escala orientativa:**
- ≥75: fácil (prensa popular, literatura juvenil)
- 50–74: normal (prensa generalista, novela comercial)
- <50: difícil (ensayo, literatura densa, texto técnico)

### 7.2 Limitaciones

El índice es un promedio global del texto. No distingue un texto con frases muy largas y muy cortas alternadas (ritmo rico) de un texto uniformemente largo (ritmo plano) si la media es la misma. Por eso el gráfico de distribución es más informativo que el índice solo.

---

## 8. Análisis ortográfico vía API

### 8.1 Decisión: JSON estructurado, no texto libre

El prompt instruye a Claude a devolver un JSON con estructura fija, no una lista de texto libre. Esto permite:
- Renderizar cada error con formato visual diferenciado (tipo, fragmento, sugerencia, nota).
- Potencialmente en el futuro resaltar el fragmento en el texto.

**Riesgo:** el modelo puede no seguir el formato. Hay un bloque `try/catch` para el `JSON.parse()` y se hace `.replace(/```json|```/g,'')` antes de parsear para neutralizar el caso más frecuente de incumplimiento (respuesta envuelta en bloque de código markdown).

### 8.2 Categorías de error solicitadas

- `ortografía` — errores de escritura.
- `gramática` — errores de concordancia, régimen, etc.
- `puntuación` — comas incorrectas, ausentes o excesivas.
- `concordancia` — género/número.
- `repetición` — la misma palabra o raíz muy cerca (problema de estilo, no de corrección).

### 8.3 Modelo utilizado

`claude-sonnet-4-20250514`. Equilibrio entre calidad y coste. Para textos cortos (párrafos de novela, máximo 500 palabras en uso típico) el coste por llamada es negligible.

---

## 9. Tokenizador

### 9.1 Regex de tokenización

```javascript
/([a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+(?:[-'][a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+)*)|([^a-záéíóúüñA-ZÁÉÍÓÚÜÑ]+)/g
```

Dos grupos alternos:
1. Una palabra española (incluyendo guiones y apóstrofes internos como en "d'Alambert").
2. Todo lo demás (espacios, puntuación, números, símbolos).

El resultado es una lista plana de tokens que cubre **todo** el texto: `words.join('') + punctuation.join('') === originalText`. Esto garantiza que al reconstruir el HTML no se pierda ni un carácter.

### 9.2 Por qué esto evita el bug de v1

En v1 se usaba `text.replace(regex, match => '<span>'+match+'</span>')` directamente sobre el HTML ya generado. El tokenizador rompe esa dependencia: la clasificación ocurre sobre el texto plano original, y el HTML se construye de cero a partir de las clasificaciones.

---

## 10. Decisiones de UI/UX

### 10.1 Dos pestañas: Editar / Analizado

El texto se escribe en un `<textarea>` limpio. El texto analizado se muestra en un `<div>` de solo lectura. La separación es necesaria porque el marcado de colores (`innerHTML`) no es compatible con la edición directa.

**Alternativa considerada:** edición directa con `contenteditable`. Descartada porque el análisis continuo al escribir requeriría reinterpretar el DOM como texto en cada keystroke, con riesgo de bugs similares al de v1 (el DOM mezclado con markup de análisis).

### 10.2 Sidebar siempre visible

Las estadísticas están siempre visibles independientemente de la pestaña activa. Los paneles de frases/verbos solo aparecen cuando la vista correspondiente está activa, para no saturar la interfaz.

### 10.3 Tipografía: Georgia para el texto, sans-serif para la interfaz

Georgia en el editor y el texto analizado da un aspecto de "máquina de escribir literaria" coherente con el uso. La interfaz (etiquetas, botones, estadísticas) usa sans-serif para máxima legibilidad a tamaños pequeños.

### 10.5 Persistencia del texto con localStorage

**Decisión:** el contenido del textarea se guarda en `localStorage` en cada keystroke (`oninput`). Al cargar la página, si hay texto guardado se restaura; si no, se muestra el texto de `SAMPLE`.

**Razón:** la herramienta se usa sobre textos en elaboración. Recargar la página (por accidente o para ver cambios de CSS/JS) no debe suponer perder el texto de trabajo.

**Alternativa descartada:** no persistir nada y confiar en que el usuario no recargue. Inaceptable para uso real.

**Alternativa descartada:** guardar con debounce (solo cada N ms). Añade complejidad innecesaria; `localStorage.setItem` en cada keystroke es negligible en rendimiento para textos de prosa.

**Clave usada:** `'analizador-texto'`. Una sola entrada, el texto completo como string plano.

### 10.4 Dark mode automático

Se usa `@media (prefers-color-scheme: dark)` con variables CSS. Sin JavaScript, sin toggle manual. La preferencia del sistema operativo se respeta automáticamente.

---

## 11. Problemas conocidos y trabajo futuro

### Problemas conocidos

| Problema | Causa | Mitigación actual |
|---|---|---|
| Sustantivos marcados como verbos | Ambigüedad morfológica de sufijos | Lista SW, filtro de stem |
| Presente de 3ª singular no detectado | Sufijos `-a`, `-e`, `-o` demasiado ambiguos | IRR para irregulares frecuentes |
| Sinalefa no implementada | Complejidad, no relevante para prosa | Documentado como limitación |
| Extranjerismos con conteo incorrecto | El algoritmo asume grafía española | Sin mitigación, raro en literatura |

### Líneas de trabajo futuro (por prioridad)

1. **Resaltado de repeticiones léxicas** — marcar en el texto las palabras repetidas dentro de una ventana de N palabras, con intensidad de color según proximidad.
2. **Detector de adverbios en -mente** — los adverbios terminados en -mente son una señal de estilo que muchos manuales recomiendan vigilar.
3. **Estadísticas por párrafo** — actualmente el análisis es sobre todo el texto. Un desglose por párrafo permitiría ver si hay zonas de ritmo plano.
4. **Exportar marcado** — guardar el HTML del texto analizado como fichero.
5. **Diccionario morfológico externo** — integrar FreeLing o similar vía API para mejorar la detección de verbos. Requiere abandonar el modelo sin-servidor.
6. **Patrones anafóricos** — detectar repetición de la misma estructura sintáctica (anáforas, enumeraciones).

---

## Historial de versiones

| Versión | Fecha | Cambios principales |
|---|---|---|
| v1 | 30 mar 2026 | Primera versión funcional. Bug de markup en verbos. Falsos positivos masivos en sustantivos. |
| v2 | 30 mar 2026 | Reescritura completa del renderizado. Arquitectura token-by-token. Expansión de SW y IRR. Toggle de proposiciones. |
| v3 | 31 mar 2026 | Colores más saturados y visibles para frases y verbos. Desdoblamiento de `indefinido` en `indefinido3s` (3ª sing.) y `indefinido` (resto). |
| v4 | 1 abr 2026 | Migración de estilos verbales de inline a clases CSS (`vt-xxx`) con custom property `--vtc`. Eliminación de `bgSolid()`. Stopwords de `SW` ordenadas alfabéticamente por categoría. |
| v5 | 1 abr 2026 | Persistencia del texto en `localStorage`. Al recargar se restaura el último texto editado; si no hay nada guardado, se usa `SAMPLE`. |
