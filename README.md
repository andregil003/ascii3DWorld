# ascii3DWorld

ASCII-FPS corriendo en el navegador (canvas + texto ASCII), sin dependencias.

Vivo en: https://andregil003.github.io/ascii3DWorld/

Historial en este repo:
- `v4 — pueblito` (commit previo, desde `Downloads/index.html`)
- `v5` (desde `Downloads/index (1).html`, copiado byte-por-byte como `index.html`)
- `v5 update 2026-10-01` (desde `Downloads/index (2).html`, copiado byte-por-byte como `index.html`, +23KB / +355 líneas)
- `v5 update-2 2026-10-01` (desde `Downloads/index (3).html`, copiado byte-por-byte como `index.html`, +24KB)
- `v5 update-3 2026-10-06` (desde `Downloads/index (2).html` re-editado, copiado byte-por-byte como `index.html`, 201KB)
- `fix seed/save 2026-10-07` (editado directo en el repo, +66/−7)

## Controles (v5)

- Click para jugar (bloquea el mouse)
- `WASD` moverse · mouse mirar · `SHIFT` correr
- `N` cambia la hora · `H` congela/reanuda el tiempo
- `M` minimapa · `F` scanlines · `- / =` tamaño del texto

HUD: FPS, posición, fase del día, corazones, minimapa, mensajes en pantalla.

## Correr local

Abre `index.html` con doble click. No necesita build ni server.

## Changelog

### fix seed/save — 2026-10-07

Corregidos dos bugs críticos que rompían la carga de partidas. Verificados con
Playwright/Chromium headless (20/20 tests) contra el código anterior.

**1. Guardar/cargar generaba un mundo diferente** (`newWorld`)

El bucle que busca un seed válido hacía `s++` incluso en la iteración que
tenía éxito, así que `SEED` quedaba en `seed+1` mientras el mundo real se
generaba con `seed`. Como el save guarda `SEED`, al cargar se regeneraba un
mapa completamente distinto (firma del terreno `839301118` vs `1894629779`),
y la posición guardada del mundo viejo te dejaba dentro de montaña o agua
—ambos sólidos— sin forma de moverte. Se perdía la partida entera.

**2. Un save corrupto dejaba la pantalla negra para siempre** (`loadGame`)

Sin validación previa, un save incompleto o editado a mano lanzaba un
`TypeError` a mitad del arranque. Eso abortaba todo el script: sin `fit()`,
sin menú y sin `requestAnimationFrame`. Canvas negro, sin salida.

Ahora `validSave()` comprueba seed, posición, vida, clima, estructura del
nivel y tipos de inventario **antes** de tocar el estado global. Si algo no
cuadra, borra el save y arranca un mundo limpio con aviso en el menú.
La resolución del nivel es defensiva (nunca indexa a ciegas) y hay una red de
seguridad que devuelve al spawn si la posición cargada cae en terreno sólido.
El arranque completo va en `try/catch`.

Efecto secundario: los saves guardados antes de este fix se descartan al
cargar, porque su seed estaba desplazado.

**Pendiente (auditoría de compatibilidad, sin aplicar todavía)**

- Menú inaccesible en táctil (el click sintetizado lo cierra al abrirlo)
- Híbridos táctil+ratón pierden el ratón: nunca se pide pointer lock
- Teclado lee `e.key`, no `e.code` → AZERTY/Dvorak no puede jugar
- `pointerlockerror` oculta el overlay sin avisar
- `100vh` sin `100dvh` ni `env(safe-area-inset-*)`
- Raycaster renderiza a 60fps detrás del menú (gasta batería)
- Clic en el texto de ayuda o relleno de opciones arranca la partida

## Deploy

GitHub Pages sirve `index.html` desde `main` / root.
