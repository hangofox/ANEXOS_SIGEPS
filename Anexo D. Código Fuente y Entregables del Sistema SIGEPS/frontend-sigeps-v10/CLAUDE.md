# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**SIGEPS** — Sistema Integrado de Gestión de Personal de Seguridad. An Angular 18 web application for managing security personnel, shift assignments, and access control. Backend is a separate Spring Boot API (not in this repo); reference copies of its controllers live under `src/app/services/controller/*.java` for checking endpoint/param shapes — these are not compiled by Angular.

## Commands

```bash
npm start                 # ng serve with proxy.config.json (proxies /api → http://localhost:8084) — use this over bare `ng serve`
npm run build:prod        # Production build → dist/sigepsv10/
ng test                   # Unit tests via Karma (Chrome)
ng test --include='**/empleados.service.spec.ts'   # Run a single spec file
ng generate component pages/nombre   # New page component (SCSS, non-standalone, skipTests by default)
```

The Angular CLI schematics are pre-configured (`angular.json`): all generated components, directives, and pipes use SCSS, are non-standalone (`standalone: false`), and skip test files by default.

## Architecture

### Module structure

This is an **NgModule-based** app (not standalone components). All declarations live in `AppModule` (`src/app/app.module.ts`), which by now imports dozens of feature components. There are no lazy-loaded feature modules — everything is eagerly loaded.

### Routing is mostly client-side, not `Router`-based

`AppRoutingModule` only defines three real routes: `/` (`IndexComponent`), `/inicio` (`InicioComponent`, guarded), and `/404`. Almost every "page" in the app (employee lists, user management, shifts, audits, etc.) is **not a route** — it's a component declared in `AppModule` and conditionally rendered inside `InicioComponent`'s template via `*ngIf="menuActivo === '...'"` (`src/app/pages/inicio/inicio.component.html`).

- `menuActivo` is a plain string on `InicioComponent`, changed by `setMenuActivo(seccion)`, which also calls `Location.replaceState()` to reflect the section in the URL bar **without** triggering Angular Router navigation (so refreshing restores the section from `location.path()` in `ngOnInit`).
- Before switching sections, call `SessionService.setContextoFuncionalidadAndRol(funcionalidad, rol)` to set the breadcrumb/permissions context.
- When adding a new feature section: declare the component in `AppModule`, add a `*ngIf` branch in `inicio.component.html` keyed to a new `menuActivo` string, and wire up the sidebar link to call `navegarA(...)`/`setMenuActivo(...)`. Do **not** add it to `AppRoutingModule`.

### CRUD component triad

Each manageable entity (empleados, usuarios, turnos, establecimientos-clientes, auditorias, etc.) follows the same three-component pattern under `src/app/pages/<dominio>/<entidad>/`:

- **`listado-*`** — the routed-to/menu-activated component. Owns search form, pagination (`paginaActual`, `tandaNumeroRegistrosporPagina`), status counters, and modal visibility state (`modalAddUpdDelVisible`, `modalVistaVisible`, `modalModo`, `<entidad>Seleccionado`). Renders `add-upd-del-*` and `vista-*` as modals, not separate routes, and passes the selected record down.
- **`add-upd-del-*`** — create/update/delete form for one record, opened as a modal from `listado-*` with a mode string (`'add'`/`'upd'`/`'del'`) and, for upd/del, the selected entity.
- **`vista-*`** — read-only detail view, also opened as a modal.

Child modals emit toast events (`{ tipo, mensaje }`) that `listado-*` collects in `recibirToast()` and shows/clears with a timer.

### Services

Feature services live under `src/app/services/<dominio>/<entidad>/` and follow a consistent REST-ish method naming convention against `environment.baseUrl`:

- `count(...)` — count matching records (filters + generic `keyword`)
- `lista(...)` / `listaPag(page, size, ...)` — unpaginated / paginated list; `listaPag` unwraps the Spring `Page` response's `.content`
- `guardar(entity)` / `modificar(entity)` / `eliminar(id)` / `consultar(id)` — create/update/delete/get-by-id, each returning a `Response<Entidad>DTO` shaped `{ mensaje: string, ... }`

- **`SessionService`** (`src/app/services/session/`) — single source of truth for reading/writing session state in localStorage (token, login flag, active nickname/functionality/role). Components should use this service rather than touching `localStorage` directly, though some older components still read `localStorage` inline in `ngOnInit`.
- **`TokensAutorizacionesService`** (`src/app/services/tokens-autorizaciones/`) — handles the `/login` POST to get a JWT token.

### Authentication flow

1. `/` renders `IndexComponent`, which hosts `<app-login>` (the `LoginComponent`)
2. `LoginComponent` validates a client-side captcha, then encodes the password with `btoa()` x10 before sending it to the API
3. On success, the JWT token and session flags are stored in `localStorage` (`tokenAutorizacion`, `isLoggedIn`, `nicknameUsuarioLogueado`, etc.)
4. `AuthInterceptor` (`src/app/interceptors/auth.interceptor.ts`) attaches the token as the `Authorization: Bearer` header on every HTTP request
5. `AuthGuardRoute` (`src/app/authguardsroute/auth-guard-route.guard.ts`) protects `/inicio` by checking `SessionService.isLoggedIn()`, which reads `localStorage`
6. Logout calls `SessionService.clearSession()`, which clears all `localStorage` and `sessionStorage`

### Environment configuration

`src/environments/environment.ts` exposes three URLs used across services:
- `baseUrl` — main backend API (proxied through the Angular dev server at `/api` locally via `proxy.config.json`, target `http://localhost:8084`)
- `baseUrlArchivosExternos` — file management server (port 8084)
- `urlServidorExpress` — Express middleware server (port 3000)

Always import from `environment` (not `environment.prod`) — the build system swaps the file at build time via `fileReplacements`.

### Interfaces

Grouped by domain under `src/app/interfaces/`, mirroring the `services/` and `pages/` domain folders (`gestion-personal`, `panel-control`, `paises-mundo`, `gestion-archivos`, etc.). Interface names use the suffix `I` (e.g., `EmpleadosI`, `ResponseTokenAutorizacionI`). Entity interfaces embed related lookup entities as nested `xxxDTO` fields (e.g., `EmpleadosI.tipoEmpleadoDTO: TipoEmpleadoI`). API mutation responses use a separate `response<Entidad>DTO.interface.ts` file.

### Page layout

- `IndexComponent` is a thin shell that composes `<app-login>` and `<app-pie-pagina>`; it contains no logic of its own
- `InicioComponent` is the post-login shell: sidebar menu, KPI cards, and — per the client-side routing above — the host for every feature section, switched via `menuActivo`
- `CabezoteComponent` and `PiePaginaComponent` are shared layout components declared in `AppModule`

### Reports (`pages/reportes-estadisticas`)

- `graficas-estadisticas` renders dashboards with `chart.js`.
- `reportes` builds PDF/Excel exports client-side with `pdfmake` and `xlsx` (+ `file-saver` to trigger the download). `pdfMake` is imported as a default singleton object (not destructured) because its methods rely on internal `this`; custom fonts are loaded once at module scope from `src/assets/fonts/fonts-vfs.js` (base64-encoded) via `addVirtualFileSystem`/`addFonts`.

### TypeScript strictness

`tsconfig.json` has `strict`, `strictTemplates`, `strictInjectionParameters`, `strictInputAccessModifiers`, `noImplicitReturns`, and `noPropertyAccessFromIndexSignature` all enabled — write new code (and component `@Input`s) to satisfy these rather than loosening them.
