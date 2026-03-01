name: regex-sorcerer
version: 1.2.0
description: Domina patrones regex para transformar y manipular texto con precisión mágica
author: OpenClaw Team
tags:
  - regex
  - texto
  - manipulación
  - automatización
  - patrón
  - coincidencia
  - refactorización
  - búsqueda-reemplazo
input:
  - text: Contenido de texto a procesar (stdin o archivo)
  - pattern: Patrón regex a aplicar
  - flags: Opciones regex opcionales (g, i, m, s, u, y)
  - operation: Uno de [match, replace, extract, split, validate, count, test]
  - replacement: Cadena de reemplazo o plantilla de transformación
  - multiline: Booleano para modo de procesamiento multilínea
output:
  - result: Salida de texto procesada
  - matches: Array de coincidencias encontradas con posiciones
  - count: Número de coincidencias encontradas
  - changed: Booleano que indica si el texto fue modificado
  - statistics: Métricas de tiempo y rendimiento
  - errors: Array de errores de validación
environment:
  - REGEX_MAX_MATCHES: Máximo de coincidencias a devolver (por defecto 1000)
  - REGEX_TIMEOUT: Tiempo límite en milisegundos (por defecto 5000)
  - REGEX_MAX_FILE_SIZE: Tamaño máximo de archivo en MB (por defecto 100)
  - REGEX_DEBUG: Habilitar registro de depuración (por defecto false)
  - REGEX_BACKUP_EXT: Extensión de archivo de respaldo (por defecto .regex-bak)
dependencies:
  - name: grep
    required: true
    version: ">=2.0"
  - name: sed
    required: true
    version: ">=4.0"
  - name: perl
    required: false
    version: ">=5.0"
  - name: ripgrep (rg)
    required: false
    version: ">=13.0"
  - name: jq
    required: false
    version: ">=1.6"
---

# Regex Sorcerer

## Propósito

Regex Sorcerer proporciona precisión quirúrgica para transformación de texto, procesamiento masivo de datos, refactorización de código, análisis de registros y tareas de limpieza automatizada de texto. Permite operaciones complejas basadas en patrones que de otro modo requerirían código imperativo verboso o edición manual.

**Casos de uso reales:**
- Extraer direcciones de correo electrónico y números de teléfono de registros no estructurados
- Renombrar variables en toda una codebase usando grupos de captura
- Convertir formatos CSV/TSV con reordenamiento de campos
- Eliminar espacios en blanco duplicados, normalizar finales de línea, limpiar formato
- Validar archivos de configuración contra requisitos de patrón
- Buscar comentarios TODO/FIXME en múltiples archivos
- Generar informes estructurados a partir de texto semiestructurado
- Transcodificar formatos de fecha (MM/DD/YYYY a ISO 8601)
- Enmascarar datos sensibles (tarjetas de crédito, SSN) para desidentificación
- Realizar búsqueda-reemplazo multi-archivo con salvaguardas de seguridad

## Alcance

### Operaciones Soportadas

- **match**: Encontrar todas las coincidencias regex con grupos de captura y posiciones
- **replace**: Reemplazar patrones con cadena de reemplazo (soporta referencias $1-$9)
- **extract**: Extraer grupos de captura en formato estructurado (JSON/CSV)
- **split**: Dividir texto en delimitador regex en array
- **validate**: Verificar si todo el texto coincide con el patrón, devolver booleano
- **test**: Verificación booleana rápida de existencia de patrón
- **count**: Contar ocurrencias de patrón sin extraer
- **analyze**: Generar análisis de patrón (complejidad, AST, errores comunes)

### Formas de Comando

```bash
# Modo match (encontrar todas las coincidencias)
regex-sorcerer match "pattern" < input.txt
regex-sorcerer match -i -g "pattern" < input.txt

# Modo replace (transformar texto)
regex-sorcerer replace "pattern" "replacement" < input.txt > output.txt
regex-sorcerer replace -i "pattern" "replacement" < input.txt
echo "text" | regex-sorcerer replace -s "\s+" " "  # comprimir espacios

# Modo extract (obtener grupos de captura)
regex-sorcerer extract "pattern" --group=1 --format=json < input.txt
regex-sorcerer extract -g "(\w+).*(\d+)" -f csv < data.txt

# Modo split
regex-sorcerer split "pattern" < input.txt
regex-sorcerer split -l -i ",\s*" < data.txt

# Modo validate
regex-sorcerer validate "pattern" < file.txt
if regex-sorcerer validate "^\d{5}-\d{4}$"; then echo "ZIP válido"; fi

# Modo test (solo booleano)
if regex-sorcerer test "ERROR" < log.txt; then echo "Tiene errores"; fi

# Modo count
regex-sorcerer count "pattern" < bigfile.log
regex-sorcerer count -i -t "^\w+@\w+\.\w+$" < emails.txt

# Modo analyze
regex-sorcerer analyze "pattern"
```

### Opciones

- `-i`, `--ignore-case`: Coincidencia insensible a mayúsculas
- `-g`, `--global`: Encontrar todas las coincidencias (por defecto: solo primera para algunas ops)
- `-m`, `--multiline`: Tratar cadena como múltiples líneas (^/$ coinciden por línea)
- `-s`, `--dotall`: El punto coincide con nueva línea (modo single-line)
- `-u`, `--unicode`: Habilitar modo Unicode
- `-l`, `--line`: Procesar línea por línea (para split/validate/test)
- `-t`, `--total`: Para count, devolver número en lugar de salida verbosa
- `-j`, `--json`: Salida en formato JSON
- `-f`, `--format`: Formato de salida: json, csv, lines, raw (para extract)
- `--group=N`: Extraer grupo de captura específico (por defecto: todos)
- `--max=N`: Limitar número de resultados
- `--timeout=N`: Anular REGEX_TIMEOUT para esta ejecución
- `--backup`: Crear respaldo .bak antes de modificar archivos
- `--dry-run`: Mostrar cambios sin aplicar (solo modo replace)
- `--line-numbers`: Incluir números de línea en salida
- `--positions`: Incluir posiciones inicio/fin en salida de coincidencias
- `--named`: Usar grupos con nombre como claves en salida JSON

## Proceso de Trabajo

### Fase 1: Validación de Patrón
1. Analizar sintaxis de patrón usando PCRE2 (Perl Compatible Regular Expressions)
2. Detectar errores comunes: cuantificadores anidados, retroceso catastrófico
3. Verificar que el patrón compila sin errores
4. Si `REGEX_DEBUG=true`, mostrar AST a stderr

### Fase 2: Adquisición de Entrada
1. Leer desde stdin O argumento de ruta de archivo
2. Verificar tamaño de archivo contra `REGEX_MAX_FILE_SIZE`
3. Detectar codificación (UTF-8 por defecto), convertir si es necesario
4. Para modo `--backup`: copiar original a `{path}{REGEX_BACKUP_EXT}`

### Fase 3: Ejecución (por operación)
- **match**: Ejecutar `re2::RE2::FindAndConsume` en bucle hasta agotar o `REGEX_MAX_MATCHES`
- **replace**: Procesar por flujo línea por línea; para modo single-line reemplazar globalmente
- **extract**: Recoger todos los grupos de captura, construir salida estructurada
- **split**: Usar `re2::RE2::Split` en vector de strings
- **validate**: Verificar `re2::RE2::FullMatch` devuelve true
- **test**: Verificar `re2::RE2::PartialMatch` devuelve true
- **count**: Incrementar contador por cada coincidencia encontrada
- **analyze**: Ejecutar análisis estático en cadena de patrón

### Fase 4: Formateo de Salida
1. Formatear según opción `--format`
2. Para JSON: escapar strings, incluir posiciones si se solicita
3. Para CSV: entrecomillar campos que contengan comas/nuevas líneas
4. Escribir a stdout; errores/advertencias a stderr
5. Código de salida:
   - 0: Éxito (coincidencias encontradas para match/test; validación pasó para validate)
   - 1: No se encontraron coincidencias O validación falló
   - 2: Error de sintaxis de patrón
   - 3: Error de entrada (archivo no encontrado, muy grande, codificación)
   - 4: Error del sistema (memoria, timeout)

### Fase 5: Post-Procesamiento
- Para replace con `--backup`: retener respaldo hasta que usuario verifique
- Para operaciones multi-archivo: continuar en errores por archivo, reportar resumen

## Reglas de Oro

1. **Siempre hacer respaldo antes de reemplazo destructivo**: Usar `--backup` al modificar archivos in-situ. Nunca operar en original sin respaldos.
2. **Probar en muestra pequeña primero**: Ejecutar coincidencia en muestra diminuta antes de reemplazo completo de archivo. Usar `regex-sorcerer test` para validación rápida.
3. **Respetar límites de línea**: A menos que se use `-m`/`-s`, los patrones operan por línea. Patrones multi-línea requieren flags explícitos.
4. **Escapar correctamente**: En shell, entrecomillar patrones para evitar interpretación del shell. Ejemplo: `'^\d{3}-\d{4}$'` no `"^\d{3}-\d{4}$"`
5. **Usar no-codicioso via `?`**: El matching codicioso por defecto causa errores. Usar explícitamente `*?` o `+?` cuando sea necesario.
6. **Manejar Unicode**: Usar `-u` para texto no-ASCII; asegurar que terminal/archivo sea UTF-8.
7. **Limitar coincidencias**: Siempre usar `--max` para archivos grandes para evitar OOM. Límite por defecto 1000 aplicado.
8. **Validar reemplazos**: Usar `--dry-run` para previsualizar cambios de replace antes de confirmar.
9. **Verificar códigos de salida**: Los scripts deben probar `$?` después de llamadas a regex-sorcerer. No asumir éxito.
10. **Solo PCRE2**: Esta herramienta usa librería RE2 (Google's regex library). No soporta backreferences en lookbehind, condicionales, o recursión.
11. **No lookbehind de longitud variable**: RE2 requiere lookbehind de ancho fijo. Patrón `(?<=\d{2,})` inválido.
12. **Referencias de grupos de captura**: En reemplazo, usar `$1` no `\1`. Para `$` literal, escapar como `$$`.
13. **Timeout de rendimiento**: `REGEX_TIMEOUT` previene DoS regex. Patrones complejos en entrada grande pueden timeout; simplificar patrón si es necesario.

## Ejemplos

### Ejemplo 1: Extraer direcciones de correo de registros
**Entrada** (`access.log`):
```
127.0.0.1 - - [01/Mar/2026:10:00:00] "GET / HTTP/1.1" 200 1234 - user@example.com
192.168.1.1 - - [01/Mar/2026:10:01:00] "POST /api HTTP/1.1" 404 567 admin@test.org
```

**Comando**:
```bash
regex-sorcerer extract -i "([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})" --named -f json < access.log
```

**Salida** (JSON):
```json
[
  {"email": "user@example.com", "line": 1},
  {"email": "admin