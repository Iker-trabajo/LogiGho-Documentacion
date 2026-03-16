---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Nombre de la Lambda

**Autor:** Nombre Apellido  
**Runtime:** Python 3.x  
**Trigger:** API Gateway | S3 | SQS | EventBridge | Otro  
**Ubicación en AWS:** `nombre-funcion-lambda`

---

## ¿Qué hace?

Descripción breve en 2-3 líneas.

---

## Estructura de archivos

```
nombre-lambda/
├── handler.py          ← punto de entrada
├── requirements.txt    ← dependencias
└── utils/
    └── helpers.py
```

---

## Handler principal

### `handler(event, context)`

**Descripción:** Punto de entrada de la Lambda.

**Evento de entrada:**
```json
{
  "campo": "valor"
}
```

**Proceso:**
1. Paso 1
2. Paso 2

**Respuesta exitosa:**
```json
{
  "statusCode": 200,
  "body": "..."
}
```

**Respuestas de error:**

| Código | Descripción |
|---|---|
| 400 | Datos inválidos |
| 500 | Error interno |

---

## Variables de entorno requeridas

| Variable | Descripción | Ejemplo |
|---|---|---|
| `NOMBRE_VAR` | Para qué se usa | `valor-ejemplo` |

---

## Dependencias

```
# requirements.txt
libreria==version
```

---

## Trigger y configuración

| Configuración | Valor |
|---|---|
| Timeout | — segundos |
| Memoria | — MB |
| Trigger | — |
| VPC | Sí / No |

---

## Changelog de la Lambda

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |

---

## Observaciones

- Observación 1
