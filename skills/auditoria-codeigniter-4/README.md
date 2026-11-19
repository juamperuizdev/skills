# Skill: auditoria-codeigniter-4

Skill de auditoría de seguridad para aplicaciones PHP sobre CodeIgniter 4.

## Instalación

El skill en sí es solo documentación (no requiere instalación). Las comprobaciones de estilo y calidad de código opcionales sí necesitan, en el proyecto auditado, estas dependencias de desarrollo:

| Paquete Composer | Binario | Para qué |
|---|---|---|
| `squizlabs/php_codesniffer` | `vendor/bin/phpcs`, `vendor/bin/phpcbf` | Lint de estilo (PSR-12) |
| `codeigniter/coding-standard` (incluye `friendsofphp/php-cs-fixer`) | `vendor/bin/php-cs-fixer` | Estándar oficial de CodeIgniter 4 |

Instalación:

```bash
composer require --dev squizlabs/php_codesniffer codeigniter/coding-standard
```

El lint (`php -l`), las búsquedas (`grep`/`Select-String`), `git` y `composer audit` no necesitan dependencias adicionales.

El skill se descubre cuando el archivo está en una de las rutas que el agente vigila, con este formato exacto:

```
.agents/skills/auditoria-codeigniter-4/SKILL.md
```

Reglas que deben cumplirse:

- El directorio fuente debe ser `.agents/skills` (minúsculas).
- En subcarpetas, el archivo debe llamarse exactamente `SKILL.md`.
- El `description` del frontmatter es lo que hace que el agente lo descubra.
- El ID del skill se deriva de la ruta (en este caso `auditoria-codeigniter-4`), no del `name`.

## Uso

1. Invocar el skill por su ID (`auditoria-codeigniter-4`) o mencionarlo en el prompt.
2. Seguir el flujo general descrito en `SKILL.md` (bloques 1 a 7).
3. Para el análisis de estilo (opcional), disponer de `.php-cs-fixer.dist.php` en la raíz del proyecto; el ejemplo está en el Bloque 1.5 de `SKILL.md`.
4. Generar el informe en `writable/logs/auditoria-YYYY-mm-dd.md`.

En proyectos grandes, ejecutar primero el linter sobre `app/` y `tests/`, excluyendo `vendor/`, `public/` y `writable/`.

## Estructura

```
auditoria-codeigniter-4/
├── SKILL.md   # Instrucciones del skill (lo que carga el agente)
└── README.md  # Este archivo (uso humano, no se carga en el contexto)
```

## Salida

El informe se guarda en `writable/logs/auditoria-YYYY-mm-dd.md`. Al estar en `writable/`, que normalmente está en `.gitignore`, no se versiona.

La tabla de hallazgos incluye una columna `Check` que marca cada hallazgo con `✔️` (resuelto o aceptado) o `❌` (abierto o parcial), para un control visual rápido.