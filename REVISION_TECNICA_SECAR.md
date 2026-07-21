# Revisión técnica de SECAR — correcciones aplicadas

Resultado de la revisión del repositorio `Viviana-M/gestion-obras-SECAR`.
Las correcciones seguras están en el parche **`secar-correcciones.patch`** (en la
raíz de este repo). Abajo: qué se corrigió, cómo aplicarlo y qué queda pendiente.

---

## ✅ Corregido en el parche (validado con `php -l`)

### Bugs / correctness
- **PSR-4:** `app/Models/cargafinanciera.php` → `CargaFinanciera.php` (rompía el
  autoload en Linux; afectaba `/contable/carga`).
- **PSR-4:** `app/jobs/financiero/` → `app/Jobs/Financiero/` (la subida de archivo
  financiero fallaba con *Class not found*).
- **`ForecastController::guardar`**: se quitó la clave `'estado'` duplicada y el
  `?? 0` muerto tras `floatval()`.
- **Login roto:** el rol `contable` redirigía a `/contable/dashboard` (ruta
  inexistente → 404). Ahora va a `/dashboard`.

### Seguridad / permisos
- **`UnBolsaController` y `TerceroManoObraController`**: se añadió `soloAdmin()`
  (mismo patrón que `UsuarioController`) en `index/store/update/toggle`. Antes
  cualquier usuario autenticado podía crear/editar registros.
- **Guardias de edición** (`puedeEditarModulo`) en los mutadores que no validaban:
  - Homologaciones: `guardar`, `actualizar`, `eliminar`, `importar` (`contabilidad`)
  - Maestro Comercial: `importar`, `guardar`, `actualizar`, `eliminar` (`operacion`)
  - `PlanoContable::habilitar`, `PlanoReclasificacion::marcar` (`contabilidad`)
  - `Forecast::guardar`, `enviar` (`operacion`)

### Rendimiento
- **Migración de índices** (`2026_07_15_000000_agregar_indices_rendimiento.php`)
  para `registro_financieros` — la tabla central que hoy se escanea completa en
  cada pantalla — y `forecast_operativo`.

### Limpieza / duplicación
- Mapa de prefijos por departamento consolidado en
  `User::PREFIJOS_DEPARTAMENTO` (antes duplicado dentro del propio modelo).
- Eliminada la vista huérfana `resources/views/financiero/carga.blade.php`.

---

## Cómo aplicar el parche

En tu copia del repositorio de SECAR:

```bash
cd gestion-obras-SECAR
git checkout -b fix/revision-tecnica

# Opción A (recomendada, conserva el commit):
git am < secar-correcciones.patch
# Opción B (solo aplica los cambios al working tree):
git apply secar-correcciones.patch

# Tras aplicar:
php artisan migrate          # crea los índices
composer dump-autoload       # refresca el autoload tras los renombres
```

Revisa, prueba en local y súbelo cuando estés conforme.

---

## ⏳ Pendiente (recomendado, requiere probar con la app corriendo)

No se incluyó en el parche porque son cambios más grandes o que conviene validar
ejecutando la app y las pruebas — no a ciegas:

1. **Gating de solo-lectura por módulo.** Los `index`/consultas no revalidan
   `puedeVerModulo` en el servidor (el menú los oculta, pero se accede por URL).
   Se puede añadir igual que los guardias de edición; es un cambio de
   comportamiento, por eso se dejó para confirmar.
2. **Extraer un `DistribucionService`** desde `DistribucionCostosController`
   (810 líneas). Además: mover el parseo de Excel de `CierreObrasController` a una
   clase `Import`, e introducir Form Requests y Policies.
3. **Reubicar módulos** para que namespace = ruta = permiso:
   `CargaFinancieraController` → `Contable\`; crear namespace `Comercial\` y mover
   ahí `MaestroComercialController` + convertir el closure `cotizaciones` en
   controlador.
4. **Consolidar duplicación restante:** `limpiarNumero()` y `clasificarCuenta()`
   (3 imports) en un trait; el mapa de estructuras de costo (`EQU-MAT-SUM, MOI…`)
   en un enum/constante compartida (el propio código advierte que las 3 copias
   "deben coincidir o clasifica mal sin avisar"); componente `<x-flash>` y helper
   de meses para las vistas.
5. **Rendimiento (2ª ola):** eliminar N+1 en las importaciones (Homologaciones,
   Maestro Comercial, Cierre de Obras) precargando con `keyBy`; `insert()` masivo
   al guardar la distribución; acotar con `whereIn` las agregaciones de
   reclasificación/reversión.
6. **Repo:** sacar `cloudflared.exe` (~54 MB) del control de versiones.

Puedo hacer cualquiera de estos en una siguiente iteración — idealmente con el
repo de SECAR dentro del alcance de la sesión para probarlo end-to-end.

---

_Generado el 2026-07-15 a partir de la revisión del código._
