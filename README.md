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
- `fix controles/móvil 2026-10-07` (editado directo en el repo, +52/−7)

## Controles (v5)

- Click para jugar (bloquea el mouse)
- `WASD` moverse · mouse mirar · `SHIFT` correr
- `N` cambia la hora · `H` congela/reanuda el tiempo
- `M` minimapa · `F` scanlines · `- / =` tamaño del texto

HUD: FPS, posición, fase del día, corazones, minimapa, mensajes en pantalla.

## Correr local

Abre `index.html` con doble click. No necesita build ni server.

## Changelog

### fix controles y móvil — 2026-10-07

Cinco correcciones de uso, verificadas con Playwright/Chromium comparando el
comportamiento antes y después (29/29 tests).

**1. En móvil no se podía volver al menú** (`tMenu`)

El navegador despacha un `click` sintetizado tras el `pointerup`. Ese click
llegaba a `#overlay` —ya visible por debajo de `#touch`— y ejecutaba `play()`,
que cerraba el menú en el mismo gesto. Como `preventDefault` no lo evita
(medido), el resultado era que **en el móvil no había forma de volver al menú**:
ni el botón, ni `Esc` (exige `!touchPlaying`), ni el botón atrás del sistema.
Solo muriendo o recargando.

Ahora `tMenu` marca una ventana de supresión de 450 ms que `overlay:click`
respeta. Traza real antes/después:

```
ANTES  pointerdown → menuOpen=true  → click → menuOpen=false (BUG)
DESPUÉS pointerdown → menuOpen=true  → click → menuOpen=true  (OK)
```

**2. Los híbridos táctil+ratón perdían el ratón** (`isTouch`, `play`)

`isTouch()` daba `true` con cualquier capacidad táctil, así que en una Surface,
un iPad con trackpad o un Chromebook, `play()` marcaba `touchPlaying` y **nunca
pedía pointer lock**. Mirar solo funcionaba con el botón pulsado, el click
izquierdo no atacaba, y el `lookpad` (58% derecho) interceptaba el ratón.

Ahora `hasFine()` comprueba `(pointer:fine)` y, si hay ratón, manda el ratón:
no se monta el HUD táctil y sí se pide pointer lock. Verificado: `locked=true`,
`pointerLockElement=#screen`, click izquierdo ataca, teclado sigue andando.

**3. AZERTY y Dvorak no podían jugar** (`keydown`/`keyup`)

El juego leía `e.key.toLowerCase()`. En AZERTY la tecla física W produce
`key="z"`, así que el juego registraba `keys={z:true}` y `inputVec()` devolvía
`[0,0]`: **imposible avanzar**. Igual con `Digit1` → `key="&"`.

Ahora se mapea por `e.code` (`KeyW`, `Digit1`…) con fallback a `e.key` para
teclados que no reporten código. Antes/después:

```
ANTES     W físico (key=z) → keys.w=false  keys.z=true   inputVec=[0,0]
DESPUÉS   W físico (key=z) → keys.w=true   keys.z=false  inputVec=[1,0]
```

**4. Clic en el texto de ayuda arrancaba la partida** (`overlay:click`)

`#opts` y `#help` son `div`, no controles. El handler arrancaba la partida
cuando no había un `button/input/label/select` bajo el cursor, así que pulsar
para leer los controles te metía al juego. Ahora se excluyen `#opts`, `#help`,
`#ovmsg` y `#menu`; el fondo del overlay sigue arrancando como antes.

**5. El raycaster renderizaba a 60fps detrás del menú** (`loop`)

Con el overlay visible, el bucle completo (`cast()` por columna, sprites,
minimapa) corría invisible tras `rgba(0,0,0,.84)`. Quemaba batería en portátil
y competía con el rAF de la UI. Ahora el menú refresca a 4fps: el fondo sigue
vivo, con una fracción del trabajo. En juego se renderiza en cada frame como
antes (verificado: `renders=17` para `framesRAF=17`).

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

- `pointerlockerror` oculta el overlay sin avisar
- `100vh` sin `100dvh` ni `env(safe-area-inset-*)` (notch, barra de direcciones)
- `#toast` desborda en pantallas angostas y solapa con el inventario
- Menú recortado sin scroll en pantallas de 500px de alto o menos
- `#mini` no aplica DPR: se ve borroso en pantallas retina
- Sin `visibilitychange`: al cambiar de app el reloj del mundo sigue corriendo
- `prefers-reduced-motion` se ignora
- `pointerlockerror` deja el juego sin control de cámara sin avisar al jugador

## Deploy

GitHub Pages sirve `index.html` desde `main` / root.
