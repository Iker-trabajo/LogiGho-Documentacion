---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: componente compartido

# Componente: Filtro de fechas

**Selector:** `app-filtro-fechas`  
**Ubicación:** `src/app/shared/components/filtro-fechas/`  
**Dónde se usa:** [Panel comercial](../views/comercial/panel-comercial/panel-comercial.md) (fecha de registro) y [Seguimiento de clientes](../views/comercial/seguimiento-clientes/seguimiento-clientes.md) (registro en Personas, contrato en Tiendas)

---

## ¿Qué hace?

Un solo botón (📅 *Registro: Cualquier fecha*) que abre un menú con atajos y un rango personalizado. Entrega `desde` y `hasta` como `aaaa-mm-dd` en **hora de Colombia** (`America/Bogota`).

```html
<app-filtro-fechas etiqueta="Registro" [desde]="fechaDesde" [hasta]="fechaHasta"
  (cambio)="fechaDesde = $event.desde; fechaHasta = $event.hasta; filtrar()"></app-filtro-fechas>
```

| Entrada / salida | Tipo | Descripción |
| --- | --- | --- |
| `etiqueta` | `string` | Qué fecha se filtra ("Registro", "Contrato") |
| `desde` / `hasta` | `string` | Rango actual (`aaaa-mm-dd`); vacío = sin filtro |
| `cambio` | `{ desde, hasta }` | Cambió el rango |

---

## Atajos

| Atajo | Rango |
| --- | --- |
| Cualquier fecha | Sin filtro |
| Hoy | Hoy |
| Ayer | Ayer |
| Últimos 7 días | Hoy y los 6 anteriores |
| Últimos 30 días | Hoy y los 29 anteriores |
| Este mes | Del día 1 a hoy |
| Mes pasado | Mes anterior completo |

**Rango personalizado:** "Desde" y "Hasta". No deja elegir fechas futuras ni un rango al revés (si "desde" queda después de "hasta", se ajusta).

---

## Comportamiento

- El botón muestra el atajo elegido o el rango corto (`01/11/26 – 11/11/26`). Con filtro queda en azul y con una **✕** para quitarlo.
- Se cierra al presionar fuera, con Esc o al abrir otra lista (evento `logi-desplegable-abierto`, igual que [Lista de opciones](lista-opciones.md)).
- Abre a la izquierda si a la derecha no cabe. En celular los atajos y el rango van uno debajo del otro.

La pantalla compara el día de Colombia de cada registro con el rango; los registros sin fecha solo pasan si no hay filtro.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Creación: atajos y rango personalizado; reemplaza las dos cajas de fecha de Seguimiento |
