---

## Autor: Adalberto González

Fecha creacion: 2026-05-26  
Estado: desarrollo

# Servicio: DireccionEstandarizadaService 

**Ubicación:** `src/app/components/direccion-estandarizada/direccion-estandar.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Convierte direcciones colombianas escritas de cualquier forma en una estructura organizada y fácil de entender con su nomenclatura correcta.

---

## Métodos

### `parse(raw: string): Partial<ParsedAddress>`

Punto de entrada principal. Normaliza el texto, detecta la zona (urbana/rural) y delega.

| Parámetro | Tipo     | Descripción                          |
| --------- | -------- | ------------------------------------ |
| `raw`     | `string` | Dirección en cualquier formato libre |

**Retorna:** Objeto con los componentes parseados, `direccionEstandarizada`, `confianza`, `correcciones` y `advertencias`.

---

### `buildAddress(f: Partial<ParsedAddress>): string`

Construye la cadena estandarizada a partir de un objeto de campos. Manda a `buildAddressRural` o `buildAddressUrbana` según `f.zona`.

| Parámetro | Tipo                      | Descripción                        |
| --------- | ------------------------- | ---------------------------------- |
| `f`       | `Partial<ParsedAddress>`  | Campos ya resueltos de la dirección |

**Retorna:** Cadena estandarizada lista para usar.

---

### `buildAddressUrbana(f: Partial<ParsedAddress>): string`

Construye la forma urbana: `TIPO_VIA NÚM_PPAL [CUADRANTE] # VÍA_GEN - ACCESO [COMPLEMENTOS] [BARRIO]`.

**Retorna:** Cadena estandarizada urbana.

---

### `buildAddressRural(f: Partial<ParsedAddress>): string`

Construye la forma rural: `VÍA_CONECTANTE NOMBRE_EJE KM n VEREDA nombre REFERENCIA`.

**Retorna:** Cadena estandarizada rural.

---

## Endpoints que consume

Este servicio no consume endpoints. Toda la lógica es local.

---

## Historial de cambios

| Fecha      | Autor              | Cambio              |
| ---------- | ------------------ | ------------------- |
| 2026-05-26 | Adalberto González | Creación del módulo |

---

## Observaciones

> Solo si hay algo no obvio: caché, manejo especial de errores, dependencias cruzadas.

- Esta acoplado totalmente a el manual entregado por inter. En caso de que nos toque adaptar el estandar a otra transportadora toca modificar el codigo.
