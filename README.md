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
- `fix robustez 2026-10-07` (editado directo en el repo, +114/−18)
- `fix hexCache 2026-10-07` (editado directo en el repo, +12/−1)

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

### fix hexCache — 2026-10-07

`hex()` se llama unas 200 veces por frame (una por columna), así que la cache está
justificada. El problema era que no tenía tope: los colores del cielo y la niebla se
recalculan con valores flotantes cada frame (`pack(190 * vis, 210 * vis, ...)`) y el
enmascarado `&252` deja 64 niveles por canal, o sea hasta **64³ = 262.144 claves
distintas** a lo largo de una partida. Eso son ~26 MB de basura acumulada sin límite.

Ahora el `Map` se vacía al superar 4096 entradas. Como la cache es puramente
aceleradora (reformatear un entero a hex es barato), perderla no afecta nada. En una
simulación de 300 frames de cielo más 1.000 colores distintos la cache se queda en
535 entradas (~52 KB) en vez de crecer sin control.

**Sin QA en navegador** (revisión de código y simulación de la lógica en node).

### fix robustez y móvil — 2026-10-07

Nueve correcciones de robustez, layout y limpieza. Verificadas por revisión de
código (`node --check` sobre el JS extraído y balance de llaves en los CSS);
**sinQA en navegador**, pendiente de prueba manual.

**Layout / móvil**

1. `100dvh` + `env(safe-area-inset-*)`. En móvil `100vh` mide el viewport con la
   barra de direcciones visible, así que al ocultarse el canvas quedaba
   desalineado. Los botones caían bajo la muesca del iPhone; ahora el inset se
   resta como padding en `#touch` y con `max()` en `#inv`/`#toast`.
2. `#toast` desbordaba. Con `white-space:nowrap` a 320px de ancho medía 692px y se
   salía por ambos lados, solapando el inventario. Ahora envuelve, con `max-width:
   min(92vw, 620px)` y tope de altura.
3. Menú recortado en pantallas bajas. `#overlay` ahora tiene `overflow-y: auto` y
   un bloque `@media (max-height: 560px)` que aprieta tipografía, menú y opciones
   para que todo quepa en landscape (844×390) sin depender del scroll.
4. Minimapa sin DPR. El buffer era 300×150 mostrado a 190px (ratio 0.63): borroso en
   cualquier retina. Ahora el buffer se escala por `devicePixelRatio` (tope 2) y el
   contexto trabaja en coordenadas lógicas con `setTransform`, así que el resto del
   dibujo no cambia.
5. `prefers-reduced-motion: reduce` respetado. Antes se ignoraba por completo;
   ahora desactiva transiciones y animaciones.

**Comportamiento**

6. `visibilitychange`. Al ocultar la pestaña o cambiar de app el bucle seguía con el
   `dt` acumulado y el reloj del mundo (`T`, `dayT`) avanzaba: al volver, la noche
   había pasado. Ahora abre el menú avisando que el tiempo se detuvo, libera el
   puntero y limpia las teclas. También se escucha `visualViewport` para el caso en
   que la barra de direcciones se oculta sin disparar `resize`.
7. `pointerlockerror` sin aviso. Si el navegador rechazaba el bloqueo (iframe sin
   `allow`, cooldown tras Esc, kiosco) el overlay se ocultaba sin decir nada y el
   juego se quedaba sin control de cámara. Ahora avisa que se puede arrastrar con
   el botón pulsado.

**Lógica / limpieza**

8. Proyectiles congelados y markers huérfanos. `goto()` no limpiaba `LV.prj`, así
   que las flechas en vuelo quedaban pegadas en el nivel abandonado para siempre.
   `detachFollowers()` tampoco retiraba el `marker` del peón, y como
   `updateMarkers()` solo actualiza los que están en `LV.act`, se acumulaban sprites
   invisibles (~4 por transición). El marker se recrea al volver. También se respetó
   el modo individual de cada peón, que `p.mode = pawnCmdMode` pisaba al guardar.
9. Opciones corruptas. Un `localStorage` editado a mano con `fov=0` daba `TAN=0` →
   `PK=Infinity` → NaN en pantalla sin error visible. Ahora los valores se validan y
   se clampean contra los `min/max` reales de los inputs, con defaults por tipo. Se
   reinforcement `resetPlayer()` con `regen`, `hurtT`, `hitDone`, `lastHit`,
   `lastHitT` y limpieza de teclas/estado táctil. `swordHit()` ahora comprueba
   línea de visión (`los()`) para no golpear a través de paredes. Y se eliminó el
   código muerto: `doors`, `MSC`, dos `if` vacíos, y el `new Date()` por frame del
   reloj de sol (ahora se recalcula solo al cambiar de minuto).

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

**Pendiente**

- Verificar el lote robustez en navegador real (móvil landscape, portrait, retina).
  Este lote se validó por revisión de código, no con tests automatizados.
- `hexCache` (`Map` sin tope) es un riesgo teórico de memoria a largo plazo.
- Sin audio: el juego no tiene sonidos de ataque ni de clima.

## Deploy

GitHub Pages sirve `index.html` desde `main` / root.
