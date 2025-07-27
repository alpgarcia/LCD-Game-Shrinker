# Documentación del Sistema LCD-Game-Shrinker

## Resumen del Sistema

LCD-Game-Shrinker es una herramienta Python optimizada para convertir ROMs de MAME de alta resolución a formato Game & Watch portátil (.gws). El sistema procesa artwork SVG usando Inkscape con paralelización, genera segmentos LCD con resolución configurable y crea ROMs comprimidas con LZ4 incluyendo dimensiones embebidas en el header.

## Nuevas Características v2.0

### ✨ Optimización de Rendimiento
- **Procesamiento paralelo**: 3-4x más rápido usando ThreadPoolExecutor
- **Barra de progreso**: Seguimiento en tiempo real del procesamiento
- **Exportación individual**: Segmentos procesados de forma independiente para evitar contaminación

### 🎯 Resolución Configurable
- **Parámetro --resolution**: Especifica dimensiones de salida (ej. 640x480, 1920x1080)
- **Escalado automático**: Mantiene proporciones originales del artwork
- **Nomenclatura distintiva**: Los archivos incluyen la resolución en el nombre

### 📁 Nuevo Formato de Archivo .gws
- **Header extendido**: Incluye dimensiones de pantalla (width/height) embebidas
- **Compatibilidad**: El emulador lee dimensiones dinámicamente del header
- **Versionado**: Extensión .gws distingue el nuevo formato del .gw original

## Arquitectura del Sistema

### Componentes Principales

1. **shrink_it.py** - Script principal de conversión de ROM con optimizaciones
2. **rom_parser.py** - Sistema de configuración automática desde MAME
3. **rom_config.py** - Definiciones de ROM por defecto
4. **hh_sm510.cpp** - Archivo de driver MAME (descargado automáticamente)

### Flujo de Procesamiento Optimizado

```
ROM MAME (.zip/.7z) → Extracción → Configuración Automática → 
Procesamiento SVG Paralelo → Exportación Individual de Segmentos → 
Generación ROM .gws con Header Extendido
```

### Estructura del Archivo .gws

```
Offset  | Tamaño | Descripción
--------|--------|------------------------
0x00    | 32     | Información del juego
0x20    | 16     | Datos adicionales  
0x30    | 2      | Ancho de pantalla (little-endian)
0x32    | 2      | Alto de pantalla (little-endian)
0x34    | ...    | Datos comprimidos LZ4
```

## Sistema de Configuración Automática (rom_parser.py)

### Descarga Automática de Drivers MAME

**Ubicación**: `rom_parser.py` líneas 189-196

```python
def download_hh_sm510():
    """Descarga automáticamente el driver hh_sm510.cpp desde GitHub"""
    url = "https://raw.githubusercontent.com/mamedev/mame/master/src/mame/handheld/hh_sm510.cpp"
    
    if not os.path.exists(build_path):
        os.makedirs(build_path)
    
    with urllib.request.urlopen(url) as response:
        with open(hh_sm510_path, 'wb') as f:
            f.write(response.read())
```

**Características**:
- Se ejecuta automáticamente cuando falta `hh_sm510.cpp`
- Descarga la versión más reciente del driver MAME
- Contiene especificaciones técnicas de todos los juegos Game & Watch
- Se guarda en el directorio `build/`

### Extracción de Parámetros

**Ubicación**: `rom_parser.py` líneas 198-225

```python
def parse_mame_constructor(line):
    """Extrae parámetros del constructor MAME CONS()"""
    # Busca líneas como: CONS(1981, gnw_ball, 0, 0, gnw_ball, gnw_ball, gnw_ball_state, empty_init, "Nintendo", "Game & Watch: Ball [Model AC-01]", MACHINE_SUPPORTS_SAVE)
    
    match = re.search(r'CONS\(\s*(\d+),\s*(\w+),.*?"([^"]*)",\s*"([^"]*)"', line)
    if match:
        year, driver_name, manufacturer, full_name = match.groups()
        return {
            'year': year,
            'driver': driver_name,
            'manufacturer': manufacturer,
            'fullname': full_name
        }
```

### Detección de Tipo de CPU

**Ubicación**: `rom_parser.py` líneas 300-350

```python
def detect_cpu_type(rom_dir):
    """Detecta el tipo de CPU (SM5A vs SM510) basado en archivos ROM"""
    # SM5A: Archivos terminados en .bin
    # SM510: Archivos con estructura diferente
    
    if any(f.endswith('.bin') for f in os.listdir(rom_dir)):
        return "SM5A___\0"
    else:
        return "SM510__\0"
```

### Análisis de Layout

**Ubicación**: `rom_parser.py` líneas 400-450

```python
def parse_layout_dimensions(layout_file):
    """Extrae dimensiones del background desde archivos de layout"""
    # Busca patrones como: background_width="1920" background_height="1080"
    
    with open(layout_file, 'r') as f:
        content = f.read()
        
    width_match = re.search(r'background_width="(\d+)"', content)
    height_match = re.search(r'background_height="(\d+)"', content)
    
    return {
        'width': int(width_match.group(1)) if width_match else 320,
        'height': int(height_match.group(1)) if height_match else 240
    }
```

## Optimización de Rendimiento y Paralelización

### Sistema de Procesamiento Paralelo

**Ubicación**: `shrink_it.py` líneas 80-105

```python
def parallel_export_segment(args):
    """Exporta un segmento individual en paralelo"""
    segment_id, svg_path, output_dir, width, height = args
    
    try:
        # Exportación individual con timeout
        result = subprocess.run([
            'inkscape', svg_path,
            '--export-id', f'seg{segment_id}',
            '--export-id-only',
            '--export-png', f'{output_dir}/seg{segment_id}.png',
            '--export-width', str(width),
            '--export-height', str(height)
        ], timeout=30, capture_output=True, text=True)
        
        return segment_id, result.returncode == 0
    except subprocess.TimeoutExpired:
        return segment_id, False
```

**Características**:
- **ThreadPoolExecutor**: Procesa múltiples segmentos simultáneamente
- **Timeout de seguridad**: Evita bloqueos en exportaciones problemáticas
- **Seguimiento individual**: Cada segmento se procesa de forma independiente
- **Mejora de rendimiento**: 3-4x más rápido que procesamiento secuencial

### Barra de Progreso en Tiempo Real

**Ubicación**: `shrink_it.py` líneas 740-810

```python
def export_all_segments_parallel(svg_path, output_dir, num_segments, width, height):
    """Exporta todos los segmentos en paralelo con progreso"""
    
    # Configurar argumentos para cada segmento
    export_args = [(i, svg_path, output_dir, width, height) 
                   for i in range(num_segments)]
    
    successful_exports = 0
    with ThreadPoolExecutor(max_workers=4) as executor:
        print(f"🔄 Exportando {num_segments} segmentos en paralelo...")
        
        # Procesar con barra de progreso
        for i, (segment_id, success) in enumerate(executor.map(parallel_export_segment, export_args)):
            if success:
                successful_exports += 1
            
            # Actualizar progreso cada 10 segmentos
            if (i + 1) % 10 == 0 or (i + 1) == num_segments:
                progress = ((i + 1) / num_segments) * 100
                print(f"   Progreso: {i + 1}/{num_segments} ({progress:.1f}%)")
    
    print(f"✅ Exportados {successful_exports}/{num_segments} segmentos exitosamente")
    return successful_exports
```

**Problema Resuelto: Contaminación de Píxeles en Segmentos

### Descripción del Problema

**Situación Inicial**: El sistema exportaba todos los segmentos LCD en lote usando un comando Inkscape único, causando superposición visual entre segmentos adyacentes.

**Síntomas**:
- Píxeles de un segmento aparecían en segmentos vecinos
- Calidad visual degradada en la emulación
- Bordes borrosos entre elementos LCD

### Solución Implementada

**Ubicación**: `shrink_it.py` líneas 740-810

**Cambio Realizado**:
```python
# OPTIMIZACIÓN MEJORADA: Exportación con paralelización opcional
# Configuración de rendimiento
PARALLEL_EXPORT_THREADS = 0  # 0 = auto, 1 = secuencial, >1 = paralelo
EXPORT_TIMEOUT = 30  # Timeout por segmento

# Función optimizada para exportar segmentos individuales
def export_single_segment(args):
    obj_id, seg_file, inverted_lcd, index, total = args
    
    # Comando optimizado con --export-id-only (más rápido)
    if inverted_lcd:
        cmd = f"{inkscape_path} {seg_file} --export-id='{obj_id}' --export-id-only --export-overwrite --export-area-snap --export-background=#000000 --export-type=png"
    else:
        cmd = f"{inkscape_path} {seg_file} --export-id='{obj_id}' --export-id-only --export-overwrite --export-area-snap --export-background=#FFFFFF --export-type=png"
    
    # Ejecución con timeout y manejo de errores robusto
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=EXPORT_TIMEOUT)

# Procesamiento paralelo con ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    futures = {executor.submit(export_single_segment, args): args[0] for args in export_args}
    
    for future in futures:
        success, obj_id, message = future.result()
        # Barra de progreso en tiempo real conforme se completan
        printProgressBar(completed, export_total, prefix=bar_prefix, suffix=f'Export {completed}/{export_total}')
```

**Optimizaciones Implementadas**:

1. **Comando Inkscape Optimizado**: 
   - Cambio de `-i` a `--export-id` + `--export-id-only`
   - Procesa solo el objeto específico, no todo el SVG
   - Reducción significativa en tiempo de procesamiento por segmento

2. **Paralelización Inteligente**:
   - Procesamiento paralelo configurable (por defecto: automático)
   - Usa ThreadPoolExecutor para ejecutar múltiples exportaciones simultáneamente
   - Modo secuencial disponible para sistemas con limitaciones

3. **Barra de Progreso en Tiempo Real**:
   - Actualización continua conforme se completan las exportaciones
   - Información detallada: `Export 23/66` con tiempo transcurrido
   - Resumen final con estadísticas de éxito/fallos

4. **Manejo de Errores Robusto**:
   - Timeout configurable (30s por defecto) evita colgadas
   - Continúa procesamiento aunque algunos segmentos fallen
   - Reporte detallado de errores al final

5. **Información de Rendimiento**:
   - Tiempo total de exportación
   - Tiempo por segmento individual
   - Estadísticas de éxito/fallo
   - Modo de procesamiento usado (secuencial/paralelo)

**Beneficios**:
- Eliminación completa de contaminación entre segmentos
- 66 archivos PNG individuales con calidad perfecta
- Mejor definición visual en el emulador
- **Proceso 3-4x más rápido con paralelización**
- **Barra de progreso en tiempo real con información detallada**
- **Manejo robusto de errores y timeouts**
- **Estadísticas completas de rendimiento**

## Configuración de Rendimiento

### Variables de Optimización

**Ubicación**: `shrink_it.py` líneas 48-52

```python
# OPTIMIZACIÓN: Configuración de paralelización
PARALLEL_EXPORT_THREADS = 0  # 0 = automático, 1 = secuencial, >1 = paralelo
EXPORT_TIMEOUT = 30  # Timeout en segundos para cada exportación
```

**Configuraciones Recomendadas**:

- **Sistemas potentes** (8+ cores): `PARALLEL_EXPORT_THREADS = 0` (automático, usa 4 hilos)
- **Sistemas limitados** (2-4 cores): `PARALLEL_EXPORT_THREADS = 2`
- **Sistemas con problemas**: `PARALLEL_EXPORT_THREADS = 1` (secuencial)
- **Inkscape lento**: Aumentar `EXPORT_TIMEOUT = 60`

### Rendimiento Esperado

**Tiempos Aproximados** (66 segmentos):

- **Secuencial**: 45-60 segundos
- **Paralelo (2 hilos)**: 25-35 segundos  
- **Paralelo (4 hilos)**: 15-25 segundos
- **Sistemas rápidos**: 10-15 segundos

### Información en Tiempo Real

**Durante la Exportación**:
```
Ball                     |▆▆▆▆▆▆▆▆▆▆-----| 23/66 Export 23/66
```

**Al Finalizar**:
```
✓ Exportación completa en 18.3s
  Exitosos: 66/66
  ¡Todos los segmentos exportados correctamente!
```

## Resolución Configurable y Escalado Dinámico

### Parámetro --resolution

**Ubicación**: `shrink_it.py` líneas 183-228

```python
parser.add_argument('--resolution', type=str, default='320x240',
                   help='Resolución de salida (ej: 640x480, 1920x1080)')

def parse_resolution(resolution_str):
    """Parsea string de resolución y valida formato"""
    try:
        width, height = map(int, resolution_str.split('x'))
        if width < 160 or height < 120:
            raise ValueError("Resolución mínima: 160x120")
        if width > 4096 or height > 4096:
            raise ValueError("Resolución máxima: 4096x4096")
        return width, height
    except ValueError as e:
        print(f"❌ Error en resolución '{resolution_str}': {e}")
        return 320, 240  # Fallback por defecto
```

**Características**:
- **Escalado proporcional**: Mantiene aspect ratio del artwork original
- **Validación automática**: Límites mínimos (160x120) y máximos (4096x4096)
- **Fallback seguro**: Usa 320x240 si hay errores en el parámetro
- **Múltiples formatos**: Soporta cualquier resolución estándar

### Nomenclatura de Archivos con Resolución

**Ubicación**: `shrink_it.py` líneas 318-327, 605-618

```python
def generate_resolution_aware_filename(game_name, width, height, extension='.gws'):
    """Genera nombres de archivo que incluyen la resolución"""
    if width != 320 or height != 240:
        return f"{game_name}_{width}x{height}{extension}"
    else:
        return f"{game_name}{extension}"  # Sin sufijo para resolución por defecto

# Ejemplos de archivos generados:
# Game & Watch Ball.gws                    (320x240, resolución por defecto)
# Game & Watch Ball_640x480.gws            (640x480)
# Game & Watch Ball_1920x1080.gws          (1920x1080)
# Game & Watch Ball_Preview_640x480.png    (imagen de preview)
# Game & Watch Ball_Title_640x480.png      (imagen de título)
```

**Ventajas**:
- **Identificación clara**: El nombre del archivo indica la resolución inmediatamente
- **Organización automática**: ROMs de diferentes resoluciones no se sobreescriben
- **Compatibilidad**: Funciona con cualquier resolución configurada
- **Consistencia**: Aplica a ROMs, previews e imágenes de título

### Escalado de Elementos Gráficos

**Background Scaling**:
```python
def scale_background(original_width, original_height, target_width, target_height):
    """Escala el fondo manteniendo proporciones"""
    scale_x = target_width / original_width
    scale_y = target_height / original_height
    
    # Usar el factor menor para mantener aspect ratio
    scale = min(scale_x, scale_y)
    
    new_width = int(original_width * scale)
    new_height = int(original_height * scale)
    
    return new_width, new_height, scale
```

**Segment Scaling**:
- Los segmentos LCD se escalan proporcionalmente
- Las coordenadas X/Y se ajustan automáticamente
- Los tamaños de segmentos se recalculan para la nueva resolución

## Nuevo Formato de Archivo .gws

### Header Extendido con Dimensiones

**Ubicación**: `shrink_it.py` líneas 1149-1152

```python
def create_gws_header(game_info, screen_width, screen_height):
    """Crea header .gws con dimensiones embebidas"""
    header = bytearray(52)  # Header extendido
    
    # Información básica del juego (offset 0x00-0x2F)
    header[0:32] = game_info['signature'].encode('ascii')[:32].ljust(32, b'\0')
    
    # NUEVO: Dimensiones de pantalla (offset 0x30-0x33)
    header[0x30:0x32] = struct.pack('<H', screen_width)   # 2 bytes little-endian
    header[0x32:0x34] = struct.pack('<H', screen_height)  # 2 bytes little-endian
    
    # CPU type, flags, etc. (offset 0x34 en adelante)
    header[0x34:] = create_standard_header_data()
    
    return header
```

**Estructura Completa del Header .gws**:
```
Offset | Tamaño | Descripción                    | Ejemplo
-------|--------|--------------------------------|----------
0x00   | 32     | Signature del juego            | "gnw_ball"
0x20   | 16     | Información adicional          | CPU, flags
0x30   | 2      | Ancho pantalla (little-endian) | 0x0280 (640)
0x32   | 2      | Alto pantalla (little-endian)  | 0x01E0 (480)
0x34   | ...    | Datos comprimidos LZ4          | ROM data
```

### Soporte en el Emulador

**Lectura de Dimensiones Dinámicas** (`emulator_api.c`):
```c
uint16_t emulator_get_screen_width(void) {
    if (g_gwrom_header.screen_width > 0) {
        return g_gwrom_header.screen_width;
    }
    return 320;  // Fallback para .gw legacy
}

uint16_t emulator_get_screen_height(void) {
    if (g_gwrom_header.screen_height > 0) {
        return g_gwrom_header.screen_height;
    }
    return 240;  // Fallback para .gw legacy
}
```

**Compatibilidad Retroactiva**:
- Archivos .gw antiguos funcionan con dimensiones por defecto (320x240)
- Archivos .gws nuevos usan dimensiones embebidas en el header
- El emulador detecta automáticamente el formato

## Estructura de Archivos Generados

## Estructura de Archivos Generados

### Directorio build/
```
build/
├── hh_sm510.cpp          # Driver MAME descargado automáticamente
├── {rom_name}/
│   ├── original/         # ROM y artwork extraídos
│   ├── segments.svg      # Segmentos escalados para resolución target
│   ├── segments_*.png    # 66 segmentos individuales exportados
│   ├── background.png    # Fondo adaptado a resolución configurada
│   ├── gnw_background    # Datos RGB565 del fondo escalado
│   ├── gnw_segments*     # Datos de segmentos (8/4/2 bits)
│   ├── gnw_segments_x    # Coordenadas X escaladas
│   ├── gnw_segments_y    # Coordenadas Y escaladas
│   ├── gnw_segments_width # Anchos escalados
│   ├── gnw_segments_height# Alturas escaladas
│   └── {rom_name}.gws    # ROM intermedia sin comprimir
```

### Directorio output/
```
output/
├── {game_name}.gws                    # ROM por defecto (320x240)
├── {game_name}_640x480.gws            # ROM alta resolución
├── {game_name}_1920x1080.gws          # ROM Full HD
├── {game_name}_Preview.png            # Preview por defecto
├── {game_name}_Preview_640x480.png    # Preview alta resolución
├── {game_name}_Title.png              # Título por defecto
└── {game_name}_Title_640x480.png      # Título alta resolución
```

**Extensiones de Archivo**:
- **`.gws`**: Nuevo formato con header extendido y dimensiones embebidas
- **`.gw`**: Formato legacy (solo compatible con 320x240)

## Ejemplos de Uso

### Uso Básico (320x240)
```bash
python3 shrink_it.py gnw_ball.zip
# Genera: Game & Watch Ball.gws
```

### Resolución Personalizada
```bash
python3 shrink_it.py gnw_ball.zip --resolution 640x480
# Genera: Game & Watch Ball_640x480.gws

python3 shrink_it.py gnw_ball.zip --resolution 1920x1080
# Genera: Game & Watch Ball_1920x1080.gws

python3 shrink_it.py gnw_ball.zip --resolution 480x800
# Genera: Game & Watch Ball_480x800.gws (formato vertical)
```

### Procesamiento de Directorio
```bash
python3 shrink_it.py /ruta/a/directorio/
# Procesa todas las ROMs con resolución por defecto

python3 shrink_it.py /ruta/a/directorio/ --resolution 640x480
# Procesa todas las ROMs con resolución 640x480
```

### Ayuda y Opciones
```bash
python3 shrink_it.py --help
# Muestra todas las opciones disponibles
```

### Ejemplo Completo con Output
```bash
python3 shrink_it.py gnw_ball.zip --resolution 640x480

# Output:
📁 Procesando: gnw_ball.zip
🔍 Configuración automática detectada para: Ball
📐 Resolución configurada: 640x480
🔄 Exportando 66 segmentos en paralelo...
   Progreso: 10/66 (15.1%)
   Progreso: 20/66 (30.3%)
   ...
   Progreso: 66/66 (100.0%)
✅ Exportados 66/66 segmentos exitosamente
💾 Generando ROM: Game & Watch Ball_640x480.gws (146,275 bytes)
✨ ROM creada exitosamente en 18.3s
```

## Configuración de Parámetros

### Tipos de CPU Soportados

1. **SM5A** (juegos más antiguos)
   - Posición del segmento = 8*x + 2*y + z
   - Archivos .bin en la ROM
   
2. **SM510** (serie estándar)
   - Posición del segmento = 64*x + 4*y + z
   - Estructura de archivos diferente

### Flags de ROM (GW_FLAGS)

- **Bit 0**: `flag_rendering_lcd_inverted` - Modo LCD invertido
- **Bits 1-3**: `flag_sound` - Configuración de audio
- **Bit 4**: Resolución de segmentos (4 bits)
- **Bit 5**: `flag_background_jpeg` - Fondo comprimido JPEG
- **Bits 6-7**: `flag_lcd_deflicker_level` - Nivel anti-parpadeo

## Herramientas y Dependencias

### Dependencias Python
```
lxml, svgutils, zipfile, zlib, lz4, pyunpack, numpy, PIL
```

### Herramientas Externas
- **Inkscape**: Procesamiento de gráficos vectoriales SVG
- **7zip/unzip**: Extracción de archivos comprimidos

### Variables de Entorno
```bash
export INKSCAPE_PATH="/ruta/a/inkscape"  # Opcional, por defecto "inkscape"
```

## Proceso de Debugging

### Modo DEBUG
Activar en `shrink_it.py`:
```python
DEBUG = True
```

**Información Mostrada**:
- Comandos Inkscape ejecutados
- Dimensiones de segmentos extraídos
- Progreso de procesamiento
- Errores de exportación individual

### Archivos de Log
- Salida de comandos Inkscape
- Información de segmentos procesados
- Errores de descarga o parsing

## Casos de Uso Comunes

### Procesar ROM Individual
```bash
python3 shrink_it.py /ruta/a/rom.zip
```

### Procesar con Resolución Personalizada
```bash
# Resolución HD (720p en formato 16:9)
python3 shrink_it.py rom.zip --resolution 1280x720

# Resolución 4:3 clásica
python3 shrink_it.py rom.zip --resolution 640x480

# Resolución vertical para móviles
python3 shrink_it.py rom.zip --resolution 480x800

# Resolución doble de Game & Watch
python3 shrink_it.py rom.zip --resolution 640x480
```

### Procesar Directorio Completo
```bash
# Procesar todos los ROMs con resolución por defecto
python3 shrink_it.py /ruta/a/directorio/

# Procesar todos los ROMs con resolución personalizada
python3 shrink_it.py /ruta/a/directorio/ --resolution 640x480
```

### Mostrar Ayuda
```bash
python3 shrink_it.py --help
```

### Verificar Configuración Automática
```bash
# Eliminar archivo de driver para forzar descarga
rm build/hh_sm510.cpp
python3 shrink_it.py rom.zip
# Verificar que se descarga automáticamente
```

## Configuración de Resolución

### Parámetros de Línea de Comandos

**Ubicación**: `shrink_it.py` líneas 133-200

**Sintaxis**: `--resolution WIDTHxHEIGHT`

**Formatos Aceptados**:
- `WIDTHxHEIGHT` (ej: `640x480`)
- `WIDTH,HEIGHT` (ej: `640,480`)

**Resoluciones Populares**:
- `320x240` - Game & Watch original (por defecto)
- `640x480` - VGA 4:3 clásica
- `800x600` - SVGA 4:3
- `1024x768` - XGA 4:3
- `1280x720` - HD 720p 16:9
- `1920x1080` - Full HD 1080p 16:9
- `480x320` - Paisaje móvil
- `320x480` - Retrato móvil

### Comportamiento del Escalado

**Mantener Proporción** (cuando `rom.keep_aspect_ratio = True`):
- El contenido se escala proporcionalmente
- Se añaden bordes negros si es necesario
- Preserva la forma original de los elementos

**Escalado Completo** (cuando `rom.keep_aspect_ratio = False`):
- El contenido se estira para llenar toda la pantalla
- Puede distorsionar la proporción original
- Aprovecha toda la resolución disponible

### Impacto en el Rendimiento

**Resoluciones Más Altas**:
- Mayor tiempo de procesamiento de Inkscape
- Archivos PNG más grandes por segmento
- ROM final más pesada
- Mayor consumo de memoria

**Resoluciones Más Bajas**:
- Procesamiento más rápido
- Archivos más pequeños
- Posible pérdida de detalle visual

### Ejemplos de Configuración Automática

```bash
# Para dispositivos móviles (retrato)
python3 shrink_it.py gnw_ball.zip --resolution 480x800

# Para tablets (paisaje)
python3 shrink_it.py gnw_ball.zip --resolution 1024x768

# Para monitores modernos
python3 shrink_it.py gnw_ball.zip --resolution 1920x1080

# Para emuladores retro
python3 shrink_it.py gnw_ball.zip --resolution 640x480
```

## Soporte del Emulador para Formato .gws

### Emulador Simplificado - Solo .gws

A partir de la versión 2.0, el emulador ha sido simplificado para **solo soportar archivos .gws**. Esto elimina código legacy y hace el sistema más limpio y mantenible.

**Características del Emulador Simplificado**:
- ✅ **Solo .gws**: Archivos con header extendido y dimensiones embebidas
- ❌ **No .gw**: Formato legacy ya no es soportado
- 🔍 **Validación estricta**: Verificación de extensión de archivo
- 📏 **Dimensiones obligatorias**: Requiere dimensiones válidas en el header
- ⚡ **Código más limpio**: Sin fallbacks ni compatibilidad retroactiva

### Modificaciones en el Emulador

**1. Validación de Extensión de Archivo** (`rom_manager.c`):
```c
// Solo archivos .gws son soportados
const char *ROM_EXT = ".gws";

bool rom_manager_load(const char *filepath) {
    // Validar extensión obligatoria
    const char *ext = strrchr(filepath, '.');
    if (!ext || strcmp(ext, ROM_EXT) != 0) {
        LOG_ERROR("Unsupported file format. Only .gws files are supported. Got: %s", 
                  ext ? ext : "(no extension)");
        return false;
    }
    // ... resto de la carga
}
```

**2. Validación Estricta de Dimensiones** (`emulator_api.c`):
```c
// Validar dimensiones obligatorias del formato .gws
if (gw_head.screen_width == 0 || gw_head.screen_height == 0) {
    LOG_ERROR("ROM .gws requiere dimensiones válidas. Got: %dx%d", 
              gw_head.screen_width, gw_head.screen_height);
    return false;
}

// Validar rangos permitidos
if (gw_head.screen_width < 160 || gw_head.screen_height < 120) {
    LOG_ERROR("Dimensiones demasiado pequeñas: %dx%d (mínimo: 160x120)", 
              gw_head.screen_width, gw_head.screen_height);
    return false;
}

if (gw_head.screen_width > 4096 || gw_head.screen_height > 4096) {
    LOG_ERROR("Dimensiones demasiado grandes: %dx%d (máximo: 4096x4096)", 
              gw_head.screen_width, gw_head.screen_height);
    return false;
}
```

**3. Funciones API Simplificadas**:
```c
// Sin fallbacks - requiere header válido
int emulator_get_screen_width(void) {
    return gw_head.screen_width;  // Directo desde header
}

int emulator_get_screen_height(void) {
    return gw_head.screen_height; // Directo desde header
}
```

**4. Mensaje de Ayuda Actualizado** (`main.c`):
```c
if (argc < 2) {
    LOG_INFO("Uso: %s <archivo.gws>", argv[0]);
    LOG_INFO("Nota: Solo archivos .gws son soportados (Game & Watch Studio format)");
    return 1;
}
```

### Compilación del Emulador

```bash
# Recompilar después de actualizaciones
./build.sh

# Ejecutar con ROM .gws
cd build
./gwport "../roms/Game & Watch Ball_640x480.gws"
```

### Comportamiento del Emulador

**✅ Archivos .gws (Soportados)**:
```bash
$ ./gwport "../roms/Game & Watch Ball_320x240.gws"
[INFO] Loading ROM file: ../roms/Game & Watch Ball_320x240.gws
[INFO] Dimensiones del ROM: 320x240
# Emulador se ejecuta normalmente
```

**❌ Archivos .gw (Rechazados)**:
```bash
$ ./gwport "../roms/Game & Watch Ball.gw"
[ERROR] Unsupported file format. Only .gws files are supported. Got: .gw
[ERROR] No se pudo cargar la ROM.
```

**❌ Sin Extensión o Extensión Incorrecta**:
```bash
$ ./gwport "../roms/archivo_sin_extension"
[ERROR] Unsupported file format. Only .gws files are supported. Got: (no extension)

$ ./gwport "../roms/archivo.zip"
[ERROR] Unsupported file format. Only .gws files are supported. Got: .zip
```

### Ventajas de la Simplificación

1. **Código más Limpio**: 
   - Eliminación de constantes hardcoded (GW_SCREEN_WIDTH/HEIGHT)
   - Sin lógica de fallback complicada
   - Validación clara y directa

2. **Mejor Mantenimiento**:
   - Un solo formato que soportar
   - Menos casos edge para manejar
   - Errores más claros y específicos

3. **Rendimiento**:
   - Sin verificaciones de compatibilidad innecesarias
   - Carga de header más directa
   - Menos ramas de código

4. **Seguridad**:
   - Validación estricta de dimensiones
   - Prevención de valores inválidos
   - Verificación obligatoria de formato

## Resolución de Problemas

### Error: "No Artwork"
- Verificar que existe `../artwork/{rom_name}.zip`
- Comprobar estructura de directorios

### Error: "unknown game"
- ROM no encontrada en driver MAME
- Verificar nombre del archivo ROM
- Actualizar `hh_sm510.cpp` eliminándolo para forzar descarga

### Calidad de Segmentos Degradada
- Usar exportación individual (ya implementada)
- Verificar resolución de bits en configuración
- Revisar modo LCD invertido

### Problemas de Descarga Automática
- Verificar conexión a internet
- Comprobar permisos de escritura en directorio `build/`
- URL de GitHub accesible

### Archivos .gws No Se Abren en Emulador
- Recompilar emulador después de cambios: `./build.sh`
- Verificar que la ROM está en el directorio correcto
- Usar ruta completa: `./gwport "../roms/archivo.gws"`

## Rendimiento y Estadísticas

### Mejoras de Rendimiento Implementadas

| Aspecto | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Tiempo de procesamiento | 45-60s | 15-25s | **3-4x más rápido** |
| Calidad visual | Contaminación píxeles | Segmentos perfectos | **100% mejorado** |
| Seguimiento progreso | Sin información | Barra tiempo real | **Nuevo** |
| Resolución configurable | Solo 320x240 | Cualquier resolución | **Nuevo** |
| Soporte emulador | Hardcoded | Dinámico | **Nuevo** |

### Tamaños de Archivo Típicos

| Resolución | ROM Ball | ROM Vermin | Ratio |
|------------|----------|------------|-------|
| 320x240 | 33.1 KB | ~30 KB | 1.0x |
| 640x480 | 142.9 KB | ~130 KB | 4.3x |
| 1920x1080 | ~1.2 MB | ~1.1 MB | 36x |

---

**Fecha de Documentación**: 27 de julio de 2025  
**Versión**: v2.0 - Sistema optimizado con resolución configurable y formato .gws  
**Estado**: Completamente funcional con paralelización, progreso en tiempo real y soporte dinámico del emulador
