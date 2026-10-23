---
autor: Yadilton Torres
fecha_creacion: 2026-10-09
ultima_actualizacion: 2026-10-09
estado: desarrollo
nivel: 1
---

# Consultar Guía (modal del header)

**Autor:** Yadilton Torres

**Selector:** `app-consulta-guia-modal`

**Ubicación:** `SitioLogiGho/src/app/components/consulta-guia-modal`

---

## ¿Qué hace?

Modal que se abre desde el botón **"Consultar guía"** del header, disponible en todas las pantallas. El usuario escribe o pega el número de una guía de transportadora y el modal muestra:

- Los datos del pedido para confirmar que es la guía correcta: transportadora, tienda, estado (con color) y fecha.
- El link del PDF de la guía (`UrlGuia`) con los botones **Abrir** y **Copiar**.
- La **vista previa del PDF** a la derecha (solo en computador).

Cada usuario solo ve las guías de **sus tiendas asignadas**. Es un componente de **solo lectura**: no inserta ni modifica nada en la base de datos.

A diferencia de la vista [Consultar Guía de Logística](../views/logistica/info-guias/informacion-guia.md), que muestra toda la información del preenvío en una pantalla propia, este modal es un acceso rápido al **PDF** de la guía sin salir de la pantalla en la que se está.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login |
| Rol específico | Ninguno: el botón está en el header, visible para todos los roles |
| Datos visibles | Solo guías de las tiendas del usuario (`sessionStorage.tiendas_asignadas`). Con `Todas` ve cualquier guía |

---

## Estructura de archivos

```
consulta-guia-modal/
├── consulta-guia-modal.component.ts
├── consulta-guia-modal.component.html
├── consulta-guia-modal.component.scss
└── consulta-guia-modal.component.spec.ts
```

---

## Dónde se usa

Lo abre el **header** (`layout/default-layout/default-header`):

| Archivo | Cambio |
|---|---|
| `default-header.component.html` | Botón `.btn-consulta-guia` (icono propio de hoja + código de barras + lupa, y el texto "Consultar guía"), un separador `.separador-header` antes de los avisos y la etiqueta `<app-consulta-guia-modal>` |
| `default-header.component.ts` | Propiedad `consultaGuiaAbierta` y métodos `abrirConsultaGuia()` / `cerrarConsultaGuia()` |
| `default-header.component.scss` | Estilos del botón. En pantallas de menos de 1200 px se oculta el texto y queda solo el icono con un tooltip |

> En celular el botón se ve solo con el icono, junto a los avisos (ver [Barra superior](../layout/default-header.md)).

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `isOpen` | `boolean` | Controla si el modal es visible. Lo maneja el header |
| `@Output` | `close` | `EventEmitter<void>` | Se emite al cerrar (botón ✕, fondo oscuro o Escape) |
| `@ViewChild` | `guiaInput` | `ElementRef<HTMLInputElement>` | Campo del número, para devolverle el foco |
| `@ViewChild` | `modal` | `ElementRef<HTMLElement>` | Caja del modal, para que Tab no saque el foco de ella |
| `@HostListener` | `document:keydown.escape` | — | Cierra el modal con Escape |
| `@HostListener` | `document:keydown` | — | Mantiene el foco dentro del modal con Tab / Shift+Tab |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `estado` | `EstadoConsulta` | Controla qué bloque muestra el template (ver tipo abajo) |
| `txtGuia` | `string` | Texto del campo, con `[(ngModel)]` |
| `urlGuia` | `string` | Link del PDF de la guía encontrada |
| `datosPedido` | `DatosPedido \| null` | Transportadora, tienda, estado y fecha del pedido |
| `historial` | `ItemHistorial[]` | Últimas 5 guías encontradas en la sesión (chips de "Recientes") |
| `urlVistaPrevia` | `SafeResourceUrl \| null` | Link del PDF marcado como seguro para el iframe. `null` si no se muestra la vista previa |
| `consultaActual` | `number` | Número de la búsqueda en curso. Si llega una respuesta de una búsqueda anterior, se descarta |

### Tipo EstadoConsulta

```typescript
type EstadoConsulta = 'inicial' | 'cargando' | 'encontrado' | 'sinPdf' | 'noEncontrado' | 'otraTienda' | 'error';
```

### Estados del modal

| Estado | Cuándo ocurre | Qué muestra |
|---|---|---|
| `inicial` | Al abrir el modal | Campo vacío y texto de ayuda |
| `cargando` | Mientras responde el servidor | Spinner en el botón y "Buscando guía..." |
| `encontrado` | La guía es del usuario y tiene link | Datos, link, Abrir, Copiar y vista previa |
| `sinPdf` | La guía es del usuario pero no tiene link | Datos y "La guía existe, pero aún no tiene PDF generado." |
| `noEncontrado` | La guía no existe | "No se encontró la guía." |
| `otraTienda` | La guía existe pero es de otra tienda | "Esta guía no pertenece a tus tiendas." (sin ningún dato de la guía) |
| `error` | Campo vacío, letras, tiendas sin cargar, demora o fallo de la consulta | El mensaje del caso, en rojo, y el modal tiembla |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico` | `metodoGenerico?coleccion=PedidosInter&Numeropreenvio={guia}&Tienda={tiendas}&campos=...&mcomp=2` | Al buscar una guía |
| `ConsumoGenericoService` | `consultarGenerico` | `metodoGenerico?coleccion=PedidosInter&Numeropreenvio={guia}&campos=Numeropreenvio&lote=1&mcomp=2` | Solo si no se encontró y el usuario no tiene `Todas`, para saber si existe en otra tienda |
| `DecompressionService` | `decompressZstd` | — | Descomprimir la respuesta (`mcomp=2`) |

Las dos consultas pasan por un solo método privado, `consultarPedidosInter(filtros)`.

**Campos que se piden** (`CAMPOS_CONSULTA`), los únicos que usa el modal:
`UrlGuia, Transportadora, Tienda, NombreTienda, Estado Logigho, Estado, Fecha de creación, FechaCarga`.

> **Rendimiento:** la búsqueda usa el índice `Numeropreenvio_1` de `PedidosInter`. Medido en el navegador: **113 ms** y **~1 KB** de respuesta. El backend busca el número como número y como texto a la vez (ver [ApiLambdaCrudGenericoAOT](../../backend/lambdas-dotnet/lambdas/ApiLambdaCrudGenericoAOT/ApiLambdaCrudGenericoAOT.md)), así que encuentra la guía sin importar cómo esté guardada.

---

## Flujo principal

```
Botón "Consultar guía" (header)
  -> isOpen = true
  -> ngOnChanges: limpiar(), leer el historial del usuario, foco en el campo

Enter o botón Buscar -> onEnter() / buscar()
  -> si ya hay una guía mostrada y el número no cambió: Enter abre el PDF
  -> limpiarGuia(): quita espacios, saltos de línea y guiones
  -> vacío o con letras -> mostrarError()
  -> si la guía está en Recientes -> se muestra sin ir al servidor
  -> sin tiendas cargadas -> mostrarError("Estamos cargando tus tiendas...")
  -> estado = 'cargando'
  -> consultarPedidosInter({ campos, Tienda: tiendas, Numeropreenvio })
  -> si llegó tarde (hay otra búsqueda) -> se descarta
  -> sin pedido:
       -> usuario con "Todas"       -> 'noEncontrado'
       -> existeLaGuia() = true     -> 'otraTienda'
       -> existeLaGuia() = false    -> 'noEncontrado'
  -> con pedido y UrlGuia http(s) -> 'encontrado', vista previa, se guarda en Recientes
  -> con pedido sin link          -> 'sinPdf'
  -> error 504 o TimeoutError     -> "La búsqueda está tardando más de lo esperado..."
  -> otro error                   -> "No se pudo consultar la guía. Intenta de nuevo."
  -> al final: foco de vuelta al campo
```

---

## Solo guías de mis tiendas

1. La búsqueda manda `Tienda = sessionStorage.tiendas_asignadas`, el mismo filtro que usa la pantalla de Pedidos de la tienda. Si el valor contiene `Todas`, el backend quita el filtro y busca en todas.
2. Si no hay resultado y el usuario no tiene `Todas`, `existeLaGuia()` pregunta si la guía existe en cualquier tienda pidiendo **solo** `Numeropreenvio` (`lote=1`), sin link, tienda ni estado. Así se puede decir "no es de tus tiendas" sin mostrar nada de la otra tienda.
3. Si esa segunda consulta falla, se toma como que no existe (`noEncontrado`).

> **Seguridad:** este filtro se hace en el frontend. Protege el uso normal de la página, pero una persona podría quitar el filtro modificando la petición desde las herramientas del navegador, porque hoy el backend genérico no valida las tiendas del usuario. La protección completa sería validar las tiendas en el backend.

**Cómo probarlo sin otro usuario:** F12 → Application → Session Storage → cambiar `tiendas_asignadas` por el nombre exacto de una tienda y, sin recargar, buscar una guía de otra tienda. Al recargar, LogiGho vuelve a escribir las tiendas reales.

---

## Recientes (historial)

- Guarda las **últimas 5 guías encontradas** en `sessionStorage`, con la clave `consultaGuiaHistorial:<correo>`. Así, si otra persona inicia sesión en la misma pestaña, no ve las guías anteriores. Sin correo no se guarda nada.
- Solo vive en el navegador de quien busca y se borra al cerrar la pestaña. No se envía al servidor.
- Un chip de Recientes muestra la guía **sin consultar al servidor**. El botón **Limpiar** (con icono de basurita) las borra.
- Al leerlo, se descartan los ítems dañados o con un formato viejo.

---

## Vista previa del PDF

- Se muestra en un `iframe` a la derecha y el modal se ensancha a dos columnas (en pantallas de menos de 820 px quedan una debajo de la otra).
- Solo con links **https** (un http se bloquearía por contenido mixto). Es el único lugar donde se usa `bypassSecurityTrustResourceUrl`.
- **No se muestra en celular** (`soportaVistaPrevia()`): Chrome de Android no tiene visor de PDF dentro de la página (`navigator.pdfViewerEnabled` es `false`) y el recuadro quedaba como un archivo suelto. También se omite en pantallas táctiles de hasta 820 px. En esos casos el PDF ni se descarga y queda el botón **Abrir**.
- Si el iframe no avisa que terminó de cargar, a los 8 segundos se quita el texto "Cargando vista previa...".

---

## Accesibilidad y teclado

- **Enter** busca; con la guía ya mostrada y el mismo número, abre el PDF.
- **Escape** cierra el modal.
- **Tab / Shift+Tab** no salen del modal (`onTab()` + `elementosEnfocables()`). El iframe del PDF tiene `tabindex="-1"`, porque desde adentro del visor el modal ya no podría retener el foco.
- Cerrado, el modal tiene `visibility: hidden`: no recibe foco ni lo anuncian los lectores de pantalla.
- Se respeta `prefers-reduced-motion`: sin temblor ni animaciones de entrada.
- En celular (≤ 576 px): menos relleno, campo de 16 px (evita el zoom de iPhone), botón de cerrar de 40 px y Abrir/Copiar a todo el ancho. El tip de Enter se oculta en pantallas táctiles.

---

## Pruebas

`consulta-guia-modal.component.spec.ts` — **31 pruebas**, con los servicios simulados (no llaman al servidor real):

```
npx ng test --include=src/app/components/consulta-guia-modal/consulta-guia-modal.component.spec.ts --watch=false --browsers=ChromeHeadless
```

Cubren: validaciones del campo, limpieza de espacios y guiones, parámetros de la consulta, estados `encontrado` / `sinPdf` / `noEncontrado` / `error`, mensaje de demora (504 y `TimeoutError`), guía de otra tienda, usuario con `Todas`, tiendas sin cargar, doble Enter, Recientes por usuario (máximo 5, ítems dañados), Enter que abre el link, colores del estado, foco con Tab y celular sin vista previa.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-05 | Yadilton Torres | Creación del botón "Consultar guía" en el header y del modal: búsqueda por `Numeropreenvio` en `PedidosInter` y link con Abrir / Copiar |
| 2026-10-06 | Yadilton Torres | Datos del pedido, distinción entre "no existe" y "sin PDF", limpieza de espacios y guiones, Recientes en `sessionStorage` y vista previa del PDF |
| 2026-10-07 | Yadilton Torres | Recientes por usuario, `visibility: hidden` al cerrar y 18 pruebas unitarias |
| 2026-10-08 | Yadilton Torres | Solo los campos necesarios (`campos`, respuesta de ~1 KB), mensaje claro cuando la búsqueda tarda (504 / `TimeoutError`) y foco atrapado con Tab |
| 2026-10-09 | Yadilton Torres | Solo guías de las tiendas del usuario (estado `otraTienda`), sin vista previa en celular, retoques para celular, botón Limpiar con basurita y limpieza de código (`mostrarError`, `consultarPedidosInter`, `limpiarResultado`). 31 pruebas |

---

## Observaciones

- **Filtro de tiendas en frontend:** ver la nota de seguridad en *Solo guías de mis tiendas*. La validación completa debe hacerse en el backend.
- **Modal movido al `body`:** en `ngOnInit` el componente se mueve a `document.body`, porque el header es `header-sticky` y su z-index dejaba el fondo oscuro por debajo del menú lateral. En `ngOnDestroy` se quita del `body`.
- **Respuestas que llegan tarde:** si el usuario busca otra guía o cierra el modal mientras espera, la respuesta anterior se descarta gracias a `consultaActual`.
- **Guías con letras:** el campo solo acepta dígitos. Guías como las de Hoko (`HK...`) no se pueden buscar hoy.
- **Doble Enter:** mientras hay una búsqueda en curso, un segundo Enter se ignora.
