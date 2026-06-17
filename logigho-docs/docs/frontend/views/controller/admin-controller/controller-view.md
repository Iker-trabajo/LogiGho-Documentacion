---

## Autor: Logigho

Fecha creacion: 2026-06-17  
Estado: desarrollo  
Tipo: vista

# Vista: ControllerComponent

**Selector:** `app-controller`  
**Ubicación:** `src/app/views/controller/controller`  
**Acceso:** Autenticado | Rol: `Controller` `CEO` `Desarrollador`

---

## ¿Qué hace?

Este módulo funciona como el panel principal de trabajo para los usuarios con rol de Controller. Desde una única pantalla, permite consultar y gestionar la información que requiere revisión, seguimiento o aprobación dentro de la operación diaria.

---

## Ruta

| Ruta               | Guard       | Parámetros de URL |
| ------------------ | ----------- | ----------------- |
| `/app/controller`  | `AuthGuard` | —                 |

---

## Servicios y endpoints

| Servicio                  | Método                    | Endpoint                                          | Cuándo              |
| ------------------------- | ------------------------- | ------------------------------------------------- | ------------------- |
| `ConsumoGenericoService`  | `consultarGenerico()`     | `GET metodoGenerico?coleccion=TiendaDocuments`    | Al inicializar      |
| `DecompressionService`    | `decompressGzip()`        | —                                                 | Tras cada respuesta |
| `DocumentosService`       | *(deprecated)*            | `GET tiendadocuments` *(reemplazado, retorna 500)*| —                   |

---

## Flujo principal

```
ngOnInit()
  -> fetchTableDataDocumentosTienda()
      -> consumogenericoServices.consultarGenerico('TiendaDocuments')
      -> decompressionService.decompressGzip()
      -> sanitizar y deduplicar registros
      -> generar columnas y filas para la tabla
```

---

## Historial de cambios

| Fecha      | Autor   | Cambio                                                                                          |
| ---------- | ------- | ----------------------------------------------------------------------------------------------- |
| 2026-06-17 | Logigho | `fetchTableDataDocumentosTienda` migrado de `DocumentosService` (500) a `ConsumoGenericoService` + Gzip, igual al patrón del resto de la vista |

---

## Observaciones
