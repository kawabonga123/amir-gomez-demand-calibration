# amir-site — A+ Growth (Amir Gómez)

Sitio de Amir Gómez / marca "A+ Growth" (consultor de marketing de adquisición). Rehecho de cero con Innovatron (vanilla Three.js, single-file) tras patinar con una versión Next/R3F anterior.

## Dónde vive todo

- **Código**: `index.html` (único archivo — HTML+CSS+JS, Three.js r160 vía import map de CDN). No hay build step, no hay `npm install`.
- **Assets**: `assets/` (fotos del equipo, tratadas con duotono cálido).
- **Preview local**: `.claude/launch.json` ya configurado — `http-server` en el puerto 8137.
- **Docs de rondas de trabajo** (histórico, más detalle que este archivo):
  - `PLAN-POBLADO.md` — población inicial con contenido real del deck comercial.
  - `PLAN-REDISENO-V2.md` — rediseño F1-F6 tras la auditoría /taste (frostGlass, diales de calibración, poses de cámara).
  - `PLAN-CALIBRACION-FINAL.md` — auditoría y plan más reciente (2026-07-09): fixes de legibilidad/superposición/hero. **Leer este último antes de tocar el sitio** — tiene las reglas duras, los gotchas de tooling, y el estado fase-por-fase más actualizado.

## Deploy — 2 repos GitHub, 2 credenciales distintas

- **Repo que sirve el sitio en vivo**: `github.com/kawabonga123/amir-gomez-demand-calibration` (remote git `personal`) → `https://kawabonga123.github.io/amir-gomez-demand-calibration/`. Push normal (`git push personal main`), usa la sesión default de `gh`/git en esta PC.
- **Repo espejo/backup**: `github.com/AgusLaboral/amir-gomez-demand-calibration` (remote git `origin`) — privado. Necesita el token de la cuenta AgusLaboral embebido en la URL (la sesión default de git NO tiene permiso ahí):
  ```bash
  GH_TOKEN=$(powershell.exe -NoProfile -Command "& 'C:\Users\Agus\.claude\skills\github-backup\scripts\Get-AgusLaboralToken.ps1'" | tr -d '\r')
  git push "https://oauth2:${GH_TOKEN}@github.com/AgusLaboral/amir-gomez-demand-calibration.git" main
  ```
- **Siempre pushear a los DOS** tras cada commit relevante (mantiene ambos en sync; el primero es el que Agus comparte con Amir).
- **Verificar en vivo** con `curl` grepeando un string nuevo del commit — Pages tarda ~45-90s en propagar normalmente.
- ⚠️ **Gotcha real (2026-07-09)**: pushear 2 commits muy rápido seguidos (sin esperar que el deploy anterior termine) puede dejar el deployment de GitHub Pages atascado ("in progress deployment... please cancel <sha> first"), bloqueando TODOS los pushes siguientes indefinidamente. Si pasa: esperar unos minutos (suele resolverse solo) o forzar `gh api -X POST repos/<owner>/<repo>/pages/builds`. Prevención: esperar la verificación en vivo del deploy anterior antes de pushear el siguiente.

## Arquitectura del sitio (para no repetir bugs ya resueltos)

- **Un solo objeto-eje 3D** recorre todo el sitio: Hero → Signal (nube de diales) → Method (3 placas) → Work (6 casos, 3 pares) → Team/Services/Contact (DOM puro, sin 3D). La cámara viaja a lo largo de este eje según el scroll (`updateFromScroll()`).
- **Todo texto que acompaña un objeto 3D es `TextGeometry` HIJA de ese objeto** (nunca DOM dockeado por proyección de cámara — esa arquitectura se probó y se abandonó por generar bugs de desincronización repetidos). Ver regla dura de memoria del usuario `[[feedback-texto-siempre-contenido]]`.
- **El texto de título/hook/número usa material SÓLIDO** (`MeshBasicMaterial`, sin matcap/frostGlass) — su contraste NUNCA debe depender de brillo de partículas, bloom, hover o ángulo de cámara. Ver `PLAN-CALIBRACION-FINAL.md` fase P0-B.
- **Focus por proximidad**: cada placa/dial se atenúa según qué tan lejos está del punto de la cámara (`registerFadeGroups()`/`applyFocusFade()`) — evita que se vean 4-6 placas superpuestas a la vez ("túnel de placas"). Ver fase P0-A.
- **`frostGlass()`** es el material de vidrio compartido (matcap + fresnel + refracción de backdrop) — cualquier material nuevo que necesite verse "detrás del vidrio" debe empujarse al array `plateMaterials` para que `renderBackdrop()` lo actualice cada frame, si no `tBackdrop` queda null y el material se ve negro/apagado (bug real, ya pasó 2 veces).

## Reglas duras (no repetir bugs ya resueltos — ver memoria del proyecto para el detalle completo)

1. Texto siempre legible, sin excepción — nunca depende de fondo/brillo variable.
2. Ningún texto flota suelto sin ser geometría 3D hija de su objeto.
3. Nunca cursor custom pegado al puntero.
4. Mobile es compuerta pass/fail — todo fix se verifica también en 375×812.
5. Nada se reporta "listo" sin evidencia (captura real o medición numérica vía `?debug` + `window.__amirDebug`).

## Gotchas de tooling (verificación)

- **Screenshots de la escena WebGL son flaky** (preview MCP y navegadores automatizados) — pueden salir negros o colgarse. El modo `?shot` (render síncrono, sin rAF) es lo más confiable: `?shot`, `?shot&stage=0..2`, `?shot&case=0..5`, `?shot&cloud=0..3`.
- **Cache**: navegar siempre con un query param nuevo (`?v=x`) tras editar, para no verificar contra una versión vieja servida por caché.
- **GLSL en template strings**: todo número interpolado necesita `.toFixed(3)` — un entero sin punto (`1` en vez de `1.0`) aborta el shader ENTERO en silencio (sin error de consola).
- **TextGeometry**: el parámetro de profundidad se llama `height`, NO `depth`. Centrar verticalmente con `-(min+max)/2`, NUNCA con `-h/2` (solo funciona si `boundingBox.min.y===0`, casi nunca cierto — bug real encontrado y corregido 2026-07-09).
- Detalle completo de más gotchas (materiales compartidos, `requestAnimationFrame` congelado en tabs de fondo, etc.) en `PLAN-CALIBRACION-FINAL.md`.

## Cómo retomar

1. Leer `PLAN-CALIBRACION-FINAL.md` completo (regla dura + gotchas + estado fase-por-fase) antes de tocar código.
2. Si el plan tiene fases pendientes, seguir en orden — no saltear verificación visual.
3. Verificar SIEMPRE en `https://kawabonga123.github.io/amir-gomez-demand-calibration/` tras cada cambio (no asumir que el push alcanza — confirmar con `curl`).
