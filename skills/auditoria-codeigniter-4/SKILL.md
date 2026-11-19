---
name: auditoria-codeigniter-4
description: Audita la calidad y seguridad de aplicaciones CodeIgniter 4 + PHP.
version: 1.3.0
author: Juampe Ruiz
---

# Skill de Auditoría de Seguridad para CodeIgniter 4 + PHP

Auditoría de seguridad para aplicaciones PHP sobre el framework CodeIgniter 4. El objetivo es detectar brechas antes de que lleguen a producción, de forma sistemática y reproducible, en proyectos heredados y nuevos.

Este skill no se limita a un escaneo automático: combina comprobaciones manuales, reglas de código y verificación de dependencias. Nada de aquí está pensado para reemplazar un revisor humano.

Nota de versión: todas las comprobaciones se refieren a CodeIgniter 4 (no a CI3). Cuando un patrón de CI3 aparece en un proyecto heredado, se marca como hallazgo.

---

## Qué cubre

- **Sintaxis y calidad de código** — errores de sintaxis, uso incorrecto de API nativa de PHP y de CI4, nomenclatura.
- **Fugas de credenciales y secretos** — claves, tokens, passwords, API keys y secrets escritos en el código, en `.env` o en archivos versionados.
- **Código muerto, deprecated y duplicado** — métodos y variables no utilizadas, código repetido, APIs obsoletas de CI4 y de PHP.
- **CRUD y tratamiento de datos** — SQL sin parametrizar, entrada no filtrada, falta de validación, XSS, CSRF, subida de archivos, mass assignment.
- **Control de acceso y sesiones** — rutas, filtros, autenticación, cookies y ciclo de vida de la sesión.
- **Configuración y exposición** — entorno, debug, CSP, CORS, cifrado, hostnames, auto-routing.
- **Exploits conocidos y dependencias** — vulnerabilidades en PHP, MySQL, CI4 y librerías Composer.

## Qué no cubre

- Aplicaciones que no usan CodeIgniter 4.
- Auditorías de rendimiento y cobertura de tests. La comprobación de estilo y nomenclatura se cubre solo como calidad de código (Bloque 1), de forma opcional.
- Seguridad física de servidores, configuración de red o despliegue (DNS, TLS, firewall). Estos puntos se pueden tratar en bloques aparte si se necesita.
- El comportamiento de seguridad de las capas no PHP (JS, motor de base de datos, infraestructura).

---

## Requisitos

El skill en sí es documentación y no necesita instalación. Las comprobaciones base (lint con `php -l`, búsquedas con `grep`/`Select-String`, `git`, `composer audit`) no requieren dependencias adicionales.

Las comprobaciones de estilo y calidad de código (Bloque 1) usan herramientas de desarrollo que deben estar instaladas en el proyecto auditado:

| Herramienta | Paquete Composer | Binario | Para qué |
|---|---|---|---|
| PHP_CodeSniffer | `squizlabs/php_codesniffer` | `vendor/bin/phpcs`, `vendor/bin/phpcbf` | Lint de estilo (PSR-12); no incluye el estándar de CI4 |
| PHP-CS-Fixer | `friendsofphp/php-cs-fixer` (dependencia de `codeigniter/coding-standard`) | `vendor/bin/php-cs-fixer` | Aplica el estándar oficial de CodeIgniter 4 |
| Estándar CodeIgniter 4 | `codeigniter/coding-standard` | — | Reglas del estándar oficial para PHP-CS-Fixer |

Instalación (solo desarrollo):

```bash
composer require --dev squizlabs/php_codesniffer codeigniter/coding-standard
```

Consideraciones:

- El estándar de CI4 se distribuye para PHP-CS-Fixer, no como ruleset de PHP_CodeSniffer. Por eso `phpcs --standard=CodeIgniter4` no es válido (el paquete antiguo `codeigniter4/coding-standard` para PHPCS está archivado).
- Si el proyecto versiona `vendor/` para despliegue (entornos idénticos), estas herramientas viajarán a producción; es aceptable siempre que se impida su acceso por web (ver Bloque 7).

---

## Convenciones

### Severidad de los hallazgos

Cada hallazgo se clasifica con esta escala. No se usa ninguna otra etiqueta.

| Severidad | Criterio | Ejemplos típicos |
|---|---|---|
| Crítica | Explotable sin autenticación y con impacto directo en datos o en la ejecución | RCE, SQLi explotable, auth bypass, secreto de producción expuesto |
| Alta | Explotable con condiciones simples; afecta a datos, cuentas o integridad | XSS almacenado, SQLi autenticado, IDOR, subida de ficheros sin validar |
| Media | Requiere interacción o contexto; impacto limitado | CSRF ausente, XSS reflejado, sesión sin regenerar, cookies sin flags |
| Baja | Endurecimiento o buenas prácticas; impacto marginal | cabeceras ausentes, mensajes de error verbosos, `esc()` ausente en salida no privilegiada |

### Estado y marcado de los hallazgos

Cada hallazgo tiene un estado y una marca visual que resume de un vistazo si sigue pendiente. La marca ocupa una columna propia (`Check`) en la tabla de hallazgos, para mejor control visual.

| Marca | Estado | Significado |
|---|---|---|
| ✔️ | Resuelto | El hallazgo está corregido y verificado; no requiere acción. |
| ✔️ | Aceptado | El riesgo se ha aceptado formalmente y de forma documentada; no se corregirá. |
| ❌ | Abierto | El hallazgo está pendiente de corrección. |
| ❌ | Parcial | El hallazgo está mitigado solo en parte; sigue requiriendo acción. |

Reglas de marcado:

- `✔️` equivale a un estado `Resuelto` o `Aceptado`.
- `❌` equivale a un estado `Abierto` o `Parcial`.
- No se usan otras marcas. Si un hallazgo cerrado se reabre, vuelve a `❌`.
- El estado y la marca se actualizan en cada re-ejecución de la auditoría (informe incremental).

### Compatibilidad de comandos

El skill está pensado para ejecutarse en Linux/macOS y Windows. Para cada comprobación se indican las dos variantes cuando difieren. En lugar de depender del shell, se prefiere ejecutar PHP directamente cuando sea posible.

Rutas relevantes del proyecto (ajustar si procede): `app/`, `tests/`. Se excluyen siempre `vendor/`, `public/`, `writable/` y `node_modules/`.

---

## Flujo general

1. Preparar el entorno de auditoría (versiones, herramientas).
2. Ejecutar las comprobaciones de sintaxis (Bloque 1).
3. Buscar credenciales y secretos (Bloque 2).
4. Buscar código muerto, deprecated y duplicado (Bloque 3).
5. Auditar CRUD, entrada y salida de datos (Bloque 4).
6. Verificar versiones y dependencias (Bloque 5).
7. Auditar control de acceso y sesiones (Bloque 6).
8. Revisar configuración del framework y exposición (Bloque 7).
9. Recopilar resultados, priorizar por severidad y generar el informe.

---

## Bloque 1 — Sintaxis y calidad de código

### 1.1 Sintaxis PHP válida

Linux/macOS:

```bash
find app tests -name '*.php' -print0 | xargs -0 -n1 php -l
```

Windows (PowerShell):

```powershell
Get-ChildItem -Recurse -Include *.php app, tests | ForEach-Object { php -l $_.FullName }
```

Criterio de paso:

- Sin errores de sintaxis en ningún archivo.
- Si falla un archivo, marcarlo y revisarlo individualmente.

### 1.2 Uso correcto de funciones nativas de PHP

Buscar errores comunes:

- Uso incorrecto de `empty()`, `isset()`, `array_keys()`, `count()`.
- Confusión entre `==` y `===` en comparaciones de seguridad.
- Uso de `extract()` sobre datos de usuario.
- Uso de `eval()` con entradas no filtradas.
- Uso de `create_function()` o `assert()` con cadenas dinámicas.
- Concatenación de SQL en lugar de Query Builder.

Comprobar que no se usan funciones eliminadas o deprecadas en la versión de PHP del proyecto.

### 1.3 Uso correcto de CodeIgniter 4

- Uso correcto de Query Builder: `$this->db->table('…')->where(…)->get()`.
- Uso correcto de Models: `$MyModel->insert()`, `$MyModel->update()`.
- Si se usa Raw Query (`$this->db->query(…)`), verificar que los parámetros se pasan enlazados (por posición o por nombre), nunca interpolados en la cadena.
- Uso correcto de los helpers: `form`, `security`, `url`, `text`.
- No usar APIs de CI3 (ver Bloque 4.1). Síntomas: `Call to undefined method`, referencias a `CI_DB`, `$this->load`, `$this->input`.
- Uso correcto de los filtros y validadores (`CodeIgniter\Validation`).
- No usar `$this->request->getPost()` o `request()->getPost()` sin filtrar ni validar.
- Uso correcto de `esc()` (función global de CI4) con el contexto adecuado: `html`, `js`, `css`, `url`, `attr`.

### 1.4 Verificar la nomenclatura

- Métodos, clases, variables y constantes que no siguen el estándar.
- Nombres que no describan lo que hacen (evitar `x`, `dato`, `tmp` para algo con función concreta).
- Constantes en mayúsculas.
- Verificar la definición de constantes en `app/Config/Constants.php`.

Criterio de paso: no hay errores ortográficos en funciones o métodos; el código es consistente en nomenclatura y respeta el estándar.

### 1.5 Estilo y nomenclatura con el estándar oficial de CI4 (opcional)

El estándar oficial es `codeigniter/coding-standard`, que se aplica con PHP-CS-Fixer (no con PHP_CodeSniffer). En la raíz del proyecto se usa un archivo `.php-cs-fixer.dist.php` que limita el análisis a `app/` y excluye `Views` (mezclan HTML y PHP):

```php
<?php

use CodeIgniter\CodingStandard\CodeIgniter4;
use Nexus\CsConfig\Factory;
use PhpCsFixer\Finder;

$finder = Finder::create()
    ->files()
    ->in([__DIR__ . '/app'])
    ->exclude(['Views'])
    ->name('*.php');

return Factory::create(new CodeIgniter4(), [], [
    'finder'    => $finder,
    'cacheFile' => 'writable/cache/.php-cs-fixer.cache',
])->forProjects();
```

Comprobar sin modificar y aplicar:

```bash
vendor/bin/php-cs-fixer fix --dry-run --diff
vendor/bin/php-cs-fixer fix
```

Criterio de paso: el informe recoge los archivos que incumplen el estándar. Aplicar el `fix` es una decisión del equipo y queda fuera del alcance de la auditoría de seguridad.

---

## Bloque 2 — Fugas de credenciales y secretos

### 2.1 Buscar patrones de secretos

Linux/macOS:

```bash
grep -rniE "password|passwd|secret|token|api[_-]?key|apikey|client[_-]?secret|db[_-]?password|private[_-]?key|bearer|certificate|jwt" . \
  --include="*.php" --include="*.env" --include="*.json" --include="*.yml" --include="*.yaml" \
  --include="*.md" --include="*.txt" \
  --exclude-dir=vendor --exclude-dir=writable --exclude-dir=public --exclude-dir=node_modules
```

Windows (PowerShell):

```powershell
Get-ChildItem -Recurse -Include *.php,*.env,*.json,*.yml,*.yaml,*.md,*.txt |
  Where-Object { $_.FullName -notmatch '\\(vendor|writable|public|node_modules)\\' } |
  Select-String -Pattern 'password|passwd|secret|token|api[_-]?key|apikey|client[_-]?secret|db[_-]?password|private[_-]?key|bearer|certificate|jwt'
```

Ampliar los patrones según las integraciones reales del proyecto. Son habituales y suelen pasar desapercibidos: `firebase`, `service_account`, `shopify`, `woocommerce`, `mailchimp`, `stripe`, `paypal`, `smtp`, `ssh`, `oauth`.

En proyectos con integraciones de terceros, revisar también:

- Archivos JSON de credenciales (por ejemplo cuentas de servicio de Firebase/Google): deben estar fuera de `public/` y en `.gitignore`.
- Claves de API de SDKs (WooCommerce, Shopify, Mailchimp, etc.): nunca en el código, siempre en `.env`.

### 2.2 Revisar archivos `.env`

- El archivo `.env` debe estar ignorado en `.gitignore`.
- No debe haberse hecho commit.
- Las credenciales deben resolverse por variables de entorno, no por valores hardcodeados.
- Revisar que no haya credenciales reales en el código fuente.

### 2.3 Revisar archivos de configuración

- `app/Config/Database.php`
- `app/Config/App.php`
- `app/Config/Session.php`
- `app/Config/Cookie.php`
- `app/Config/Email.php`
- `app/Config/Encryption.php` (clave de cifrado)
- `app/Config/Boot/*`

Verificar que los valores por defecto no contengan secretos reales. Las claves deben inyectarse desde `.env` (por ejemplo `encryption.key`, `database.default.password`).

### 2.4 Verificar `composer.json` y `composer.lock`

- `composer.json` no debe contener credenciales en la sección `config`.
- `composer.lock` no debe versionar secretos.
- Si hay repositorios privados, el token debe estar en `COMPOSER_AUTH` o en el entorno, no en el archivo.

### 2.5 Verificar `phpunit.xml`, `.env.testing` y artefactos

- Buscar credenciales en archivos de tests.
- Estos archivos deben estar en `.gitignore`.
- Revisar artefactos comunes: `.env.*`, `*.pem`, `*.key`, `*.p12`, `service-account*.json`, `firebase*.json`.

---

## Bloque 3 — Código muerto, deprecated y duplicado

### 3.1 Buscar código muerto

- Métodos de clase que no se llaman.
- Variables y constantes asignadas pero no leídas.
- Ramas de código inalcanzables (condiciones que siempre son falsas).
- Propiedades `$this->algo` que nunca se usan.
- Métodos heredados que no se sobrescriben y nunca se usan.

Herramientas recomendadas (estáticas, mantenidas y compatibles con PHP 8.4):

- `phpstan analyse app --level=6` (con la extensión de CI4 si está disponible).
- `psalm --show-info=false`.
- Reglas de código sin uso: PHPStan con `--level` alto detecta variables/miembros no usados.

No usar `phpcpd` ni `phpdcd`: están archivados y sin mantenimiento; no funcionan de forma fiable en PHP 8.4.

Criterio de paso: se identifica y se elimina (o se documenta como deliberado) el código muerto.

### 3.2 Buscar deprecated

- Especificadores de versión de PHP incompatibles en `composer.json`.
- Funciones eliminadas en la versión de PHP del proyecto.
- Métodos de CI4 obsoletos (comparar con el changelog de la versión instalada).
- APIs de CI3 presentes en código (ver Bloque 4).

### 3.3 Buscar duplicados

- Bloques de código repetidos.
- Métodos que hacen lo mismo con distinto nombre.
- Misma función/ID repetida en distintos archivos.

La detección de duplicados es manual: no hay herramienta mantenida que los detecte de forma fiable en PHP 8.4 (`phpcpd` está archivado). Para estilo y nomenclatura, usar el estándar oficial (ver Bloque 1.5).

Criterio de paso: no hay código deprecated ni duplicados significativos sin justificación.

---

## Bloque 4 — CRUD, entrada y salida de datos

### 4.1 Consultas SQL

- Buscar `query(` con interpolación de variables (no enlazadas).
- Buscar `getWhere(` y `CI_DB`: son APIs de CI3, no de CI4 (hallazgo si aparecen).
- Buscar concatenación de strings en consultas.
- Buscar identificadores dinámicos sin lista blanca, que Query Builder NO escapa:
  - `->orderBy($input)` con columna procedente del usuario.
  - `->select("...$userInput...")`, `->groupBy()`, `->having()`.
- Verificar que las respuestas de `*_search`, filtros de tabla y ordenación validan contra listas permitidas.

### 4.2 Entrada de datos

- Buscar `getGet()`, `getPost()`, `getPut()`, `getVar()` sin filtrado ni validación.
- Verificar el uso correcto de `filter_var()` / `filter_input()`.
- Verificar que se aplican reglas de `Validation` de CI4 antes de usar la entrada.

### 4.3 Mass assignment (Models)

- Revisar `$allowedFields` en `app/Models/*`. Si está vacío, CI4 permite asignación masiva completa.
- Verificar que `$allowedFields` solo incluya campos que el usuario puede modificar (excluir `id`, `role`, `is_admin`, `created_at`, etc.).
- Revisar `$useTimestamps` y campos protegidos.

### 4.4 XSS y CSRF

- CSRF (CI4): revisar `app/Config/Security.php`.
  - `$csrfProtection` debe estar activo (`'cookie'` o `'session'`).
  - `$regenerate` recomendado en `true`.
  - `$tokenName`, `$headerName` coherentes con los formularios y JS.
  - El filtro `csrf` debe estar aplicado en `app/Config/Filters.php` a las rutas de escritura.
- XSS:
  - Buscar variables impresas en vistas sin `esc()` (atributos, HTML, JS, CSS, URL).
  - En CI4 no existe la función global `xss_clean()` de CI3; si aparece, es código heredado. El equivalente es `Security::xss_clean()` o, preferiblemente, `esc()` en la salida.
  - Verificar `Content-Security-Policy` (Bloque 7) como defensa en profundidad.
- Buscar `onInvalid` en filtros sin configuración segura.

### 4.5 Subida de archivos

- Buscar `move_uploaded_file()` y `$this->request->getFile()`.
- Verificar validaciones de extensión, MIME real y tamaño en `app/Config/Mimes.php`.
- Verificar el directorio de destino: no debe ser ejecutable ni accesible directamente con extensión de script.
- Buscar `copy()`, `rename()` o `system()` con entradas del usuario.

### 4.6 Inyección, LFI, RFI, SSRF

- SQLi: en cada ruta que lea o escriba datos.
- LFI/RFI: en cada ruta que incluya archivos (`include`, `require` con entrada de usuario).
- SSRF: en cada request externo (`CURLRequest`, `file_get_contents()` con URL del usuario). Validar destino contra lista blanca.

### 4.7 Evaluación por punto de inserción

Para cada punto de escritura de datos, verificar:

- Entrada validada.
- Salida escapada.
- Sin inyección.
- Sin exposición de datos sensibles.

Criterio de paso: toda entrada está validada y toda salida con datos de usuario está escapada; no hay SQLi, XSS, CSRF, LFI/RFI ni SSRF explotables.

---

## Bloque 5 — Exploits conocidos, versiones y dependencias

### 5.1 Versiones

```bash
php -v
composer show codeigniter4/framework
mysql --version   # o la versión que reporte el servidor de BD
```

- PHP: verificar compatibilidad con la versión de CI4 instalada.
- CI4: consultar el changelog de seguridad de la rama instalada.
- MySQL/MariaDB: verificar estado de soporte y CVEs conocidas.

### 5.2 Dependencias Composer

- `composer audit --locked` para vulnerabilidades conocidas.
- `composer show` para listar versiones de todos los paquetes.
- `composer outdated --direct`.
- Revisar paquetes críticos: `vlucas/phpdotenv`, `doctrine`, `symfony`, `monolog`, SDKs de terceros.

### 5.3 Riesgos de supply chain

- Dependencias fijadas a `dev-master` o ramas sin versión estable.
- Restricciones demasiado laxas (`^4.0`) que permiten saltos de versión mayores.
- Verificar que `composer.lock` esté versionado y que coincida con `composer.json`.

### 5.4 Buscar secretos en el repositorio

```bash
git log -p --all -S "password"
git log -p --all -G "(?i)(secret|token|api[_-]?key)"
git log --all --full-history -- "*.env" "*.pem" "*.key"
```

Herramientas específicas recomendadas: `gitleaks detect` o `trufflehog git file://.`.

Criterio de paso: versiones dentro de soporte, `composer audit` sin vulnerabilidades conocidas y sin secretos en el historial.

---

## Bloque 6 — Control de acceso y sesiones

### 6.1 Rutas y filtros

- Todas las rutas deben estar definidas en `app/Config/Routes.php` o en `app/Config/Routes/`.
- Buscar rutas duplicadas (mismo método HTTP + misma URI).
- Cada ruta debe apuntar a un controlador y método existentes.
- Verificar `app/Config/Filters.php`: qué filtros existen y cuáles están activos globalmente.
- Aplicar filtros a rutas privadas mediante `['filter' => '…']`. Usar el nombre real del filtro del proyecto (no asumir `logged-in`).
- Verificar que ninguna ruta privada quede sin filtro.
- Verificar el método HTTP: las escrituras deben ser POST/PUT/PATCH/DELETE, nunca GET.
- Eliminar rutas obsoletas o que apunten a código inexistente.

### 6.2 Autenticación

- Si el proyecto usa `codeigniter4/shield`, revisar su configuración y grupos.
- Si la autenticación es propia (no hay Shield), auditar la lógica de login:
  - Hashing de contraseñas (preferible el helper `password_hash`/`password_verify`, no MD5/SHA1).
  - `session_regenerate_id(true)` tras login correcto (evita fixation).
  - Limitación de intentos / bloqueo temporal (fuerza bruta).
  - Recuperación de contraseña: tokens de un solo uso con caducidad.
  - Logout: destruir la sesión y regenerar el ID.

### 6.3 Sesiones y cookies

- `app/Config/Session.php`:
  - `$driver` correcto (`FileHandler`, `DatabaseHandler`, `RedisHandler`…). Si es base de datos, verificar la tabla.
  - `$matchIP` (considerar `true` en entornos sensibles).
  - `$expiration` y `$timeToUpdate` razonables.
  - No almacenar datos sensibles en claro en la sesión.
- `app/Config/Cookie.php`:
  - `$httponly = true`, `$secure = true` en producción y `$samesite` configurado (`Lax` o `Strict`).
- Verificar que la sesión no se exponga en la URL (no usar `?session_id=`).

Criterio de paso: solo lo público es accesible sin sesión; las rutas privadas están filtradas; el login regenera el ID y las cookies tienen flags seguros.

---

## Bloque 7 — Configuración del framework y exposición

Comprobaciones sobre `app/Config/` y el entorno de ejecución:

- **Entorno** (`CI_ENVIRONMENT` en `.env`): en producción debe ser `production`. Nunca `development` en producción.
- **Errores** (`app/Config/Exceptions.php` y `.env`): en producción, sin trazas ni rutas en pantalla. Verificar el umbral de logging.
- **Debug Toolbar** (`app/Config/Toolbar.php`): no debe estar activa ni visible en producción.
- **Auto-routing** (`app/Config/Routing.php`): `$autoRoute` debe ser `false` salvo justificación. Si está activo, cualquier método público de un controlador puede quedar expuesto.
- **Hostnames** (`app/Config/App.php`): `$allowedHostnames` debe estar definido en producción para prevenir host header injection. (`app/Config/Hostnames.php` solo contiene constantes de TLDs de dos partes; no es esta configuración.)
- **Cifrado** (`app/Config/Encryption.php`): `$key` debe venir de `.env` (`encryption.key`), no por defecto. Verificar el `$driver`/`$cipher`.
- **CSP** (`app/Config/ContentSecurityPolicy.php`): revisar directivas (`defaultSrc`, `scriptSrc`, `reportOnly`). En producción conviene CSP activa.
- **CORS** (`app/Config/Cors.php`): `$allowedOrigins` no debe ser `*` con credenciales; revisar qué rutas aplican el filtro `cors`.
- **Honeypot** (`app/Config/Honeypot.php`): útil como protección de formularios si hay spam/bots.
- **`$baseURL`** (`app/Config/App.php`): debe ser la URL real (evita redirecciones abiertas y problemas de CSRF/cookies).
- **`$forceGlobalSecureRequests`** y `$proxyIPs`: revisar su coherencia con el despliegue.
- **Módulos** (`app/Config/Autoload.php`, `app/Config/Modules.php`): comprobar que no se autocargan módulos inesperados.

Criterio de paso: el entorno de producción no filtra errores ni rutas, el debug está apagado, el auto-routing desactivado y las cabeceras/cifrado configurados.

---

## Checklist ejecutivo

### Bloque 1 — Sintaxis y calidad de código

- [ ] `php -l` pasa en todos los archivos relevantes.
- [ ] Sin errores ortográficos en funciones/métodos.
- [ ] Funciones nativas de PHP usadas correctamente.
- [ ] Query Builder y filtros CI4 usados correctamente.
- [ ] Nomenclatura consistente.
- [ ] Estilo comprobado con el estándar de CI4 (PHP-CS-Fixer), si aplica.

### Bloque 2 — Fugas de credenciales

- [ ] Sin secretos literales en el código.
- [ ] `.env` en `.gitignore` y sin commitear.
- [ ] Credenciales resueltas desde variables de entorno.
- [ ] Artefactos de credenciales fuera de `public/` y no versionados.

### Bloque 3 — Código muerto, deprecated, duplicado

- [ ] Sin métodos ni variables no usados.
- [ ] Sin código deprecated.
- [ ] Sin duplicados significativos.

### Bloque 4 — CRUD, entrada y salida de datos

- [ ] Sin consultas SQL sin parametrizar.
- [ ] Identificadores dinámicos (`orderBy`, `select`) contra lista blanca.
- [ ] `$allowedFields` de los Models restringido.
- [ ] Entradas validadas y salidas escapadas.
- [ ] CSRF y XSS cubiertos.
- [ ] Subida de ficheros validada.
- [ ] Sin LFI/RFI, SSRF ni SQLi.

### Bloque 5 — Versiones y dependencias

- [ ] PHP, CI4 y MySQL dentro de soporte.
- [ ] `composer audit` sin vulnerabilidades conocidas.
- [ ] Sin dependencias `dev-master` ni restricciones excesivamente laxas.
- [ ] Sin secretos en el historial de git.

### Bloque 6 — Control de acceso y sesiones

- [ ] Rutas cubiertas, sin duplicados y con controlador/método existentes.
- [ ] Filtros de sesión aplicados a todas las rutas privadas.
- [ ] Login regenera el ID de sesión; logout destruye la sesión.
- [ ] Sin datos sensibles en sesión.
- [ ] Cookies con HttpOnly, Secure y SameSite.
- [ ] Timeout y `timeToUpdate` correctos.

### Bloque 7 — Configuración y exposición

- [ ] `CI_ENVIRONMENT=production` en producción.
- [ ] Sin trazas de error ni rutas en pantalla.
- [ ] Debug Toolbar desactivada.
- [ ] `$autoRoute` desactivado.
- [ ] `$allowedHostnames` definido.
- [ ] Clave de cifrado desde `.env`.
- [ ] CSP y CORS revisados.

---

## Herramientas y comandos recomendados

| Tarea | Linux/macOS | Windows (PowerShell) |
|---|---|---|
| Lint de PHP | `find app tests -name '*.php' -print0 \| xargs -0 -n1 php -l` | `Get-ChildItem -Recurse -Include *.php app, tests \| % { php -l $_.FullName }` |
| Buscar patrones de secretos | `grep -rniE "password\|secret\|token\|jwt" app app/Config` | `Select-String -Path app\*.php -Pattern 'password\|secret\|token\|jwt' -Recurse` |
| Versiones | `php -v && composer show codeigniter4/framework` | igual |
| Audit de Composer | `composer audit --locked` | igual |
| Entradas no validadas | `grep -rn "getPost()" app/Controllers` | `Select-String -Path app\Controllers\*.php -Pattern 'getPost\(\)' -Recurse` |
| Secretos versionados | `git log -p --all -S "password"` | igual |
| Análisis estático | `vendor/bin/phpstan analyse app` | igual |
| Estilo CI4 (PHP-CS-Fixer) | `vendor/bin/php-cs-fixer fix --dry-run --diff` | igual |
| Lint PSR-12 (PHP_CodeSniffer) | `vendor/bin/phpcs --standard=PSR12 app` | igual |
| Secretos (histórico) | `gitleaks detect` / `trufflehog git file://.` | igual |

Alternativas de análisis: PHPStan o Psalm para código muerto y tipos; PHP-CS-Fixer con el estándar oficial de CI4 (`codeigniter/coding-standard`) para estilo; PHP_CodeSniffer (PSR-12) como lint alternativo; `pygount` para contar líneas solo si el repo es grande.

---

## Pitfalls

1. **No confundir `getPost()` con validación.** CI4 no valida automáticamente; el desarrollador debe aplicarlo.
2. **`esc()` no es gratis.** Hay que usarlo en cada salida con el contexto correcto.
3. **Query Builder no protege de todo.** Los `where()` con arrays/bindings son seguros; los `query()` interpolados y los identificadores dinámicos (`orderBy`, `select`) no.
4. **El archivo `.env` no es un secreto.** Puede ser leído por cualquiera con acceso al servidor si los permisos están mal aplicados.
5. **Git no borra secretos.** Si un secreto se commiteó, hay que rotarlo y revocarlo, no solo eliminarlo del código.
6. **`composer audit` no reemplaza la revisión manual.** Una dependencia vulnerable no implica compromiso; depende de cómo se usa.
7. **Las reglas de seguridad de CI4 se aplican en el servidor, no en el cliente.** No confiar en el navegador para filtrar entradas.
8. **Ausencia de Shield no significa ausencia de auth.** En proyectos sin `codeigniter4/shield` la autenticación es propia y debe auditarse explícitamente.
9. **Restos de CI3 son hallazgos, no solo ruido.** `getWhere`, `CI_DB`, `csrf_protect`, `xss_clean`, `$this->load` indican código migrado a medias.

---

## Reporte de salida

Cada auditoría genera un archivo Markdown en `writable/logs/` con el nombre:

```
writable/logs/auditoria-YYYY-mm-dd.md
```

Donde `YYYY-mm-dd` es la fecha de ejecución de la auditoría (por ejemplo, `writable/logs/auditoria-2026-10-14.md`).

### Plantilla del informe

```markdown
# Auditoría CodeIgniter 4 — <proyecto>

- Fecha: YYYY-mm-dd
- PHP: <versión>
- CodeIgniter: <versión>
- MySQL/MariaDB: <versión>
- Alcance: <directorios y bloques revisados>

## Resumen

- Hallazgos totales: N (críticos: n, altos: n, medios: n, bajos: n)

## Comandos ejecutados

| Comando | Salida (resumen) |
|---|---|
| `php -l ...` | sin errores |

## Hallazgos

| # | Bloque | Severidad | Ubicación (archivo:línea) | Descripción | Recomendación | Estado | Check |
|---|---|---|---|---|---|---|---|
| 1 | 4 | Alta | app/Controllers/X.php:42 | ... | ... | abierto | ❌ |

## Checklist ejecutivo

- [x] Bloque 1 — ...
- [ ] Bloque 2 — ...

## Anexos

- Lista de archivos revisados.
- Salidas completas de los comandos relevantes.
```

### Reglas del informe

- Cada bloque debe dejar rastro de comprobación: comando ejecutado y su salida, o lista de archivos revisados.
- Cada hallazgo debe indicar archivo, línea, severidad, descripción y estado (`abierto`, `parcial`, `resuelto` o `aceptado`).
- La columna `Check` resume el estado con `✔️` (resuelto o aceptado) o `❌` (abierto o parcial), para un control visual rápido.
- El informe debe ser entendible por un desarrollador no experto en seguridad.
- El checklist ejecutivo se marca con `[X]` o `[ ]` por cada bloque.