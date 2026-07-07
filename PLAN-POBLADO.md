# Plan: poblar amirgomez.com con el contenido real (deck A+ Growth)

> Fuente: `A+Growth x Urban USA - Propuesta 2026 (2).pdf` (15 págs). Planificado 2026-07-07 (Fable),
> ejecuta Sonnet por fases. **No implementar la "caja explosión eléctrica" del final — sigue vetada.**

## 0. Decisiones tomadas (Agus puede vetar cualquiera antes de ejecutar)

1. **La marca del sitio pasa a ser "A+ Growth"** (no "AMIR GÓMEZ"). El deck se firma como A+ Growth
   con Amir como Growth Lead + equipo (Maia, Pilar). El dominio amirgomez.com no cambia. El logo es
   un wordmark tipográfico: "A+ Growth" con el **"+" en naranja** — se replica con Space Grotesk Bold
   (sin asset gráfico, es texto). Nav brand: `A+ GROWTH` (el `+` con `color:var(--brand)`).
2. **Paleta de marca sobre fondo oscuro** (NO migrar a light — el bloom/WebGL necesita oscuridad,
   y rehacer el look entero es un riesgo sin upside). Mapeo:
   - `--brand: #e8551e` — el naranja del logo (#d8551e) subido ~1 paso de luminancia para no
     apagarse sobre negro. En superficies planas chicas (nav hover, subrayados) puede usarse el
     exacto #d8551e.
   - `--fg: #f2efe9` — ya es casi idéntico al crema del deck (#f1efeb). Cero cambio.
   - `--bg: #050608` se mantiene.
   - **El cian (#4dffd0) SE ELIMINA.** No es de marca. Nueva semántica del concepto caos→orden:
     el caos es **gris cálido tenue** (partículas apagadas, ~#8a8078), la señal calibrada es
     **naranja marca encendido** — "A+ Growth convierte ruido gris en señal naranja". Esto invierte
     la rampa actual (ámbar→cian) por **carbón/crema→naranja**. Semánticamente mejor: la marca ES
     la señal, no el caos.
   - Afecta: `--amber`/`--cyan` en :root, `.hero .cy`, CTA, haze gradient, colores del shader de
     partículas (mix caos/orden), colores del hilo (thread fragmentShader), `STAGES[].color`
     (rampa nueva: #6b645c → #97887c → #c96a3a → #e8551e → #ff7a3d), matcaps de títulos, scrim igual.
3. **Idioma: inglés** (coherente con lo construido; los casos son de USA; los números hablan solos).
4. **Sección Resultados v1 en DOM** (tipografía gigante + count-up), NO ribbon 3D todavía: el
   presupuesto de frames mobile se calibró recién ayer; el ribbon es una mejora posterior si el
   presupuesto lo banca. El eje/hilo 3D sigue pasando por detrás — no queda plana.

## 1. Arquitectura final del sitio (orden = prioridad de lectura)

```
1. HERO        (existe, retocar copy + color)
2. RESULTS     (NUEVA — autoridad primero, antes de pedir 5 pantallas de scroll)
3. METHOD      (existen las 5 placas — remapear contenido al proceso real)
4. WORK/CASES  (NUEVA — 6 casos del deck sobre el mismo eje)
5. TEAM        (NUEVA — Amir protagonista + Maia + Pilar, fotos del deck)
6. SERVICES    (NUEVA — liviana, lista, sin más vidrio)
7. CONTACT     (NUEVA — CTA simple + footer. SIN caja eléctrica.)
```
Nav: `Results / Method / Work / Team / Contact` (5 links — verificar que no desborde en 375px,
ya hay media queries).

## 2. Contenido exacto por sección

### HERO (retoque)
- H1 se mantiene: "Calibrate demand / before you / **scale spend.**" (`.cy` → naranja marca).
- Sub nueva (mete la prueba temprano): *"We turn scattered paid media, funnels and analytics into
  one clean acquisition signal. 10 years, 300+ funnels shipped across the US, Europe and LatAm."*
- CTA: "Request a growth audit →" (antes "signal audit" — "growth" conecta con la marca).
- Eyebrow: "A+ GROWTH — DEMAND CALIBRATION".

### RESULTS (nueva, post-hero) — los claims más fuertes del deck, en este orden
| Número | Claim corto |
|---|---|
| **$1M+** | margin closed from $2,744 in ad spend (webinar funnel) |
| **10×** | lower cost per qualified lead ($50 → $5) |
| **7×** | ROAS after scaling UGC+AI creative (from 2×) |
| **63,000** | leads at $4.5 avg. from a single social funnel |

- Formato: 4 números gigantes (clamp ~4rem–9rem, Space Grotesk 700, color crema con el símbolo
  en naranja), caption mono debajo. Stack vertical en mobile, 2×2 o fila en desktop.
- Count-up con IntersectionObserver (respetar `REDUCE`: sin animación, valor final directo).
- El hilo/partículas 3D pasan detrás (la sección es `pointer-events:none` salvo links, como el resto).
- Línea de marcas (texto mono gris, sin logos gráficos): `ETHAN ALLEN · DIRT KING · GIANT PARTNERS ·
  NATIONWIDE TRAILERS`.

### METHOD (remapear las 5 placas existentes al proceso real del deck, pág. 7)
Títulos 3D nuevos (2 líneas cortas, caben en el clamp de 2.6 de ancho) + captions:
1. `UNDERSTAND / THE SYSTEM` — *"Deep discovery: business model, audience, funnel assets. We map
   the system before touching a dollar."*
2. `CREATIVE / AT VOLUME` — *"Strategy first, then creative volume — ads varied up to 12× more
   often than standard, always on-brand."*
3. `SHIP THE / FUNNEL` — *"Meticulous measurement plan, then fast implementation. Speed is part
   of the strategy."*
4. `MEASURE / EVERYTHING` — *"Business results, not vanity metrics. Everything measurable gets
   measured — and improved."*
5. `THE SIGNAL / COMPOUNDS` — *"Winners get budget, losers get killed. The signal tells you what
   to scale next."*
- `STAGES[].color` toma la rampa nueva (ver §0.2). El resto del sistema (frostGlass, docking,
  proximidad táctil) no se toca.

### WORK/CASES (6 casos, pág. 9–14) — el número es el protagonista
| Caso | Gancho | Descripción corta |
|---|---|---|
| Webinar + VSL | **$1M+ / $2.7K** | B2B+B2C lending funnel. 5 deals closed, campaign still live. |
| Lead Magnet B2B | **$50 → $5 CPL** | Full funnel incl. the lead magnet; ads varied 12× more often. |
| UGC + AI | **2× → 7× ROAS** | 20× more UGC volume for a US legal company. |
| LeadGen B2C | **63K leads @ $4.5** | $335K/yr social campaign for a home-decor leader. |
| PPC B2B | **3.7× revenue** | Google+Microsoft Ads, 18 months, TOFU/BOFU split, 2× ROI. |
| SEO + GEO | **5.3× organic** | +410K organic visitors for nationwidetrailers.com (2024). |
- **Formato**: extender el eje 3D hacia abajo (threadCurve + STAGES-like array segundo, o array
  `CASES` propio): placas frosted **apaisadas** (RoundedBox ~4.4×2.4×0.22) alternando X, con el
  **número gancho como TextGeometry** (números cortos = geometría barata) hijo de la placa, y la
  descripción como caption DOM dockeada debajo (mismo patrón `dockStageCopies`, generalizarlo).
- ⚠️ Cuidados: NO cajas dentro de cajas (nada de marcos internos); el costo mobile — las placas
  nuevas usan el MISMO `frostGlass` compartido y entran al mismo backdrop render (verificar FPS
  con tier low, si duele: en tier low los casos pueden ser DOM puro con el eje detrás).
- El hilo se extiende: `TUBE_BOTTOM` baja, más puntos en threadPts, partículas re-samplean la
  curva (N no sube — se estiran, la densidad baja levemente: aceptable, la señal ya está "ordenada").

### TEAM (pág. 6)
- Amir protagonista: "Amir Gómez — Growth Lead. 10 years in online marketing, 300+ successful
  B2B and B2C funnels, multinational experience leading high-performance growth teams."
- Maia Gaitan — Acquisition Lead ("5+ years running 360° digital campaigns. Meta, Google, GA4,
  SEO, web.") y Pilar Lucero — Full-Funnel Data Analysis ("Process and results analysis,
  constant improvement, growth research and tactics.").
- **Fotos**: extraerlas del PDF con PyMuPDF (`page.get_images()`, pág. 6) a `amir-site/assets/`.
  Tratamiento obligatorio (son selfies): **B/N + contraste + grano** (CSS filter grayscale+contrast
  y el grain global ya existente encima) para que se vean intencionales y consistentes.
- Formato: DOM puro (el eje 3D ya terminó arriba) — 3 columnas desktop / stack mobile. Amir con
  foto más grande. Línea de cierre del deck: *"The best idea wins. 100% collaborative."*

### SERVICES (scope genérico del deck, pág. 4 — NO el scope Urban USA que es propuesta puntual)
Lista mono con hover (sin placas — evitar fatiga de vidrio): `Full-funnel paid media / Creative
& UGC (+AI) / SEO + GEO / Funnels, webinars & VSL / Data & full-funnel analytics`.

### CONTACT / FOOTER
- Claim de cierre (pág. 15): *"Let's build something extraordinary."*
- CTA primario: mailto `amir@amirgomez.com` (mismo estilo botón naranja del hero).
- Secundario: WhatsApp `+54 9 3541 370209` (link `wa.me/5493541370209`, texto mono).
- Footer mínimo: `A+ Growth · amirgomez.com` — SIN caja eléctrica (vetada hasta que Agus la pida).

## 3. Orden de ejecución (fases commiteables, cada una con verify+push+Pages vivo)

- **F1 — Rebrand**: tokens de color (§0.2), logo/nav "A+ GROWTH", copy hero, colores de shaders
  (partículas, hilo, STAGES, matcaps, haze, CTA). Chica y de máximo impacto visual.
- **F2 — Method remap**: títulos 3D + captions nuevos (§2). Solo strings + rampa de color.
- **F3 — Results**: sección DOM nueva post-hero + count-up + línea de marcas. Reajustar
  `updateFromScroll` (el scrollHeight crece → el mapeo progress→cámara debe seguir anclando las
  placas del método; anclar por posición de sección, no por % global — revisar antes de tocar).
- **F4 — Work/Cases**: extensión del eje + 6 placas apaisadas + captions (la fase más pesada;
  verificar tier low y mobile ANTES de pulir).
- **F5 — Team + Services + Contact**: extraer fotos del PDF, DOM puro, footer.

Verificación por fase (patrón ya probado): consola sin errores → `?shot&stage=N` + medición
numérica (proyección vs DOM) → tier check 375×812 y 1440×900 → commit → push `personal` →
curl del Pages hasta ver string nuevo. El screenshot del preview no es confiable en esta escena.

## 4. Riesgos señalados

- **F3 rompe el mapeo scroll→cámara** si se hace naive (progress es % del documento entero).
  Solución: derivar `camTargetY` de `getBoundingClientRect()` de #method, no de progress global.
- **F4 y el presupuesto mobile**: +6 placas frosted + geometría de texto = presión sobre el tier
  low recién calibrado. Gate: si en tier low el frame budget no da, casos en DOM para ese tier.
- **Fotos del deck son de baja producción** — sin el tratamiento B/N+grano van a bajar la
  percepción de calidad de todo el sitio. No saltearse el tratamiento.
