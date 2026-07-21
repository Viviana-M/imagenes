# Contexto del proyecto — SECAR (Gestión de Obras)

> Documento de contexto para retomar el proyecto en cualquier sesión.
> Repositorio: `Viviana-M/gestion-obras-SECAR` (rama por defecto: `main`).
> Cárgalo/pásalo al asistente al iniciar para ponerlo al día del proyecto.

---

## 1. Qué es SECAR

Aplicación web interna de **SECAR** para la **gestión de obras** (proyectos de
construcción / mantenimiento / instalaciones), con foco fuerte en el
**control financiero y contable** de cada obra: carga de movimientos contables,
homologación de cuentas, distribución de costos, forecast operativo, cierre
contable de obras y reportes financieros.

## 2. Stack técnico

| Área | Tecnología |
|------|-----------|
| Framework | **Laravel 12** (PHP **8.2+**) |
| Vistas | **Blade** + **Tailwind CSS** (build con **Vite**) |
| Autenticación | **Laravel Breeze** (login con logo SECAR, recuperación de clave por correo con plantilla propia) |
| Excel | `maatwebsite/excel` (importar/exportar) |
| PDF | `barryvdh/laravel-dompdf` (reportes / planos) |
| Colas | Queue jobs (`QUEUE_CONNECTION=database`) |
| BD (dev) | **SQLite** por defecto (`DB_CONNECTION=sqlite`); sesión y caché en base de datos |
| Dev tooling | Pint (formato), Pail (logs), Sail, PHPUnit, Collision |

**Arranque local:** `composer setup` (instala, copia `.env`, genera key, migra,
`npm install`, `npm run build`) y luego `composer dev` (levanta `serve` +
`queue:listen` + `pail` + `vite` en paralelo).

## 3. Estructura de módulos

Rutas protegidas por middleware `auth` + `UsuarioActivo`. Los módulos viven bajo
namespaces de controlador: `Financiero`, `Contable`, `Operativo`, `Admin`,
`Auth` y raíz.

### 3.1 Financiero (`App\Http\Controllers\Financiero`)
- **Dashboard** financiero con modal de detalle y detalle por cuenta.
- **Históricos** (con detalle y detalle por cuenta).
- **Estados financieros**.
- **Comparativo**.
- Rutas: `financiero.dashboard`, `financiero.detalle`, `financiero.detalle.cuenta`,
  `financiero.historicos*`, `financiero.estados`, `financiero.comparativo`.

### 3.2 Contable (`App\Http\Controllers\Contable` + carga en `Financiero`)
- **Homologaciones**: mapeo del plan de cuentas **14 ↔ 61** (importar Excel,
  alta manual, editar, eliminar). Con vigencia.
- **Carga financiera**: importador **BIABLE** (movimientos contables), vía queue jobs.
- **Cierre de obras** contable (por Excel o manual).
- **Plano contable**: distribución **14 → 61** (habilitar, descargar).
- **Reclasificaciones** y **Plano de reversión**.

### 3.3 Operativo (`App\Http\Controllers\Operativo`)
- **Forecast operativo**: matriz de costos (guardar / enviar).
- **Distribución de costos**: reparto con **versiones** y **trazabilidad de planos**
  (borrador/enviado, edición habilitable, consulta de versiones).
- **Distribución de áreas** (módulo en desarrollo): reparto por bolsa,
  prefijos MTO/INS.
- **Maestro comercial**: ficha de proyectos (importar Excel / alta manual / editar).

### 3.4 Comercial
- **Cotizaciones**: pantalla base (aún sin lógica completa).

### 3.5 Administración (`App\Http\Controllers\Admin`, solo admin)
- **Usuarios**: alta/edición, activar/desactivar, roles y permisos por módulo.
- **UN / Bolsas** (`un_bolsas`): maestros por departamento (códigos INS.../MTO...).
- **Terceros de mano de obra**: maestro por departamento.

### 3.6 Perfil y preferencias
- Edición de perfil (Breeze) y preferencia de menú colapsado.

## 4. Roles, permisos y departamentos

- **Roles** (`users.rol`, enum): `financiero`, `operativo`, `comercial`,
  `contable`, `admin`. Default `operativo`.
- **Permisos por módulo** (`users.permisos_modulos`, JSON): nivel por módulo
  `ver` | `editar`. Es el esquema nuevo; convive con el viejo
  `modulos_permitidos` y con el rol como respaldo (ver `User::mapaPermisos()`).
  - `esAdmin()` → acceso total.
  - `puedeVerModulo($clave)` / `puedeEditarModulo($clave)` / `nivelModulo($clave)`.
- **Departamentos** (filtro de obra, no módulos): `dep_mantenimiento`,
  `dep_instalaciones`. Determinan qué **prefijos de código de obra** ve el usuario:
  - mantenimiento → `C, R, MO, GM`
  - instalaciones → `GI, O`
- `users.activo` (boolean) + middleware `UsuarioActivo` controlan el acceso.

## 5. Modelo de datos (tablas principales)

| Tabla / Modelo | Rol | Campos clave |
|----------------|-----|--------------|
| `users` / `User` | Usuarios, roles y permisos | `rol`, `sede`, `activo`, `modulos_permitidos`, `permisos_modulos`, `menu_colapsado` |
| `registro_financieros` / `RegistroFinanciero` | Movimientos contables por obra | `codigo_proyecto`, `cuenta_contable`, `valor_debito`, `valor_credito`, `mes`, `anio`, `origen` (biable/manual) |
| `homologaciones` / `Homologacion` | Mapeo cuentas 14↔61 | `cuenta_14` (única), `cuenta_61`, `estructura` (EQU-MAT-SUM, MOI…), vigencia |
| `ficha_proyectos` / `FichaProyecto` | Maestro comercial de obras | `codigo_proyecto` (único), `cliente`, `nombre_obra`, `valor_contratado`, `costo_estimado`, `margen_ofertado` %, `utilidad_ofertada`, responsables, `area` |
| `distribuciones` / `Distribucion` | Distribución de costos | `mes`, `anio`, `estado` (borrador/enviado), `edicion_habilitada`, `departamento` |
| `distribucion_versiones` / `DistribucionVersion` | Versionado de distribuciones | — |
| `distribucion_envios` / `DistribucionEnvio` | Envíos de distribución | — |
| `aplicaciones_costo` / `AplicacionCosto` | Aplicación de costos (ligada a distribución) | `distribucion_id` |
| `forecast_operativo` / `ForecastOperativo` | Matriz de costos forecast | campos operativos |
| `movimiento_contable` / `MovimientoContable` | Movimientos contables | — |
| `carga_financieras` / `cargafinanciera` | Cargas (importador BIABLE) | — |
| `proyectos_cerrados` / `ProyectoCerrado` | Cierre contable de obras | — |
| `saldos_balance` / `SaldoBalance` | Saldos de balance | — |
| `obras_estado` / `ObraEstado` | Estado de obras | — |
| `obra_clientes` / `ObraCliente` | Clientes por obra | — |
| `observaciones_obra` / `ObservacionObra` | Observaciones de obra | — |
| `un_bolsas` / `UnBolsa` | Maestro UN/bolsas | `codigo` (INS.../MTO...), `departamento`, `activo` |
| `terceros_mano_obra` / `TerceroManoObra` | Maestro terceros mano de obra | `departamento` |

**Convención de prefijos de código de obra:** `C, R, MO, GM` (mantenimiento),
`GI, O` (instalaciones); `MTO`/`INS` en bolsas.

## 6. Organización del código

```
app/
  Http/Controllers/{Financiero,Contable,Operativo,Admin,Auth}/...
  Http/Middleware/UsuarioActivo.php      # bloquea usuarios inactivos
  Models/...                              # ver tabla arriba
  Mail/ResetPasswordSecar                # correo de recuperación con diseño SECAR
resources/views/{financiero,contable,operativo,comercial,admin,auth,layouts,emails,components}/
  layouts/navigation.blade.php           # menú lateral (usa puedeVerModulo)
routes/
  web.php                                 # rutas de la app (todos los módulos)
  auth.php                                # rutas de autenticación (Breeze)
database/migrations/                      # 35 migraciones (esquema completo)
```

## 7. Estado actual (según historial de commits)

Trabajo reciente centrado en el **módulo de distribución de áreas**:
- Base del módulo: maestros, pantalla de reparto por bolsa, ajuste de prefijos MTO/INS.
- Fase 1: maestros de **UN/bolsas** y **terceros de mano de obra** en Administración.
- Participación sobre ingreso; ajuste de prefijo `O` en instalaciones.
- Distribución por departamentos, resumen Excel/PDF, trazabilidad de planos, roles de cargo.

Commits previos consolidaron: administración de usuarios con permisos por módulo,
módulos financieros divididos (activos/históricos/estados), cierre contable de
obras, dashboard financiero con modal, forecast operativo e importador BIABLE.

## 8. Notas / pendientes conocidos

- **Comercial → Cotizaciones**: pantalla base, sin lógica completa.
- **README** del repo es el genérico de Laravel (no describe SECAR); este archivo
  cumple ese rol de documentación funcional.
- Hay un binario `cloudflared.exe` (~54 MB) versionado en la raíz — probablemente
  para exponer el entorno local con un túnel; convendría sacarlo del repo
  (`.gitignore`) y no versionarlo.
- BD por defecto es SQLite (desarrollo); para producción se configuraría MySQL/otro
  vía `.env`.

---

_Última actualización de este contexto: 2026-07-15. Generado a partir de rutas,
controladores, modelos y migraciones del repositorio._
