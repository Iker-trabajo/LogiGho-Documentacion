---

## Autor: Adalberto González

Fecha creacion: 2026-06-10  
Estado: desarrollo  
Tipo: vista

# Vista: MetricasMuroComponent

**Selector:** `app-metricas-muro`  
**Ubicación:** `src/app/views/administracion/metricas-muro/metricas-muro`  
**Acceso:** Autenticado | Rol: `CEO` `Administrador` `Red social`

---

## ¿Qué hace?

Este módulo proporciona una vista consolidada del comportamiento y la actividad de la red social interna, permitiendo a la dirección de la compañía realizar seguimiento a los principales indicadores de participación.

---

## Ruta

| Ruta                            | Guard      | Parámetros de URL |
| ------------------------------- | ---------- | ----------------- |
| `/app/administracion/metricas-muro` | `AuthGuard` + rol CEO | —        |

---

## Decoradores

*Este componente no tiene `@Input` ni `@Output`.*

---

## Propiedades clave

| Propiedad          | Tipo                    | Descripción                                                                 |
| ------------------ | ----------------------- | --------------------------------------------------------------------------- |
| `kpis`             | `kpiCard[]`             | 6 tarjetas de métricas principales — índice 5 corresponde a vistas en video |
| `topVideosClics`   | `VideoClicMetrica[]`    | Top 5 videos ordenados por número de vistas únicas                          |
| `totalClicsVideo`  | `number`                | Suma total de vistas en todos los videos                                    |
| `publicaciones`    | `Publicacion[]`         | Copia privada de todas las publicaciones — base de todos los cálculos       |

---

## Servicios y endpoints

| Servicio                 | Método              | Endpoint                                       | Cuándo          |
| ------------------------ | ------------------- | ---------------------------------------------- | --------------- |
| `ConsumoGenericoService` | `consultarGenerico` | `GET metodoGenerico?coleccion=Publicaciones`   | Al inicializar  |
| `ConsumoGenericoService` | `consultarGenerico` | `GET metodoGenerico?coleccion=Comunidades`     | Al inicializar  |
| `ConsumoGenericoService` | `consultarGenerico` | `GET metodoGenerico?coleccion=Users`           | Al inicializar  |
| `DecompressionService`   | `decompressZstd`    | —                                              | Al descomprimir respuestas Zstd |

---

## Flujo principal

```
ngOnInit()
  -> cargarDatos()   // lanza en paralelo:
      -> cargarPublicaciones()   // GET + Zstd
      -> cargarComunidades()     // GET + Zstd
      -> cargarUsuarios()        // GET + Zstd
  -> calcularMetricas()
      -> calcularKpis()          // totales generales + total vistas video (kpis[5])
      -> calcularBarrasMeses()   // publicaciones por mes (últimos 6)
      -> calcularBarrasUsuarios()// usuarios registrados por mes
      -> calcularDonut()         // distribución por tipo de contenido
      -> calcularReacciones()    // desglose de emojis
      -> calcularTopCreadores()  // ranking por reacciones
      -> calcularComunidadesMetrica() // comunidades por miembros y posts
      -> calcularPublicacionesRecientes() // tabla últimas publicaciones
      -> calcularClicsVideo()    // top 5 videos por vistas + barra porcentual
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                                                        |
| ---------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| 2026-06-10 | Adalberto González | Se añadió la sección de vistas por video: nuevo KPI con el total de vistas, y tabla con el top 5 de videos más vistos con barra de progreso y conteo por video. |

---

## Observaciones