# Diagramas — Conversaciones y Mensajes Pancake

Fuente Mermaid (`.mmd`) de todos los diagramas de este proceso. Se editan **acá**, no directamente en las páginas — las páginas los incluyen vía `--8<--` (`pymdownx.snippets`), así 1 solo archivo alimenta cualquier página que lo referencie y el `git diff` de un cambio de diagrama queda claro y aislado.

| Archivo | Tipo | Dónde se usa | Qué muestra |
|---|---|---|---|
| `contexto-c4.mmd` | Flowchart (estilo contexto/C4 nivel 1) | [Visión general](../overview.md) | Nivel más alto: quién usa el sistema, con qué sistemas externos habla (Pancake, EventBridge) y dónde persiste (Mongo, MySQL). Se usa `flowchart` en vez del tipo nativo `C4Context` de Mermaid porque ese renderer es rígido y poco legible (cajas fijas chiquitas, sin control de tipografía) |
| `orquestacion-estados.mmd` | State diagram (`stateDiagram-v2`) | [Orquestación](../orquestacion.md) | Los estados reales del Step Function `Step_Conversaciones_Pancake`, 1:1 con la definición ASL (mismos nombres de estado) |
| `flujo-sincronizar-conversaciones.mmd` | Flowchart | [Lambda: Sincronizar Conversaciones](../../lambdas/ApiLambdaSicronizarConversacionesPancake/ApiLambdaSincronizarConversacionesPancake.md) | Flujo de datos dentro de 1 invocación (1 página): watermark, paginado, regeneración de token |
| `flujo-sincronizar-mensajes.mmd` | Flowchart | [Lambda: Sincronizar Mensajes](../../lambdas/ApiLambdaSincronizarMensajes/ApiLambdaSincronizarMensajes.md) | Flujo de datos con el paralelismo (80 tareas), el intercalado por página y la coordinación de tokens |

## Cómo editar un diagrama

1. Editar el `.mmd` correspondiente en esta carpeta (sintaxis [Mermaid](https://mermaid.js.org/)).
2. No hace falta tocar la página que lo incluye — el `--8<--` lee el archivo en cada build.
3. Verificar con `mkdocs build --strict` (falla si el `--8<--` apunta a un archivo que no existe, gracias a `check_paths: true` en `mkdocs.yml`).

## Cómo agregar un diagrama nuevo

1. Crear el `.mmd` acá con el contenido Mermaid puro (sin las comillas de bloque, eso lo pone la página que lo incluye).
2. En la página destino, agregar un bloque de código `mermaid` cuyo contenido sea la directiva de inclusión (`snippets`, 2 guiones + `8<` + 2 guiones, seguido de la ruta entre comillas relativa a `docs/`) apuntando al `.mmd` nuevo — copiar el patrón exacto de cualquiera de los bloques ya usados en [Visión general](../overview.md) u [Orquestación](../orquestacion.md) y solo cambiar el nombre de archivo.
3. Agregar la fila correspondiente a la tabla de arriba.

> ⚠️ Por eso este archivo **no muestra la directiva literal** como ejemplo: el preprocesador de `snippets` la lee de cualquier `.md`, incluso dentro de bloques ilustrativos, e intenta resolverla — si el archivo de ejemplo no existe, rompe el build.

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-08 | Iker Acevedo | Creación de los 4 diagramas iniciales (contexto C4, estados de orquestación, flujo de datos de ambas lambdas). |
