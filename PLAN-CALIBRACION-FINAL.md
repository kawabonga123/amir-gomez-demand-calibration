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
- **Fase 5 — P1-F (nav)**: **diferida al recorrido visual de Cierre** (Fable, 2026-07-09) — es explícitamente condicional ("solo si tras la Fase 2 sigue molestando"), y el harness de scroll sintético (scrollTo + dispatchEvent de 'scroll' + forzar convergencia de cámara) no dio lecturas confiables en este entorno (mismo gotcha ya documentado: rAF se congela en tab de fondo, la convergencia de `updateFromScroll` no se refleja de forma sincrónica y predecible vía eval). Se decide directamente con captura real de scroll en el Cierre en vez de seguir un diagnóstico sintético que no converge — con P0-A ya aplicado (placas no enfocadas al 8% de opacidad), la hipótesis es que el solape con el nav ya es mínimo.
- **Fase 6 — P2-G (copy)**: PREGUNTAR a Agus antes de aplicar (dio "sin cambiar" la última vez; confirmar el OK).
- **Cierre**: recorrido completo desktop (1440×900 y 1568×783) + mobile (375×812) con capturas de CADA sección; checkpoint en esta misma carpeta (actualizar este archivo marcando qué se hizo); commit+deploy final verificado.
  - **BLOQUEO DE INFRAESTRUCTURA (2026-07-09, sin resolver)**: tras el commit de la Fase 4 (72690fe), el deploy a GitHub Pages quedó en un estado roto que NO se resuelve con reintentos — confirmado con el Deployments API (`gh api repos/.../deployments`): el deployment de 72690fe llegó a estado `"failure"` (no solo lento), y un build legacy forzado a mano (`POST pages/builds`) quedó en `"building"` sin ningún avance por 20+ minutos. Los reintentos (2 pushes seguidos, 1 build forzado, 1 intento de cancelar) generaron varios mails de "Run failed" a Agus — se PARÓ de reintentar a pedido explícito de Agus ("me llegan constantes mails de fail... no te distraigas"). El código de las Fases 1-4 está commiteado y pusheado a ambos remotos (`personal`/kawabonga123 y `AgusLaboral`), pero la ÚLTIMA versión en vivo en `https://kawabonga123.github.io/amir-gomez-demand-calibration/` corresponde a un commit ANTERIOR a esta ronda (verificar con `curl ... | grep HERO_CAM_Y` — si no aparece, sigue sin actualizarse). **Próximo paso recomendado**: revisar a mano en GitHub (Settings→Pages del repo, o la pestaña Actions) en vez de más reintentos automáticos — puede necesitar desactivar/reactivar Pages o simplemente más tiempo real para que el backend de GitHub libere el lock.
  - **Evidencia visual de esta vuelta**: capturas reales obtenidas pese a herramientas degradadas (preview MCP: timeout persistente en `preview_screenshot` durante toda la sesión, incluso tras reiniciar el server 3 veces; claude-in-chrome: ventana de automatización fija en 640×305, sin poder redimensionar) — 1 captura real de una placa enfocada de Method ("SHIP & MEASURE", legible, sola en cuadro — confirma P0-A+P0-B visualmente) + 1 captura real del Hero en scroll=0 (carga limpia, sin errores). El resto de la evidencia de Fases 3-4 es numérica exacta (getBoundingClientRect, frustum, bounding boxes) en los 4 viewports exactos que pide el plan — más precisa que una captura para esos casos puntuales, pero no reemplaza el recorrido visual completo pedido. **Pendiente real**: recorrido de capturas reales de las 4 secciones restantes (Signal, Work, Team, Contact) en los 4 viewports, apenas la herramienta de captura se estabilice.

## DEFINICIÓN DE TERMINADO (todas juntas)

1. En cualquier punto del scroll se lee UN objeto protagonista; los demás no compiten (P0-A).
2. TODO título/número/texto se lee de un vistazo en cualquier momento (P0-B) — desktop y mobile.
3. Hero sin ningún solape a 1440×900, 1568×783, 375×812, 375×667 (P0-C).
4. Placas con ritmo interno consistente y pares alineados (P1-D); captions de diales inequívocos (P1-E).
5. Cero errores de consola en el recorrido completo.
6. Deployado en `https://kawabonga123.github.io/amir-gomez-demand-calibration/` y verificado EN VIVO con captura.
7. Evidencia visual (capturas) de cada punto anterior en el reporte final.
