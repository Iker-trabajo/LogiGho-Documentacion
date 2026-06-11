---

## Autor: Adalberto González

Fecha creacion: 2026-06-10  
Estado: desarrollo  
Tipo: vista

# Vista: MuroCompartidoComponent

**Selector:** `app-muro-compartido`  
**Ubicación:** `src/app/views/muro-compartido/muro-compartido`  
**Acceso:** Autenticado | Rol: todos los usuarios registrados

---

## ¿Qué hace?

Este módulo corresponde a la red social corporativa de LogiGho, un espacio diseñado para fomentar la comunicación y la interacción entre los colaboradores de la organización.

---

## Ruta

| Ruta                   | Guard      | Parámetros de URL |
| ---------------------- | ---------- | ----------------- |
| `/app/muro-compartido` | `AuthGuard`| —                 |

---

## Decoradores

| Decorador    | Nombre          | Tipo          | Descripción                              |
| ------------ | --------------- | ------------- | ---------------------------------------- |
| `@ViewChild` | `fileInput`     | `ElementRef`  | Input oculto para subir imágenes         |
| `@ViewChild` | `videoInput`    | `ElementRef`  | Input oculto para subir videos           |
| `@ViewChild` | `panelEdicion`  | `ElementRef`  | Panel de edición de publicación          |

---

## Propiedades clave

| Propiedad                  | Tipo                    | Descripción                                                              |
| -------------------------- | ----------------------- | ------------------------------------------------------------------------ |
| `publicaciones`            | `Publicacion[]`         | Lista completa de publicaciones cargadas desde MongoDB                   |
| `clicsVideo`               | `Array<{usuarioId, fechaCreacion}>` | Campo en `Publicacion` — registra quién vio el video y cuándo |
| `videoSonidoActivado`      | `string \| null`        | ID de la publicación cuyo video tiene sonido activo en este momento      |
| `cargandoPublicaciones`    | `boolean`               | Flag que evita cargas simultáneas al mismo endpoint                      |
| `comunidadSeleccionada`    | `Comunidad \| null`     | Filtra las publicaciones por comunidad cuando está definida              |
| `usuariosRestringidos`     | `UsuarioRestringido[]`  | Lista de usuarios bloqueados — impide publicar/comentar si está activo   |
| `estadoPublicacion`        | `string`                | `'ACTIVO'` habilita crear publicaciones; cualquier otro valor lo bloquea |

---

## Servicios y endpoints

| Servicio                  | Método               | Endpoint                                           | Cuándo                              |
| ------------------------- | -------------------- | -------------------------------------------------- | ----------------------------------- |
| `ConsumoGenericoService`  | `consultarGenerico`  | `GET metodoGenerico?coleccion=Publicaciones`       | Al cargar y al actualizar el muro   |
| `ConsumoGenericoService`  | `insertarGenerico`   | `POST metodoGenerico?coleccion=Publicaciones`      | Al crear una publicación            |
| `ConsumoGenericoService`  | `actualizarGenerico` | `PUT metodoGenerico?coleccion=Publicaciones`       | Al reaccionar, comentar, dar vista  |
| `ConsumoGenericoService`  | `consultarGenerico`  | `GET metodoGenerico?coleccion=PerfilesUsuario`     | Al cargar avatares de usuarios      |
| `ConsumoGenericoService`  | `consultarGenerico`  | `GET metodoGenerico?coleccion=Comunidades`         | Al inicializar                      |
| `ConsumoGenericoService`  | `consultarGenerico`  | `GET metodoGenerico?coleccion=Notificaciones`      | Polling periódico de notificaciones |
| `PutObjectService`        | `uploadFile`         | S3 / CloudFront                                    | Al adjuntar imagen o video          |
| `DecompressionService`    | `decompressZstd`     | —                                                  | Al procesar respuestas comprimidas  |

---

## Flujo principal

```
ngOnInit()
  -> cargarUsuario()          // Lee emailaddress y nombre de sessionStorage
  -> cargarConfiguracionMuro() // Banner y título del muro
  -> cargarPublicaciones()    // GET Publicaciones con Zstd, filtra activas
      -> cargarAvatares()     // GET PerfilesUsuario para enriquecer cada publicación
  -> cargarComunidades()      // GET Comunidades
  -> verificarPermisos()      // Consulta estados ACTIVO/INACTIVO y restricciones

// Al activar sonido en un video:
activarSonidoVideo(publicacion)
  -> verifica si usuarioId ya está en publicacion.clicsVideo[]
  -> si no está: registrarClicVideo(publicacion)
      -> construye clicsActualizados = [...clicsExistentes, nuevoClic]
      -> PUT Publicaciones con el array actualizado
      -> actualiza objeto local para refrescar el badge sin recargar
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                     |
| ---------- | ------------------ | ------------------------------------------------------------------------------------------ |
| 2026-06-10 | Adalberto González | Se agregó conteo de vistas por video: cada vez que un usuario activa el sonido de un video se guarda su email y la fecha en el campo `clicsVideo`. |

---

## Observaciones

- Actualmente, el módulo presenta un alto nivel de acoplamiento, concentrando gran parte de su funcionalidad en una única implementación que supera las 4.600 líneas de código.
