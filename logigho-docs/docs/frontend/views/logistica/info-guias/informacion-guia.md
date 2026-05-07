---
autor: Adalberto González
fecha_creacion: 2026-05-04
ultima_actualizacion: 2026-05-07
estado: Desarrollo
nivel: 2
---

# Vista: Consultar Guía

**Autor:** Adalberto González

**Selector:** `app-informacion-guia`

**Ubicación:** `SitioLogiGho/src/app/views/logistica/informacion-guia`

---

## ¿Qué hace?

Vista de consulta de información de preenvíos por número de guía. Permite al usuario ingresar o escanear (lector físico de código de barras) el número de guía de un pedido y visualizar en pantalla toda la información relevante del preenvío: destinatario, dirección, transportadora, contrapago, valores y observaciones de contenido. Es un módulo de solo lectura; no realiza inserciones ni modificaciones en la base de datos.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/app/logistica/informacion-guias` |
| **Título de página** | `Informacion Guias` |
| **Guard** | `AuthGuard` |
| **Rol requerido** | Administrador, Desarrollador, CEO, COO (configurable en colección `module`) |
| **Parámetros de URL** | Ninguno |

---

## Estructura de archivos

```
informacion-guia/
├── informacion-guia.component.ts
├── informacion-guia.component.html
├── informacion-guia.component.scss
└── informacion-guia.component.spec.ts
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título "Consultar Guía" y subtítulo descriptivo |
| 2 | **Buscador** | Input para número de guía con prefijo `#`, botón "Buscar" (solo desktop) y hint de ayuda. En móvil se reemplaza por botón "Activar cámara" |
| 3 | **Estado inicial** | Pantalla vacía con ícono e instrucción al usuario antes de la primera búsqueda |
| 4 | **Estado cargando** | Skeleton animado con shimmer que representa la estructura del resultado mientras se espera la respuesta del backend |
| 5 | **Estado encontrado** | Tarjeta de resultado con: barra superior (número de guía + badge de estado + descripción), grid de dos columnas (Destinatario / Envío) con `align-items: start` para no estirar la columna derecha, y recuadro de Contenido y Observaciones |
| 5a | **Tabla de productos** | Dentro del recuadro de observaciones: tabla de hasta 12 productos con columna Producto, Cantidad, Precio unitario y Total. Incluye skeleton mientras carga, fila de total general y estado vacío si el pedido no tiene productos. Con scroll horizontal en mobile |
| 6 | **Estado no encontrado** | Tarjeta de error con ícono y mensaje cuando la guía no existe en PedidosInter |

---

## Propiedades del componente

### Estado y datos

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `txtGuia` | `string` | `''` | Texto del input, enlazado con `[(ngModel)]`. Se limpia automáticamente al terminar la consulta |
| `estado` | `EstadoConsulta` | `'inicial'` | Controla qué bloque renderiza el template. Ver tipo más abajo |
| `pedido` | `any \| null` | `null` | Documento de PedidosInter retornado por el backend. `null` mientras no hay resultado activo |
| `searchTimeout` | `ReturnType<typeof setTimeout> \| null` | `null` | Handle del timer de debounce. Se nulifica explícitamente tras `clearTimeout()` para evitar referencias colgantes |
| `productos` | `any[]` | `[]` | Lista de productos del pedido obtenida de la colección `Productos` en paralelo |
| `cargandoProductos` | `boolean` | `false` | Controla el skeleton de la tabla de productos mientras se resuelven las consultas paralelas |

### Tipo EstadoConsulta

```typescript
type EstadoConsulta = 'inicial' | 'cargando' | 'encontrado' | 'noEncontrado';
```

### Detección de dispositivo y escáner

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isMobile` | `boolean` | `false` | `true` si el user-agent corresponde a Android, iPhone o iPad |
| `isActivar` | `boolean` | `false` | Controla la visibilidad del escáner de cámara en móvil |
| `formatsEnabled` | `BarcodeFormat[]` | `[CODE_128, EAN_13, UPC_A]` | Formatos de código de barras aceptados por el escáner ZXing |

---

## Interfaz de datos (PedidosInter)

Campos del documento de MongoDB que utiliza esta vista. La colección completa tiene más campos; solo se documentan los que se renderizan.

```typescript
interface PedidoInter {
  Numeropreenvio: string | number;   // Número de guía — campo índice de búsqueda
  Estado: string;                    // Estado del pedido (ej. "Pagada", "Entregada")
  'Descripcion Asociada': string;    // Descripción legible del estado actual
  NombreCompleto: string;            // Nombre del destinatario
  Telefono: string | number;         // Teléfono de contacto
  Ciudad: string;
  Departamento: string;
  Direccion: string;
  Alto: string;                      // Dimensiones del paquete
  Ancho: string;
  Largo: string;
  Peso: string;
  Transportadora: string;
  Trayecto: string;                  // "Urbano" | "Nacional"
  FechaDeEntregaEstimada: string;
  AplicaContraPago: string;          // "SI" | "NO"
  TotalRecaudo: string | number;     // Solo relevante si AplicaContraPago === "SI"
  ValorDeclarado: number;
  DiceContener: string;              // Descripción del contenido del paquete
  Observaciones: string;             // Instrucciones especiales del envío
}
```

> **Nota sobre el índice:** La colección tiene un índice `Numeropreenvio_1` (REGULAR, ~54.5 MB, ~2M usos) que garantiza búsquedas en O(log n) sin full scan.

---

## Flujo de inicialización

```
ngOnInit()
  → detectDevice()         — detecta si es móvil por user-agent
  → setTimeout 0ms         — foco automático en el input #guia-input
```

---

## Flujo de consulta

```
onKeyPress(Enter) | (click) Buscar | onScanSuccess(result)
  → consultarGuia()
      → strip leading zeros + trim
      → check caché / prefetch             — si el dato ya está en caché se resuelve sin mostrar skeleton
      → estado = 'cargando', pedido = null
      → GET metodoGenerico?coleccion=PedidosInter&Numeropreenvio={value}
      → decompressGzip(data.Resultado)
      → si vacío  → estado = 'noEncontrado', playErrorSound()
      → si existe → pedido = resultado[0], estado = 'encontrado', playBeepSound()
                 → extraerPares(pedido)     — extrae hasta 12 pares IdStock/CantidadStock
                 → Promise.all(consultas a Productos por idproducto)
                 → calcula total = precioproveedor × cantidad por cada producto
      → catch(e)  → console.error, estado = 'noEncontrado'
      → txtGuia = ''
      → setTimeout 0ms — refoco del input para siguiente escaneo
```

---

## Métodos

### `ngOnInit(): void`
Inicializa el componente: detecta el dispositivo y establece el foco en el input.

---

### `consultarGuia(): Promise<void>`
Método principal. Consulta PedidosInter por `Numeropreenvio` y actualiza el estado de la vista según el resultado.

**Proceso:**
1. Normaliza `txtGuia`: elimina ceros a la izquierda con `/^0+/` y espacios.
2. Si el valor está vacío, retorna sin hacer nada.
3. Comprueba caché/prefetch **antes** de cambiar el estado — si el dato ya está disponible, omite el skeleton por completo.
4. Cambia el estado a `'cargando'` y consulta el backend con `consultarGenerico`.
5. Descomprime con `decompressGzip`.
6. Si el array resultado está vacío → `'noEncontrado'` + sonido de error.
7. Si hay resultado → asigna `resultado[0]` a `pedido` → `'encontrado'` + beep → llama a `extraerPares()` para cargar la tabla de productos.
8. En caso de excepción → loguea en consola → `'noEncontrado'`.
9. Limpia el input y restaura el foco.

---

### `onKeyPress(event: KeyboardEvent): void`
Escucha el evento `keypress` del input. Llama a `consultarGuia()` cuando la tecla es `Enter`. Permite el uso con lector físico de código de barras.

---


### `getStatusClass(estado: string): string`
Convierte el valor del campo `Estado` de MongoDB al nombre de clase CSS del badge de estado.

| Valor MongoDB | Clase CSS |
|---|---|
| `'Pagada'` | `'pagada'` |
| `'Entregada'` | `'entregada'` |
| `'Pendiente'` | `'pendiente'` |
| `'Devuelta'` | `'devuelta'` |
| Cualquier otro | `'pendiente'` (fallback) |

---

### `getAudioContext(): AudioContext`
Devuelve una instancia compartida de `AudioContext`. Si ya existe la reutiliza; si no, la crea y la almacena. Evita el memory leak que se producía cuando cada llamada a `playBeepSound` / `playErrorSound` creaba un contexto nuevo sin cerrar el anterior.

---

### `playBeepSound(): void`
Reproduce un tono corto de 440 Hz (tipo "confirmación") usando la Web Audio API a través de `getAudioContext()`. Duración: ~100 ms.

---

### `playErrorSound(): void`
Reproduce un tono de 180 Hz tipo sawtooth (tipo "error") usando la Web Audio API a través de `getAudioContext()`. Duración: ~600 ms.

---

### `extraerPares(pedido: any): void`
Lee hasta 12 campos `IdStock` / `CantidadStock` del documento de PedidosInter y construye la lista de pares `{ idStock, cantidad }`. Con esos pares lanza consultas en paralelo (`Promise.all`) a la colección `Productos` filtrando por `idproducto`. Calcula `total = precioproveedor × cantidad` por cada línea (el campo en MongoDB se llama `precioproveedor`, todo en minúsculas) y acumula el total general. Actualiza `productos` y baja `cargandoProductos` al finalizar.

---

## Subcomponentes

Esta vista no utiliza subcomponentes.

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | Consulta PedidosInter por Numeropreenvio y la colección Productos por idproducto |
| `DecompressionService` | `decompressGzip()` | Descomprime la respuesta base64 del backend |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=PedidosInter&Numeropreenvio={value}` | Cada vez que se escanea o ingresa un número de guía |
| `GET` | `metodoGenerico?coleccion=Productos&idproducto={id}` | En paralelo (hasta 12 peticiones) tras encontrar un pedido, para cargar la tabla de productos |

---

## Estados de la vista

| Estado | Cuándo ocurre | Qué muestra |
|---|---|---|
| `inicial` | Al cargar el componente | Ícono de caja + instrucción al usuario |
| `cargando` | Desde que se dispara la consulta hasta que el backend responde | Skeleton con shimmer animado |
| `encontrado` | Backend retorna al menos un documento | Tarjeta con datos del preenvío |
| `noEncontrado` | Backend retorna array vacío o lanza excepción | Tarjeta de error con ícono rojo |

---

## Registro en el menú (colección `module`)

El módulo queda visible en el menú lateral cuando existe un documento activo en la colección `module` con la siguiente estructura:

```json
{
  "ID": "<siguiente ID en la secuencia>",
  "Nombre": "Informacion Guias",
  "ruta": "/app/logistica/informacion-guias",
  "iconComponent": "cilSearch",
  "moduloPadre": "<ID del módulo padre de Logística>",
  "roles": ["Administrador", "Desarrollador", "CEO", "COO"],
  "estado": "ACTIVO"
}
```

---

## Changelog

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-05-04 | Adalberto González | Creación del componente: consulta de guías con estados inicial, cargando, encontrado y noEncontrado |
| 2026-05-04 | Adalberto González | Integración de lector físico de código de barras vía evento keypress Enter |
| 2026-05-04 | Adalberto González | Implementación de skeleton loading y `ChangeDetectorRef` para render inmediato |
| 2026-05-05 | Adalberto González | Implementación de timer con debounce en el input del módulo |
| 2026-05-06 | Adalberto González | Implementación de función prefetch para precargar datos en el front y obtener resultados instantáneos |
| 2026-05-07 | Iker Acevedo | **Bug fixes:** eliminado memory leak de `AudioContext` (instancia única vía `getAudioContext()`); corregido flash de skeleton en hits de caché moviendo el check antes de `estado = 'cargando'`; eliminada variable `prefetchCargando` sin uso; `searchTimeout` se nulifica tras `clearTimeout()`; eliminado `console.log` del prefetch expuesto en producción; eliminado `ChangeDetectorRef.detectChanges()` que bloqueaba el hilo (limpiados import y constructor) |
| 2026-05-07 | Iker Acevedo | **Rendimiento:** debounce reducido de 1200 ms → 500 ms; `searchTimeout` tipado como `ReturnType<typeof setTimeout> \| null` |
| 2026-05-07 | Iker Acevedo | **Nueva funcionalidad:** tabla de productos del pedido en el recuadro de observaciones — extrae hasta 12 pares `IdStock`/`CantidadStock` vía `extraerPares()`, consulta la colección `Productos` en paralelo con `Promise.all`, calcula `total = precioproveedor × cantidad`, muestra skeleton mientras carga, fila de total general y estado vacío; scroll horizontal en mobile |
| 2026-05-07 | Iker Acevedo | **UI/UX:** `obs-text` limitado a `max-height: 100px` con scroll interno; grid del resultado cambiado a `align-items: start` para no estirar la columna derecha; añadidos `overflow: hidden` y `word-break: break-word` en `.obs-box` |

---

## Observaciones

- **Ceros a la izquierda:** El campo `Numeropreenvio` en MongoDB es de tipo `$numberLong`. El componente elimina ceros a la izquierda con `/^0+/` antes de consultar, siguiendo el mismo patrón del módulo `DevolucionInventarioComponent`.
- **Solo lectura:** Este módulo no realiza inserciones ni actualizaciones. Si en el futuro se requieren acciones (registrar novedad, cambiar estado), se deberá agregar un endpoint PUT y el formulario correspondiente.
- **Campo `precioproveedor`:** En MongoDB el campo se almacena en minúsculas (`precioproveedor`), no en PascalCase. `extraerPares()` lee exactamente ese nombre para el cálculo de totales.
- **AudioContext compartido:** `getAudioContext()` sigue el patrón singleton por instancia de componente. Si el componente se destruye y se vuelve a crear, el contexto anterior queda elegible para GC ya que la referencia privada se pierde.
