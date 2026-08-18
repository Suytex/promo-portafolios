# Arquitectura y migración — Suytex Portafolio Experience

Referencia del patrón canónico de las landings `promo-*`. Documenta cómo está
construida esta landing y qué decisiones se tomaron al migrarla de dark a
light, para replicarlo en `trading`, `crypto` y `apex`.

Sitio: `portafolios.suytex.com` · GitHub Pages desde `main` / `(root)`.

---

## 1. Principio de diseño

**Todo el contenido vive en `assets/content.txt`. `index.html` no se toca para
cambiar texto.** Francisco edita ese archivo desde la UI de GitHub y la página
se actualiza sola. No hay build step, no hay dependencias instaladas, no hay
rutas absolutas: `.nojekyll` + `CNAME` + dos archivos.

```
/
├── .nojekyll
├── CNAME                 portafolios.suytex.com
├── index.html            shell + CSS + JS de render
├── assets/content.txt    TODO el contenido editable
└── docs/MIGRATION.md     este archivo
```

---

## 2. Pipeline de render

Todo ocurre en el cliente, en un solo `<script>` al final del `<body>`.

```
fetch('./assets/content.txt')
        │
        ▼
domReady()                    espera DOMContentLoaded si aún no pasó, para
        │                     garantizar que el sprite de icons.js ya esté
        ▼
parseFrontmatter(text)        separa el bloque --- ... --- inicial
        │                     → { meta: {brand, whatsapp}, body }
        ▼
parseSections(body)           parte por <!-- section: x --> con un split
        │                     regex → [{ type, content }, ...]
        ▼
buildSection(section)         marked.parse(content) envuelto en
        │                     <section class="section section--{type}">
        │                       <div class="container">…</div>
        │                     La sección cta recibe además id="reserva".
        ▼
#app.innerHTML = …            una sola escritura al DOM
        │
        ▼
enhanceForWho()               ┐
enhancePricing()              │ post-proceso por sección,
enhanceFAQ()                  │ cada uno con su try/catch
enhanceCTA(meta)              │
enhanceHero(meta)             ┘
        │
        ▼
enhanceIcons()                último: rellena los marcadores de ícono
        │
        ▼
footer.style.display = ''
```

### Por qué cada `enhance*` lleva su propio try/catch

El markdown lo edita una persona, no un compilador. Si alguien reordena un
`<h3>` o borra un párrafo, el post-proceso de **esa** sección falla, deja un
`console.warn`, y **el resto de la página sigue renderizando**. Sin el
try/catch un error en `enhancePricing` se llevaría el FAQ y el CTA con él.

`enhanceHero` era la única sin try/catch; se corrigió en la migración.

### Orden: `enhanceIcons()` va al final

Los otros `enhance*` **mueven nodos** (`appendChild`, `insertBefore`) y en dos
casos reescriben con `innerHTML`/`textContent`. Los marcadores de ícono
sobreviven a los movimientos de nodos, así que rellenarlos al final es lo más
seguro — y además cubre el botón que `enhanceCTA` crea desde cero.

### Contratos frágiles a respetar

Dos `enhance*` dependen del **orden y la forma** del markdown, no de clases:

- **`enhancePricing`** clasifica los `<p>` de la sección por índice:
  `0 → .price-main`, `1 → .price-subtitle`, `2 → .price-seats`,
  `3 → .price-note`, resto → `.price-closing`. Agregar un párrafo en medio
  descoloca los estilos.
- **`enhanceFAQ`** detecta una pregunta cuando un `<p>` contiene **solo** un
  `<strong>` (`p.textContent === strong.textContent`), y toma el `<p>`
  siguiente como respuesta. El patrón obligatorio es pregunta en negrita,
  respuesta en texto plano, alternando.

`enhanceForWho` es más tolerante: busca los dos primeros `<h3>` hijos directos
y reparte todo lo que va entre ellos.

---

## 3. Secciones

13 marcadores `<!-- section: x -->` + el `<footer>` (que vive fuera de `#app`)
= 14 bloques renderizados. El orden lo define `content.txt`, no el HTML.

| # | Sección | Patrón visual |
|---|---|---|
| 1 | `hero` | Pantalla completa, eyebrow + título grande + lead + botón pill |
| 2 | `pain` | Card grid — **2×2 forzado** en desktop (>1024px) |
| 3 | `problem` | Quote: `h2` atenuado + `<strong>` grande como declaración |
| 4 | `transformation` | Card grid auto-fit |
| 5 | `outcomes` | Filas con sangría francesa (ícono absoluto a la izquierda) |
| 6 | `method` | Card grid auto-fit (7 cards) + subtítulo tipo caption |
| 7 | `philosophy` | Quote, misma estructura que `problem` |
| 8 | `format` | Card grid auto-fit |
| 9 | `includes` | Filas con sangría francesa |
| 10 | `for-who` | 2 columnas sí/no construidas por JS |
| 11 | `pricing` | Card destacada (radius-lg, borde `--su-line`) |
| 12 | `faq` | Acordeón construido por JS |
| 13 | `cta` | Cierre + botón WhatsApp, `id="reserva"` |
| 14 | `footer` | Fuera de `#app`, marca + año |

**Alternancia de fondos:** `#app .section:nth-of-type(even)` recibe
`--su-bg-alt`. Es por orden de aparición, así que si se reordenan las secciones
en `content.txt` el ritmo se mantiene solo. Sin líneas divisorias.

### Card grid vs. sangría francesa

Dos recetas, y la diferencia importa:

- **Card grid** (`pain`, `transformation`, `method`, `format`): ícono en
  bloque arriba, etiqueta, cuerpo.
- **Sangría francesa** (`outcomes`, `includes`, `for-who`): ícono en
  `position: absolute` y el texto con `padding-left`.

La sangría **no usa flexbox a propósito**. En un `<li>` como
`<span icon></span> <strong>Etiqueta</strong> texto`, flex convertiría el
`<strong>` y el texto suelto en items separados y los pondría en columnas
distintas. Con el ícono absoluto el texto envuelve alineado y no se parte.

---

## 4. Consumo del design system

Tres recursos de `Suytex/suytex-design` por CDN, **siempre con tag fijo**:

```html
<html lang="es" data-theme="light">
…
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/suytex.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/theme-light.css">
…
<script src="https://cdn.jsdelivr.net/gh/Suytex/suytex-design@v1.2.0/icons.js"></script>
```

**Nunca `@main`.** Un `@main` significa que un commit en el design system
puede romper una landing en producción sin que nadie toque la landing. El tag
se sube a mano, revisando.

Reglas del acoplamiento:

- `theme-light.css` **exige `data-theme="light"` en el `<html>`**. Sin ese
  atributo no aplica nada: todos sus selectores están calificados por él.
- `suytex.css` ya hace `@import` de las fuentes (Space Grotesk + Inter). La
  landing **no** debe cargar su propio `<link>` de Google Fonts: sería una
  petición duplicada.
- El orden importa: `suytex.css` primero, `theme-light.css` después.
- El `<style>` inline de la landing va **después** de los dos `<link>`, así
  que en empates de especificidad gana la landing.

### Qué aporta el DS y qué escribe la landing

El DS da **tokens y 9 clases `.su-*`**. No da layout. La landing escribe:
contenedor, padding de sección, grids, acordeón FAQ, columnas sí/no, botones,
y estados de carga/error. Color, tipografía, radius y espaciado salen todos de
`var(--su-*)`.

**Botones:** el DS no tiene clase de botón. La receta se copió de
`preview/light.html` del propio design system (pill, primario relleno
`--su-accent`, secundario con borde `--su-line`). Añadido propio: `gap` para
el ícono, que el preview no contempla.

### Trampa de especificidad: `.su-title`

`theme-light.css` fija `font-size: clamp(2.125rem, 5vw, 3.5rem)` dentro de
`[data-theme="light"] .su-title`, con especificidad **(0,2,0)**. Aplicar
`.su-title` a los 13 `h2` de sección los dejaría **todos a tamaño hero**.

La landing declara su propia escala de sección igualando esa especificidad
(`[data-theme="light"] #app .section h2`) y reserva el tamaño grande para el
`h2` del hero. Si replicás esto en otra landing, no uses `.su-title` crudo en
los encabezados de sección.

---

## 5. Íconos

24 íconos Lucide en un sprite SVG que `icons.js` inyecta en el `<body>`.

**Marcador en `content.txt`:**

```markdown
- <span class="su-icon" data-su-icon="brain"></span> **Un método** Para decidir qué comprar.
```

`marked` pasa el HTML inline tal cual. El marcador queda legible y editable a
mano, que es el punto: `content.txt` sigue siendo el archivo que se edita sin
tocar código. Para cambiar un ícono se cambia el nombre en el atributo.

### Por qué NO se usa el auto-render declarativo

`icons.js` trae un auto-render que rellena cualquier `[data-su-icon]` al
cargar. **En esta landing no sirve**, y por una razón estructural:

> `renderAutoIcons()` corre **una sola vez**, en `DOMContentLoaded`. Esta
> landing pinta su contenido **después**, cuando resuelve el `fetch()` de
> `content.txt`. Cuando los marcadores entran al DOM, esa pasada ya ocurrió.

Verificado en navegador: un marcador presente en el HTML inicial renderiza; el
mismo marcador inyectado después de `load` queda vacío. Y `renderAutoIcons` no
está expuesto en `window`, así que no se puede volver a disparar.

**Solución:** `enhanceIcons()` recorre los marcadores y usa el helper
`window.suIcon(name)`, que sí está exportado.

```js
function enhanceIcons() {
  try {
    if (typeof window.suIcon !== 'function') return;
    document.querySelectorAll('[data-su-icon]').forEach(el => {
      const name = el.getAttribute('data-su-icon');
      if (!name || el.querySelector('svg')) return;
      el.innerHTML = window.suIcon(name);
    });
  } catch (e) { console.warn('enhanceIcons failed:', e); }
}
```

### La clase `.su-icon` es obligatoria

No es decorativa. Los atributos `fill`/`stroke` del sprite viven en el `<svg>`
raíz del sprite, y **no se heredan al shadow tree de un `<use>`** desde otro
`<svg>`. Sin `.su-icon`, el ícono hereda `fill: black` / `stroke: none` y sale
como **una mancha negra sólida**. Verificado en navegador.

`.su-icon` aporta el `fill: none` + `stroke` + 24×24 que lo hacen funcionar.
La landing solo le agrega `display: inline-flex` (en un `<span>` inline el
`width`/`height` se ignoraría) y ajustes de alineación vertical.

**Dentro de un botón relleno** el ícono debe seguir al texto, no al acento:
`[data-theme="light"] #app .btn-primary .su-icon { stroke: currentColor }`.

### Límite del set: 24 íconos

No hay bandera, globo ni idioma. El bullet "Prefieres aprender en español"
**quedó sin ícono** deliberadamente — se descartó usar `map`, que es geografía,
no idioma. La sangría francesa hace que ese bullet siga alineado con sus
hermanos, así que no se nota como hueco.

De los 24, se usan 23. Sobra `calendar`.

---

## 6. `for-who`: neutro + glifo, no semáforo verde/rojo

La versión dark usaba `border-top` verde `#22c55e` en la columna SÍ y rojo
`#ef4444` en la NO.

**`theme-light.css` no tiene tokens de éxito ni de error.** Su capa semántica
es `bg / bg-alt / surface / fg / fg-muted / fg-subtle / accent / line`. No hay
verde ni rojo a dónde mapear, y el DS es explícito en que **el azul es acento,
nunca relleno**.

Conservar el semáforo habría significado hardcodear dos colores fuera del
sistema — exactamente el problema que la migración vino a resolver. La
diferenciación se hace con recursos del propio DS:

| Columna | Glifo | Stroke |
|---|---|---|
| SÍ | `check-circle` | `--su-accent` (azul) |
| NO | `x-circle` | `--su-fg-subtle` (neutro) |

Los `border-top` de color desaparecieron. La forma del ícono carga la
semántica, no el color — que además es más accesible: no depende de percibir
la diferencia entre verde y rojo.

> El `--su-fg-subtle` del ícono NO es el mismo caso que el del footer
> (§7). Como elemento gráfico no textual su umbral WCAG es 3:1, y ahí
> `3.33:1` cumple. Por eso se dejó.

---

## 7. Historial de migración

### Punto de partida: dark con tokens ad-hoc

- ~520 líneas de CSS inline con **12 tokens propios** (`--bg: #0a0a0a`,
  `--gold: #d4a855`, `--text: #f0ede8`, …). Cero relación con el design
  system: no lo consumía en absoluto.
- 5 `radial-gradient` de glow dorado calibrados para fondo negro.
- **40 emoji** como única iconografía (39 en `content.txt` + 1 en el JS del
  botón CTA). Sin ningún asset de imagen en el repo.
- Titulares en uppercase con `letter-spacing` amplio y pesos Inter 800/900.
- Un solo breakpoint (640px).
- 17 colores hardcodeados fuera de los tokens, incluido el bloque de error
  JS con paleta dark inline.

### `b9d2658` — migración a light-first

**Se eliminó:**

- Los 12 tokens dark propios y los 17 hardcodes.
- Los 5 radial-gradient dorados.
- La paleta dark inline del bloque de error → reescrito con tokens.
- El `<link>` propio de Inter 800/900 (duplicaba lo que ya importa
  `suytex.css`).
- CSS muerto: `.section--hero h2 em` (no hay cursiva en el hero) y
  `.section--outcomes li::before` (era `content:''; display:none`).
- Los 40 emoji.

**Se ganó:**

- Consumo real del design system, con tag fijo `@v1.2.0`.
- CSS inline de ~520 a ~430 líneas, y lo que queda es **solo** layout.
- Iconografía vectorial coherente, tematizable por token.
- Breakpoint intermedio en 1024px, además del de 640px.
- `enhanceHero` con try/catch, como las otras cuatro.
- WhatsApp corregido: `809 639 9999` → `809 408 9999`.

**Dos bugs que encontró el QA en navegador, no la lectura del código:**

1. **Franja fantasma en el acordeón.** El FAQ pasó de `max-height` fija a
   `grid-template-rows: 0fr → 1fr` (auto-dimensiona, sin número mágico que
   recorte cuando el tema light agranda el cuerpo de texto). Pero el
   `padding-bottom` de un grid item **no se comprime**: cada pregunta cerrada
   arrastraba 22px muertos. El padding se movió al estado abierto.
2. **Pill de altura fija a 375px.** La etiqueta larga del CTA no cabe en
   `height: 48px`. En ≤640px el botón pasa a `height: auto` y envuelve.

### `c016cac` — copy y grid

- **Guion huérfano.** 28 bullets tenían el patrón `**Etiqueta** — texto`. Como
  `<strong>` renderiza en bloque, ese `— ` quedaba abriendo la línea de abajo
  y se leía como ruido. Se borró solo el guion que seguía al cierre de `**`;
  las 7 rayas intra-frase se conservaron.
- **`pain` a 2×2.** Con `auto-fit` sus 4 cards quedaban 3 arriba y 1 huérfana
  abajo. Ahora `repeat(2, 1fr)` por encima de 1024px; tablet y móvil mantienen
  el auto-fit.

### `6b94df3` — contraste AA en el footer

`--su-fg-subtle` (`#86868B`) no alcanza 4.5:1 como texto. Se cambiaron los
tres usos donde era `color` de texto:

| Contexto | Antes | Ahora |
|---|---|---|
| `footer span` sobre `--su-bg-alt` | 3.33:1 | **4.66:1** |
| `.loading` sobre `--su-bg` | 3.62:1 | **5.07:1** |
| `.error-state small` sobre `--su-bg` | 3.62:1 | **5.07:1** |

El `stroke` de `--su-fg-subtle` en el ícono de `for-who-no` quedó intacto: no
es texto.

> **Ojo al replicar:** el footer es el caso más justo del grupo (4.66 vs 5.07)
> porque va sobre `--su-bg-alt`, no sobre blanco. El `5.07:1` que documenta
> `theme-light.css` para `--su-fg-muted` aplica sobre `--su-bg`. Si el footer
> cambia de fondo, hay que recalcular.

---

## 8. Checklist para replicar en otra landing

1. `<html lang="es" data-theme="light">` — sin el atributo, `theme-light.css`
   no hace nada.
2. Los 3 recursos del DS con **tag fijo**, en orden, y el `<style>` después.
3. No cargar Google Fonts aparte.
4. No usar `.su-title` en los `h2` de sección: declarar escala propia
   igualando especificidad (0,2,0).
5. Íconos por `enhanceIcons()` + `window.suIcon()`, nunca por el auto-render.
   La clase `.su-icon` es obligatoria.
6. Cada `enhance*` con su try/catch.
7. `--su-fg-subtle` **no** se usa como color de texto.
8. Cero colores fuera del DS. Si falta un token semántico (éxito/error),
   diferenciar por forma, no inventar el color.

### QA mínimo, en navegador

Leer el código no alcanza: los dos bugs de la migración solo aparecieron al
renderizar. Verificar:

- Todas las secciones renderizan y el footer se muestra.
- Los marcadores de ícono están **todos** rellenos, cero `<use>` roto, cero
  emoji en el DOM.
- El acordeón abre y cierra de verdad (medir altura, no solo la clase).
- El CTA apunta al número correcto, con `target="_blank"` y
  `rel="noopener noreferrer"`.
- 375 / 768 / 1024 / 1440px sin desborde horizontal.
- Cero errores de consola.

**Nota de entorno:** las sesiones remotas suelen tener `cdn.jsdelivr.net` y
Google Fonts bloqueados por policy de red. El QA se hace interceptando esas
URLs hacia copias locales de los archivos del tag; la tipografía en las
capturas será fallback del sistema, así que el render tipográfico real se
confirma sobre el sitio publicado.
