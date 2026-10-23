---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: estándar obligatorio

# Crear una vista

Aplica a **toda vista nueva** desde que se apruebe el estándar, y a toda vista existente que se migre
(ver [Migración de vistas](migracion-vistas.md)).

## 1. Orden de trabajo

Primero la lógica, después lo visual:

1. `models/`: tipos de la vista.
2. `repository/`: llamadas al API (con las reglas de [seguridad por diseño](seguridad-por-diseno.md)).
3. `helpers/`: reglas puras con sus pruebas.
4. `state/`: store y facade, con pruebas del facade.
5. `<vista>.page.ts` y `<vista>.experiencias.ts`.
6. `experiencias/clasica/`: la versión obligatoria.
7. Versiones para otras experiencias, cuando el diseño exista.
8. Ficha de documentación con la [plantilla](../../plantillas/PLANTILLA-VISTA-EXPERIENCIAS.md).

## 2. Ruta

```ts
{
  path: 'pedidos',
  loadComponent: () => import('./pedidos/pedidos.page').then(m => m.PedidosPage),
  data: { title: 'Pedidos', vista: 'tienda.pedidos' },
}
```

La URL es la misma en todas las experiencias. Una experiencia nunca agrega ni cambia rutas.

## 3. Registro de versiones

```ts
// pedidos.experiencias.ts
import { VersionesVista } from '@core/experiencias/experiencias.models';

export const VERSIONES_PEDIDOS: VersionesVista = {
  vista: 'tienda.pedidos',
  versiones: {
    clasica: () => import('./experiencias/clasica/pedidos-clasica.component').then(m => m.PedidosClasicaComponent),
    // Se agrega una línea por cada experiencia que tenga versión propia de esta vista.
  },
};
```

El tipo `VersionesVista` exige `clasica` y solo acepta ids de experiencias registradas, así que un error de
escritura no compila:

```ts
// core/experiencias/experiencias.models.ts
export type CargadorComponente = () => Promise<Type<unknown>>;

export interface VersionesVista {
  vista: string;
  versiones: { clasica: CargadorComponente } & Partial<Record<ExperienciaId, CargadorComponente>>;
}
```

## 4. Página (host)

```ts
// pedidos.page.ts
@Component({
  selector: 'app-pedidos-page',
  imports: [NgComponentOutlet, CargandoVistaComponent],
  providers: [PedidosStore, PedidosFacade], // el cerebro vive y muere con la página
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (version(); as componente) {
      <ng-container *ngComponentOutlet="componente" />
    } @else {
      <app-cargando-vista />
    }
  `,
})
export class PedidosPage {
  protected readonly version = versionActiva(VERSIONES_PEDIDOS);

  constructor() {
    inject(PedidosFacade).iniciar();
  }
}
```

`versionActiva()` devuelve la versión de la experiencia activa o, si no existe, la `clasica` (respaldo).
La página casi no tiene código y es igual en todas las vistas.

## 5. Facade y versión visual

```ts
// state/pedidos.facade.ts
@Injectable()
export class PedidosFacade {
  private readonly repository = inject(PedidosRepository);
  private readonly store = inject(PedidosStore);

  // Lecturas: las versiones visuales solo leen
  readonly cargando = this.store.cargando;
  readonly filas = this.store.filasVisibles;
  readonly resumen = this.store.resumenPorEstado;
  readonly permisos = this.store.permisos;

  // Acciones: las versiones visuales solo llaman
  async iniciar(): Promise<void> { /* repository -> store */ }
  aplicarFiltros(cambio: Partial<FiltrosPedidos>): void { /* ... */ }
  seleccionarEstado(estado: EstadoPedido | null): void { /* ... */ }
}
```

```ts
// experiencias/clasica/pedidos-clasica.component.ts
@Component({
  selector: 'app-pedidos-clasica',
  imports: [ClasicaTarjetaComponent, ClasicaTablaComponent],
  templateUrl: './pedidos-clasica.component.html',
  styleUrl: './pedidos-clasica.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class PedidosClasicaComponent {
  protected readonly pedidos = inject(PedidosFacade);
  /** Estado solo visual. */
  protected readonly filtrosAbiertos = signal(false);
}
```

## 6. Prohibiciones en una versión visual

Una carpeta `experiencias/<id>/` de una vista **no puede**:

1. Inyectar `Repository`, `Store`, `HttpClient`, `ConsumoGenericoService` ni servicios de datos.
2. Leer o escribir `sessionStorage` o `localStorage`.
3. Decidir permisos con nombres de rol. Usa `facade.permisos()`.
4. Calcular reglas de negocio (totales, conteos, filtros, formatos de moneda). Van en `helpers/`.
5. Usar colores fijos (`#hex`, `rgb()`), `!important` o `::ng-deep` nuevo.
6. Importar archivos de otra experiencia.
7. Usar `innerHTML` o `bypassSecurityTrust*` con datos del usuario o del API.

## 7. Pruebas mínimas

| Qué | Cómo |
|---|---|
| `helpers/` | Una prueba por regla, sin TestBed |
| `state/*.facade.ts` | Con repository falso: iniciar, cada acción, errores |
| Cada versión visual | Se monta con un facade falso y pinta sin errores |
| Registro | Cada cargador resuelve a un componente |

## 8. Checklist de terminado

- [ ] La ruta usa `<vista>.page.ts` y declara `data.vista`.
- [ ] Existe `experiencias/clasica/`.
- [ ] Ninguna versión rompe las prohibiciones de la sección 6 (ESLint en verde).
- [ ] Las reglas de negocio están en `helpers/` con pruebas.
- [ ] Los permisos de la vista se calculan en un solo lugar del facade.
- [ ] El SCSS usa solo tokens (Stylelint en verde).
- [ ] Cumple la [seguridad por diseño](seguridad-por-diseno.md).
- [ ] Ficha de la vista creada y entrada en el changelog.

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
