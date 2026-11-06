---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-06
Estado: desarrollo
Tipo: componente

# Componente: Mapa "La Red" (escena WebGL)

**Selector:** — (clases TypeScript, sin Angular)  
**Ubicación:** `src/app/views/pages/registro/la-red/scene` y `la-red/host`  
**Acceso:** Público (lo usan el [registro](registro.md) y el [inicio de sesión](../../../components/login.md))

---

## ¿Qué hace?

Dibuja el mapa de Colombia en 3D con Three.js: los 1.103 municipios como puntos, las rutas entre ciudades, camiones, aviones y barcos en movimiento, la silueta del país, los ríos y los países vecinos. Reacciona a lo que la persona hace en el formulario y sirve de fondo vivo del login. Si el equipo no aguanta WebGL, se cambia solo por un mapa SVG (`PosterMap`).

---

## Piezas

| Pieza | Archivo | Para qué |
|---|---|---|
| `MapaLaRed` | `host/MapaLaRed.ts` | Punto de entrada: decide WebGL o póster, carga la escena de forma diferida y maneja el mouse (ciudades, países vecinos, sello "Próximamente") |
| `NetworkScene` | `scene/NetworkScene.ts` | La escena: cámara, capas y la API `NetworkSceneApi` (`scene/types.ts`) |
| `Choreographer` | `host/choreography.ts` | Traduce los eventos del formulario en llamadas a la escena y repite lo ya respondido si la escena carga tarde |
| `Points`, `Routes`, `Trucks`, `Borders`, `Labels`, `UserNode` | `scene/*.ts` + `shaders/*.glsl` | Municipios, rutas con pulso, vehículos, silueta/ríos, etiquetas y el punto de la persona |
| `Contexto` | `scene/Contexto.ts` | Países vecinos y mares; aperturas, puntos que laten, líneas al pasar el mouse y viaje del cometa (`DURACION_VIAJE = 0.4` s) |
| `QualityGovernor` | `scene/QualityGovernor.ts` | Mide los cuadros por segundo y baja la calidad por niveles (resolución, brillo, ondas) hasta pasar al póster |
| `PackageView` | `host/PackageView.ts` | La caja que se arma con las respuestas y vuela al punto al enviar |
| `PosterMap` | `fallback/PosterMap.ts` | Mapa SVG equivalente, sin WebGL |

Los datos salen de `assets/la-red/data` (`municipios.bin`, `borders.bin`, `vecinos.json`, `mar.json`) y se proyectan con `scene/projection.ts` (1 unidad = alto de la Colombia continental).

---

## API de la escena

| Método | Lo llama | Efecto |
|---|---|---|
| `spawnNode(label)` | Nombre | Aparece el punto de la persona |
| `moveNodeTo(code)` | Ciudad | El punto viaja al municipio y la cámara se acerca |
| `verifyNode()` | Correo válido | Arco de verificación |
| `emitSignal()` | Teléfono | Onda desde el punto |
| `igniteRoutes(nivel)` | Cuándo empieza / éxito | Enciende las rutas (0 a 3) |
| `finalWave()` / `playLaunch()` | Envío, confirmación, login | Onda final y salida de camiones |
| `vecinoEn()`, `resaltarVecino()`, `viajarAVecino()` | Mouse sobre el mapa | Países vecinos |
| `setQuality(tier)` / `setReducedMotion(on)` | Regulador / preferencias | Calidad y movimiento |

---

## Rendimiento

- **Fuera de la zona de Angular:** la escena corre con `NgZone.runOutsideAngular`.
- **Regulador de calidad:** baja por niveles (`high` → `mid` → `low` → `poster`), y un nivel que falla no se vuelve a intentar en la visita. Con `?fps=1` se ve un medidor, y `?quality=` / `?nivel=` fijan el nivel para pruebas.
- **Clases para equipos lentos:** el fondo (`.lared-app[data-nivel]`) quita el vidrio, la luz del borde de la tarjeta y las animaciones de la caja.
- **En celular:** las luces del fondo (`.lared-map::before/::after`) dejan de animarse.
- **Vuelo de la caja:** usa solo `transform` y `opacity`, y pausa las sombras y las animaciones internas mientras vuela.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-03 | Yadilton Torres | Escena "La Red" con municipios, rutas, vehículos, vecinos y regulador de calidad. |
| 2026-11-06 | Yadilton Torres | Vecinos con "Síguenos en redes" (hace parpadear el enlace del pie), viaje del cometa de 0,4 s, vuelo de la caja en arco suave y luces fijas en celular. |

---

## Observaciones

- **Comentarios en los `.glsl`:** solo en ASCII. Algunos drivers fallan con tildes en los shaders.
- **Pruebas:** la escena no tiene pruebas de dibujo (es WebGL). Sí las tienen `Contexto` y `QualityGovernor`.
