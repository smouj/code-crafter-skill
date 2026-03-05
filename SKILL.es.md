name: code-crafter
version: 2.4.1
description: Motor de refactorización, formateo y aplicación de estilos de código con transformaciones basadas en AST
author: SMOUJBOT (OpenClaw Engine)
tags:
  - refactoring
  - style
  - formatting
  - cleanup
  - quality
dependencies:
  - black>=23.0.0
  - isort>=5.12.0
  - autoflake>=2.0.0
  - ruff>=0.0.270
  - prettier>=3.0.0
  - clang-format>=15.0.0
  - gofmt (system)
  - rustfmt (system)
language_priority:
  - python
  - typescript
  - javascript
  - cpp
  - go
  - rust
  - proto
minimum_openclaw_version: "1.8.0"
```

# Code Crafter

Motor de refactorización, formateo y aplicación de estilos de código con AST.

## Purpose

Casos de uso reales:
- Limpiar código legado con formato mixto antes de mergear PR
- Aplicar guía de estilos del equipo (Google, PEP8, Airbnb) en monorepo
- Actualizar APIs obsoletas de forma segura en todo el codebase
- Extraer bloques de código repetidos en funciones reutilizables
- Simplificar condicionales complejos y expresiones anidadas
- Eliminar imports no usados y código muerto automáticamente
- Convertir patrones ES5 a ES6+ (arrow functions, const/let, template literals)
- Estandarizar comillas de strings, trailing commas y finales de línea
- Preparar código para revisión de deuda técnica reduciendo complejidad ciclomática

## Alcance

Los comandos se ejecutan en el directorio actual a menos que se especifique una ruta.

### Formatting commands

```
$ code-crafter format <path>
--language <py|ts|js|cpp|go|rs|proto>
--style <pep8|google|airbnb|llvm|gofmt|rustfmt>
--line-length <int>
--dry-run (preview only)
--check (exit 1 if changes needed)
--exclude <pattern> (glob, repeatable)
--backend <local|github-actions>
$ code-crafter format-all [--staged] [--since <branch>]
$ code-crafter fix-imports <path> --profile <isort-profile>
$ code-crafter lint <path> --fix [--ignore <codes>]
```

### Refactoring commands

```
$ code-crafter extract-function <file> --region <start>:<end> --name <new_name>
$ code-crafter inline-variable <file> --variable <name>
$ code-crafter rename-symbol <file> --old <name> --new <new_name> [--scope <local|global>]
$ code-crafter simplify-conditionals <file> --target <line>|<region>
$ code-crafter expand-math <file> --target <line>|<region>
$ code-crafter convert-to-arrow <file> --function <name>
```

### Analysis commands

```
$ code-crafter complexity <path> --threshold <n> [--format <text|json|csv>]
$ code-crafter duplicate-code <path> --min-lines <n> --min-similarity <0.0-1.0>
$ code-crafter dead-code <path> --confidence <high|medium|low>
```

### Project-wide operations

```
$ code-crafter apply-rule <rule_file> --path <path>
$ code-crafter batch <operations_file.yaml> --report <report.json>
```

## Proceso de Trabajo

### 1. Preparation
- Parsear `pyproject.toml`, `.prettierrc`, `.clang-format`, etc. para configuración global del proyecto
- Cargar parsers específicos del lenguaje basados en la extensión del archivo
- Crear respaldo: `cp <file> <file>.codecrafter.bak.<timestamp>`
- Detectar archivos modificados via `git diff --name-only` cuando se usa `--staged`

### 2. Formatting phase (idempotent)
- Ejecutar Black/Prettier con opciones de estilo detectadas
- Aplicar isort/rustfmt después del formateador principal para evitar conflictos
- Verificar sintaxis con compilador del lenguaje (python -m py_compile, tsc --noEmit, etc.)
- Revisar cambios de formato; si no hay, omitir commit

### 3. Refactoring phase (destructive, requires review)
- Aplicar transformaciones AST en el código fuente original
- Preservar comentarios y docstrings a menos que se eliminen explícitamente
- Generar unified diff para aprobación manual si no se usa `--auto`
- Validar: sin errores de sintaxis, sin imports rotos

### 4. Verification
- Ejecutar suite de tests si existe: `pytest`, `npm test`, `go test`, `cargo test`
- Build check: `python -m build`, `npm run build`, `go build ./...`
- Lint resultado final con `ruff check`, `eslint`, `golangci-lint`
- Fallar si algún paso de verificación falla a menos que se use `--force`

### 5. Commit (optional)
- Stage de todos los archivos cambiados: `git add -u`
- Plantilla de mensaje de commit:
  ```
  refactor: [Breve descripción]

  - Auto-formateado con code-crafter
  - Aplicadas X refactorizaciones
  - Todos los tests pasan

  Verificado: <timestamp>
  ```
- Push si se usa flag `--push` con upstream configurado

## Reglas de Oro

1. **Always backup** - archivos `.codecrafter.bak.*` conservados hasta limpieza explícita
2. **Never mix formatting and refactoring in same pass** - fases separadas evitan diffs no revisables
3. **Respect existing configs** - leer configuración del proyecto antes de aplicar defaults
4. **Never remove code marked `# noqa`**, `// @ts-ignore`, o `// codecrafter:skip`
5. **Always preserve imports order** reconocido por isort/profile
6. **Exit 0 only if all verifications pass** cuando se usa `--check` o modo `--strict`
7. **Require `--force` for destructive refactorings** (inline-variable, rename-symbol)
8. **Log every action to `.codecrafter.log`** con comando completo y hashes de archivos
9. **Reject modifications to generated files** (protobuf, thrift, `*-generated.go`, `dist/`, `build/`)
10. **Suggest manual review** para reducciones de complejidad > 20% o eliminación de duplicados

## Examples

### Example 1: Format Python with Google style
```
$ code-crafter format src/ --language python --style google --dry-run
[DRY RUN] google style, line-length=88
✓ src/utils.py: reformatted (was 142 lines → 138 lines)
✗ src/models.py: no changes needed
Verification: python -m py_compile on reformatted files: PASSED
Run without --dry-run to apply
```

### Example 2: Extract function from repetitive block
```
$ code-crafter extract-function src/auth.py --region 45:67 --name validate_token_payload
Extract 23 lines into validate_token_payload()
Created new function at line 45
Updated 3 call sites
Backup: src/auth.py.codecrafter.bak.20260305_142310
```

### Example 3: Fix imports and lint TypeScript
```
$ code-crafter fix-imports src/components/ --profile google
$ code-crafter lint src/components/ --fix --ignore "N801,N802"
Fixable lint errors: 12/15 applied
Remaining: 3 (non-fixable naming violations)
```

### Example 4: Find and report complex functions
```
$ code-crafter complexity lib/ --threshold 10 --format json
[
  {
    "file": "lib/processor.py",
    "function": "process_transaction",
    "line": 234,
    "complexity": 18,
    "suggested_threshold": 10
  }
]
```

### Example 5: Batch operations via YAML
```
$ cat batch.yaml
- format:
    path: src/
    language: python
    style: pep8
- lint:
    path: src/
    fix: true
- complexity:
    path: src/
    threshold: 15
    format: json
$ code-crafter batch batch.yaml --report batch-report.json
Applied 3 operations
Report: batch-report.json
```

## Rollback Commands

### Per-file immediate revert
```
$ code-crafter rollback <file> [--to-backup <timestamp>]
# Restores latest .codecrafter.bak.* unless specific timestamp given
```

### Git-based revert
```
$ git checkout -- <file>  # if not yet committed
$ git revert <commit_sha>  # if already committed
```

### Batch rollback all Code Crafter changes
```
$ code-crafter rollback-all --since "2 hours ago"
# Finds all files with .codecrafter.bak.* and reverts
# WARNING: unsaved work may be lost
```

### Cleanup backups
```
$ code-crafter cleanup-backups --older-than 7d
$ code-crafter cleanup-backups --all
```

## Solución de Problemas

**Issue**: "Language not supported"
- Fix: Instalar formateador requerido (ej., `npm install -g prettier`, `apt install clang-format`)
- Check: `code-crafter --list-languages`

**Issue**: Conflict between formatters (ej., Black luego isort reordena)
- Fix: Set `--profile black` para isort o usar `pyproject.toml` para configurar ambos

**Issue**: "Refactoring changed semantics"
- Restaurar desde backup: `code-crafter rollback <file>`
- Re-ejecutar con `--dry-run` para inspeccionar diff antes de aplicar
- Reportar bug con caso mínimo de reproducción

**Issue**: "Verification failed: tests broken"
- Abortar: `code-crafter abort` (cancela batch actual)
- Investigar: `code-crafter diff <file>` muestra cambios pre/post
- Usar `--skip-tests` solo para ejecuciones de formato cosmético

**Issue**: "Skipping generated file"
- Agregar `--allow-generated` para override (usar con precaución)
- O marcar archivo con comentario: `// codecrafter: allow-generated`

## Environment Variables

- `CODECRAFTER_BACKUP_DIR`: Ubicación personalizada de backups (por defecto: mismo directorio)
- `CODECRAFTER_MAX_BACKUPS`: Número de backups a retener (por defecto: 5, 0=ilimitado)
- `CODECRAFTER_VERBOSE`: Set `1` para logging detallado
- `CODECRAFTER_GIT_INTEGRATION`: Set `0` para deshabilitar auto-staging/committing
- `CODECRAFTER_STYLE_OVERRIDE`: Forzar estilo para todos los lenguajes (ej., `google`)

## Exit Codes

- 0: Success (todas las verificaciones pasaron)
- 1: Changes needed (con `--check`)
- 2: Lint/format errors remain after `--fix`
- 3: Syntax/compilation error
- 4: Test suite failure
- 5: Backup/restore failure
- 6: Unsupported language or missing dependency
- 7: User abort (ej., keyboard interrupt)

## Logging

Todas las operaciones se agregan a `.codecrafter.log`:
```
[2026-03-05 14:23:01] START: format src/ --style google
[2026-03-05 14:23:02] BACKUP: src/utils.py → src/utils.py.codecrafter.bak.20260305_142302
[2026-03-05 14:23:03] FORMAT: src/utils.py (black)
[2026-03-05 14:23:04] VERIFY: py_compile PASSED
[2026-03-05 14:23:04] END: format SUCCESS
```

## File Patterns (Reserved)

Estos patrones se **siempre omiten** a menos que se use `--force`:
- `dist/`, `build/`, `target/`, `node_modules/`, `__pycache__/`
- `*.min.js`, `*.pyc`, `*.o`, `*.so`
- `*-generated.go`, `*.pb.go`, `pkg/gen/`
- `vendor/`, `third_party/`, `external/`

## Performance Notes

- Format: ~50 files/sec (Python), ~200 files/sec (JS/TS)
- Refactor: ~10 operations/sec (AST-bound)
- Usar `--parallel N` para override de concurrencia (por defecto: CPU count)
- Para > 1000 archivos, ejecutar por-directorio para evitar presión de memoria

## Safety Checklist

Antes de ejecutar refactoring destructivo:
1. `git status` limpio o cambios commitados
2. Tests pasando en código actual
3. `--dry-run` diff revisado manualmente
4. Backup confirmado en log
5. Pipeline CI/CD al tanto de cambios potenciales
```