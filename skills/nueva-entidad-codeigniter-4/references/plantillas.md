# Plantillas de código

Marcadores: `{{Entity}}` (Client), `{{entity}}` (client), `{{plural}}` (clients), `{{Title}}` (Cliente), `{{Titles}}` (Clientes), `{{icon}}` (fa-solid fa-suitcase), `{{PATH_CONST}}` (PATH_CLIENT), `{{table}}` (vc_client).

Sustituye y adapta los campos a los del migration/SQL. Mantén el estilo de 4 espacios y los docblocks.

---

## 1. Migration (si no existe o le falta `image`)

```php
<?php

namespace App\Database\Migrations;
use CodeIgniter\Database\Migration;

class {{Entity}} extends Migration
{
    public function up()
    {
        $this->forge->addField([
            'id' => [
                'type' => 'INT', 'constraint' => 10,
                'unsigned' => true, 'auto_increment' => true,
            ],
            'status' => [
                'type' => 'TINYINT', 'constraint' => 1,
                'unsigned' => true, 'default' => 1
            ],
            // ... resto de campos del SQL/migration ...
            'image' => [
                'type' => 'VARCHAR', 'constraint' => 100,
                'null' => true, 'default' => null
            ],
            'created_at' => [
                'type' => 'DATETIME', 'null' => true, 'default' => null
            ],
            'updated_at' => [
                'type' => 'DATETIME', 'null' => true, 'default' => null
            ]
        ]);

        $this->forge->addKey('id', true);
        $this->forge->createTable('{{table}}');
    }

    public function down()
    {
        $this->forge->dropTable('{{table}}');
    }
}
```

---

## 2. Modelo — `app/Models/{{Entity}}Model.php`

```php
<?php

namespace App\Models;
use CodeIgniter\Model;

class {{Entity}}Model extends Model {

    protected $table = '{{table}}';
    protected $primaryKey = 'id';
    protected $useAutoIncrement = true;
    protected $returnType = 'object';
    protected $useSoftDeletes = false;

    protected $allowedFields = [
        'status',
        // ... campos editables ...
        'image'
    ];

    protected $useTimestamps = true;
    protected $createdField = 'created_at';
    protected $updatedField = 'updated_at';
    protected $deletedField = '';

    protected $validationRules = [
        'id' => 'permit_empty|integer',
        'status' => 'permit_empty|in_list[0,1]',
        'name' => 'required|string|min_length[3]|max_length[100]',
        'email' => 'permit_empty|valid_email|max_length[100]|is_unique[{{table}}.email,id,{id}]',
        'image' => 'permit_empty|string|max_length[100]'
    ];

    protected $validationMessages = [
        'name' => [
            'required' => 'El nombre es obligatorio',
            'min_length' => 'El nombre debe tener al menos 3 caracteres',
            'max_length' => 'El nombre no puede exceder los 100 caracteres'
        ],
        'email' => [
            'valid_email' => 'El email no es válido',
            'is_unique' => 'Este email ya está registrado'
        ],
        'image' => [
            'max_length' => 'La imagen no puede exceder los 100 caracteres'
        ]
    ];

    protected $skipValidation = false;

}
```

---

## 3. Constante — `app/Config/Constants.php`

```php
defined('{{PATH_CONST}}') || define('{{PATH_CONST}}', 'public/uploads/{{plural}}/');
```

---

## 4. Controlador — `app/Controllers/Backend/{{Entity}}.php`

```php
<?php

namespace App\Controllers\Backend;
use App\Controllers\BaseController;
use CodeIgniter\HTTP\ResponseInterface;

class {{Entity}} extends BaseController {

    protected string $title = '{{Titles}}';
    protected string $parent = '{{Titles}}';
    protected string $page = '{{plural}}';
    protected string $icon = '{{icon}}';
    protected string $folder = 'backend/';

    /**
     * Función que muestra la página principal
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function index() : string {

        // Datos para la vista
        $data = [ 'title' => $this->title, 'page' => $this->page, 'icon' => $this->icon ];

        return view($this->folder . $this->role . '/layout/header', $data) .
            view($this->folder . $this->role . '/{{plural}}/index', $data) .
            view($this->folder . $this->role . '/layout/footer', $data);

    }

    /**
     * Función de búsqueda de elementos y paginación
     * @param Int $pagina Número de página
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function search($pagina = NULL) : ResponseInterface {

        try {

            // Models
            $MyModel = new \App\Models\{{Entity}}Model();

            // Establezco los parámetros de la búsqueda
            $parametros = [];
            $arrayExc = [ 'orderBy', 'orderSort', 'registros' ];
            foreach (request()->getPost() as $k => $v) { if ($v != '' && !in_array($k, $arrayExc)) { $parametros[$k] = $v; } }

            // Campos de búsqueda por texto
            $parametrosLike = [];
            foreach ([ 'name', 'contact', 'phone', 'email' ] as $campo) {
                if (isset($parametros[$campo])) { $parametrosLike[$campo] = $parametros[$campo]; unset($parametros[$campo]); }
            }

            // Orden por defecto
            $orderBy = request()->getPost('orderBy') ?: 'name';
            $orderSort = request()->getPost('orderSort') ?: 'ASC';

            // Paginación
            $limite = (request()->getPost('registros')) ? request()->getPost('registros') : 0;
            $inicio = (!isset($pagina)) ? 0 : ($pagina-1) * $limite;

            if (request()->getPost('registros') != "") {

                $uri = service('uri');

                $config['base_url'] = base_url($uri->getSegment(1).'/'.$uri->getSegment(2)) . "/";
                $config['per_page'] = request()->getPost('registros');

                (!isset($pagina)) ? $inicio = 0 : $inicio = ($pagina-1) * $config['per_page'];
                $limite = $config['per_page'];

            }

            // Realizo la búsqueda
            $resultado['resultado'] = $MyModel->where($parametros)->like($parametrosLike)->orderBy($orderBy, $orderSort)->findAll($limite, $inicio);
            if (!$resultado['resultado']) { throw new \Exception('Sin resultado(s)'); }
            $resultado['total'] = $MyModel->where($parametros)->like($parametrosLike)->countAllResults();

            if (isset($config['per_page']) && ($resultado['total'] > $config['per_page'])) {
                $pager = service('pager');
                $resultado['pagination'] = $pager->makeLinks(($pagina) ?? 1, $config['per_page'], $resultado['total'], 'pagination');
            }

            foreach ($resultado['resultado'] as &$item) {

                // Nombre
                $nombre = esc($item->name);
                $item->name = '<a href="'.base_url('admin/{{plural}}/'.$item->id.'/form/').'">'. $nombre .'</a>';

                // Imagen
                $item->image = !empty($item->image)
                    ? '<img class="image" style="max-width:60px; height:auto;" src="'.base_url({{PATH_CONST}} . $item->image).'" alt="'. $nombre .'">'
                    : '-';

                // Estado
                $item->status = ($item->status == 1)
                ? '<div class="cl-toggle-switch"><label class="cl-switch"><input type="checkbox" checked onclick="switchToggle(\'status\', ' . $item->id . ', 0)"><span></span></label></div>'
                : '<div class="cl-toggle-switch"><label class="cl-switch"><input type="checkbox" onclick="switchToggle(\'status\', ' . $item->id . ', 1)"><span></span></label></div>';

                // Actions
                $item->actions = '<a href="'.base_url('admin/{{plural}}/'.$item->id.'/form/').'" class="button is-small is-dark"><span class="icon"><i class="fa-solid fa-pen"></i></span></a> <a class="button is-danger is-small" onclick="javascript:eliminar('.$item->id.');"><span class="icon"><i class="fa-solid fa-trash"></i></span></a>';

            }

            return response()->setJSON([ 'status' => true, 'message' => $resultado['total'] . ' resultado(s) encontrados', 'object' => $resultado ]);

        } catch (\Exception $e) {
            return response()->setJSON([ 'status' => false, 'message' => $e->getMessage() ]);
        }

    }

    /**
     * Función que muestra el formulario de inserción y modificación de un elemento
     * @param Int $id Id del elemento a modificar
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function form($id = null) : string {

        // Models
        $MyModel = new \App\Models\{{Entity}}Model();

        // Datos para la vista
        $data['parent'] = $this->parent;
        $data['title'] = (isset($id)) ? 'Editar {{Title}}' : 'Nuevo {{Title}}';
        $data['page'] = $this->page;
        $data['icon'] = $this->icon;

        if (isset($id)) {
            $data['{{entity}}'] = $MyModel->find($id);
            if (!$data['{{entity}}']) { throw \CodeIgniter\Exceptions\PageNotFoundException::forPageNotFound('Elemento no encontrado'); }
        }

        return view($this->folder . $this->role . '/layout/header', $data) .
            view($this->folder . $this->role . '/{{plural}}/form', $data) .
            view($this->folder . $this->role . '/layout/footer', $data);

    }

    /**
     * Función para el guardado de un elemento
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function save() : ResponseInterface {

        try {

            // Models
            $MyModel = new \App\Models\{{Entity}}Model();

            // Establezco los parámetros del formulario
            $parametros = [];
            foreach (request()->getPost() as $k => $v) {
                if (in_array($k, [ 'action', 'id' ], true)) { continue; }
                $parametros[$k] = ($v !== '') ? $v : null;
            }

            // Valores por defecto de campos no nulos
            if (!isset($parametros['status']) || $parametros['status'] === null) { $parametros['status'] = 1; }

            // Imagen
            $file = request()->getFile('image');
            if ($file && $file->getError() !== UPLOAD_ERR_NO_FILE) {

                if (!$file->isValid() || $file->hasMoved()) {
                    return response()->setJSON([ 'status' => false, 'message' => 'La imagen no es válida' ]);
                }

                $imagen = \App\Helpers\File::uploadSingle('image', ROOTPATH . {{PATH_CONST}}, null, ARCHIVOS_IMAGEN);
                if (!$imagen) {
                    return response()->setJSON([ 'status' => false, 'message' => 'Error al subir la imagen. Compruebe el formato del archivo' ]);
                }

                $parametros['image'] = $imagen;

            }

            switch (request()->getPost('action')) {

                case 'update':

                    // Incluyo el id para que las reglas is_unique ignoren el propio registro
                    $parametros['id'] = (int) request()->getPost('id');

                    $resultado = $MyModel->update((int) request()->getPost('id'), $parametros);
                    if (!$resultado) { throw new \Exception($MyModel->errors() ? implode('<br/>', $MyModel->errors()) : 'Error en la actualización'); }
                    $mensaje = 'Actualización correcta';
                    break;

                default:

                    $resultado = $MyModel->insert($parametros);
                    if (!$resultado) { throw new \Exception($MyModel->errors() ? implode('<br/>', $MyModel->errors()) : 'Error en la inserción'); }
                    $mensaje = 'Inserción correcta';
                    break;

            }

            return response()->setJSON([ 'status' => true, 'message' => $mensaje ]);

        } catch (\Exception $e) {
            return response()->setJSON([ 'status' => false, 'message' => $e->getMessage() ]);
        }

    }

    /**
     * Función para la eliminación de un elemento seleccionado
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function delete() : ResponseInterface {

        // Models
        $MyModel = new \App\Models\{{Entity}}Model();

        $id = request()->getPost('id');
        if (!$id) { return response()->setJSON([ 'status' => false, 'message' => 'Elemento no encontrado' ]); }

        $arrayIds = [];
        if (!is_array($id)) { array_push($arrayIds, $id); } else { $arrayIds = $id; }

        foreach ($arrayIds as $key => $value) {

            $resultado = $MyModel->delete($value);
            if (!$resultado) { return response()->setJSON([ 'status' => false, 'message' => 'Error en la eliminación' ]); }

        }

        return response()->setJSON([ 'status' => true, 'message' => 'Eliminación correcta' ]);

    }

    /**
     * Función para eliminar la imagen asociada a un elemento
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function deleteImage() : ResponseInterface {

        // Models
        $MyModel = new \App\Models\{{Entity}}Model();

        $id = request()->getPost('id');
        if (!$id) { return response()->setJSON([ 'status' => false, 'message' => 'Elemento no encontrado' ]); }

        $item = $MyModel->find($id);
        if (!$item) { return response()->setJSON([ 'status' => false, 'message' => 'Elemento no encontrado' ]); }

        // Elimino el archivo físico del servidor
        if (!empty($item->image) && file_exists(ROOTPATH . {{PATH_CONST}} . $item->image)) { unlink(ROOTPATH . {{PATH_CONST}} . $item->image); }

        // Actualizo el registro eliminando la imagen
        $resultado = $MyModel->update($id, [ 'image' => null ]);
        if (!$resultado) { return response()->setJSON([ 'status' => false, 'message' => 'Error en la eliminación' ]); }

        return response()->setJSON([ 'status' => true, 'message' => 'Eliminación correcta' ]);

    }

    /**
     * Función para actualizar el estado de un elemento seleccionado
     * @author Juampe Ruiz
     * @version 1.0
     */
    public function status() : ResponseInterface {

        $MyModel = new \App\Models\{{Entity}}Model();
        $resultado = $MyModel->update((int) request()->getPost('id'), [ request()->getPost('switch') => request()->getPost('value') ]);
        if (!$resultado) { return response()->setJSON([ 'status' => false ]); }
        return response()->setJSON([ 'status' => true ]);

    }

}
```

---

## 5. Rutas — `app/Config/Routes/Backend.php`

```php
    // {{Titles}}
    $routes->group('{{plural}}', [ 'filter' => 'SessionFilter' ], function($routes) {
        $routes->get('', 'Backend\{{Entity}}');
        $routes->get('new', 'Backend\{{Entity}}::form');
        $routes->get('(:num)/form', 'Backend\{{Entity}}::form/$1');
        $routes->post('save', 'Backend\{{Entity}}::save');
        $routes->post('delete', 'Backend\{{Entity}}::delete');
        $routes->post('delete-image', 'Backend\{{Entity}}::deleteImage');
        $routes->post('status', 'Backend\{{Entity}}::status');
        $routes->post('search/(:num)', 'Backend\{{Entity}}::search/$1');
        $routes->post('search', 'Backend\{{Entity}}::search');
    });
```

---

## 6. Menú — `app/Views/backend/admin/layout/menu.php`

```php
            <li><a id="menu-{{plural}}" href="<?= base_url('admin/{{plural}}/') ?>"<?= $clase('{{plural}}') ?>><span class="icon"><i class="{{icon}}"></i></span>{{Titles}}</a></li>
```

---

## 7. Vista índice — `app/Views/backend/admin/{{plural}}/index.php`

```php
<script>

    $(function(){

        // Búsqueda inicial
        buscar();

        // Enviar el formulario de búsqueda al presionar la tecla Enter
        $(document).on('keypress', function(e){ if (e.which == 13){ e.preventDefault(); buscar(); } });

        // Enviar el formulario de búsqueda si cambia algún desplegable de la vista
        $(document).on('change', 'select', function(){ buscar(); });

        // Clic en paginación
        $(document).on('click', '.pagination-link', function(e){
            e.preventDefault();
            buscar($(this).attr('data-ci-pagination-page'));
            $('html, body').animate( { scrollTop : 0 }, 300 ); // Scroll hacia arriba
        });

    });

    function buscar(pagina) {

        let formData = $('#form-busqueda').serialize();
        let formUrl = (pagina) ? '<?= base_url(uri_string()) ?>search/' + pagina : '<?= base_url(uri_string()) ?>search';

        let callback = function (data, textStatus) {

            if (!data.status) {
                showToastify('error', data.message);
                $('.numero-resultados').html('0');
                $('.div-paginacion').html('');
                $('.table tbody').html('<tr><td colspan="99" class="has-text-centered has-text-weight-bold"><br/>'+data.message+'<br/>&nbsp;</td></tr>');
                return false;
            }

            let resultados = data.object.resultado;
            let append = '';

            // Limpio el contenido
            $('.numero-resultados').html(data.object.total);
            $('.table tbody').html('');

            // Paginación de resultados
            $('.div-paginacion').html('');

            if ( $('select[name="registros"]').val() != "" ) { $('.div-paginacion').html(data.object.pagination); }

            resultados.forEach(item => {

                append += `

                    <tr>

                        <td><input type="checkbox" name="id[]" value="${item.id}" /></td>
                        <td class="has-text-centered transparente">${item.id}</td>
                        <td class="has-text-centered">${item.image}</td>
                        <td>${item.name}</td>
                        <td class="has-text-centered">${item.status}</td>
                        <td class="has-text-centered">${item.actions}</td>

                    </tr>

                `;

            });

            $('.table tbody').html(append);

        };

        $.ajax({
            type: "POST",
            url: formUrl,
            dataType: "JSON",
            data: formData,
            success: callback,
            error: function (x, t, e) {
                $('.numero-resultados').html('0');
                $('.table tbody').html('<tr><td colspan="99" class="has-text-centered has-text-weight-bold"><br/>Error al realizar la búsqueda<br/>&nbsp;</td></tr>');
                showToastify('error', (x.responseJSON && x.responseJSON.message) ? x.responseJSON.message : 'Error al realizar la búsqueda');
            }
        });

        return false;

    }

</script>

<section class="section content">

    <!-- Filtros de búsqueda -->
    <div class="box">

        <form id="form-busqueda">

            <div class="columns">

                <div class="column">
                    <div class="field">
                        <label class="label is-small">Nombre</label>
                        <div class="control has-icons-left">
                            <input type="text" class="input is-small" name="name" placeholder="Nombre" />
                            <span class="icon is-left"><i class="fa-solid fa-search"></i></span>
                        </div>
                    </div>
                </div>

                <div class="column is-narrow">
                    <div class="field">
                        <label class="label is-small">Estado</label>
                        <div class="select is-small">
                            <select name="status">
                                <option value="">[ Todos ]</option>
                                <?php foreach (App\Helpers\Filter::$status as $k => $v) { ?>
                                    <option value="<?= $k ?>"><?= $v ?></option>
                                <?php } ?>
                            </select>
                        </div>
                    </div>
                </div>

                <div class="column is-narrow">
                    <div class="field">
                        <label class="label is-small">Ordenar por</label>
                        <div class="select is-small">
                            <select name="orderBy">
                                <option value="name" selected>Nombre</option>
                                <option value="created_at">Fecha de creación</option>
                            </select>
                        </div>
                    </div>
                </div>

                <div class="column is-narrow">
                    <div class="field">
                        <label class="label is-small">Dirección</label>
                        <div class="select is-small">
                            <select name="orderSort">
                                <option value="ASC" selected>Ascendente</option>
                                <option value="DESC">Descendente</option>
                            </select>
                        </div>
                    </div>
                </div>

                <div class="column is-narrow">
                    <div class="field">
                        <label class="label is-small">Resultados</label>
                        <div class="select is-small">
                            <select name="registros">
                                <option value="">[ Todos ]</option>
                                <?php foreach (App\Helpers\Filter::$pagination as $k => $v) { ?>
                                    <option value="<?= $k ?>"><?= $v ?></option>
                                <?php } ?>
                            </select>
                        </div>
                    </div>
                </div>

                <div class="column is-narrow">
                    <label class="label is-small">&nbsp;</label>
                    <div class="buttons">
                        <a class="button is-small is-dark" onclick="javascript:buscar();" title="Buscar"><span class="icon"><i class="fas fa-search"></i></span></a>
                        <a class="button is-small is-dark" onclick="javascript:restablecer_filtros();" title="Restablecer filtros"><span class="icon"><i class="fas fa-sync"></i></span></a>
                    </div>
                </div>

            </div>

        </form>

    </div>

    <div class="resultados">Total: <span class="numero-resultados"></span> resultados</div>

    <div class="box">

        <div class="buttons">
            <a class="button is-small is-success" href="<?= base_url('admin/{{plural}}/new/') ?>"><span class="icon"><i class="fas fa-plus"></i></span> <span>Nuevo {{Title}}</span></a>
        </div>

        <div class="table-container">
            <table class="table is-striped is-fullwidth is-narrow is-hoverable is-vcentered">
                <thead>
                    <tr>
                        <th class="has-text-centered is-narrow"><input type="checkbox" name="selectAll" /></th>
                        <th class="has-text-centered is-narrow">Id</th>
                        <th class="has-text-centered is-narrow">Imagen</th>
                        <th>Nombre</th>
                        <th class="has-text-centered is-narrow">Estado</th>
                        <th class="has-text-centered" width="100"></th>
                    </tr>
                </thead>
                <tbody>
                </tbody>
            </table>
        </div>

        <div class="buttons mt-6">
            <a class="button is-small is-danger" onclick="javascript:eliminar_masivo();"><span class="icon"><i class="fas fa-trash"></i></span> <span>Eliminar</span></a>
        </div>

    </div>

    <div class="div-paginacion"></div>

</section>
```

---

## 8. Vista formulario — `app/Views/backend/admin/{{plural}}/form.php`

Una sola columna: box **Datos** y debajo box **Imagen**. Si el campo `state` debe ser selectize, manten el bloque JS indicado.

```php
<script>

    $(function(){

        // Selectize de provincias, alimentado desde el helper Country
        $('select[name="state"]').selectize({
            create: false,
            maxItems: 1,
            allowEmptyOption: true,
            sortField: 'text',
            placeholder: 'Seleccionar ...'
        });

    });

    function guardar() {

        let form = $('#form').get(0);
        let formData = new FormData(form);

        // Comprobación de campos obligatorios
        let campos_vacios = $('#form .required').filter(function() { return $(this).val() === ""; }).length;
        if (campos_vacios > 0) { showToastify('error', 'Faltan campos obligatorios'); return false; }
        if ($('select[name="status"]').val() == '') { showToastify('error', 'Debe seleccionar el estado'); return false; }

        $('#modal-loader').addClass('is-active');

        let callback = function(data, textStatus) {

            $('#modal-loader').removeClass('is-active');

            if (data.status) {
                showToastify('success', data.message);
                setTimeout(function() { window.location.href = '<?= base_url('admin/{{plural}}/') ?>'; }, 500);
            } else {
                showToastify('error', data.message);
            }
        };

        $.ajax({
            type: "POST",
            enctype: "multipart/form-data",
            processData: false,
            contentType: false,
            cache: false,
            url: '<?= base_url('admin/{{plural}}/save') ?>',
            dataType: "JSON",
            data: formData,
            success: callback,
            error: function(x, t, e) { $('#modal-loader').removeClass('is-active'); showToastify('error', (x.responseJSON && x.responseJSON.message) ? x.responseJSON.message : 'Error al guardar'); }
        });

        return false;
    }

</script>

<section class="section content">

    <form id="form">

        <input type="hidden" name="action" value="<?= isset(${{entity}}->id) ? 'update' : 'add' ?>" />
        <input type="hidden" name="id" value="<?= isset(${{entity}}->id) ? ${{entity}}->id : '' ?>" />

        <div class="box p-6">

            <h3 class="has-text-weight-light has-text-primary">Datos</h3>

            <div class="columns">

                <div class="column is-narrow">
                    <div class="field">
                        <label class="label">Estado <span class="has-text-danger">*</span></label>
                        <div class="select is-fullwidth">
                            <select name="status">
                                <option value="">Seleccionar ...</option>
                                <?php foreach (App\Helpers\Filter::$status as $k => $v) { ?>
                                    <option value="<?= $k ?>" <?php if (isset(${{entity}}->status) && $k == ${{entity}}->status) { echo 'selected="selected"'; } ?>><?= $v ?></option>
                                <?php } ?>
                            </select>
                        </div>
                    </div>
                </div>

                <div class="column">
                    <div class="field">
                        <label class="label">Nombre <span class="has-text-danger">*</span></label>
                        <div class="control has-icons-left">
                            <input type="text" class="input required" name="name" value="<?= isset(${{entity}}->name) ? esc(${{entity}}->name) : ''; ?>" placeholder="Nombre" maxlength="100">
                            <span class="icon is-left"><i class="{{icon}}"></i></span>
                        </div>
                    </div>
                </div>

            </div>

            <!-- Más filas de columnas con el resto de campos del migration -->

            <div class="columns">

                <div class="column">
                    <div class="field">
                        <label class="label">Provincia</label>
                        <div class="control">
                            <select name="state" id="state" placeholder="Seleccionar ...">
                                <option value="">Seleccionar ...</option>
                                <?php foreach (App\Helpers\Country::$state as $k => $v) { ?>
                                    <option value="<?= esc($k) ?>" <?php if (isset(${{entity}}->state) && $k == ${{entity}}->state) { echo 'selected="selected"'; } ?>><?= esc($v) ?></option>
                                <?php } ?>
                            </select>
                        </div>
                    </div>
                </div>

            </div>

        </div>

        <div class="box p-6">

            <h3 class="has-text-weight-light has-text-primary">Imagen</h3>

            <div class="field">
                <label class="label">Imagen</label>
                <div class="file has-name">
                    <label class="file-label">
                        <input class="file-input" type="file" name="image" accept="image/*" onchange="attachmentName('image')">
                        <span class="file-cta">
                            <span class="file-icon"><i class="fas fa-upload"></i></span>
                            <span class="file-label">Seleccionar imagen...</span>
                        </span>
                        <span class="file-name" id="attachment-image"><?= !empty(${{entity}}->image) ? esc(${{entity}}->image) : 'Sin archivo'; ?></span>
                    </label>
                </div>
            </div>

            <?php if (!empty(${{entity}}->image)) { ?>
                <div class="field">
                    <label class="label">Imagen actual</label>
                    <a href="<?= base_url({{PATH_CONST}} . ${{entity}}->image) ?>" data-fancybox title="Ampliar imagen"><img style="max-width:200px;" src="<?= base_url({{PATH_CONST}} . ${{entity}}->image) ?>" alt="<?= esc(${{entity}}->name) ?>"></a>
                </div>
                <a class="button is-small is-danger" onclick="javascript:eliminarImagen(<?= ${{entity}}->id ?>, '<?= esc(${{entity}}->image) ?>');" title="Eliminar imagen"><span class="icon"><i class="fa-solid fa-trash"></i></span> <span>Eliminar imagen</span></a>
            <?php } ?>

        </div>

        <div class="buttons">
            <a class="button is-dark" onclick="javascript:guardar();"><span class="icon"><i class="fa-solid fa-save"></i></span> <span>Guardar</span></a>
            <a class="button" href="<?= base_url('admin/{{plural}}/') ?>" title="Volver"><span class="icon"><i class="fa-solid fa-arrow-left"></i></span> <span>Volver</span></a>
        </div>

    </form>

</section>
```
