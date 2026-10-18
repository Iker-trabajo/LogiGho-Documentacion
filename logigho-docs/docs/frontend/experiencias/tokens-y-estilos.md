---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: estándar de estilos

# Tokens y estilos

Un **token** es una variable CSS con significado: `var(--lg-color-primario)` en vez de `#1e40af`. Todas las
experiencias definen **los mismos nombres** de tokens con **sus propios valores**. Así, cualquier componente neutro
o vista en respaldo se ve integrada en cualquier experiencia.

## Estado actual (2026-10-17)

| Hecho | Dato |
|---|---|
| Paleta existente | `src/scss/_theme.scss` define `--logigho-*` |
| Usos de la paleta | 3.450 `var(--logigho-*)` |
| Colores escritos a mano | 6.020 hexadecimales en 308 archivos SCSS |
| Más repetidos | `#fff` (755), `#3d4fd6` (328), `#1e40af` (294), `#0e2c46` (265), `#ccc` (252) |
| Fuera de la paleta | `#3d4fd6` y `#121b60` se usan cientos de veces y no están en `--logigho-*` |

## Niveles

| Nivel | Ejemplo | Quién lo usa |
|---|---|---|
| **Primitivo** | `--logigho-azul: #1e40af` | Solo los archivos `_tokens.scss` de cada experiencia |
| **Semántico** | `--lg-color-primario`, `--lg-superficie`, `--lg-texto` | Todo el código de componentes y vistas |
| **De componente** | `--lg-tarjeta-fondo` | Un componente del UI kit, apuntando a semánticos |

## Cómo se aplica una experiencia

El motor pone el atributo `data-experiencia` en `<html>`. Cada experiencia define sus valores dentro de ese
selector:

```scss
// experiencias/clasica/estilos/_tokens.scss
:root[data-experiencia='clasica'] {
  --lg-color-primario: var(--logigho-azul);
  --lg-color-primario-hover: var(--logigho-azul-oscuro);
  --lg-color-acento: var(--logigho-naranja);
  --lg-fondo: var(--logigho-fondo);
  --lg-superficie: var(--logigho-blanco);
  --lg-borde: var(--logigho-borde);
  --lg-texto: var(--logigho-texto);
  --lg-texto-suave: var(--logigho-texto-claro);
  --lg-exito: var(--logigho-verde);
  --lg-alerta: var(--logigho-naranja-claro);
  --lg-peligro: var(--logigho-rojo);
  --lg-radio-md: 12px;
  --lg-espacio-md: 16px;
  --lg-fuente-base: Roboto, 'Helvetica Neue', sans-serif;

  // Puente con CoreUI: sus componentes siguen la experiencia
  --cui-primary: var(--lg-color-primario);
  --cui-body-bg: var(--lg-fondo);
  --cui-body-color: var(--lg-texto);
}

:root[data-experiencia='clasica'][data-tema='oscuro'] {
  // Solo se reasignan semánticos
}
```

Los archivos `_tokens.scss` de todas las experiencias se incluyen en el estilo global (son pequeños). Los estilos
del shell, del UI kit y de las vistas viven en sus componentes y se cargan solo con la experiencia activa.

## Lista oficial de tokens semánticos

Se mantiene en código, en `core/experiencias/tokens-semanticos.ts` (un arreglo con los nombres), junto con una
prueba que verifica que cada experiencia registrada define todos los tokens de la lista. Una experiencia con tokens
incompletos no pasa CI. Esta página documenta el significado de cada token cuando la lista se apruebe.

## Reglas

1. En componentes y vistas **solo** tokens semánticos o de componente. Nunca primitivos ni hex.
2. Sin `!important` y sin `::ng-deep` nuevo.
3. Espacios, radios y tipografía también desde tokens.
4. Contraste mínimo WCAG AA (4.5:1 para texto normal) en cada experiencia y tema.
5. Fuentes empaquetadas en el proyecto (como hoy con `@fontsource`), no desde CDN.

## Cómo se hace cumplir

Stylelint con `color-no-hex` y `declaration-no-important`, activo en `experiencias/**`, `views/**/experiencias/**`
y en todo archivo nuevo. El código viejo queda exento hasta que se migre, para no bloquear el trabajo diario.

## Decisiones pendientes

| Tema | Pendiente |
|---|---|
| `#3d4fd6` y `#121b60` | ¿Se agregan a la paleta o se unifican con colores existentes? |
| Tema oscuro en `clasica` | ¿Se ofrece desde el inicio? |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
