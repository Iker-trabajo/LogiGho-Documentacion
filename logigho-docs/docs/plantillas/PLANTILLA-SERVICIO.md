---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Nombre del Servicio

**Autor:** Nombre Apellido  
**Tipo:** `@Injectable({ providedIn: 'root' })`  
**Ubicación:** `SitioLogiGho/src/app/core/services/nombre.service.ts`

---

## ¿Qué hace?

Descripción breve en 2-3 líneas.

---

## Métodos

### `nombreMetodo(param: tipo): Observable<tipo> | Promise<tipo> | tipo`

**Descripción:** Qué hace.

**Parámetros:**
| Parámetro | Tipo | Descripción |
|---|---|---|
| `param` | `string` | Descripción |

**Retorna:** Descripción del retorno

**Ejemplo de uso:**
```typescript
this.nombreService.nombreMetodo(param).subscribe(resultado => {
  // hacer algo
});
```

---

## Dependencias

| Servicio / Módulo | Uso |
|---|---|
| `HttpClient` | Llamadas HTTP al API |
| — | — |

---

## Endpoints que consume

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/api/v1/recurso` | Obtiene lista |
| `POST` | `/api/v1/recurso` | Crea recurso |

---

## Changelog

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |

---

## Observaciones

- Observación 1
