## 6. Servicio: DespachoPdfService

**Ubicación:** `src/app/views/logistica/relacion-despacho/services/despacho-pdf.service.ts`  
**Scope:** `providedIn: 'root'` — singleton global, inyectado en `RelacionDespachoComponent`

---

### ¿Qué hace?

Este servicio crea el archivo PDF final con la relación de despacho. Reúne las guías, las organiza por transportadora y las prepara para descargarlas o imprimirlas de forma ordenada.

- **Cabecera:** ícono RLDICONO (izquierda) · "RELACIÓN DE DESPACHO" (centro) · logo de la transportadora (derecha).
- **Cards de resumen:** fecha de generación, cantidad de guías y total a recaudar.
- **Tabla:** #, Guía, Tienda, Cliente (+teléfono), Departamento, Recaudo, Dice contener.
- **Bloque de firmas:** transportador, responsable/tienda y placa del vehículo (solo última página del grupo).

Usa `jsPDF` + `html2canvas` renderizando dentro de un `<iframe>` oculto para aislar los estilos globales de la app.

---

### Métodos

| Método | Visibilidad | Descripción |
|---|---|---|
| `generarPDF(rows)` | público | Orquesta todo el flujo: enriquece, carga logos, construye HTML, renderiza y descarga |
| `enriquecerConTransportadora(rows)` | privado | Consulta `PedidosInter` por cada guía para obtener transportadora y teléfono |
| `construirHtml(rows, icono, logos)` | privado | Genera el HTML completo paginado por transportadora |
| `cargarImagenBase64(src)` | privado | Convierte una imagen local a base64 para evitar CORS en html2canvas |
| `renderizarHtmlACanvas(html)` | privado | Inyecta el HTML en un iframe oculto y lo captura con html2canvas |
| `guardarPDF(canvas)` | privado | Corta el canvas en páginas A4 y descarga el PDF como JPEG 95 % |

---

### Constantes

| Constante | Valor | Descripción |
|---|---|---|
| `ICONO_SRC` | `assets/images/RLDICONO.png` | Ícono corporativo de cabecera |
| `POR_PAGINA` | `18` | Filas máximas antes de paginar dentro de un mismo grupo |
| `LOGOS_TRANSPORTADORA` | Array | Mapeo clave fuzzy → ruta de logo por transportadora |

**Logos disponibles:**

| Clave (fuzzy) | Archivo |
|---|---|
| `SERVIENTREGA` | `logo-servientrega-blanco.svg` |
| `TCC` | `tcc.svg` |
| `COORDINADORA` | `coordinadora.svg` |
| `ENVIA` | `envia.png` |
| `INTERRAPIDISIMO` | `inter.png` |

La búsqueda es fuzzy: `transportadora.toUpperCase().includes(clave)`. Si no hay coincidencia, se muestra el nombre en texto blanco.

---

### Servicios y endpoints

| Servicio | Endpoint | Cuándo |
|---|---|---|
| `ConsumoGenericoService` | `GET PedidosInter?Numeropreenvio=` | Por cada guía al enriquecer transportadora |
| `DecompressionService` | — | Descomprime respuesta Zstd de cada consulta |

---

### Flujo principal

```
generarPDF(rows)
  ├─► Promise.all
  │     ├─► cargarImagenBase64(RLDICONO) → base64
  │     └─► enriquecerConTransportadora(rows)
  │           └─► GET PedidosInter × n guías (paralelo)
  │
  ├─► cargar logos de transportadoras únicas (paralelo)
  │
  ├─► construirHtml() → HTML con secciones por transportadora
  │     └─► por cada grupo: chunks de POR_PAGINA filas
  │           └─► última página: bloque de firmas
  │
  ├─► renderizarHtmlACanvas()
  │     ├─► iframe oculto fuera del viewport
  │     ├─► espera fonts.ready (máx. 3 s)
  │     └─► html2canvas(scale: 2) → canvas
  │
  └─► guardarPDF(canvas)
        ├─► corta en slices A4
        └─► jsPDF.save('RelacionDespacho_fecha_hora.pdf')
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación con agrupación por transportadora, logos y bloque de firmas |

---

### Observaciones

- Los logos se convierten a base64 **antes** de pasarlos al iframe porque `html2canvas` no puede cargar imágenes locales desde un iframe sin ese paso (restricción de CORS del browser).
- `LOGOS_TRANSPORTADORA` es un `Array` (no un `Record`) para evitar el error TS2538 al indexar con string dinámico.
- El iframe se posiciona en `top: -99999px` para que sea invisible pero esté en el DOM y pueda ser medido por `html2canvas`.
- `guardarPDF` usa JPEG 95 % en lugar de PNG para reducir significativamente el tamaño del archivo final.
