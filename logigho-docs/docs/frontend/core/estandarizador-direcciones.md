---

## Autor: Adalberto González

Fecha creacion: 2026-05-26  
Ultima actualizacion: 2026-10-15  
Estado: desarrollo

# Servicio: DireccionEstandarizadaService 

**Ubicación:** `src/app/components/direccion-estandarizada/direccion-estandar.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Parsea direcciones colombianas en formato libre y las descompone en sus partes estandarizadas según el manual DANE/IGAC / Inter Rapidísimo. Detecta automáticamente si la dirección es urbana o rural y aplica el pipeline de parsing correspondiente. No consume ningún endpoint; toda la lógica es local.

También arma las direcciones que el [Constructor](../components/estandar-direcciones.md) construye campo por campo, incluidas las de **manzana, barrio o vereda** (sin `#`), con sus avisos y su confianza.

Zonas que maneja (`estandarDireccion.zona`):

| Zona | Cuándo | Ejemplo de resultado |
|---|---|---|
| `urbana` | Dirección con vía (`CL`, `KR`…). Si trae manzana y no placa, la manzana ocupa el lugar de la placa | `KR 43A # 73 - 77 AP 301` · `CL 3 MZ 1 LT 4 CS 6` |
| `rural` | Trae `KM`, `VEREDA`, `CORREGIMIENTO`… o una vía rural (`CT`, `AU`, `VI`…) | `AU MEDELLIN KM 12 VEREDA EL HATO` |
| `conjunto` | Texto libre que empieza por `MZ` o `LT` (urbanización sin calle) | `MZ 8 LT 2 CS 4 BARRIO EL POZON` |
| `sector` | Solo desde el Constructor, tarjeta "Manzana, barrio o vereda" | `VEREDA LA ESPERANZA CASA 5` |

---

## Constantes

| Constante | Contenido |
|---|---|
| `COMPLEMENTO_LABELS` | Orden de los complementos, de mayor a menor unidad: `MZ → LT → TO → BL → CS → PI → AP → IN → LC → OF → GJ → DP → BG`. Lo usan la zona urbana y la de conjunto, así la manzana nunca queda después de la casa |
| `TIPOS_LUGAR` (exportada) | Tipos de lugar de la zona `sector`. `alInicio` indica si el lugar abre la dirección (Vereda, Corregimiento, Finca, Parcelación, Batallón) o va al final como el barrio (Barrio, Urbanización, Conjunto, Sector) |

---

## Métodos

### `parse(raw: string): Partial<estandarDireccion>`

Punto de entrada principal. Normaliza el texto, detecta la zona (conjunto, rural o urbana) y delega.

| Parámetro | Tipo     | Descripción                          |
| --------- | -------- | ------------------------------------ |
| `raw`     | `string` | Dirección en cualquier formato libre |

**Retorna:** Objeto con los componentes parseados, `direccionEstandarizada`, `confianza`, `correcciones` y `advertencias`.

Con manzana:

- **Calle + manzana sin placa** (`CL 3 MZ 1 LT 4 CS 6`): no avisa "falta el distanciador"; lo que sobre después de una coma es el barrio. Avisa si falta el lote o la casa.
- **Sin calle** (`MZ 8 LT 2 CS 4, EL POZON`): el barrio se toma de la palabra `BARRIO` o de lo que va después de la primera coma. Si no hay barrio avisa que es obligatorio. Los `#`, `-`, `,` y `.` sueltos que dejaba el formulario viejo (`MZ 05 # LOTE 5 - ETAPA 4`) se quitan.

---

### `buildAddress(f: Partial<estandarDireccion>): string`

Construye la cadena estandarizada a partir de un objeto de campos. Manda a `buildAddressRural`, `buildAddressConjunto`, `buildAddressSector` o `buildAddressUrbana` según `f.zona`.

| Parámetro | Tipo                      | Descripción                        |
| --------- | ------------------------- | ---------------------------------- |
| `f`       | `Partial<estandarDireccion>`  | Campos ya resueltos de la dirección |

**Retorna:** Cadena estandarizada lista para usar.

---

### `buildAddressUrbana(f: Partial<estandarDireccion>): string`

Construye la forma urbana: `TIPO_VIA NÚM_PPAL [CUADRANTE] # VÍA_GEN - ACCESO [COMPLEMENTOS] [BARRIO]`.

**Retorna:** Cadena estandarizada urbana.

---

### `buildAddressRural(f: Partial<estandarDireccion>): string`

Construye la forma rural: `VÍA_CONECTANTE NOMBRE_EJE KM n VEREDA nombre REFERENCIA`.

**Retorna:** Cadena estandarizada rural.

---

### `buildAddressConjunto(f: Partial<estandarDireccion>): string`

Construye la forma de urbanización sin calle: `MZ n LT n [COMPLEMENTOS] [REFERENCIA] BARRIO nombre`.

**Retorna:** Cadena estandarizada de conjunto.

---

### `buildAddressSector(f: Partial<estandarDireccion>): string`

Construye la dirección de la tarjeta "Manzana, barrio o vereda": `[LUGAR si va al inicio] [CALLE REF] MZ n LT n CASA n KM n [LUGAR si va al final] [REFERENCIA]`.

| Datos | Resultado |
|---|---|
| Barrio Belén, MZ 8, LT 2, casa 4 | `MZ 8 LT 2 CASA 4 BARRIO BELÉN` |
| Vereda La Esperanza, casa 5 | `VEREDA LA ESPERANZA CASA 5` |
| Finca El Recreo, km 3 | `FINCA EL RECREO KM 3` |
| Barrio Olaya, calle de referencia Calle 3, MZ 1, casa 6 | `CALLE 3 MZ 1 CASA 6 BARRIO OLAYA` |

La casa se escribe completa (`CASA`), no `CS`, porque `CS` no está en las palabras clave que acepta el formulario de Base Ventas para estas direcciones.

**Retorna:** Cadena estandarizada, vacía si no hay datos.

---

### `advertenciasSector(f: Partial<estandarDireccion>): string[]`

Avisos de la dirección por lugar: falta el nombre del lugar, o no hay ningún número (manzana, lote, casa, km o calle de referencia). Sin número, Base Ventas la rechaza.

**Retorna:** Lista de advertencias, vacía si está completa.

---

### `calcConfianzaSector(f: Partial<estandarDireccion>): number`

Confianza de la dirección por lugar: nombre del lugar 40 + al menos un número 40 + sin avisos 20.

**Retorna:** Número de 0 a 100.

---

## Confianza

| Zona | Cómo se calcula |
|---|---|
| `urbana` | Vía 20 + número 25 + generadora 25 + placa 20 + sin avisos 10. Con calle + manzana sin placa, la manzana vale lo de la generadora y el lote o la casa lo de la placa |
| `rural` | Vía conectante 20 + KM 35 + vereda 30 + sin avisos 15 |
| `conjunto` | Manzana 25 + lote o casa 20 + **barrio 45** + sin avisos 10. Sin barrio queda por debajo de 50 (roja) |
| `sector` | Ver `calcConfianzaSector` |

---

## Endpoints que consume

Este servicio no consume endpoints. Toda la lógica es local.

---

## Pruebas

`direccion-estandar.service.spec.ts` (17 pruebas): direcciones normales que no deben cambiar, calle + manzana, sin calle con y sin barrio, limpieza de direcciones del formulario viejo y los ejemplos de la guía de Base Ventas para la dirección por lugar.

```bash
npx ng test --include=src/app/components/direccion-estandarizada --watch=false --browsers=ChromeHeadless
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio              |
| ---------- | ------------------ | ------------------- |
| 2026-05-26 | Adalberto González | Creación del módulo |
| 2026-10-15 | Yadilton Torres    | Manzana en texto libre (calle + manzana y sin calle con barrio), orden de complementos de mayor a menor, zona `sector` con `TIPOS_LUGAR`, `buildAddressSector`, `advertenciasSector` y `calcConfianzaSector`, limpieza de `#` y `-` sueltos y pruebas |

---

## Observaciones

> Solo si hay algo no obvio: caché, manejo especial de errores, dependencias cruzadas.

- Esta acoplado totalmente a el manual entregado por inter. En caso de que nos toque adaptar el estandar a otra transportadora toca modificar el codigo.
- Las direcciones que llegan muy dañadas en texto libre (`MZ Mz 23`, `LOTE - 30`, rellenos `NA`, "Manzaba") salen sin `#` ni `-` sueltos, pero pueden quedar con datos repetidos. El Constructor siempre arma la dirección limpia.
- En texto libre, el barrio solo se reconoce con la palabra `BARRIO` o después de una coma; `URBANIZACION X` queda como referencia y aparece el aviso de barrio.
- **Pendiente:** `buildAddressSector` y el barrio de la urbana no quitan símbolos no permitidos (`( ) / ! , .`) ni tildes de lo que escribe la persona.