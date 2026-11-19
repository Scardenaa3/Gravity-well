# Gravity Well

Videojuego de navegación espacial en **Jack** sobre la arquitectura **Hack** (Nand2Tetris).
Examen Parcial 4 — Organización de Computadores — EAFIT 2026.

**Video (funcionamiento y arquitectura):** _pegar aquí el enlace de YouTube_

## Equipo
- Santiago Cárdenas Angarita
- Jaime Jesús Cotez Nuñez

## Concepto
Una nave vuela entre planetas que la atraen con su gravedad. No se puede frenar ni apuntar libremente:
hay que **aprovechar la atracción de los planetas para curvar la trayectoria**, recoger cápsulas de energía
y evitar chocar. La dificultad sube con el puntaje (más velocidad de desplazamiento y planetas más juntos).

**Diferenciador:** la mecánica central es la atracción gravitacional entre cuerpos, no el rebote simple
(Pong, Breakout), el movimiento en rejilla (Snake) ni el salto de trayectoria fija.

## Jugador y récord
- Antes de cada partida se pide el **nombre del jugador** (máximo 8 letras o dígitos, se convierten a mayúsculas).
  ENTER sin escribir nada conserva el nombre anterior.
- El marcador muestra jugador, puntaje, cápsulas, nivel, impulso y el **récord** vigente.
- Si el jugador supera el récord, aparece **NEW RECORD!** en vivo en el marcador y, al terminar, en la pantalla final.
- La pantalla de título y la final muestran quién tiene el récord.
- El récord vive en RAM: dura mientras el programa esté cargado. Hack no tiene archivos, así que al recargar el emulador se reinicia.

## Controles
| Tecla | Acción |
|---|---|
| ← → ↑ ↓ | Empuje en esa dirección |
| Espacio | Impulso frontal (recarga de 45 frames; el HUD muestra `BOOST OK` / `BOOST --`) |
| ESC | Salir (en la pantalla de título) |

## Cómo compilar y ejecutar
1. Abrir la carpeta `GravityWell` con el **JackCompiler** de Nand2Tetris (genera un `.vm` por clase).
2. Abrir el **VMEmulator**, cargar la carpeta `GravityWell` (File → Load Program).
3. Poner *Animate* en **No animation** y la velocidad al máximo; hacer clic en la pantalla para darle foco al teclado.
4. Ejecutar (Run).

## Requisitos del examen → dónde se cumplen
| Requisito | Implementación |
|---|---|
| Interactividad por teclado sin bloqueo | `Ship.applyInput` usa `Keyboard.keyPressed()` (devuelve de inmediato); nunca `readChar()` |
| Fondo móvil / entorno dinámico | `Starfield` (2 capas con paralaje) + planetas y cápsulas en scroll continuo |
| Comportamiento físico modelado | `Ship.pullFrom` (gravedad 1/r, aceleración, inercia) y `Ship.move` (integración y rebote con pérdida de energía) |
| Puntaje visible y dinámico | `Game.drawHud`: SCORE, CAPS, LEVEL y BOOST, actualizado solo cuando cambia |

## Arquitectura
```
Main ──► Game ──┬──► Ship         física de la nave, entrada, rebotes
                ├──► Planet [3]   masa, tabla de filas, dibujo incremental
                ├──► Capsule [2]  coleccionables
                ├──► Starfield    fondo de 2 capas
                ├──► Config       parámetros (balance del juego)
                └──► Rng          generador pseudoaleatorio
```

Ciclo de cada frame (`Game.step`):
```
teclado (no bloqueante) → planetas (scroll, gravedad, colisión) → estrellas
   → cápsulas (scroll, recogida) → nave (integrar, rebotar, dibujar) → puntaje/nivel/HUD → Sys.wait
```

## Física
- **Punto fijo:** Jack no tiene decimales. Posición y velocidad se guardan en 1/16 de píxel (`pixel = p / 16`).
- **Gravedad:** en un universo 2D la fuerza cae como 1/r: `a = m / d`, con componentes
  `ax = m·dx / d²` y `ay = m·dy / d²` (sin raíz cuadrada).
- **Alcance limitado (100 px):** garantiza `dx² + dy² ≤ 20000 < 32767`, así que nunca hay desbordamiento de 16 bits.
- **Integración:** `v += a`, se limita `|v| ≤ 5 px/frame`, y luego `p += v`.
- **Rebote:** en los 4 bordes se conserva 3/4 de la velocidad.

## Decisiones de bajo nivel (memoria de video y ciclos)
- `Screen.drawPixel/drawRectangle` escriben directamente en el mapa de memoria de pantalla (RAM[16384..24575], 256 filas × 32 palabras).
- **Dibujo incremental de planetas:** al desplazarse se borra solo la franja derecha y se pinta la izquierda de cada fila, en lugar de redibujar el círculo. Sin parpadeo y con menos instrucciones.
- **Tabla precalculada** con el semiancho de cada fila del planeta (una sola vez en `Planet.init`).
- **Recorte horizontal propio:** `Screen.drawCircle` produce error si el círculo no cabe entero en pantalla, y los planetas entran y salen por los lados.
- **Textos creados una sola vez:** en Jack cada texto entre comillas que se ejecuta reserva un String nuevo en el heap y nunca se libera (no hay recolector de basura). Todos los textos se crean en `Game.initTexts()` y solo se reutilizan; hacerlo dentro del bucle agotaba el heap (error ERR6).
- La captura del nombre usa `Keyboard.keyPressed()` sobre un String de 8 caracteres reservado una sola vez, en lugar de `Keyboard.readLine()` (que reserva memoria en cada llamada).
- Los objetos se reciclan (no se crean ni destruyen durante la partida), así que no hay fugas de memoria del heap.
- El HUD se redibuja solo cuando cambia algo.
- El generador aleatorio es un congruencial lineal de 15 bits; la semilla sale del tiempo que tarda el jugador en pulsar una tecla.

## Ajustes rápidos (`Config.jack`)
| Función | Efecto |
|---|---|
| `frameMs()` | Pausa por frame. Subirla si el juego va demasiado rápido, bajarla si va lento |
| `numStars()` | Bajar a 20–25 si el emulador va lento |
| `thrust()`, `boost()`, `maxSpeed()` | Manejo de la nave |
| `gravityRange()` | Alcance de la gravedad (máximo seguro: 100) |
| `numPlanets()`, `numCapsules()` | Densidad del escenario |

## Archivos
`Main.jack` · `Game.jack` · `Ship.jack` · `Planet.jack` · `Capsule.jack` · `Starfield.jack` · `Config.jack` · `Rng.jack`
