---
autor: Adalberto González
fecha_creacion: 2026-05-04
ultima_actualizacion: 2026-05-04
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
| 5 | **Estado encontrado** | Tarjeta de resultado con: barra superior (número de guía + badge de estado + descripción), grid de dos columnas (Destinatario / Envío) y recuadro de Contenido y Observaciones |
| 6 | **Estado no encontrado** | Tarjeta de error con ícono y mensaje cuando la guía no existe en PedidosInter |

---

## Propiedades del componente

### Estado y datos

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `txtGuia` | `string` | `''` | Texto del input, enlazado con `[(ngModel)]`. Se limpia automáticamente al terminar la consulta |
| `estado` | `EstadoConsulta` | `'inicial'` | Controla qué bloque renderiza el template. Ver tipo más abajo |
| `pedido` | `any \| null` | `null` | Documento de PedidosInter retornado por el backend. `null` mientras no hay resultado activo |

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
      → estado = 'cargando', pedido = null
      → cdr.detectChanges()          — fuerza render inmediato del skeleton
      → GET metodoGenerico?coleccion=PedidosInter&Numeropreenvio={value}
      → decompressGzip(data.Resultado)
      → si vacío  → estado = 'noEncontrado', playErrorSound()
      → si existe → pedido = resultado[0], estado = 'encontrado', playBeepSound()
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
3. Cambia el estado a `'cargando'` y fuerza detección de cambios para que el skeleton aparezca inmediatamente.
4. Consulta el backend con `consultarGenerico`.
5. Descomprime con `decompressGzip`.
6. Si el array resultado está vacío → `'noEncontrado'` + sonido de error.
7. Si hay resultado → asigna `resultado[0]` a `pedido` → `'encontrado'` + beep.
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

### `playBeepSound(): void`
Reproduce un tono corto de 440 Hz (tipo "confirmación") usando la Web Audio API. Duración: ~100 ms.

---

### `playErrorSound(): void`
Reproduce un tono de 180 Hz tipo sawtooth (tipo "error") usando la Web Audio API. Duración: ~600 ms.

---

## Subcomponentes

Esta vista no utiliza subcomponentes.

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | Consulta la colección PedidosInter por Numeropreenvio |
| `DecompressionService` | `decompressGzip()` | Descomprime la respuesta base64 del backend |
| `ChangeDetectorRef` | `detectChanges()` | Fuerza la actualización del template al estado 'cargando' para que el skeleton aparezca instantáneamente |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=PedidosInter&Numeropreenvio={value}` | Cada vez que se escanea o ingresa un número de guía |

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
| 2026-05-05 | Adalberto González | Implementación de Timer en el imput de el modulo |
| 2026-05-06 | Adalberto González | Implementación de funcion preFech para pre cargar datos en el front y tener resultados instantaneos |

---

## Observaciones

- **Ceros a la izquierda:** El campo `Numeropreenvio` en MongoDB es de tipo `$numberLong`. El componente elimina ceros a la izquierda con `/^0+/` antes de consultar, siguiendo el mismo patrón del módulo `DevolucionInventarioComponent`.
- **Solo lectura:** Este módulo no realiza inserciones ni actualizaciones. Si en el futuro se requieren acciones (registrar novedad, cambiar estado), se deberá agregar un endpoint PUT y el formulario correspondiente.
