# LogiGho Docs

Repositorio de documentación técnica del sistema LogiGho.

## Estructura

```
docs/
├── arquitectura/     → Vision general, diagramas, decisiones técnicas
├── backend/          → Servicios .NET (Clean Architecture) y Lambdas Python
├── frontend/         → SitioLogiGho en Angular
├── api/              → Contratos, autenticación, manejo de errores
├── guias/            → Setup, variables de entorno, flujo de trabajo
└── changelog/        → Historial de cambios y nuevas funcionalidades
```

## Cómo ver la documentación localmente

```bash
pip install -r requirements.txt
mkdocs serve
```

Abre http://localhost:8000

## Cómo contribuir

1. Crea o edita el archivo `.md` correspondiente en `docs/`
2. Si es una sección nueva, agrégala al `nav` en `mkdocs.yml`
3. Haz commit con un mensaje descriptivo
