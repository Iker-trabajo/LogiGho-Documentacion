# Cómo documentar en este repo

## Regla principal

> Cuando terminas una funcionalidad, documentas. No después. En ese mismo momento.

## Pasos

1. Identifica a qué sección pertenece lo que desarrollaste
2. Abre el archivo `.md` correspondiente en `docs/`
3. Si no existe el archivo, créalo y agrégalo al `nav` en `mkdocs.yml`
4. Escribe la documentación (sin perfeccionismo — algo es mejor que nada)
5. Agrega la entrada al [Changelog](../changelog/index.md)
6. Commit

## ¿Qué documentar?

- **Qué hace** el servicio / componente / lambda
- **Cómo se usa** (parámetros, ejemplos)
- **Por qué** se hizo así (si la decisión no es obvia)
- **Dependencias** con otros servicios

## Lo que NO necesitas hacer

- Documentar cada línea de código
- Ser perfecto al escribir
- Esperar a tener todo listo para documentar

## Ver la documentación localmente

```bash
pip install mkdocs-material
mkdocs serve
```
