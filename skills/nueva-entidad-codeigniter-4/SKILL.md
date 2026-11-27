---
name: nueva-entidad-codeigniter-4
description: Genera el modelo, controlador, vistas de administración, rutas, menú y constantes para una nueva entidad partiendo de un migration o de un SQL en este proyecto CodeIgniter 4. Úsalo cuando pidan crear una entidad, un CRUD de administración o una sección nueva del backend.
---

# Nueva entidad (CRUD backend) en CodeIgniter 4

## Cuándo usar

El usuario pide crear una entidad nueva (por ejemplo `Client`) y/o dice «basándote en el migration/SQL y en otras entidades del proyecto». Sirve tanto si el migration ya existe como si sólo hay un `CREATE TABLE`.

## Contexto del proyecto (leer antes de escribir)

**No es Laravel.** Es CodeIgniter 4.7 / PHP 8.5. Equivalencias:

| Concepto | Ruta real |
|---|---|
| Modelo | `app/Models/<Entity>Model.php` |
| Controlador admin | `app/Controllers/Backend/<Entity>.php` |
| Vistas admin | `app/Views/backend/admin/<plural>/index.php` y `form.php` |
| Migration | `app/Database/Migrations/AAAA-MM-DD-HHMMSS_<Entity>.php` |
| Rutas | `app/Config/Routes/Backend.php` (requerido desde `app/Config/Routes.php`) |
| Validación | Dentro del Model (`$validationRules` / `$validationMessages`) |
| Auth | Filter `SessionFilter` aplicado al grupo `admin` |

Referencias a copiar el estilo: `News`, `Employment`, `Administrator` (modelo/controlador/vistas). `Employment` y `News` son los que tienen imagen.

## Flujo

1. **Explorar primero.** Busca el migration nuevo o el SQL con `git status`, `glob` y `grep`. Si hay que buscar patrones, usa un subagent `explore` con nivel `very thorough`.
2. **Analizar la fuente.** Extrae: nombre de tabla (`vc_...`), primary key, campos, tipos, `null`, defaults, y qué campos son texto libre, estado (`status TINYINT(1)`), imagen y unique.
3. **Decidir los filtros de texto.** Los campos de búsqueda textual van a `$parametrosLike`; el resto a `$parametros` (where exacto). `status` siempre va como filtro exacto.
4. **Crear en este orden:**
   1. Migration (si falta o está incompleta; p. ej. añadir `image`).
   2. Modelo `app/Models/<Entity>Model.php`.
   3. Constante `PATH_<ENTITY>` en `app/Config/Constants.php` si hay imagen/adjuntos.
   4. Controlador `app/Controllers/Backend/<Entity>.php`.
   5. Vistas `app/Views/backend/admin/<plural>/index.php` y `form.php`.
   6. Grupo de rutas en `app/Config/Routes/Backend.php`.
   7. Entrada en `app/Views/backend/admin/layout/menu.php` con `id="menu-<page>"`.
5. **Verificar siempre:**
   ```powershell
   php -l <cada archivo PHP creado/modificado>
   php spark routes | Select-String "<plural>"
   ```

## Detalles por capa

### Migration

- Usa `forge`, prefijo `vc_`, `status TINYINT(1) DEFAULT 1`, `created_at` / `updated_at` DATETIME nullable.
- Un campo imagen es `VARCHAR(100) NULL DEFAULT NULL` (ver `Employment`, `News`, `NewsCategory`).
- Si el usuario pide subida de imagen y el migration no tiene `image`, **añádelo**. Advierte que si ya se ejecutó hay que migrar de nuevo o añadir la columna a mano.
- Añade `addUniqueKey()` sólo si el usuario lo pide o si la validación `is_unique` tiene sentido en la BD.

### Modelo

- `$table`, `$primaryKey = 'id'`, `$useAutoIncrement = true`, `$returnType = 'object'`, `$useSoftDeletes = false`.
- `$allowedFields` con todos los campos editables, **sin** `id` y sin timestamps.
- `$useTimestamps = true`, `$createdField`, `$updatedField`, `$deletedField = ''`, `$skipValidation = false`.
- `$validationRules`: `id permit_empty|integer`, `status permit_empty|in_list[0,1]`, `name required|string|max_length[N]`, campos opcionales `permit_empty|string|max_length[N]`, `email ...|valid_email`, imagen `permit_empty|string|max_length[100]`.
- Unicidad: `is_unique[vc_tabla.campo,id,{id}]`. En `update()` hay que meter `$parametros['id'] = (int) request()->getPost('id')` porque la validación ocurre antes de `doProtectFields()`.
- Mensajes de validación en español, uno por regla como en `NewsModel`.

### Controlador

Propiedades: `$title`, `$parent`, `$page`, `$icon`, `$folder = 'backend/'`. El `$this->role` viene de `BaseController` (sesión) y las vistas se montan como `view($this->folder . $this->role . '/<plural>/index', $data)`.

Métodos obligatorios: `index()`, `search($pagina = NULL)`, `form($id = null)`, `save()`, `delete()`, `deleteImage()` (si hay imagen), `status()`.

- `search()`: excluye `orderBy`, `orderSort`, `registros` del POST; mueve los campos LIKE a `$parametrosLike`; aplica `where()->like()->orderBy()->findAll($limite, $inicio)`; calcula `countAllResults()`; genera paginación con `makeLinks(..., 'pagination')`. Devuelve `{status, message, object:{resultado,total,pagination}}`.
- **Orden por defecto** se pone en el `<select name="orderBy">` con `selected` y en `orderSort` con `ASC selected`. Añade además un fallback en el controlador: `$orderBy = request()->getPost('orderBy') ?: 'name'; $orderSort = request()->getPost('orderSort') ?: 'ASC';`.
- En el bucle de resultados, convierte cada campo a HTML: nombre como `<a href=".../form/">`, imagen con `<img class="image" style="max-width:60px; height:auto;">`, switch con `switchToggle('status', id, value)`, actions con `eliminar(id)`.
- `save()`: construye `$parametros` desde POST descartando `action`/`id`; vacío → `null`; default `status = 1`; sube imagen con `\App\Helpers\File::uploadSingle('image', ROOTPATH . PATH_<ENTITY>, null, ARCHIVOS_IMAGEN)`; `insert` / `update` con mensajes «Inserción correcta» / «Actualización correcta».
- `delete()`: acepta `id` escalar o array (borrado masivo).
- `deleteImage()`: `unlink(ROOTPATH . PATH_<ENTITY> . $item->image)` y `update($id, ['image' => null])`.
- `status()`: `update((int) post('id'), [post('switch') => post('value')])`.

### Vistas

`index.php`: script con `$(function(){ buscar(); ... })`, eventos `keypress`, `change` en `select` y click en `.pagination-link`; `buscar(pagina)` serializa `#form-busqueda` y llama a `base_url(uri_string()) . 'search[/pagina]'`; la paginación se lee de **`data.object.pagination`** (ojo: `Administrator/index.php` tiene el bug `paginacion`).

Filtros típicos: campos de texto (`name`, `contact`, `phone`, `email`), `status` con `App\Helpers\Filter::$status`, `orderBy`, `orderSort`, `registros` con `App\Helpers\Filter::$pagination`, botones `buscar()` y `restablecer_filtros()`.

Tabla: checkbox `id[]`, Id, Imagen, campos, switch, actions; botón «Nuevo X» a `admin/<plural>/new/`; botón `eliminar_masivo()`.

`form.php`: hidden `action` (`update`/`add`) e `id`; `guardar()` con `new FormData(form)`, valida `.required`, `#modal-loader`, POST a `admin/<plural>/save` con `processData:false, contentType:false` y redirige al listado.

**Formato preferido del usuario:** una sola columna de boxes apilados: primero el box **Datos** y debajo el box **Imagen**. Los campos dentro de Datos pueden seguir usando `<div class="columns">`. En el box Imagen: input file con `onchange="attachmentName('image')"` y `<span id="attachment-image">`, vista previa (ancho limitado) con `data-fancybox` y botón `eliminarImagen(id, archivo)`.

Selectize (ej. provincia `state`): el `<select>` se nutre de `App\Helpers\Country::$state` y se inicializa en el `$(function(){})` del form con `$('select[name="state"]').selectize({ create:false, maxItems:1, allowEmptyOption:true, sortField:'text', placeholder:'Seleccionar ...' })`. El CSS/JS ya se carga en `app/Views/backend/scripts.php`. Para provincias usa `Country::$state`; para países, `Country::$country`.

### Rutas

Dentro del `group('admin', ['filter' => 'SessionFilter'], ...)`:

```php
$routes->group('<plural>', [ 'filter' => 'SessionFilter' ], function($routes) {
    $routes->get('', 'Backend\<Entity>');
    $routes->get('new', 'Backend\<Entity>::form');
    $routes->get('(:num)/form', 'Backend\<Entity>::form/$1');
    $routes->post('save', 'Backend\<Entity>::save');
    $routes->post('delete', 'Backend\<Entity>::delete');
    $routes->post('delete-image', 'Backend\<Entity>::deleteImage'); // si hay imagen
    $routes->post('status', 'Backend\<Entity>::status');
    $routes->post('search/(:num)', 'Backend\<Entity>::search/$1');
    $routes->post('search', 'Backend\<Entity>::search');
});
```

`delete-image` **debe estar al nivel del grupo**: `common.js` hace POST a `window.location.href + '../../delete-image'` desde `.../<plural>/<id>/form/`.

### Menú y preferencias del usuario

Añade `<li><a id="menu-<page>" href="<?= base_url('admin/<plural>/') ?>"<?= $clase('<plural>') ?>>...` en `layout/menu.php`. El `footer.php` activa `$('#menu-<?= $page ?>')`.

Los iconos son Font Awesome (`fa-solid fa-...`). Si el usuario pide un icono concreto, ponlo tanto en el menú como en `$icon` del controlador.

## Trampas conocidas

- `permit_empty` con valor vacío **omite** el resto de reglas (incluido `is_unique`): verificado en `vendor/codeigniter4/framework/system/Validation/Validation.php::processPermitEmpty`. Los campos opcionales pueden ser `null` sin problema.
- `BaseModel::validate()` usa `cleanValidationRules`, por lo que `update()` parcial (`status`, `image`) no falla por reglas `required` de otros campos.
- `like($array)` combina los filtros con **AND**, igual que en `Administrator::search()`; es el comportamiento esperado en los listados.
- No uses `data.object.paginacion` (bug de `Administrator/index.php`); usa `pagination`.
- Si el usuario pide imagen múltiple, usa `uploadMultiple`; para una sola imagen, `uploadSingle`.

## Plantillas

Lee `references/plantillas.md` para los esqueletos de código (modelo, controlador, rutas, index y form) con marcadores `{{Entity}}`, `{{plural}}`, `{{PATH_CONST}}`, etc.
