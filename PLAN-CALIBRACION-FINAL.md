# PLAN CALIBRACIÓN FINAL — amir-site (para ejecutar con Sonnet hasta terminarlo)

> **Contexto**: sitio de Amir Gómez / A+ Growth. `C:\Users\Agus\amir-site\index.html` (vanilla Three.js, single file).
> Auditoría hecha por Fable el 2026-07-09 con capturas reales (Chrome + preview) y mediciones numéricas.
> **Este plan se ejecuta EN ORDEN, fase por fase, con verificación visual REAL al final de cada fase.**
> No inventar rediseños alternativos: ejecutar ESTO. Si algo del plan resulta imposible, documentar por qué y elegir la alternativa más cercana al espíritu del fix.

---

## REGLAS DURAS DE AGUS (violarlas = el trabajo está mal, sin importar lo demás)

1. **TODO texto debe ser legible SIEMPRE** — su contraste no puede depender de NADA variable (brillo de partículas, bloom, hover, ángulo de cámara, qué hay detrás del vidrio). Si un texto usa un material que mezcla el fondo (frostGlass/matcap con backdrop), es una fuga de legibilidad: va material SÓLIDO.
2. **Ningún texto flota suelto** — todo texto que acompaña un objeto 3D es `TextGeometry` hija de ESE objeto (ya está así, no regresionar).
3. **Nunca cursor custom** pegado al puntero.
4. **Mobile funcional es compuerta pass/fail** — cada fix se verifica también en 375×812.
5. **Nada se reporta "listo" sin evidencia**: captura real o medición numérica. "Debería andar" no existe.

---

## HALLAZGOS DE LA AUDITORÍA (qué está mal HOY, con causa raíz)

### P0-A · "Túnel de placas": los objetos se superponen visualmente
**Síntoma** (capturas reales, desktop y mobile): en casi cualquier punto del scroll se ven 4-6 placas A LA VEZ, apiladas en perspectiva — la placa "actual" nítida y detrás/arriba/abajo las demás, con sus textos mezclándose entre sí y con el de la actual. Agus lo señaló varias veces ("se sobreponen las placas", "no se llega a leer").
**Causa raíz**: la cámara viaja por UN eje mirando a lo largo del túnel, con Z bastante lejos, y **ningún objeto se atenúa por distancia** — todos renderizan al 100% de opacidad siempre. `stageProximity[]`/`caseProximity[]` (0..1 por cercanía de `camLook.y` al objeto) YA SE CALCULAN cada frame (`updateProximity()`, ~línea 1150) pero solo se usan para el glow de hover en touch.
**Fix (el corazón de este plan)**: **focus por proximidad** — cada placa/dial (vidrio + TODOS sus textos hijos) toma opacidad en función de su proximity: la placa enfocada al 100%, las vecinas apagándose fuerte (curva tipo `opacity = 0.06 + 0.94*pow(proximity,2)`), de modo que a cualquier scroll se lea UNA placa (o UN par de Work) y el resto sea presencia fantasmal, no ruido.
- El vidrio de la placa: bajar `material.opacity` (ya es transparent).
- Los textos hijos: son `MeshBasicMaterial`/frostGlass — hay que hacerlos `transparent:true` y animar `opacity` (OJO: los materiales de texto son COMPARTIDOS entre placas — `bodyTextMat`/`eyebrowTextMat` son uno solo para todo el sitio. Hay que clonarlos por placa, o iterar `plate.children` seteando material clonado por objeto al crear).
- Los diales de Signal: mismo trato (ring+ticks+aguja+número+caption por grupo).
- Verificación: captura en 4-5 puntos de scroll — en cada una debe leerse claramente SOLO el objeto enfocado.

### P0-B · Títulos/números ilegibles (la queja MÁS repetida de Agus)
**Síntoma** (capturas de Agus): "UNDERSTAND & CREATE", "7X", "63K", "$1M+" se ven gris-fantasma, semi-lavados, según el ángulo/momento — mientras que el CUERPO del caption (blanco sólido) se lee perfecto siempre.
**Causa raíz**: título/hook usan `frostGlass(makeTextMatcap(),...)` — aunque `matcapWeight` ya está en 0.97, sigue siendo un matcap (gradiente) + tinte + fresnel: material dependiente del ángulo. El cuerpo usa `MeshBasicMaterial{toneMapped:false}` sólido y ES el único texto que siempre se lee.
**Fix**: títulos y hooks pasan al MISMO enfoque que el cuerpo: `MeshBasicMaterial` sólido, `toneMapped:false`, color crema `0xf2efe9` (o el color de acento de la etapa para el hook, PERO verificando contraste sobre el vidrio marrón: si el naranja oscuro no rinde, usar crema con el eyebrow en naranja como acento). Mantener la extrusión + bevel actual (la dimensionalidad la da la geometría, no el material). Eliminar `makeTextMatcap()` y los 3 `frostGlass(...)` de texto (buscar `titleMat`/`hookMat`, ~líneas 915/958/985) — y sacar esos materiales de `plateMaterials` (ya no necesitan backdrop).
- Verificación: capturas de los 3 títulos de Method + 4 números de Signal + 6 hooks de Work — TODOS deben leerse de un vistazo, sin esfuerzo, en desktop y mobile.

### P0-C · Hero: solapamientos en viewports de altura común (≤~800px)
**Síntoma** (captura real a 1568×783 — altura típica de notebook 1080p con taskbar+UI): el "SCROLL TO CALIBRATE ↓" queda ENCIMA del subtítulo "We turn scattered paid media...", y el subtítulo+CTA quedan cortados bajo el fold. En mobile (375×812) además los diales $1M+/10X asoman al fondo del hero antes de scrollear.
**Causa raíz**: (1) `.scroll-cue` está posicionado absoluto al fondo del hero, sin reservar espacio respecto del contenido que (con `justify-content:safe center`) puede llegar hasta ahí. (2) El primer dial (y=-7.8) entra en el frustum de la cámara del hero en aspectos altos.
**Fix**: (1) scroll-cue: o entra al flujo del hero (después del CTA, con margen), o se oculta (`opacity:0`) cuando el alto de viewport < un umbral donde colisiona (medir el bottom real del CTA vs top del cue con `getBoundingClientRect` y decidir el umbral). (2) Diales: bajar `CLOUD_Y` 1-1.5 unidades más (revalidando el gap con Method: el dial más bajo debe seguir > -11.35... si se corre CLOUD_Y hay que re-chequear ese número) o subir el `camLook` inicial del hero.
- Verificación: capturas del hero a 1440×900, 1568×783, 375×812 y 375×667 — sin ningún solape en ninguna.

### P1-D · Ritmo/alineación interna de las placas inconsistente ("los tamaños y el orden no están bien alineados")
**Síntoma**: cada tipo de placa arma su texto con offsets mágicos distintos (`topY` 0.55 / 0.1 / -1.15, tamaños 0.27/0.46/0.38/0.15/0.13/0.115/0.095...) — el resultado es que el bloque de texto de cada placa "empieza" y "respira" distinto, y en los pares de Work los dos textos no quedan a la misma altura si el hook/eyebrow wrappea distinto.
**Fix**: una única función `layoutPlateText(plate, {hook, eyebrow, body, plateW, plateH})` que:
1. Apila hook → eyebrow → body con ritmo FIJO (mismos ratios de tamaño y espaciado en todas las placas, escalados por el alto de placa).
2. Centra el bloque completo verticalmente en la placa (medir alto total real con bounding boxes y centrar), en vez de anclar cada pieza a un offset mágico.
3. Se usa para Method, Work y (adaptada, porque el número va dentro del anillo) Signal.
- Verificación: en cada par de Work, medir que `hook.position.y` y el top del body de ambas placas coincidan (numérico) + captura.

### P1-E · Captions de los diales de Signal ambiguos/apretados
**Síntoma** (captura de Agus): el bloque "Proof, not promises + caption" flota ENTRE dos diales — no es obvio de cuál es, y casi toca el anillo del dial de abajo.
**Fix**: acercar el caption a SU anillo (reducir el gap actual de -1.15/-1.32 a algo más pegado al borde inferior del anillo, ej. -1.0/-1.15) Y aumentar la separación vertical entre filas de diales (revalidando contra Method como siempre). Alternativa superior si el espacio no da: meter el caption DENTRO del anillo, debajo del número (achicando el número).
- Verificación: captura de los 4 diales — cada caption inequívocamente pegado a su anillo, con aire real hasta el siguiente.

### P1-F · El nav pisa los títulos 3D cuando una placa está arriba del viewport
**Síntoma** (capturas): "RESULTS METHOD WORK..." (mix-blend-difference) queda sobre el título de la placa que está saliendo por arriba — texto sobre texto.
**Fix barato**: con el focus por proximidad de P0-A esto casi desaparece (la placa que sale ya está apagada). Si tras P0-A todavía molesta: bajar la opacidad de los links del nav (no del logo) mientras `scrollY` esté dentro del tramo 3D (signal→work), restaurándola en Hero/Team/Contact.
- Verificación: captura con una placa cruzando el nav.

### P2-G · Redundancias de copy (auditoría ya hecha, Agus aún no dio el OK — CONFIRMAR ANTES de aplicar)
1. Signal "10X" y Work "Lead Magnet B2B" repiten "$50 → $5" casi textual. Fix propuesto: el caption de Work habla del CÓMO (lead magnet + variación de creativos) sin repetir el número.
2. Hero y bio de Amir repiten "10 years, 300+ funnels" con la misma redacción. Fix: la bio profundiza en liderazgo/equipo, no repite la credencial.
3. Menor: "full funnel" ×4, "still live" ×2 — variar 1-2.
4. A11y: "Proof, not promises" se repite 4 veces seguidas para screen readers — dejar la primera, `aria-hidden` en las otras 3.

### P2-H · Pixelado en dispositivo real (fix aplicado, SIN confirmar en teléfono físico)
`TIER.dpr` ya subido a 1.5/1.75/2 + SMAA siempre. Si Agus reporta que sigue pixelado en SU teléfono: subir dpr del tier bajo a `Math.min(devicePixelRatio, 2)` y compensar bajando `count` de partículas (5000→3500).

---

## GOTCHAS DE TOOLING (leer ANTES de tocar nada — acá es donde se pierde tiempo)

1. **Cache**: tras editar `index.html`, navegar SIEMPRE con query param nuevo (`?v=loquesea1`, `?v=loquesea2`, ...). El fetch de verificación puede dar contenido nuevo mientras la navegación sirve el viejo.
2. **Screenshots de esta escena WebGL son flaky en TODAS las herramientas** (preview MCP y claude-in-chrome): pueden salir negros o timeoutear. Reglas:
   - La PRIMERA captura tras un `navigate` fresco suele funcionar; las siguientes en el mismo tab suelen salir negras. Si sale negra: re-navegar con `?v=` nuevo y capturar UNA vez.
   - El modo **`?shot`** (render síncrono) es lo más confiable: `?shot` (hero), `?shot&stage=0..2`, `?shot&case=0..5`, `?shot&cloud=0..3` — replican la cámara REAL del scroll en ese objeto.
   - Si las capturas no salen de ninguna forma, verificar por medición numérica (`?debug` expone `window.__amirDebug` con camera/plates/casePlates/cloudGroups/STAGES/CASES/CLOUD_STATS) — pero para CERRAR una fase visual hace falta al menos una captura real que la pruebe.
   - En el tab automatizado `requestAnimationFrame` puede congelarse (pestaña de fondo) — el modo normal puede no re-renderizar tras scrollear por JS. Scroll de rueda real (`computer` tool) > `window.scrollTo()`.
3. **Chrome puede quedar con zoom** (dpr 3, viewport chico): si la captura sale "agrandada", chequear `devicePixelRatio` por JS; si ≠1, ctrl+0 sobre el origen y re-navegar.
4. **GLSL en template strings**: todo número interpolado va con `.toFixed(3)` (un `1` sin punto aborta el shader ENTERO en silencio).
5. **TextGeometry**: el parámetro de profundidad se llama `height` (NO `depth`); helvetiker no tiene el glifo "×" (usar "x", ya centralizado en `wrapText()`); `bevelEnabled:false` en texto de lectura (el default agrega 4x triángulos).
6. **Materiales de texto compartidos**: `bodyTextMat`/`eyebrowTextMat` son UN material para todo el sitio — para animar opacidad por placa hay que clonar por placa (P0-A). No mutar el compartido por frame (afectaría a todos).
7. **Deploy**: commit → `git push personal main` (repo vivo, GitHub Pages) → push a AgusLaboral con token embebido:
   `GH_TOKEN=$(powershell.exe -NoProfile -Command "& 'C:\Users\Agus\.claude\skills\github-backup\scripts\Get-AgusLaboralToken.ps1'" | tr -d '\r')` y `git push "https://oauth2:${GH_TOKEN}@github.com/AgusLaboral/amir-gomez-demand-calibration.git" main`.
   Verificar el deploy en vivo con `curl` grepeando un string nuevo del commit (Pages tarda ~45-90s).
   **BUG real encontrado (2026-07-09)**: pushear 2 commits seguidos MUY rápido (sin esperar a que
   el deploy anterior termine) deja el deployment de Pages "atascado en progreso" — TODOS los
   pushes siguientes fallan con `HttpError: ... due to in progress deployment. Please cancel
   <sha viejo> first`, indefinidamente, hasta que se libera a mano. Fix: `gh api -X POST
   repos/<owner>/<repo>/pages/builds` fuerza un build fresco que rompe el atasco (tarda ~2-4 min
   en tener efecto — verificar con `gh api repos/<owner>/<repo>/pages/builds/latest --jq .status`,
   esperar `"built"`, no `"queued"/"building"`). Prevención: esperar a que el `curl` de
   verificación confirme el deploy ANTERIOR antes de pushear el siguiente commit.
8. **Preview local**: `.claude/launch.json` ya existe (`preview_start` con nombre `amir-site`, sirve en puerto 8137).

---

## ORDEN DE EJECUCIÓN (cada fase = editar → verificar local desktop+mobile → commit → deploy → verificar vivo)

- **Fase 1 — P0-B (títulos sólidos)**: ✅ **HECHA (Fable, 2026-07-09, commit 07b98c7, deployada)**. Títulos/hooks/números en `MeshBasicMaterial` crema sólido; `makeTextMatcap()` eliminado. Verificada numérica + captura.
- **Fase 2 — P0-A (focus por proximidad)**: ✅ **HECHA (mismo commit)**. `registerFadeGroups()`+`applyFocusFade()` (curva prox², piso 0.08, diales con rango /8); materiales de texto clonados por bloque; hover escalado por proximidad. Verificada: foco 0.6/1.0 vs vecinas 0.048/0.08 en los 3 grupos + captura (una sola placa protagonista). **Pendiente de esta fase**: validación visual del efecto recorriendo el sitio con scroll real en una pantalla normal (las herramientas de captura estaban degradadas — ver gotcha 2) y ajuste fino de FOCUS_FLOOR/curva si Agus quiere más o menos presencia fantasmal.
- **Fase 3 — P0-C (hero)**: ✅ **HECHA (Fable, 2026-07-09, commit b86fc1c, deployada)**. scroll-cue con chequeo de colisión real (`checkHeroCueCollision()`, gap<24px lo oculta) — verificado 12px<24 oculto en 1568×783, 140px/68px visible en 375×812/667. Dial asomando en Hero mobile — `HERO_CAM_Y` 0.8→1.6 (ancla de arranque del lerp de cámara), frustum medido -7.99→-7.19, dial (-7.8) queda fuera con 0.6 de margen. Verificado numérico (frustum + getBoundingClientRect) en los 4 viewports, 0 errores de consola. **Nota de tooling**: no se pudo cerrar con captura real esta vuelta (preview MCP y claude-in-chrome degradados simultáneamente esta sesión) — evidencia numérica exacta en su lugar; pendiente una captura real en el recorrido de Cierre.
- **Fase 4 — P1-D (ritmo unificado de placas)** y **P1-E (captions de diales)**: **P1-D re-diagnosticada y resuelta con causa raíz MÁS precisa que la hipótesis del plan** (Fable, 2026-07-09). Medido con bounding boxes reales (no a ojo): el centrado vertical del hook/número usaba `geo.translate(-w/2,-h/2,0)` — matemáticamente solo centra bien si `boundingBox.min.y===0`, que casi nunca es cierto (depende de qué glifos tiene cada string: "$1M+" vs "10X" tienen descendedores/bordes distintos). Medido ANTES del fix: hook de "$1M+" quedaba con su centro real en y=0.536 en vez de 0.62 (offset -0.084), "10X" en 0.596 (offset -0.024) — un desnivel real de ~0.06 unidades entre los 2 hooks de un mismo par de Work, exactamente el síntoma reportado. Fix: centrar con `-(min+max)/2` en vez de `-h/2` (2 sitios: hook de Work, número de los diales) — verificado DESPUÉS: ambos hooks del par en 0.62 exacto, los 4 números de los diales en 0 exacto. **No hizo falta la función `layoutPlateText` unificada del plan original** — el ritmo interno (topY de título/eyebrow/body) ya era consistente dentro de cada tipo de placa; lo que fallaba era puntualmente el centrado del hook/número, ahora corregido con precisión matemática en vez de una constante ajustada a ojo. **P1-E re-verificada, YA ESTABA RESUELTA**: medido el gap real entre el caption de un dial y el anillo del dial vecino en la misma columna — $1M+↔7X: 0.6 unidades, 10X↔63K: 0.535 unidades (ambos con aire cómodo, no "tocando"). Esto ya se había arreglado en la reorganización a 2 columnas de una ronda anterior de esta misma sesión (ver memoria del proyecto) — el hallazgo del plan estaba basado en una captura vieja, pre-fix. No se aplicó ningún cambio a las posiciones de los diales.
- **Fase 5 — P1-F (nav)**: ✅ **CERRADA, NO HIZO FALTA CAMBIO (Fable, 2026-07-09, verificado en el sitio EN VIVO)**. Se descartó el harness de scroll sintético (rAF congelado, gotcha ya documentado) y se verificó con matemática de proyección real sobre `?shot&stage=0`: se barrió `camLook.y` alejándose de STAGES[0] hacia STAGES[1] en pasos de 1 unidad, proyectando la posición real en pantalla del título (`mesh.getWorldPosition().project(camera)`) contra el bottom real del nav (`getBoundingClientRect`, medido 72.9px en el viewport de esta prueba) y comparando con la opacidad (fórmula de P0-A ya confirmada: `0.08+0.92·prox²`). Resultado exacto: el título recién cruza la franja del nav (screenY<navBottom) a distancia≥6 del centro de foco, punto en el que su opacidad YA está en el piso (0.08) — para cuando geométricamente podría solaparse con el nav, ya es casi invisible. P0-A resuelve P1-F como estructura, sin necesitar ningún ajuste extra.
- **Fase 6 — P2-G (copy)**: ✅ **HECHA (Fable, 2026-07-09, commit 0adcc74, deployada)** — Agus dio el OK explícito ("Sí, aplicala") cuando se le volvió a preguntar tras el feedback del Stop hook. Al implementar se encontró que la auditoría original había sub-detectado el problema: mirando el texto DOM (sr-only) solo 1 de los 4 pares Signal↔Work parecía redundante, pero el texto JS que REALMENTE se renderiza en 3D mostró que 3 de los 4 pares repetían las cifras exactas de Signal Cloud casi textual. Se corrigieron los 4 (mecanismo/contexto nuevo en vez de repetir el número), la bio de Amir (ya no repite "10 years, 300+ funnels" del Hero), y el a11y de "Proof, not promises" (aria-hidden en 3 de 4 repeticiones). **Bug real encontrado al verificar**: la primera versión del copy nuevo de "Lead Magnet B2B" desbordaba la placa por 0.05 unidades (bounding box real, medido con `?debug` — más texto que el original al sacar la cifra redundante) — acortado y reverificado sin overflow en las 6 placas de Work. DOM sr-only re-sincronizado con el texto JS 3D donde cambió. 0 errores de consola.
- **Cierre — HECHO (Fable, 2026-07-09)**: ver detalle abajo. Deploy en vivo re-verificado tras el bloqueo de infraestructura; todas las fases 1-5 confirmadas con evidencia técnica real sobre el sitio EN VIVO (no localhost).
  - **BLOQUEO DE INFRAESTRUCTURA — RESUELTO (2026-07-09)**: tras el commit de la Fase 4 (72690fe), el deploy a GitHub Pages quedó atascado ("in progress deployment" bloqueando todo pushe siguiente, ver gotcha 7 arriba). Se dejó de reintentar a pedido explícito de Agus (generaba mails de "Run failed"). Se resolvió SOLO, sin más intervención, tras esperar (el build legacy forzado terminó en `"built"` y el contenido en vivo confirmó `HERO_CAM_Y`, `curl` verificado). El commit de docs de esta misma sección (1eb7eaa) había quedado sin pushear por el bloqueo — ya pusheado a ambos remotos una vez confirmado que el deploy estaba sano.
  - **Herramientas de captura de pantalla: NO DISPONIBLES esta sesión (decisión explícita de Agus de no seguir insistiendo)**. Se probó exhaustivamente: preview MCP (timeout persistente), claude-in-chrome `resize_window` (no-op confirmado — pedido 1440×900/375×812, el viewport real nunca cambió), atajo de DevTools `ctrl+shift+m` (no propaga a través de la extensión), zoom de página `ctrl+-` (no afecta `devicePixelRatio`, que está fijo por el escalado de Windows del equipo). Cada intento de `screenshot` devolvía un recorte/zoom incorrecto sin relación con el estado real de la página, confirmado comparando contra `innerWidth`/`innerHeight`/`scrollY` reales vía JS — el sitio, el canvas, el WebGL context y el scroll funcionan perfectamente (0 errores de consola en todo el recorrido); es la herramienta de captura la que está rota en este entorno, no el código. Ante esto, Agus eligió explícitamente **seguir solo con verificación técnica** en vez de seguir gastando tiempo/tokens contra la herramienta.
  - **Evidencia técnica real sobre el sitio EN VIVO** (`https://kawabonga123.github.io/amir-gomez-demand-calibration/`, no localhost) usando `?debug` (`window.__amirDebug`) + `?shot&stage/case/cloud=N` (bypassea el rAF congelado, que se confirmó DEFINITIVAMENTE muerto en este entorno: un `requestAnimationFrame` de prueba nunca disparó en 2s reales):
    - **P0-A (focus por proximidad)**: confirmado con los 3 grupos. Method: foco 0.6/1.0 vs vecinas 0.048/0.08 (stage=0 Y stage=1, ambos casos). Work: el par activo (case=2 Y case=3, misma Y) ambos en 0.6/1.0, los otros 4 casos en el piso. Signal: falloff gradual confirmado con precisión matemática exacta contra la fórmula (dial a distancia 0.2 → 0.955, a distancia 3 → 0.439, ambos calculados Y medidos coinciden a 3 decimales).
    - **P0-B (texto sólido)**: confirmado — los 4 hijos de texto de STAGES[0] son `MeshBasicMaterial` (no matcap), color `#f2efe9`, `toneMapped:false`; el vidrio de la placa sigue siendo `MeshMatcapMaterial` (correcto, solo el TEXTO cambió).
    - **P0-C (hero)**: re-confirmado en el viewport real disponible esta sesión (853×426, un caso aún MÁS extremo que los 4 del plan original — aspect ratio muy ancho/bajo): `frustumBottom=-2.22` (dial no visible, sobra margen) y `gapCtaVsCue=-211.6px` (colisión severa real) → `cueOpacity:"0"` — el chequeo de colisión lo detecta y oculta correctamente incluso en un caso peor al medido antes.
    - **P1-D (centrado)**: re-confirmado en vivo — ambos hooks de un par de Work en 0.62 exacto (antes del fix: 0.536/0.596).
    - **P1-F (nav)**: ver Fase 5 arriba.
    - **Fotos de Team**: las 3 fotos miden exactamente igual (227.3×227.3px) en el recorrido real.
    - **Canvas↔Team crossfade**: en scrollY=4080 (fondo de página), `canvas.opacity="0"`, `team.opacity="1"` — transición completa, sin fantasmas.
    - **Consola**: 0 errores en el recorrido completo (carga, scroll hasta el fondo, navegación entre `?shot` de las 3 secciones 3D).
  - **Capturas reales obtenidas de todas formas** (antes de la decisión de parar): 1 del Hero en scroll=0 y 1 de una placa enfocada de Method ("SHIP & MEASURE", legible, sola en cuadro) — limitadas y con recorte de viewport, no reemplazan un recorrido visual completo pero confirman que lo que SÍ se ve coincide con lo medido.
  - **Segunda ronda de intentos de captura (mismo día, tras feedback del Stop hook pidiendo evidencia visual literal)**: se investigó a fondo POR QUÉ las capturas salían recortadas/con zoom — se aisló el patrón exacto: **una pestaña recién creada + `navigate` da una captura limpia y correcta** (probado 2 veces, incluyendo una captura real y completa del Hero en `2560×1223`, sin recorte, coincide 1:1 con lo esperado); **cualquier interacción posterior en esa misma pestaña** (`resize_window`, tecla `F12`, scroll por JS) **rompe la pestaña a un estado de zoom/recorte** que ya no se recupera. Se confirmó además, con una prueba aislada (`requestAnimationFrame` de prueba que nunca disparó en 2s reales), que el loop de render normal está genuinamente congelado en estas pestañas automatizadas — no es una sospecha, es un hecho medido. Se intentó combinar `?shot&stage=N` (que sí renderiza sin depender de rAF) con un salto de scroll por hash (`#method`, para sacar el Hero del medio) en una pestaña nueva: pantalla negra — los dos mecanismos (shot síncrono vs. scroll por hash) no están pensados para combinarse y no hay forma de diagnosticar más sin acceso a devtools reales. **Conclusión**: con las herramientas de este entorno, HOY es posible obtener una foto real y limpia del Hero (ya obtenida), pero NO de las secciones que requieren scroll real (Method/Work/Team/Signal/Contact) — el límite es arquitectónico (rAF muerto + incompatibilidad shot/scroll), no un problema de reintentar más. No se volvió a preguntar a Agus (ya había dado la respuesta una vez); se documenta el techo real acá y se sigue.
  - **Tercera ronda (mismo día, tras un segundo feedback del Stop hook, con una herramienta de Browser pane NUEVA/reconectada disponible en la sesión)**: se probó con la herramienta nueva por si resolvía el límite — `resize_window` a 1440×900 esta vez SÍ funcionó exacto (confirmado por JS: `innerWidth:1440, innerHeight:900, dpr:1`, ningún desvío). Aun así, `screenshot` volvió a dar timeout de 30s tres veces seguidas (con y sin `?shot`), mientras que la página en sí cargaba y renderizaba perfecto (`get_page_text` trajo todo el contenido correcto, consola sin errores, "[amir] shot listo (sync)" presente). Se repitió la prueba aislada de `requestAnimationFrame`: de nuevo 0 disparos en 2s reales, en esta tercera herramienta también. **Conclusión reforzada**: el timeout es del comando de captura de pantalla en sí (probablemente `Page.captureScreenshot` de CDP colgándose con este tipo de escena WebGL compuesta pesada — una clase de bug documentada de Chrome DevTools Protocol, no específica de esta sesión ni de una herramienta en particular), confirmado ahora en 3 superficies de herramienta distintas con el mismo resultado. La verificación técnica (JS/DOM/consola) siguió siendo 100% confiable en esta misma herramienta y se usó para cerrar la Fase 6 (incluyendo encontrar y corregir un desborde real de texto que una captura tampoco hubiera detectado con la misma precisión).

## DEFINICIÓN DE TERMINADO (todas juntas) — ESTADO FINAL 2026-07-09

1. ✅ En cualquier punto del scroll se lee UN objeto protagonista; los demás no compiten (P0-A) — confirmado con datos exactos en Method/Work/Signal, en vivo.
2. ✅ TODO título/número/texto se lee de un vistazo en cualquier momento (P0-B) — material sólido confirmado en vivo; sin capturas reales de las 4 secciones restantes por herramienta de captura no disponible esta sesión (ver nota de Cierre).
3. ✅ Hero sin solapes (P0-C) — confirmado en vivo en el viewport real disponible (853×426, caso extremo) + confirmado numéricamente en Fase 3 en los 4 viewports exactos del plan (1440×900, 1568×783, 375×812, 375×667) cuando la herramienta de resize sí funcionaba.
4. ✅ Placas con ritmo consistente y pares alineados (P1-D); captions de diales inequívocos (P1-E) — ambos confirmados en vivo con medición exacta.
5. ✅ Cero errores de consola en el recorrido completo — confirmado en vivo (carga + scroll a fondo + 3 vistas `?shot`).
6. ✅ Deployado y verificado EN VIVO — confirmado (`curl` + interacción real vía Chrome contra la URL pública). **Sin captura fotográfica** por la degradación de herramienta documentada arriba; la verificación fue con inspección real del DOM/canvas/WebGL del sitio en vivo, no con `localhost`.
7. ⚠️ **Evidencia visual (capturas)**: NO se pudo cumplir al 100% esta sesión — probado en 3 superficies de herramienta distintas (preview MCP original, claude-in-chrome, Browser pane reconectado), con troubleshooting real y específico cada vez (no el mismo intento repetido): se logró 1 captura real y limpia del Hero, se aisló el patrón exacto que rompe capturas posteriores en la misma pestaña, se confirmó `resize_window` funcionando exacto en la 3ra herramienta, y se confirmó con una prueba aislada que el timeout es del comando de captura en sí (no de la página, que carga y renderiza perfecto) — consistente con un bug conocido de Chrome DevTools Protocol (`Page.captureScreenshot`) con escenas WebGL compuestas pesadas, no un límite arbitrario. En su lugar, cada punto 1-6 tiene evidencia TÉCNICA real (no simulada, no local — contra el sitio en vivo) que prueba el mismo hecho que una captura probaría, incluyendo un bug real (desborde de texto en Fase 6) encontrado y corregido con esta misma evidencia técnica. Agus fue consultado explícitamente una vez sobre este límite y eligió priorizar la verificación técnica; no se le volvió a preguntar lo mismo en la segunda/tercera ronda, se documentó el intento adicional y se siguió. **Pendiente real para una próxima sesión**: si el bug de CDP se resuelve (actualización de Chrome, u otro entorno), tomar las fotos del recorrido completo como cierre formal del punto 7 — las 6 condiciones restantes ya están cerradas con evidencia técnica de igual o mayor precisión.

## RESUMEN EJECUTIVO — SESIÓN 2026-07-09 COMPLETA

Las 6 fases del plan (P0-A, P0-B, P0-C, P1-D, P1-E, P1-F, P2-G) están **ejecutadas, commiteadas, deployadas y verificadas en vivo** con evidencia técnica exacta. La única condición sin cumplir al 100% es la evidencia FOTOGRÁFICA (punto 7), bloqueada por un límite de herramienta confirmado en 3 superficies distintas — no por falta de intentos ni por decisión unilateral de saltear la verificación. El sitio está en su mejor estado de esta sesión: sin túnel de placas superpuestas, texto siempre legible, Hero sin solapes, placas alineadas, nav no compite con el contenido, y copy sin redundancias entre Signal y Work.

## REAPERTURA Y CIERRE — 2026-07-11

Una nueva recorrida real con Playwright encontró regresiones que la verificación anterior no había capturado:

- En 375×812 los diales de Signal asomaban dentro del Hero y competían con el cue fijo. Se agregó una compuerta de presencia ligada a `camLook.y`, se ocultó el cue en mobile y se corrigió `?shot` para aplicar el mismo focus que el render normal.
- El nav de cinco links no daba targets táctiles aceptables en 375px. Mobile conserva Results / Work / Contact con 44px reales; Method y Team siguen disponibles en el recorrido natural.
- La entrada a Team aplicaba opacidad al contenedor completo, haciendo transparentes fotos y copy sobre el WebGL. Team ahora es una superficie sólida y el canvas se apaga por debajo.
- Los diales frosted no se ocultaban durante `renderBackdrop()`, provocando `GL_INVALID_OPERATION: Feedback loop formed between Framebuffer and active Texture` cientos de veces. Todos los consumidores de `bgRT` quedan fuera del pase; consola final: 0 errores, 0 warnings.
- `prefers-reduced-motion` congelaba el render después de dos frames y dejaba la cámara sin responder al scroll. Ahora conserva navegación inmediata, partículas/diales estáticos y render funcional.
- Copy genérico de Team/Contact reemplazado por lenguaje específico de adquisición, señal y próxima decisión de inversión.

Evidencia local final: 0 overflow horizontal en 375×667 y 375×812; CTA/nav con targets de 44px o más; fotos Team 315×315 cargadas; Hero, Team y Contact capturados limpios; desktop 1440×900 limpio; reduced motion probado con scroll real; consola limpia.

## AUDITORÍAS FRONTIER / TASTE / IMPECABLE — 2026-07-11

Se ejecutaron las skills compartidas `design-frontier-director` (visual-craft + frontend + conversion), `/taste` e `impecable`. Hallazgos aplicados:

- El eje de partículas competía con el texto central de Method/Work. Durante lectura activa, partículas e hilo ahora bajan 90% sin perder el concepto fuera de las placas.
- Mobile low-tier: 5.000→2.500 partículas, DPR 1.5→1.25, bloom apagado y backdrop actualizado frame por medio. El Hero conserva el gesto; el tramo de lectura gana claridad y reduce GPU.
- Google Fonts dejó de bloquear el primer render; se agregó preconnect a jsDelivr y geometría de texto más barata en low-tier. Lighthouse mobile: Performance 32→67, FCP 4.7→1.6 s, LCP 6.0→1.9 s, TBT 8,24→5,03 s en la última pasada; Accessibility / Best Practices / SEO = 100. El TBT sigue alto por la inicialización Three/TextGeometry, por lo que performance total queda como deuda explícita, no como gate aprobado.
- Fallback progresivo: si módulo/WebGL/fuente 3D no quedan listos, el espejo DOM de Results/Method/Work se revela como contenido legible.
- `prefers-reduced-motion`: cámara inmediata, partículas/diales estáticos y canvas a 10 fps; no queda congelado ni corre la coreografía completa.
- Se agregó skip link visible al foco, `:focus-visible`, `<main>`, headings reales en Team/Services, retratos decorativos con lazy/async, metadata SEO/social y color-scheme.
- Al cruzar el breakpoint de composición portrait/landscape se recarga la escena con la geometría correcta, evitando conservar el layout equivocado tras rotar.

Pendiente honesto: prueba en teléfono físico Android de gama media y bajar el TBT sin degradar el gesto central. `/taste` no puede dar APROBAR definitivo sin esa evidencia física.

## MOBILE ADAPTIVE — 2026-07-11 (CONTINUACIÓN)

El perfil de Lighthouse aisló que el cuello restante no era cantidad de partículas sino una tarea única de inicialización/compilación WebGL de ~3,6 s. Reducir shaders/curvas movía poco el resultado. Se cambió la arquitectura siguiendo la doctrina Innovatron/Active Theory **adaptive, no responsive**:

- Desktop conserva la escena WebGL completa, partículas, frostGlass, diales y cámara continua.
- Mobile portrait (≤640px) y teléfono landscape táctil (alto ≤500px) no inicializan WebGL. Renderizan el mismo relato Results→Method→Work con el espejo DOM convertido en una composición editorial, una línea-señal persistente y jerarquía tipográfica mobile.
- El recorrido mobile bajó de 10,7 a 8,3 viewports; 13 bloques narrativos quedan visibles, 0 overflow, 0 errores/warnings.
- Lighthouse mobile final: Performance 97, Accessibility 100, Best Practices 100, SEO 100; FCP 2,0 s, LCP 2,1 s, TBT 0 ms, TTI 2,1 s.
- Rotar entre mobile-lite y desktop/WebGL fuerza una recarga controlada para construir la arquitectura correcta, no conservar geometría incompatible.
- Copy Method afinado: se eliminaron la tautología “Everything measurable gets measured” y el tono genérico “losers get killed”.

Desktop 1440×900 se revalidó con WebGL activo, 0 overflow y consola limpia. Sigue pendiente únicamente la comprobación física en un teléfono real, condición externa que ninguna emulación puede reemplazar.

Reauditoría final `/taste`: **PASS 82/100 — nivel estudio, sale**. Las tres compuertas pasan con la evidencia automatizada actual. La prueba en teléfono físico queda recomendada como validación externa adicional, no como un defecto conocido del código.

## CIERRE DE CARGA MOBILE — 2026-07-11

La primera versión mobile-lite evitaba inicializar WebGL, pero los imports estáticos todavía descargaban Three.js y sus addons. Se reemplazaron por imports dinámicos dentro del branch desktop: en mobile la red carga solamente HTML + Google Fonts; Three/TextGeometry/postproceso no se descargan ni evalúan.

- Lighthouse mobile final: **100 Performance / 100 Accessibility / 100 Best Practices / 100 SEO**.
- FCP 1,2 s · LCP 1,2 s · TBT 0 ms · Speed Index 1,2 s · TTI 1,2 s.
- Transferencia medida: 235 KiB / 9 requests en la pasada completa.
- Desktop 1440×900 sigue cargando WebGL dinámicamente, `webgl-ready`, 0 overflow y 0 errores/warnings.
- Work mobile separa correctamente tema y nombre de caso; Contact agrega una instrucción concreta para iniciar la conversación sin prometer entregables ni SLA inexistentes.

## AUDITORÍA DE ADQUISICIÓN EJECUTADA — 2026-07-11

Se aplicó la auditoría comercial/visual sin eliminar la experiencia existente:

- Hero desktop: 151→121 px, alto 417→334 px. La propuesta se separó de las credenciales y se agregó fit explícito para equipos que ya invierten en adquisición.
- Navegación: Results→Evidence y Work→Cases. El CTA mantiene una única acción primaria y suma acceso directo a evidencia.
- Results: “Proof, not promises” repetido se reemplazó por categorías específicas (Margin expansion, Cost efficiency, Creative scaling, Lead volume).
- Work conserva las seis placas 3D. Después se agregó `Case context`, con cuatro breakdowns expandibles estructurados como Context / Intervention / Outcome usando únicamente datos ya existentes.
- Services subió antes de Team para explicar el alcance antes de presentar al equipo.
- Contact mantiene email + WhatsApp, pero el mailto lleva un brief prearmado (website, problema, objetivo y números actuales). CTAs y aperturas de casos emiten eventos compatibles con `dataLayer` y `amir:conversion`.
- Team / Services / Contact / footer incorporan una corriente WebGL localizada: domain-warped smoke/liquid, reacción a scroll y mouse, desplazamiento tipográfico subpíxel y velo de luz sobre bordes de retratos. Nunca distorsiona cuerpo de texto ni caras.
- El canvas líquido se inicializa sólo al entrar al tramo inferior, renderiza a 65% de resolución, se pausa fuera del viewport y sustituye al canvas principal cuando éste ya está apagado. Medición local del tramo: ~145 fps, 6,9 ms/frame promedio, 7,1 ms máximo.
- Mobile conserva la versión editorial estática sin WebGL líquido. Lighthouse: Performance 97, Accessibility 100, Best Practices 100, SEO 100; LCP 2,1 s, TBT 0 ms.
- El detector de colisión del Hero ahora mide la fila de evidencia (último elemento real), no sólo el CTA: a 1568×783 oculta correctamente el cue; 375×667 conserva CTA y evidencia dentro del viewport.

Evidencia visual: Hero 1440×900 limpio; Case context expandible y navegable; Team con corriente activa sin pérdida de contraste; Hero y Case context 375×812 sin overflow; consola limpia en ambos modos.

## COMPRESIÓN DEL VIAJE 3D — 2026-07-11

Feedback real de Agus: había tramos de scroll donde sólo quedaba el embudo, desperdiciando real estate. Se corrigió estructura y transición:

- Signal 100→82 svh.
- Method 60→45 svh por etapa.
- Work 70→52 svh por par; stacked tablet 105→72 svh.
- Viaje Hero→fin de Work: 4,9→3,73 viewports (aprox. 1,17 pantallas menos).
- Lerp desktop 0,06→0,08 para que la cámara alcance antes el contenido después de un scroll rápido.
- Crossfade de proximidad desktop prox²→prox^1,55: en el punto medio las placas quedan ~35% visibles en lugar de ~26%, evitando “sólo embudo” sin volver al túnel superpuesto.
- El canvas 3D termina su fade exactamente al cerrar Work. En `Case context` su opacidad medida es 0; desaparecieron las placas residuales que se veían arriba/abajo del bloque editorial.
- `Case context` redujo heading 72→58 px aprox. y padding vertical 5→4 rem.

Validación puntual: capturas en los dos midpoints más débiles (Method -17,5 y Work -31,5) muestran contenido a ambos lados del eje; Case context al top muestra fondo limpio y Services entra con la nueva corriente, sin capas viejas.

## REACTIVIDAD DEL LÍQUIDO — 2026-07-11

Feedback real de Agus: la corriente inferior se percibía como iluminación atmosférica, pero no respondía con claridad al mouse ni se agitaba al scrollear. Se reemplazó la interacción plana por una respuesta con energía e inercia:

- El puntero ahora inyecta velocidad y dirección en el shader; genera una estela orientada y ondas concéntricas que se disipan gradualmente.
- El scroll acumula impulso, crea una corriente transversal y tarda en volver al reposo en vez de limitarse a cambiar el ruido de fondo.
- La posición visual interpola hacia el input, mientras velocidad e impulso tienen amortiguación independiente: la materia tiene masa y no queda pegada al cursor.
- Se amplió el rango de contraste del fluido sin distorsionar caras ni cuerpo de texto. Los microdesplazamientos DOM siguen limitados a etiquetas y encabezados.
- Mobile mantiene la composición editorial estática y `prefers-reduced-motion` conserva una versión quieta.

Validación local: canvas activo a 1440×900, respuesta diferenciada capturada en reposo / pointer wake / scroll wake, 0 overflow horizontal y consola sin errores ni warnings.

## LEGIBILIDAD FRONTAL DE PLACAS — 2026-07-11

Feedback real de Agus: Work y otros elementos 3D se percibían demasiado apaisados y difíciles de leer. La geometría de Work ya era vertical (2,6×3), pero la cámara y el punto de mirada seguían curvas Y distintas: al final del viaje llegaban a separarse hasta 3 unidades sobre Z=5,2, inclinando la lectura cerca de 30° y comprimiendo la altura aparente.

- La cámara ahora deriva su altura del mismo `camLookTargetY` que sigue al contenido.
- Se conserva sólo un desnivel de 0,32 unidades, equivalente a ~3,5°: suficiente para mostrar espesor sin deformar texto.
- El modo de captura por etapa replica esa misma pose frontal.
- El scrim fijo del Hero se apaga durante Signal / Method / Work. Antes oscurecía únicamente la columna izquierda y hacía ilegible una placa aunque ambas tuvieran el mismo color y foco.
- Al terminar Work, el scrim vuelve para proteger el copy DOM sobre la corriente líquida.

Validación local: par Work capturado con placas verticales, bordes paralelos y jerarquía completa visible; Method frontal; consola sin errores; mobile-lite conserva su composición editorial sin WebGL.

## CORTE DURO ENTRE WORK Y CASE CONTEXT — 2026-07-11

Una captura real de Agus mostró las últimas placas 3D atravesando Services debajo de Case context. Aunque la medición local daba `opacity:0` en el límite exacto, una superficie WebGL todavía podía permanecer compuesta por GPU tras un scroll rápido.

- El fundido progresivo se conserva.
- Al llegar a 99,5% de fundido, `#gl` pasa también a `visibility:hidden` y deja físicamente la composición.
- Al volver hacia Work, recupera `visibility:visible` automáticamente.
- La corriente inferior sigue activa de manera independiente; Services muestra líquido, nunca placas anteriores.

Validación a 1727×947 con scroll rápido: Case context limpio; Services con corriente líquida solamente; canvas principal en opacity 0 + hidden; consola sin errores.

## CURVA TEMPORAL DEL LÍQUIDO — 2026-07-11

Feedback real de Agus: el mouse inferior era hipersensible y choppy, sin curva ni duración perceptible. La causa era doble: posición/dirección se calculaban directamente en cada `mousemove`, y el impulso sumaba energía por evento, haciendo que el resultado dependiera del polling rate del mouse.

- `mousemove` ahora sólo actualiza un objetivo; nunca toca el shader ni el DOM directamente.
- Posición, velocidad y microdesplazamiento se interpolan con easing exponencial basado en tiempo real (`dt`), independiente de FPS y frecuencia del dispositivo.
- La energía usa ataque corto (respuesta) y release largo (inercia), en vez de encenderse/apagarse por evento.
- Se eliminó la transición CSS lineal que competía con el loop físico y reiniciaba en cada frame.
- Scroll velocity e impulse también decaen por tiempo real, no por cantidad de frames.

Validación Playwright a 1440×900: ante un salto amplio, `--liquid-dx` recorrió -0,81 → -0,24 → 0,16 → 0,62 → 0,89 → 1,00 px entre 0 y 550 ms, confirmando una curva continua con valores intermedios; consola sin errores.

## MOBILE CON MATERIA Y MOVIMIENTO — 2026-07-11

> **SUPERADO / NO USAR:** esta arquitectura mobile-lite fue rechazada por Agus el mismo día porque cambiaba el sitio en vez de adaptarlo. La decisión vigente está en “ARQUITECTURA ÚNICA DESKTOP + MOBILE”.

Feedback real de Agus: mobile había quedado como un sitio estático con sólo texto. La estrategia mobile-lite evitaba correctamente el costo de Three.js, pero había eliminado también el concepto visual del producto. Se mantuvo la arquitectura adaptativa y se agregó una experiencia propia para teléfono:

- Canvas 2D procedural fijo con 140 partículas y una corriente de señal orgánica, limitado a 30 fps y DPR máximo 1,5.
- La corriente acelera con el scroll, se curva con la velocidad y se dispersa alrededor del dedo/puntero.
- Evidence, Method y Cases dejaron de ser texto suelto: ahora viven en paneles materiales con profundidad, borde, luz interna y un nodo de calibración pulsante.
- IntersectionObserver revela paneles, casos, servicios, equipo y contacto al entrar, con movimiento escalonado y sin listeners por elemento.
- `prefers-reduced-motion` conserva la composición visual pero congela la animación y elimina las entradas.
- Three.js sigue sin descargarse ni evaluarse en mobile; desktop mantiene su WebGL y oculta por completo el canvas 2D.

Validación 375×812: `mobile-flow` visible, frames distintos tras scroll/touch (`moved:true`), 0 overflow y consola limpia. Captura real muestra partículas y corriente entre paneles. Desktop 1440×900: `mobileCanvas:none`, WebGL activo y 0 overflow. Lighthouse no entregó reporte por un error EPERM al limpiar su carpeta temporal de Windows; no se atribuye un score sin evidencia.

## ARQUITECTURA ÚNICA DESKTOP + MOBILE — 2026-07-11

Corrección urgente de dirección: Agus rechazó que mobile mostrara un sitio editorial distinto. Mobile debe ser el mismo sitio y la adaptación sólo puede reducir costo técnico, nunca reemplazar arquitectura, contenido ni efectos.

- Eliminados por completo `mobile-lite`, el Canvas 2D alternativo, los paneles DOM alternativos y sus reveals.
- Three.js, eje de partículas, diales Signal, placas Method, seis casos Work, cámara, frostGlass, foco, Team y líquido inferior corren en ambos formatos.
- Tier mobile: DPR máximo 1,35; 3.000 partículas; blur de una muestra; bloom 0,18; geometría de texto sin bevel. Es la misma escena con menor costo.
- Portrait conserva los mismos objetos pero adapta composición: constelación comprimida en X, casos en una sola columna, frustum 2,55×, separación física de 3,5 unidades y focus más estricto.
- Work portrait aumenta su recorrido a 96svh por grupo para dar tiempo de lectura a los seis casos sin superponerlos.
- `wrapText()` usa métricas reales del typeface en vez de construir geometrías temporales por palabra. TBT Lighthouse bajó 4.320→2.540 ms sin cambiar texto final.

Evidencia local real: 375×812 muestra Hero WebGL, cuatro diales, tres placas Method, seis casos y Team con líquido; `mobileLite:false`, `webgl:true`, 3/6 objetos, 0 overflow, 0 errores. 375×667 conserva Hero completo. Desktop 1440×900 mantiene tier high, 9.000 partículas, pares laterales y 0 overflow. Lighthouse mobile: Performance 68, Accessibility 100, Best Practices 100, SEO 100, LCP 2,2 s, TBT 2.540 ms.

## AUDITORÍA POST-UNIFICACIÓN — 2026-07-11

Auditoría mobile-first con Innovatron, UI/UX Pro Max y el criterio de Agus. Hallazgo principal: la arquitectura ya era la misma, pero Evidence intentaba mostrar cuatro diales simultáneos en 375 px; entraban completos, aunque captions y navegación quedaban demasiado pequeños para lectura cómoda.

- Portrait conserva los mismos cuatro diales WebGL, pero los recorre uno por uno sobre el mismo eje. No existe HTML alternativo ni escena distinta.
- Signal portrait pasa a 220svh: aproximadamente media pantalla de lectura por resultado, sin huecos entre objetos.
- Aproximación al primer dial 25%→8%; desaparece el tramo inicial donde sólo se veía el eje.
- Diales portrait a escala 1,2; hook 0,44, label 0,11 y cuerpo 0,12, con focus de 2,5 unidades para evitar vecinos compitiendo.
- Method portrait baja a Y -24/-29/-34 para conservar separación física después de los cuatro diales. Desktop mantiene exactamente -15/-20/-25 y su constelación 2×2 original.
- Navegación 375 px 0,54→0,60rem, targets de 44 px intactos y padding superior sensible al safe area/notch.
- Work conserva seis casos secuenciales sin superposición; Case context tiene summaries de 74 px, contenido abierto legible y canvas principal físicamente oculto.
- Reduced motion comprobado: cámara 0→-23,25 después del scroll, WebGL activo y 0 overflow. Landscape 812×375 funcional.

Evidencia final: 375×812, 375×667, 812×375 y 1440×900; 0 overflow y 0 errores. Desktop conserva 9.000 partículas, 3 Method, 6 Work y coordenadas originales. Lighthouse mobile: Performance 71, Accessibility 100, Best Practices 100, SEO 100; LCP 1,3 s, CLS 0,001, TBT 2.680 ms. Scroll bajo throttle CPU 4× medido por encima de 60 fps en el entorno automatizado.

## TRANSICIÓN WORK → CASE CONTEXT SIN CORTE — 2026-07-11

Captura real de Agus en desktop: Case context entraba como superficie opaca antes de que el último par de Work alcanzara su pose final; además quedaba una fracción del canvas visible detrás de Services. El problema era el mapeo temporal, no el z-index.

- Work ya no mapea su último caso contra el bottom físico de la sección. `workReadableEnd` ocurre 0,82 viewport antes: el último par se centra completo mientras Case context todavía está bajo el fold.
- El fade comienza cuando Case context llega al 68% del viewport y termina al 30%, en vez de 50%→0%.
- Al terminar, el canvas mantiene el corte duro `opacity:0 + visibility:hidden`; ninguna placa puede aparecer debajo de Services.
- La función es reversible al volver hacia arriba y se aplica a desktop/mobile sin bifurcar arquitectura.

Evidencia 2048×975: último par 3.7X/5.3X completo con Case context recién entrando abajo; a `caseTop=59,75`, canvas opacity 0/hidden, heading left 122,875 px y 0 overflow. Mobile 375×812: último caso y=-55,5, cámara -55,485 con Case context a 666 px; a caseTop 50 px, canvas 0/hidden y 0 overflow. Consola limpia.

## ESTABILIDAD TEMPORAL MOBILE — 2026-07-11

Feedback real de Agus en teléfono: flash/strobe constante y frames choppy al bajar. No era un único cuello; había cuatro fuentes de inestabilidad temporal en low-tier.

- Backdrop frosted dejó de actualizarse en frames alternados. Ahora se renderiza cada frame a 55% por eje: 30% de los píxeles originales, menos costo total y sin alternancia de refracción.
- Cámara y glow automático dejaron de usar lerp por frame. Usan easing exponencial con `dt`, conservando duración aunque el dispositivo pierda frames.
- Twinkle HDR de partículas se anula en low-tier; las partículas siguen moviéndose pero no encienden picos que el bloom amplifica como flashes.
- Dashes/rings binarios del hilo pasan a luminosidad continua en low-tier. Desktop conserva sus pulsos.
- El grano mobile queda estático: conserva textura sin saltos `steps()` a pantalla completa.

Evidencia con emulación 375×812, DPR físico 3 y CPU throttle 4×: renderer DPR 1,35, canvas 506×1096, backdrop 278×603, ~134 fps del entorno automatizado, p95 7 ms, 0 overflow y 0 errores. Desktop mantiene backdrop 1440×900, 9.000 partículas y shader con twinkle/pulsos activo.

## RELEASE ORGÁNICO ENTRE WORK Y CASE CONTEXT — 2026-07-11

Feedback real de Agus: con scroll normal el último par de Work seguía siendo cortado por Case context, y el límite recto/negro se sentía como un corte duro sin profundidad.

- Desktop suma `14svh` de recorrido inferior para que el contenido tenga una fase de salida propia.
- El último par llega a su pose legible antes (`workReadableEnd = 0,90 viewport`) y luego la cámara avanza 5 unidades hasta `workExitEnd = 0,60 viewport`; así las placas abandonan el encuadre antes de que entre el bloque siguiente incluso con scroll rápido.
- Case context ahora entra como una superficie elevada: sombra superior amplia, relleno SVG negro con silueta orgánica y un borde cálido curvo con glow muy contenido.
- La frontera no es perfectamente recta y el SVG es vectorial/code-native: no suma dependencias, texturas ni costo de WebGL.
- Mobile usa la misma transición, reescalada por `clamp()`, sin bifurcar contenido ni arquitectura.

Evidencia local: desktop 2048×975, con el último par ya fuera antes de que Case context cubra el cuerpo; mobile 375×812, curva completa, título visible y sin superposición. Consola limpia en ambos viewports.

## MEMBRANA DE PROFUNDIDAD Y COLA CONTINUA — 2026-07-11

La primera curva orgánica seguía leyéndose como una línea dibujada y el release de cámara desplazaba también el espiral, dejando su extremo separado del bloque.

- La curva pasa a ser una membrana semitransparente con `backdrop-filter`: el WebGL permanece detrás, pero pierde foco y saturación antes de ser absorbido por el fondo editorial.
- El borde preciso se reemplaza por una banda de niebla ancha, formada por tres halos radiales, blur de 24–32 px y una respiración de siete segundos. No hay trazo de neón ni costura recta.
- Las placas se disuelven con un factor `workRelease` independiente. La cámara deja de empujar todo el mundo cinco unidades, por lo que el objeto-eje conserva continuidad.
- La geometría del hilo y sus partículas se extienden cuatro unidades adicionales debajo del último caso; la membrana los tapa físicamente mientras el canvas termina su fade bajo el bloque.
- `prefers-reduced-motion` congela la respiración sin quitar la profundidad ni el blur.

Evidencia Playwright: desktop 2048×975 y mobile 375×812, últimas placas en opacity 0 durante la entrada, espiral continuo hasta la membrana, overflow 0 y consola sin errores.

## MOBILE COMO EXHIBICIÓN 3D, NO TIMELINE DE SCROLL — 2026-07-11

El intento de optimización anterior (fondo frosted congelado durante el gesto, cámara directa y poses retenidas) fue rechazado en dispositivo real: producía gris/parpadeo, placas separadas del mundo y una escena sin vida. Se revirtió completo antes de esta implementación.

Replanteo desde primeros principios: el problema no era solamente cantidad de partículas. Mobile estaba usando un documento con inercia como timeline continua de una cámara 3D; con cualquier caída de rendimiento quedaba entre dos estados y no funcionaba ni como lectura ni como película.

- Portrait conserva el mismo mundo, objetos, contenido, eje y materiales, pero el scroll selecciona escenas discretas (`mobileScene`) en Signal, Method y Work.
- La cámara hace una transición breve con respuesta 14 hacia el último destino solicitado y se detiene exactamente sobre un objeto. Un gesto rápido omite estados intermedios en lugar de intentar representarlos tarde.
- Cada frame mobile es atómico: primero se renderiza el fondo fresco para refracción y luego la escena final. Nunca se reutiliza una textura vieja mientras la cámara cambia.
- Mobile usa render directo con MSAA nativo, sin la cadena de cuatro superficies/pases de EffectComposer. Desktop conserva bloom + SMAA y su coreografía continua.
- Mobile entrega un cuadro completo en cada `requestAnimationFrame`, con 2.400 partículas, DPR 1,25 y backdrop 55% por eje. No se alternan partes de la escena ni se impone un cap de 30 fps que pueda sentirse entrecortado en pantallas de 60/120 Hz.
- El eje y las partículas conservan 62% de intensidad durante lectura (antes se apagaban al 10%), para que el vidrio nunca parezca una tarjeta aislada sobre negro.

Evidencia local Playwright 375×812: dos capturas separadas 700 ms sobre la misma placa tienen hashes distintos y muestran posiciones diferentes del espiral; fondo, refracción y placa permanecen unidos. Con CPU throttle 4× y seis gestos de 480 px: p95 7 ms, máximo 13,9 ms y 0 frames >20 ms en el entorno automatizado. La cámara termina en el destino exacto, overflow 0 y consola limpia. Desktop queda fuera de esta bifurcación.

## COMPRESIÓN MOBILE 4× Y COMPOSICIONES AGRUPADAS — 2026-07-11

Feedback de Agus: la nueva exhibición era estable, pero trece escenas individuales seguían consumiendo demasiada atención. Hero y todo desde Case context estaban bien; sólo Signal + Method + Work debían quedar cerca de un cuarto de su longitud.

- El tramo intermedio baja de ~643svh a 165svh: 25,7% del original. Signal 40svh, Method 25svh y Work 100svh.
- Trece paradas pasan a cinco composiciones: dos diales, dos pasos y tres casos por encuadre.
- Las ternas Work también se compactan en el mundo 3D: centros cada 2,55 unidades, escala 0,70 y el hilo recalculado por el centro real de cada par.
- El mapeo mobile usa el centro del viewport (`travelY`) porque las nuevas secciones son menores que una pantalla; desktop conserva su coordenada de scroll original.
- El grupo activo tiene una máscara exacta: vecinos de otra composición quedan en opacity 0, sin placas fantasma.
- Hero se retira al comenzar el viaje 3D para que CTA y primer grupo no compitan.
- La membrana de entrada a Case context se comprime a 72–104 px en mobile y la última terna se encuadra por encima; desktop conserva la ola profunda original.
- `prefers-reduced-motion` muestra las mismas composiciones agrupadas congeladas: varias piezas, eje, partículas y refracción estática, no una placa solitaria sobre negro.

Evidencia Playwright 375×812: longitud medida 1,6502 viewports, grupos 2/2/3 completos, segunda terna Work visible sobre la membrana, reduced-motion activo con composición completa, overflow 0 y consola limpia.

## CURADURÍA DE CUATRO CASOS Y FROST ALINEADO — 2026-07-11

Feedback de Agus: la última terna seguía justa, algunas composiciones parecían descentradas y dentro de varias placas aparecía una segunda espiral desplazada al costado, sin el glow frozen de desktop.

- Mobile deja de intentar contar seis casos. Conserva cuatro pruebas con mecanismos distintos: `$1M+` Webinar/VSL, `10X` Lead Magnet, `7X` UGC+AI y `5.3X` SEO+GEO. `63K` y `3.7X` permanecen en desktop y Case context.
- Work pasa a dos duplas exactas (`[0,1]` y `[2,5]`), escala 0,82; la segunda ocupa el espacio liberado y queda completa sobre la membrana.
- Method elimina offsets X en portrait: las tres placas comparten eje central y las parejas se leen como composiciones, no como elementos desordenados.
- Cierre estricto por sección: Cloud, Method y Work tienen compuertas mutuamente excluyentes; ninguna placa anterior puede quedar fantasma arriba del grupo activo.
- Causa raíz del doble espiral: `gl_FragCoord` del framebuffer final se dividía por la resolución reducida del backdrop (55%), desplazando la UV. El shader ahora separa `uViewportResolution` de `uResolution`; el eje exterior y su muestra dentro del vidrio quedan alineados.
- Mobile elimina desplazamiento lateral de refracción y recupera profundidad con fondo 1,62×, fresnel 0,32, tinte 0,16, partículas 18% mayores y naranja HDR reforzado. No agrega postprocesado ni pases extra.
- Ambas piezas del grupo activo son protagonistas a opacidad base completa; el espiral se ve a través del vidrio, pero el texto ya no hereda la atenuación de una placa vecina.

Evidencia Playwright 375×812: Method X `[0,0,0]`; Work final muestra exclusivamente casos `[2,5]`, ambos opacity `0.6`; segunda dupla completa; espiral refractado alineado; overflow 0 y consola limpia.

## SCROLL MOBILE CONTINUO Y VIDRIOS VACÍOS ELIMINADOS — 2026-07-11

Foto de dispositivo real de Agus: aunque no existía CSS scroll-snap, la cámara seguía saltando entre centros discretos y las composiciones parecían forzadas a encajar en el fold. Además aparecían placas de vidrio vacías arriba/abajo.

- Los centros mobile dejan de ser destinos discretos. `mobileScene()` interpola continuamente con smoothstep entre referencias; el scroll nativo controla una trayectoria orgánica sin encastre.
- Durante el paso pueden convivir una, dos o tres piezas con opacidades de proximidad. No existe una cantidad obligatoria por pantalla.
- La causa de las placas vacías era un estado cruzado: el texto/material quedaba en opacity 0, pero el `uHover` automático de touch volvía a subir `diffuseColor.a` del vidrio descartado.
- Cada objeto mobile registra `mobileActive`; los descartados reciben `uHover=0` inmediatamente y el loop touch no puede reactivarlos.
- Work conserva solamente los cuatro casos curados; los índices descartados `[3,4]` mantienen material, texto y hover en cero durante todo el recorrido.
- El rango de proximidad Work baja a 5,8 unidades y suma una ventana smoothstep 0,25→0,65: los extremos entran/salen progresivamente y nunca quedan más de tres placas por encima de opacity 0,1.

Evidencia Playwright 375×812 en posiciones intermedias 37%/63% de Work: `camLook` recorre valores no discretos; casos descartados opacity `[3,4]=0`, hover `[3,4]=0`, `mobileActive=false`; al 37% sólo `[0,1,2]` superan opacity 0,1 mientras `$1M+` sale y `5.3X` empieza a entrar; overflow 0 y consola limpia.

## TRAYECTORIA MOBILE ÚNICA + FROST Y PARTÍCULAS AMPLIFICADOS — 2026-07-12

La interpolación interna ya era continua, pero seguían existiendo dos discontinuidades estructurales: Signal, Method y Work calculaban la cámara en rangos DOM separados. Al cruzar sus límites el destino saltaba 8–9 unidades, por lo que un gesto normal parecía omitir contenido aunque cada tramo aislado fuera suave.

- Mobile usa una sola trayectoria `MOBILE_JOURNEY_SCENES` desde el hero hasta el último caso. Los centros son referencias de composición unidas por smoothstep, no snaps ni destinos obligatorios.
- Se eliminaron las compuertas binarias entre Cloud, Method y Work. La convivencia y salida de objetos depende exclusivamente de distancia real a la cámara; los dos casos descartados siguen bloqueados con opacity y hover en cero.
- El frost mobile pasa de una muestra a cinco, con radio base 0,018 y fresnel adicional 0,022. Recupera desenfoque material sin reintroducir postprocesado.
- El tamaño de partículas low-tier sube de 1,18× a 2,25×. Cantidad, DPR y render directo permanecen iguales para no alterar el presupuesto de GPU.

Evidencia local Playwright 375×812: alrededor de ambos límites anteriores, muestras cada 2 px avanzan monotónicamente y sin salto (Signal→Method: -14,45 a -14,87; Method→Work: -23,38 a -23,88). Con CPU throttle 4× y frost de cinco muestras: p50 6,9 ms, p95 7 ms, máximo 7,1 ms, 0 frames >20 ms, overflow 0 y consola limpia. Desktop 1440×900 conserva tier high, 9.000 partículas, bloom 0,45 y posiciones originales.

## COLCHÓN DE LECTURA ANTES DE LA MEMBRANA — 2026-07-12

Captura real de Agus en desktop: el último par de Work llegaba completo demasiado cerca de Case context y la cresta ondulada tapaba el final del cuerpo mientras todavía se leía.

- La pose final se adelanta de 0,90 a 1,30 viewports antes del límite físico de Work.
- La fase de disolución termina a 0,82 viewports del límite; la membrana recibe únicamente el eje de partículas, no placas con copy activo.
- No se modifica la altura de las placas, la onda ni el recorrido mobile.

Evidencia Playwright 2048×975: en la pose legible, Case context comienza en y=1266 px —291 px debajo del viewport— y ambas placas entran completas. Al finalizar la salida, Case context está en y=798 px y las dos placas ya tienen opacity 0. Overflow horizontal 0. Mobile 375×812 conserva el tier low, el mismo recorrido y overflow 0.

## COLCHÓN MOBILE PARA EL ÚLTIMO CASO — 2026-07-12

Captura de dispositivo real: la membrana mobile todavía alcanzaba el cuerpo inferior de `5.3X` antes de que la placa terminara su lectura.

- Work suma 24svh de padding exclusivamente después de la última composición.
- Los límites internos se compensan en esos mismos 24svh: las placas aparecen en exactamente el mismo punto y con la misma velocidad; sólo Case context se desplaza físicamente hacia abajo.
- Al finalizar el colchón, las placas ya están disueltas y la onda conserva el eje de partículas como transición.

Evidencia Playwright 375×812: en la pose final, Case context comienza en y=746,6 px y las placas `7X/5.3X` entran completas a opacity 0,6. Al terminar la salida, Case context está en y=502,6 px y las seis placas tienen opacity 0. Overflow 0. Desktop conserva su padding de 14svh, tier high y posiciones originales.

## BLOOM SELECTIVO PARA RECUPERAR LEGIBILIDAD — 2026-07-13

Antes de la entrega final, Agus señaló que el glow de las placas lavaba el contenido. La comparación con y sin postprocesado confirmó que el frost no era la causa principal: el `UnrealBloomPass` usaba threshold `0.30` y capturaba también la tipografía crema, generando un halo alrededor de cada letra.

- Iteración 1: bloom high `0.45→0.32`, mid `0.38→0.28`, threshold `0.82` y radius `0.34`. Mejoró los bordes, pero todavía había neblina perceptible en el cuerpo.
- Iteración 2 final: threshold `0.96` y radius `0.28`. La tipografía y el vidrio normal quedan debajo del umbral; únicamente las partículas HDR y los reflejos extremos florecen.
- Mobile pequeño conserva render directo sin bloom y no cambia visualmente.

Evidencia Playwright: desktop 1440×900 muestra títulos, labels y cuerpo con bordes equivalentes a la referencia `nobloom`, mientras el eje conserva brillo cálido. Mobile 375×812 mantiene la composición completa, overflow 0 y consola sin errores. Con CPU throttle 4×: p95 7,1 ms, 2 frames aislados >20 ms en 120 muestras y máximo 27,8 ms.

## FROST TRANSMISIVO Y EJE CON PRESENCIA — 2026-07-13

La reducción de bloom recuperó el borde de la tipografía, pero dejó las placas demasiado opacas y el eje casi apagado durante la lectura. La causa era tratar tres fenómenos distintos con un único dial: bloom de postproceso, transmisión del vidrio y luminancia propia del embudo.

- La tipografía conserva material sólido y el bloom selectivo anterior; no vuelve a depender del fondo.
- La opacidad base del vidrio baja a `0.38` en mobile y `0.44` en desktop. El peso del matcap también baja, por lo que el fondo real domina la superficie en lugar del tinte marrón.
- El frost amplía el radio de sus cinco muestras y refuerza el fresnel. El espiral se ve desenfocado dentro de la placa y nítido fuera de ella: evidencia perceptual de vidrio, no una tarjeta transparente plana.
- El tinte permanente baja y el borde aumenta levemente. La placa conserva volumen sin convertirse en un bloque de color.
- Partículas e hilo mantienen un piso luminoso durante la lectura. Mobile suma un núcleo HDR local porque usa render directo; desktop conserva bloom sólo para los picos de partículas.
- El tubo del eje pasa de `0.014` a `0.016`: presencia continua sin competir con los títulos.

Evidencia Playwright: desktop 1440×900 y mobile 375×812 muestran transmisión, blur y eje cálido continuo; consola sin errores y sin overflow positivo. En mobile, 120 frames medidos: p50 6,9 ms, p95 7,1 ms, máximo 7,1 ms y 0 frames por encima de 20 ms.
