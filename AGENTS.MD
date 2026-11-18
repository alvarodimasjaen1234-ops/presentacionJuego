# AGENTS.MD — Contrato de calidad: Tres en Raya (Humano vs IA)

Este documento es el contrato de verificación del proyecto. Cualquier cambio en
`tres-en-raya.html` debe validarse contra las secciones de este archivo antes de darse
por terminado.

**Alcance del proyecto**

| Archivo | Descripción |
|---|---|
| `AGENTS.MD` | Guía de verificación, pruebas y auto-corrección (este archivo). |
| `tres-en-raya.html` | Juego completo: HTML5 + CSS3 + JavaScript ES6+ vanilla, sin dependencias ni red. |

**Arquitectura del código (un solo archivo)**

1. `<script id="nucleo-logica">` — núcleo puro y testeable, sin acceso al DOM:
   `LINEAS_GANADORAS`, `crearTablero()`, `obtenerLineaGanadora()`, `tableroCompleto()`,
   `casillasVacias()`, `buscarJugadaGanadora()`, `elegirJugada()` y los motores
   `elegirJugadaFacil()`, `elegirJugadaNormal()`, `elegirJugadaMinimax()` (+ `minimax()`).
   Se expone en `window.TresEnRayaCore` para depuración desde la consola.
2. `<script>` (IIFE) — capa de interfaz: estado de la partida, render del tablero,
   bloqueo de turnos, temporizador de la IA, marcador, sonido y persistencia.
3. Marcado semántico con `aria-live` para el turno/resultado, `role="grid"` en el tablero
   y botones reales (accesibles por teclado) para las 9 casillas.

**Reglas implementadas**

- El humano es siempre `X` e inicia cada ronda. La IA es `O`.
- La IA espera entre 420 ms y 620 ms antes de mover (sensación de "pensamiento").
- Dificultad `Fácil`: aleatoria, remata a veces y nunca bloquea.
- Dificultad `Normal`: remata si puede, bloquea la victoria inmediata del humano y el
  resto de jugadas son aleatorias (permite ganar al humano si construye una horquilla).
- Dificultad `Imbatible`: Minimax completo con poda alfa-beta y desempate aleatorio
  entre jugadas de valor óptimo (nunca pierde).
- El marcador se guarda en `localStorage` (con `try/catch`); "Reiniciar marcador" lo borra.

---

## Criterios de Aceptación y Validación (Checklist)

- [ ] El archivo `AGENTS.MD` existe en la raíz del proyecto.
- [ ] El jugador humano (`X`) puede hacer clic en cualquier celda vacía en su turno.
- [ ] El tablero bloquea cualquier interacción del usuario mientras la IA (`O`) está calculando o ejecutando su turno.
- [ ] El modo "Imbatible" utiliza el algoritmo Minimax correctamente (la IA gana o empata el 100% de las veces).
- [ ] El modo "Normal" permite al jugador humano ganar si juega bien.
- [ ] Se detectan correctamente las 8 combinaciones ganadoras y el estado de empate.
- [ ] El botón de reinicio limpia el tablero y restablece el estado interno sin recargar la página.

---

## Comandos de Test / Ejecución

### Abrir el juego

```bash
# Windows (PowerShell / CMD)
start tres-en-raya.html

# macOS
open tres-en-raya.html

# Linux
xdg-open tres-en-raya.html
```

### Servidor local ligero (opcional, evita restricciones de `file://`)

```bash
npx serve .
# o
python3 -m http.server 8080
# o
python -m http.server 8080
```

Luego abrir `http://localhost:8080/tres-en-raya.html` (o `http://localhost:3000/tres-en-raya.html`
según el servidor).

### Verificación rápida en la consola del navegador

1. Abrir las DevTools (`F12`) → pestaña **Console**.
2. Recargar la página: **no debe aparecer ningún** `Uncaught TypeError`,
   `ReferenceError` ni `null is not an object`.
3. Comprobar que el núcleo lógico está expuesto y es coherente:

```js
// Debe devolver un objeto con las funciones del núcleo
window.TresEnRayaCore;

// Debe devolver [0, 1, 2] (línea ganadora de la fila superior)
TresEnRayaCore.obtenerLineaGanadora(['X','X','X', null, null, null, 'O', 'O', null]);

// Debe devolver null (tablero vacío = sin ganador)
TresEnRayaCore.obtenerLineaGanadora(TresEnRayaCore.crearTablero());

// Debe devolver 2 (la IA remata su propia línea O en 0-1-2)
TresEnRayaCore.elegirJugada(['O','O',null,'X','X',null,null,null,null], 'imbatible', 'O', 'X');

// Debe ser true (hay 8 líneas ganadoras registradas)
TresEnRayaCore.LINEAS_GANADORAS.length === 8;
```

4. Comprobar que no hay referencias nulas al DOM:

```js
['tablero','estado','estado-texto','marcador-x','marcador-empates','marcador-o',
 'btn-nueva','btn-marcador','btn-sonido','dificultad-desc']
  .every(id => document.getElementById(id) !== null); // => true

document.querySelectorAll('.casilla').length; // => 9
```

### Tests automáticos del núcleo lógico (Node.js, opcional)

El núcleo está aislado en `<script id="nucleo-logica">`, por lo que puede extraerse y
probarse sin navegador. Ejemplo mínimo:

```bash
node -e "const fs=require('fs'),vm=require('vm');const html=fs.readFileSync('tres-en-raya.html','utf8');const codigo=html.match(/<script id=\"nucleo-logica\">([\s\S]*?)<\/script>/)[1];const ctx=vm.createContext({Math});vm.runInContext(codigo+';this.api={elegirJugada,obtenerLineaGanadora,crearTablero,LINEAS_GANADORAS};',ctx);console.log('lineas:',ctx.api.LINEAS_GANADORAS.length,'| minimax:',ctx.api.elegirJugada(['O','O',null,'X','X',null,null,null,null],'imbatible','O','X'));"
```

Resultado esperado: `lineas: 8 | minimax: 2`.

---

## Casos Borde y Pruebas de Borde (Edge Cases)

### 1. Doble clic rápido
Hacer doble clic muy rápido (o clic en dos celdas distintas antes de que la IA responda).
- **Esperado:** se registra **una sola `X`**. El turno cambia a la IA de forma síncrona en
  el mismo instante del primer clic y las celdas quedan `disabled` + `data-bloqueado="true"`.
- **Comprobación en consola:** tras el primer clic, `document.querySelectorAll('.casilla:disabled').length === 9`.
- **Implementación:** guardas en `puedeJugarHumano()` (`fase`, `turno`, celda vacía) +
  atributo `disabled` + bandera `estado.turno`, sin depender solo del CSS.

### 2. Clic en celda ocupada
Hacer clic sobre una `X` o una `O` existente.
- **Esperado:** no se sobrescribe la ficha, no se consume el turno, no se emite sonido.
- **Comprobación:** la celda conserva su `data-ficha` y el estado sigue siendo `Tu turno (X)`.

### 3. Clic post-victoria / post-empate
Hacer clic en celdas vacías después de que alguien haya ganado o haya empate.
- **Esperado:** el tablero permanece congelado (`fase = 'fin'`, todas las celdas `disabled`)
  hasta pulsar "Nueva partida". La línea ganadora queda resaltada.

### 4. Cambio de dificultad en mitad de partida
Cambiar el selector mientras hay fichas en el tablero.
- **Esperado:** la nueva dificultad se aplica en el siguiente turno de la IA. Si la IA está
  "pensando" en ese momento, su temporizador se cancela y se reprograma con la nueva lógica
  (nunca se producen dos movimientos de la IA ni se pierde el turno). El tablero no se limpia.

### 5. Reinicio durante el turno de la IA
Pulsar "Nueva partida" mientras la IA está pensando.
- **Esperado:** se cancela el temporizador pendiente (`clearTimeout`) y se invalida con un
  token de ronda (`idRonda`), de modo que la IA no coloca una ficha fantasma tras el reinicio.

### 6. Victoria/empate en la última casilla
La novena jugada completa el tablero y además forma línea.
- **Esperado:** se detecta primero la victoria (se comprueba `obtenerLineaGanadora()` antes
  que `tableroCompleto()`), se resalta la línea y suma el punto al ganador, no empate.

### 7. Sin `localStorage` (modo restringido / `file://`)
El navegador lanza `SecurityError` al acceder a `localStorage`.
- **Esperado:** el juego funciona igual; el marcador vive en memoria. Todo acceso está
  envuelto en `try/catch`.

### 8. Audio bloqueado por el navegador
- **Esperado:** si `AudioContext` no está disponible o está suspendido, la partida continúa
  sin errores. El botón de sonido permite silenciar en cualquier momento.

---

## Protocolo de Auto-Corregido

Antes de dar la tarea por finalizada, revisar:

1. **Funciones completas:** ninguna función declarada sin cuerpo, sin `return` esperado ni
   código muerto. Sin comentarios `// TODO` / `// FIXME` / `// XXX`.
2. **Variables sin usar:** revisar el núcleo y la capa de UI (sin imports ni dependencias
   externas, sin CDNs, sin fuentes remotas).
3. **Casos borde:** ejecutar mentalmente (o en consola) los 8 escenarios de la sección
   anterior; corregir de inmediato cualquier fallo.
4. **Consola limpia:** 0 errores no capturados, 0 advertencias propias, 0 peticiones de red.
5. **Accesibilidad mínima:** navegación con `Tab` + `Enter`/`Espacio`, foco visible,
   `aria-live` en el mensaje de estado, etiquetas `aria-label` en las 9 casillas.
6. **Responsive:** probar a 360 px, 768 px y 1280 px de ancho sin scroll horizontal.
7. **Contraste:** texto legible sobre el fondo oscuro; no depender solo del color para
   distinguir `X` de `O` (la forma de la ficha es diferente).

---

## Registro de Verificación

Ejecutado el **16 de noviembre de 2026**.

| Criterio | Resultado | Evidencia |
|---|---|---|
| `AGENTS.MD` en la raíz | ✅ | Este archivo. |
| Humano (`X`) juega en celdas vacías | ✅ | `alPulsarCasilla()` → `puedeJugarHumano()` → `realizarJugada()`. |
| Tablero bloqueado en el turno de la IA | ✅ | 9/9 celdas `disabled` durante `Pensando IA (O)`. |
| `Imbatible` usa Minimax (nunca pierde) | ✅ | Búsqueda exhaustiva del árbol de jugadas: 0 victorias humanas. |
| `Normal` permite ganar al humano | ✅ | Simulación Monte Carlo: victorias humanas > 0 (bloquea amenazas simples pero no horquillas). |
| 8 combinaciones + empate | ✅ | Test de las 8 líneas, tablero vacío sin ganador y tablero lleno sin línea. |
| Reinicio limpia estado sin recargar | ✅ | `iniciarPartida()` reinicia tablero, turno, fase, resaltados y temporizador. |
| Casos borde 1–8 | ✅ | Verificados en navegador (consola) y con tests del núcleo. |
| Consola sin errores | ✅ | 0 `Uncaught TypeError` / referencias nulas. |
