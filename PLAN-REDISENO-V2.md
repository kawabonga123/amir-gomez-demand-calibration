# Plan Rediseño V2 — de "efectos sueltos" a UNA pieza que cuenta una historia

> Auditoría `/taste` (Taste Gate Innovatron) corrida 2026-07-08 por Fable con **evidencia visual real**
> (Chrome real vía claude-in-chrome — primera vez que el sitio se VE, no solo se mide; el preview
> headless nunca pudo capturar esta escena). Comparado contra activetheory.net en vivo.
> **Ejecuta Sonnet, por fases, verificando CADA fase con screenshot real de Chrome** (método al final).

---

## 1. VEREDICTO TASTE GATE

**⛔ COMPUERTA FALLADA: Accesibilidad/contraste** — el texto 3D (títulos Method, hooks Work) se ve
verdín-grisáceo apagado sobre placas marrón chocolate. Ilegible en varias tomas. FAIL automático.

**Puntaje ponderado: ~52/100** → *"todavía es un sitio lindo con efectos — no sale"*.
Lo que más resta: Sello (4), Restricción curatorial (4), A11y (4), Momentos descuidados (2).
Lo que más suma: Concepto (7 — caos→señal ES fuerte y sobrevive sin efectos), la columna de luz (lo
más hermoso del sitio, nivel real).

### Bugs P0 vistos en pantalla (ninguno detectado por las mediciones numéricas previas)
1. **Captions de la nube sangran sobre Method**: "margin closed from just $2,744…" aparece DENTRO
   de la placa "UNDERSTAND & CREATE", y "lower cost per qualified lead…" dentro de "SHIP & MEASURE".
   El lector ve descripciones de casos pegadas a pasos del método → la historia se rompe en el minuto 1.
   *Causa*: opacidad por proximidad (`1-dist/5`) sin gate por sección — CLOUD_Y=-4 queda a dist<5 de
   STAGES[0] (y=-8). Los tres sistemas de docking (cloud/stage/case) pueden estar visibles A LA VEZ.
2. **Team y Services (DOM) chocan con las placas 3D de Work**: las fotos del equipo se dibujan sobre
   las placas 7X/63K, y la lista de servicios sobre 3.7X/5.3X. Dos mundos superpuestos sin jerarquía.
   *Causa*: el canvas es fixed y el journey 3D "termina" recién al fondo de #work, pero el DOM de Team
   ya subió — no hay handoff diseñado entre el mundo 3D y el DOM.
3. **La nube asoma en el Hero** ($1M+/10X + cristales se ven abajo del hero antes de tiempo, tapados
   por el hilo) — spoiler feo de la sección siguiente, y nunca hay UN momento donde los 4 números se
   lean compuestos y claros.
4. **El hilo pasa POR DELANTE del contenido**: la columna de partículas cruza sobre los títulos
   ("UNDERSTAND & CREATE", "THE SIGNAL COMPOUNDS" quedan parcialmente tapados). El eje compite contra
   la lectura en vez de sostenerla.

### Los 5 cortes (sin diplomacia)
1. **¿EL gesto?** La columna de luz que ordena el caos. Existe, es bueno — pero el contenido no lo habita: las placas son pizarras pegadas al costado del gesto.
2. **¿Sin WebGL sigue siendo bueno?** El copy sí; las placas no — sin vidrio serían rectángulos con 2 palabras. Falta layout interno.
3. **¿Qué sobra?** (a) cristales-confeti de la nube, (b) labels sticky huérfanos ("PROOF, NOT PROMISES" flotando solo en una esquina — el "texto flotante sin diseño" que señaló Agus), (c) pantallas de transición vacías (solo hilo), (d) bevel noventoso del texto 3D.
4. **¿Delata IA/template?** Sí: helvetiker con bevel, confeti icosaédrico random, títulos gigantes centrados sin jerarquía interna.
5. **¿Distingue del parangón?** No es copia — pero está 3 ligas abajo en composición. AT nunca deja un frame sin componer.

### Qué hace el parangón que nosotros no (visto en vivo hoy)
- **Cada frame es un cuadro**: un solo objeto luminoso protagonista, compuesto contra el vacío. El vacío de AT es *composición* (el objeto respira); el nuestro es *ausencia* (labels perdidos + hilo).
- **UN material rector luminoso e iridiscente** (vidrio refractivo con dispersión cromática) que hace de CUALQUIER cosa una joya. Nuestro vidrio quedó **marrón chocolate mate** — el tinte naranja permanente + matcap cálido + fondo oscuro lo ensucian. No se percibe refracción.
- **El eje nunca tapa el contenido**: la cinta/columna pasa por detrás o se aparta cuando hay lectura; acá el hilo cruza por delante de los títulos.
- **Arco de color**: AT arranca casi negro y revela color (rosa/oro) como clímax. Nosotros mostramos todo el naranja desde el frame 1 — no hay progresión emocional.
- **Poses de cámara diseñadas por sección** (lookAt/fov/lerp por vista, data-driven) vs nuestro único lerp lineal — de ahí nuestros "viajes vacíos".

---

## 2. LA HISTORIA (nueva columna vertebral narrativa — 6 actos, ~6.5 pantallas)

El sitio debe leerse así, acto por acto, sin que falte ni sobre nada:

| # | Acto | Mensaje que el visitante entiende | Pantallas |
|---|---|---|---|
| 1 | **HERO** | "Calibran demanda antes de escalar gasto — 10 años, 300+ funnels" | 1.0 |
| 2 | **PROOF** | "Y lo prueban: $1M+ / 10× / 7× / 63K — números con contexto legible" | 1.2 |
| 3 | **METHOD** | "Así lo hacen: 3 pasos claros, cada uno explicado dentro de su vidrio" | 2.0 |
| 4 | **WORK** | "Casos reales por tema, número + qué hicieron + para quién" | 1.8 |
| 5 | **TEAM + SERVICES** | "Quiénes son y qué venden — el mundo 3D ya cerró limpio" | 0.8 |
| 6 | **CONTACT** | "Escribile a Amir" | 0.7 |

Meta dura: **≤ 6.5 alturas de viewport en desktop (~5.900px en 900px)** — hoy hay ~8.4 (7530px).
Cómo se gana: sin pantallas puente vacías (la cámara viaja DURANTE el contenido, no entre medio),
Method 60svh/paso, Work 70svh/par, y el tramo Team→Contact compacto.

---

## 3. CAMBIOS ESTRUCTURALES (en orden de ejecución)

### F1 — Arreglar los 4 bugs P0 (sin rediseño todavía)
1. **Gate de docking por sección**: cada sistema de captions (cloud/stage/case) solo es visible si SU
   sección está activa. Derivar `activeSection` del scroll (rangos de #signal/#method/#work) y
   multiplicar la opacidad de cada grupo por su gate (con fade corto). Nunca dos sistemas a la vez.
2. **Handoff 3D→DOM**: al salir de #work (progress>0.98), TODO el mundo 3D (placas, hilo, partículas,
   nube) hace fade-out global (uniform `uWorldFade` en materiales, u opacity por grupo) en ~400ms de
   scroll. Team/Services/Contact viven sobre fondo limpio con las partículas al 10% como textura
   ambiental. El eje TERMINA con intención: el hilo converge en un punto de luz que se apaga.
3. **La nube no asoma en el Hero**: CLOUD_Y más abajo (-6.5) + su visibilidad gateada por sección
   (punto 1) + fade-in al entrar a #signal.
4. **El hilo pasa por DETRÁS del contenido**: en los tramos de curva que atraviesan placas/nube,
   z=-1.2 (detrás del vidrio — se refracta a través: EXACTAMENTE el efecto frost que queremos
   mostrar y hoy no se ve); z=-0.4 solo en tramos de transición. Además `bigZ=0` (las partículas
   delanteras) mientras una placa esté activa en frustum.

### F2 — El material: de chocolate a vidrio de verdad
Objetivo visual: vidrio ahumado CLARO y luminoso, columna refractándose visible detrás (referencia:
anillo de la intro de AT — sin copiar su iridiscencia exacta).
- `frostGlass`: bajar el tinte permanente `uNeon*0.1` → `uNeon*0.04` (el naranja permanente es lo que
  da el marrón); subir el peso del backdrop (`mix(behind, matcap, 0.28)` → behind más dominante 0.8);
  matcap base más CLARO y neutro (crema/gris perla, no pardo); `opacity` 0.6→0.5.
- Rim fresnel: más presente en reposo (0.22→0.35) — el borde del vidrio debe dibujar la silueta SIEMPRE,
  no solo al hover (en AT la silueta del vidrio es lo que compone el frame).
- Chequeo de aprobación (screenshot real): en la toma de una placa quieta se debe VER la columna de
  luz doblada/borrosa a través del vidrio. Si no se ve, no pasa.

### F3 — Tipografía 3D digna (asset prediseñado, no helvetiker)
- Generar **typeface JSON de Space Grotesk Bold** (facetype.js, gratis) → misma voz tipográfica que
  el DOM en TODO el sitio. Guardar en `assets/space-grotesk-bold.typeface.json` (local, no CDN).
- Sin bevel noventoso: `bevelThickness:0.008, bevelSize:0.006` (solo anti-alias del borde) o bevel off.
- **Color del texto 3D: crema luminoso** (`#f2efe9`, emissive sutil) — el cian en matcap lee verdín
  con ACES tonemapping (visto en pantalla). El cian queda SOLO como acento DOM (CTA/hover/case-name).
  Contraste crema-sobre-vidrio-oscuro: garantizado y elegante.

### F4 — Jerarquía interna de placa (el pedido central de Agus)
Cada placa (Method y Work) es una "diapositiva de vidrio" con layout interno completo — nada flota:
```
┌──────────────────────────────┐
│  01 / DISCOVERY & CREATIVE   ← etiqueta mono chica, naranja (DOM dockeada o GL)
│  Understand                  ← título 3D, ~55% del tamaño actual
│  & Create                    │
│  ──────                      ← hairline
│  Deep discovery on the      ← cuerpo DOM legible (1rem+), crema, DENTRO
│  business model… (2-3 líneas)│   del rect proyectado (sistema ya existe)
└──────────────────────────────┘
```
- Título 3D baja a `size:0.26` (Method) / hook Work a `0.4` — el número/título deja de ser decoración
  gigante y pasa a ser jerarquía. El cuerpo sube a protagonista de lectura.
- La etiqueta numerada (01/02/03) da el "dónde estoy" que hoy no existe.
- Los labels sticky (`.pair-theme`, `.cloud-eyebrow`) SE ELIMINAN — su contenido entra a la etiqueta
  interna de cada placa. Cero texto huérfano flotando en esquinas.

### F5 — PROOF rediseñada: los números CONTENIDOS en un objeto con sentido
Pedido: "si algo está en el aire, debe estar contenido en una forma 3D" + "objetos prediseñados o de
calidad, no formas geométricas sin sentido". Los cristales-confeti mueren. Reemplazo — **elegir Agus**
(checkpoint Innovatron de assets, decidir ANTES de que Sonnet ejecute F5):

- **Opción A (recomendada) — "Diales de calibración"**: cada número vive dentro de un ANILLO-DIAL de
  vidrio con marcas de graduación instanciadas (ticks finos, como el f-stop de una lente / dial de
  instrumento). Semántica perfecta (calibrar = dial), mismo material vidrio del sitio, procedural de
  alta factura (anillo torus fino + 60-90 ticks InstancedMesh + números 3D dentro) — barato y cohesivo.
  Los 4 diales compuestos en constelación asimétrica (no grid 2×2), cada uno con su caption en el
  layout interno del dial (etiqueta + número + 1 línea).
- **Opción B — Lente/prisma glTF CC0**: un objeto real descargado (lente de cámara o prisma, de
  poly.pizza/Sketchfab CC0), la "señal" lo atraviesa y se descompone en los 4 números. Más wow, más
  riesgo (calidad del asset, pipeline gltf-transform, licencia a verificar) y una pantalla más lenta.
- **Opción C — Placas Proof**: los 4 números en mini-placas del MISMO vidrio del sitio (2×2 asimétrico),
  cero objetos nuevos. La más segura y rápida, la menos memorable.

### F6 — Cámara con poses (del parangón) + acortamiento final
- Reemplazar el lerp lineal único por **POSES por sección** `{y, z, lookY, xTilt}` interpoladas con
  easing entre rangos de scroll (mismo patrón data-driven de AT, versión simple). La cámara llega,
  ENCUADRA la composición diseñada de esa sección, y acelera en los tramos entre poses (los viajes
  vacíos desaparecen porque duran poco scroll).
- Method 60svh/paso · Work 70svh/par · sin márgenes muertos entre secciones. Verificar meta ≤6.5 vh.
- Arco de color sutil (de AT): el Hero arranca con la columna más fría/tenue y el naranja va ganando
  saturación hacia PROOF/WORK — progresión emocional en vez de todo-naranja-siempre.

### F7 — Momentos descuidados (hoy: 2/10)
- Preloader mínimo diseñado (la columna "cargando" como línea de luz que crece — 300ms, no spinner).
- `<title>` + favicon + og:image. Footer con año y los relojes… no — footer simple pero intencional.
- 404.html de GitHub Pages con el mismo fondo + link de vuelta.

---

## 4. MÉTODO DE VERIFICACIÓN (obligatorio por fase — el hallazgo de hoy)

**El preview headless NO sirve para esta escena** (rAF congelado, screenshots timeout) — meses de
mediciones numéricas no vieron lo que un screenshot real mostró en 1 minuto. Desde ahora:
1. Sonnet edita → push a Pages (o server local) →
2. **claude-in-chrome**: `navigate` → `wait 5s` → scroll por TODAS las secciones → `screenshot` de cada una.
3. Comparar contra la definición de aprobación de la fase (ej. F2: "se ve la columna refractada a
   través del vidrio"). Si el screenshot no lo muestra, la fase NO está terminada.
4. Recién ahí commit definitivo + memoria.
Checklist visual mínimo por fase: (a) ningún texto tapado por el hilo, (b) ningún caption de una
sección visible en otra, (c) ningún elemento DOM pisando placas 3D, (d) contraste de texto legible
en el screenshot achicado al 50%.

## 5. Reglas que NO se tocan
- Cursor nativo siempre (jamás custom). Sin cajas dentro de cajas. Caja "explosión eléctrica" sigue
  vetada hasta pedido explícito. Tier system y fixes mobile previos se conservan.
- El hilo/columna de partículas NO se rediseña: es lo mejor del sitio. Solo se reubica en Z y se
  le da el rol de protagonista en las transiciones.

## 6. Decisiones tomadas
1. **Asset de PROOF: Opción A — diales de calibración** (elegida por Agus, 2026-07-08).
2. Sonnet ejecuta F1→F7 en orden, una fase = un commit verificado con screenshot real de Chrome.
