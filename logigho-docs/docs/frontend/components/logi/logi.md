---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi, el asistente de LogiGho

**Autor:** Yadilton Torres

**Selector:** `app-logi-assistant`

**Ubicación:** `SitioLogiGho/src/app/components/logi-assistant`

**Rama:** `feature/logo-animado` (commit `553a7b71`). Aún no está en PreProd ni en producción.

---

## ¿Qué hace?

Logi es el robot que acompaña al usuario en todas las pantallas internas. Reemplaza el botón de WhatsApp y hace cinco cosas:

1. **Explica:** el menú lateral, los botones importantes y para qué sirve cada pantalla.
2. **Responde:** «Pregúntale a Logi» (**Ctrl + K**) tiene 39 guías paso a paso.
3. **Avisa:** errores, formularios incompletos, campos mal escritos, falta de internet, versión nueva, sesión por vencer y notificaciones nuevas.
4. **Reporta a soporte:** guarda los últimos 10 errores y los manda al equipo técnico con diagnóstico.
5. **Cambia de entorno:** muestra la opción «Cambiar entorno» (experiencias) cuando el servidor la habilita.

Nunca muestra al usuario texto técnico, y **solo habla de los módulos que el rol del usuario puede usar** (ver [Seguridad por rol](seguridad-por-rol.md)).

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Vive en el [layout](../../layout/default-layout.md), así que aparece en todas las rutas bajo `/app` |
| Lo que muestra | Guías, preguntas frecuentes, consejos y recorridos solo de los módulos del rol. Las ayudas generales (soporte, atajos, «No veo una opción») son para todos |

---

## Uso

```html
<app-logi-assistant
  whatsappUrl="https://wa.me/573176064197"
  whatsappMensaje="¡Hola! Necesito ayuda con LogiGho."
  [menu]="navItems"
></app-logi-assistant>
```

| Entrada | Tipo | Para qué |
|---|---|---|
| `whatsappUrl` | `string` | Enlace de WhatsApp de la opción «Escribir por WhatsApp» |
| `whatsappMensaje` | `string` | Mensaje que ya va escrito (si hubo un error, se le suma la pantalla y el error) |
| `modoSerio` | `boolean` | Apaga frases pícaras |
| `menu` | `ItemMenuLogi[]` | Menú lateral del usuario, ya filtrado por su rol. Logi solo navega a estas pantallas |

---

## Estructura de archivos

```
logi-assistant/
├── logi-assistant.component.ts / .html / .spec.ts → El robot: dibujo, burbuja, menú, buscador y reacciones
├── styles/
│   └── logi-assistant.component.scss              → Dron, burbuja, menú y animaciones
├── services/
│   ├── logi.service.ts (+ .spec)               → Puente entre la app y Logi (esperas, fallas, avisos)
│   └── logi-experiencias.service.ts (+ .spec)      → Cambiar entorno: GET y PUT /preferencias/mia
├── conocimiento/                               → Solo textos y datos
│   ├── logi-conocimiento.ts (+ .spec)          → ~140 pantallas: para qué sirven, consejos y errores típicos
│   ├── logi-guias.ts (+ .spec)                 → 39 guías de «Pregúntale a Logi»
│   ├── logi-tips.ts                            → Consejo al entrar a una pantalla
│   ├── logi-ayuda-menu.ts (+ .spec)            → Frase de cada opción del menú lateral
│   └── logi-botones.ts                         → Frase de los botones importantes
└── helpers/                                    → Lógica pura
    ├── logi-errores.rules.ts (+ .spec)         → Cómo explica cada alerta
    ├── logi-soporte.rules.ts (+ .spec)         → Reporte técnico y diagnóstico
    ├── logi-campos.rules.ts (+ .spec)          → Teléfono, correo, documento, dinero…
    ├── logi-recorrido.ts                       → «Muéstrame cómo» de la pantalla
    └── logi-formas.ts                          → Colores y formas del dibujo
```

Detalle de cada carpeta: [Conocimiento](conocimiento.md) · [Helpers](helpers.md) · [Servicios](services.md).

Fuera de `logi-assistant` también participan:

| Archivo | Qué hace |
|---|---|
| `core/application/services/guards/acceso-modulos.ts` | Regla de qué rutas puede abrir el usuario. Ver [Acceso a módulos](../../core/acceso-modulos.md) |
| `infrastructure/interceptors/logi-espera.interceptor.ts` | Avisa a Logi de peticiones lentas (más de 3 s) y fallidas. `LOGI_LA_EXPLICA` evita contar dos veces un error que la pantalla ya explica |
| `infrastructure/errores/logi-error.handler.ts` | Le pasa a Logi los errores de JavaScript sin manejar |
| `core/.../notificaciones/notificacion.models.ts` | Tipos `REPORTE_SOPORTE` y `SOPORTE_RESPONDIDO` |
| `layout/.../campana-notificaciones.component.ts` | Etiquetas del reporte en la campana (Módulo, Causa probable, Qué revisar…) |
| `scss/lared/_tokens.scss` | Colores del dron `--lg-logi-*` en noche y día |
| `src/index.html` | Pantalla de carga con el dron |

---

## Diseño: el dron

- Casco con visor, dos hélices que giran, antena con luz y un paquete que se mece como péndulo.
- Colores de **La Red**; cambia solo entre noche y día.
- **Luz de estado:**

| Luz | Significa |
|---|---|
| Naranja | Normal |
| Pulsa y las hélices aceleran | Está pensando (petición lenta) |
| Verde con ✓ en el paquete | Algo salió bien |
| Roja con «!» | Algo falló |
| Hélices lentas y una «z» | Dormido (1 min sin actividad) |

- En **modo básico** de La Red o con **movimiento reducido** todo queda quieto.

---

## Personalidad

- Tranquilo: sin travesuras ni frases al azar.
- Saluda una vez por sesión.
- Da un consejo solo la primera vez que entras a cada pantalla, con al menos **90 s** entre uno y otro.
- Prioridad de mensajes: un error no se tapa con otro mensaje.
- Cada mensaje dura según su largo, entre 3 y 15 s.
- La burbuja tiene una franja de color: **azul** consejo, **ámbar** aviso, **rojo** error y **verde** bien.

---

## Ayuda

| Dónde | Qué hace |
|---|---|
| Menú lateral | Explica cada módulo al pasar el mouse. Lee el nombre sin las etiquetas «Nuevo/Beta»; si no conoce un grupo, dice lo que tiene adentro |
| Botones importantes | Consultar guía, campana, modo día/noche, Generar guía, Exportar, Desembolso y otros |
| Pregúntale a Logi (Ctrl + K) | Busca pantallas del menú y guías con las palabras del usuario. Cada guía tiene «Llévame a esa pantalla» |
| ¿Qué hago aquí? | Para qué sirve la pantalla, sus consejos y sus preguntas |
| Muéstrame cómo | Recorrido por los botones de la pantalla actual |
| Modo silencio y WhatsApp | Desde el menú de Logi |

---

## Errores que detecta y explica

- Alertas (SweetAlert y toasts), fallas de servicios, errores de JavaScript y pantallas que no abren.
- Mensajes dentro de modales y pantallas (ej. «No se encontró la guía»).
- Formularios incompletos: dice cuáles campos faltan.
- Sin internet y cuando vuelve la conexión.
- Versión nueva publicada: pide recargar.
- Sesión por vencer: avisa 5 minutos antes.
- Notificación nueva en la campana.

---

## Soporte técnico

- En **«¿Qué pasó?»** están los últimos 10 errores con **Copiar** y **Enviar a soporte**.
- **Enviar a soporte** crea una notificación para el rol **Desarrollador** (igual que «Exportar datos») con: módulo, ruta, hora, origen, error técnico, lo que Logi le explicó al usuario, **causa probable**, **qué revisar**, y código HTTP, servicio y referencia cuando existen.
- La respuesta le llega al usuario como «Soporte técnico te respondió».
- La URL del servicio se guarda sin dominio ni parámetros (pueden llevar correos o teléfonos).

---

## Cambiar entorno

Fase F5 del plan de [Experiencias](../../experiencias/plan-implementacion.md):

- Lista que da el servidor, selector de tema y confirmación si hay cambios sin guardar.
- Cambia solo si el servidor confirma, y tiene un mensaje para cada error.
- Las pantallas pueden avisar cambios sin guardar con `registrarGuardia()` (hoy ninguna lo hace).
- `preferenciasActivas`: `false` en producción (dice «Muy pronto») y `true` en PreProd.

---

## Flujo principal

```
Layout carga el menú y los módulos
  → [menu]="navItems" y AccesoModulosService.registrar(todos, delUsuario)
Usuario entra a una pantalla
  → ¿el rol la puede abrir? ──no──► Logi calla (el layout lo saca a login)
        │ sí
  → primera vez y pasaron 90 s ► consejo de la pantalla
Usuario abre Logi o presiona Ctrl + K
  → preguntas frecuentes de la pantalla (solo de sus módulos)
  → escribe ► pantallas de su menú + guías de sus módulos
  → «Llévame» ► navega solo a URLs de su menú
Interceptor / manejador de errores
  → LogiService ► Logi explica en palabras ► guarda en «¿Qué pasó?»
```

---

## Pruebas

| Archivo | Pruebas |
|---|---|
| `logi-assistant.component.spec.ts` | Seguridad: buscador, preguntas frecuentes, «¿Qué hago aquí?» y guías abiertas directo |
| `services/*.spec.ts` | Esperas, fallas y avisos; Cambiar entorno apagado sin llamar al servidor |
| `conocimiento/*.spec.ts` | Rutas bien escritas, ids únicos, búsqueda sin tildes, rotación de consejos |
| `helpers/*.spec.ts` | Explicación de alertas, texto sin tecnicismos, diagnóstico por HTTP, revisión de campos |

Junto con `acceso-modulos.spec.ts` y `default-layout.component.spec.ts` son **74 pruebas en verde**.

```bash
npx ng test --watch=false --browsers=ChromeHeadless --include="src/app/components/logi-assistant/**/*.spec.ts" --include="src/app/core/application/services/guards/*.spec.ts" --include="src/app/layout/default-layout/default-layout.component.spec.ts"
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-24 | Yadilton Torres | Carpeta `logi-pet` → `logi-assistant`, `LogiMascotComponent` → `LogiAssistantComponent` y `LogiEntornosService` → `LogiExperienciasService` |
| 2026-11-24 | Yadilton Torres | Seguridad por rol; carpetas `services/`, `conocimiento/`, `helpers/` y `styles/`; pruebas unitarias |
| 2026-11 | Yadilton Torres | Diseño del dron, guías, recorridos, errores, reportes a soporte, ayuda en todo el menú y Cambiar entorno |

---

## Pendientes

- **Backend (Iker):** el autorizador de Cognito de `/preferencias/mia` y `/experiencias` en PreProd rechaza los tokens del pool `us-east-1_NtFz4uKGk`; sin eso, Cambiar entorno no carga.
- **Decisiones (Iker):** qué manda entre el modo día/noche de La Red y el `tema` del servidor; id de Mastershop; si cerrar sesión con 401; contradicciones de la documentación (R5, R6 y tabla de `beta`).
- **Nosotros:** probar la seguridad y «Enviar a soporte» en la app con usuarios de distintos roles; confirmar si el rol de soporte es Desarrollador.
