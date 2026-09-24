# Evaluación Integral de Viabilidad y Plan de Ejecución del Programa IEEE CASS UMSA 2026

---

## PARTE 1: EVALUACIÓN GENERAL DE VIABILIDAD

### 1.1 Coherencia técnica y pedagógica del proyecto

El proyecto describe la implementación de un sistema integral de Edge-AI sustentado en el co-diseño de hardware y software, articulado a través de un flujo de datos real y coherente:

```
ADXL345 (sensor físico con ruido)
    │
    ▼
SPI Driver (C en ARM Cortex-M3)
    │
    ▼
Acelerador HW en FPGA (cálculo de características ML: varianza, etc.)
    │
    ▼
Bus AMBA → ARM Cortex-M3
    │
    ▼
Filtro de Kalman Adaptativo (C + CMSIS-DSP)
    │  ← parámetros Q, R reconfigurados por el modelo ML
    ▼
UART → PC (visualización de señal limpia)
```

La arquitectura representa un caso canónico de co-diseño HW/SW: el procesamiento matemáticamente intensivo (extracción de características) se delega a la matriz FPGA, mientras que las tareas de control, la lógica adaptativa y el filtrado residen en el procesador embebido. Desde la perspectiva formativa, la estructura expone con claridad el aporte específico de cada estrato del sistema al comportamiento global.

**Veredicto: La formulación se cataloga como técnicamente sólida y de elevado rigor pedagógico.**

---

### 1.2 Viabilidad temporal en un esquema de seis sesiones de cuatro horas (24 h acumuladas)

El margen temporal resulta sumamente ajustado; no obstante, el cumplimiento de los objetivos es factible si se adoptan medidas metodológicas preventivas. El diagnóstico por módulo se detalla a continuación:

| Módulo | Tema | Dificultad real | Tiempo asignado | Veredicto |
|--------|------|-----------------|-----------------|-----------|
| 1 | Arquitectura SoC FPGA | Media | 4 h | **Factible**, sujeto a la preinstalación de herramientas |
| 2 | Online ML en Python | Media | 4 h | **Factible**, circunscrito a simulación algorítmica pura |
| 3 | Aceleración HW en FPGA (Verilog + AMBA) | **Muy alta** | 4 h | **Esta parte está complicada; se está proponiendo una modificación** para mitigar la ambición del alcance original |
| 4 | Interfaz SPI del ADXL345 | Media-Alta | 4 h | **Factible**, condicionado a la configuración del MSS Configurator |
| 5 | Filtro de Kalman Adaptativo | Alta | 4 h | **Factible**, supeditado a la operatividad previa del driver SPI |
| 6 | Integración completa + benchmarking | **Muy alta** | 4 h | **Esta parte está complicada; se está proponiendo una modificación** para evitar la integración total desde cero |

**Los puntos críticos del cronograma se localizan en el Módulo 3 y el Módulo 6.** Sin las intervenciones sugeridas, el riesgo de no finalizar el proyecto dentro del plazo previsto es elevado.

---

### 1.3 Concordancia entre objetivos modulares y disponibilidad temporal

Los objetivos presentan una correspondencia general adecuada, con excepción de dos instancias críticas donde se requieren adaptaciones formales:

**Módulo 3 – Limitación identificada:** Los objetivos señalan formalmente la necesidad de *"Crear un periférico IP personalizado para acelerar el cálculo de características del ML en paralelo"* e *"Integrar el IP en el bus del sistema AMBA"*. El diseño íntegro de un núcleo IP con interfaz de bus AMBA APB desde cero, la articulación de bloques DSP, su síntesis y posterior enlace en el MSS del SmartFusion2 exige plazos que exceden el marco de cuatro horas para un perfil de estudiante con conocimientos básicos en VHDL/Verilog.

> **Esta parte está complicada; se está proponiendo esta modificación:** Proporcionar un **template de IP con la interfaz AMBA APB ya implementada** (que incluya wrapper de registros, decodificación de direcciones y señales de sincronización/handshaking). Bajo esta modificación, los estudiantes concentran su labor en la adaptación del **datapath interno** (la lógica encargada del cálculo de varianza mediante bloques DSP). Esta adecuación reduce la complejidad en aproximadamente un 70%, reteniendo al mismo tiempo el valor conceptual de integrar aceleradores en buses de sistema.

**Módulo 6 – Limitación identificada:** El programa establece *"Integrar todos los subsistemas... ejecutar el flujo de datos completo... analizar latencia, ocupación de área y perfiles de energía... comparar cuantitativamente con analizador lógico"*. En caso de suscitarse demoras o fallas en fases precedentes, esta sesión corre el riesgo de derivar exclusivamente en depuración de errores.

> **Esta parte está complicada; se está proponiendo esta modificación:** Disponer de un **diseño de referencia pre-integrado y funcional** en calidad de respaldo técnico. Si los grupos presentan retrasos en la integración, se procede a cargar el diseño de referencia para enfocar la sesión en el benchmarking y la caracterización funcional. Asimismo, se propone orientar el análisis de "perfiles de energía" hacia un marco conceptual/teórico, considerando que las mediciones directas requieren instrumentación dedicada no contemplada en el inventario.

---

### 1.4 Secuencia pedagógica y articulación entre módulos

El encadenamiento de dependencias técnicas y conceptuales se articula del siguiente modo:

```
Módulo 1 (SoC, herramientas, LEDs, UART)
    │  → Provee: entorno funcional, comprensión de la arquitectura
    ▼
Módulo 2 (Online ML en Python)
    │  → Provee: comprensión del algoritmo ML, cuantización punto fijo
    ▼
Módulo 3 (Acelerador HW en Verilog)
    │  → Provee: IP de hardware que calcula características
    ▼
Módulo 4 (Driver SPI para ADXL345)
    │  → Provee: datos reales del sensor
    ▼
Módulo 5 (Filtro de Kalman)
    │  → Provee: algoritmo de filtrado adaptativo
    ▼
Módulo 6 (Integración completa)
    → Producto final: sistema funcional
```

**Observación técnica sobre la secuencia:** El Módulo 2 (Python/ML) resulta **conceptualmente indispensable**, aunque se encuentra **desconectado operativamente** del flujo de síntesis en hardware, dado que el código generado no se exporta de forma automática al silicio. El nexo efectivo radica en que el estudiante determina *qué* parámetros matemáticos procesará el acelerador del Módulo 3 (varianzas y descriptores estadísticos) y *cómo* opera la cuantización a punto fijo. Para formalizar dicha articulación, el entregable del Módulo 2 se define como un **documento de especificación formal** que establezca: *"el acelerador HW calculará variable X, con Y bits de resolución, bajo el formato de punto fijo Z"*.

**Secuencia alternativa:** El Módulo 4 (desarrollo del driver SPI) posee independencia respecto al acelerador y suministra los datos empíricos que justifican la aceleración por hardware; no obstante, el esquema propuesto se sostiene válidamente empleando patrones de prueba sintéticos durante el Módulo 3.

---

### 1.5 Idoneidad técnica de la FPGA seleccionada

La verificación de requerimientos frente a las especificaciones del manual de usuario de la plataforma Polaris arroja los siguientes datos:

| Requisito del curso | Recurso en SmartFusion2 M2S005 | Verificado |
|---|---|---|
| Procesador ARM Cortex-M3 | Integrado en el MSS | ✅ Sección 3 del manual |
| Matriz FPGA para acelerador HW | 6,060 4-LUTs, 6,060 DFFs | ✅ Sección 2 |
| Bloques DSP para ML | 11 multiplicadores de 18×18 | ✅ Sección 2 |
| Memoria para firmware | 64 KB eSRAM + 128 KB eNVM | ✅ Sección 3 |
| SPI para ADXL345 | 2 periféricos SPI en el MSS | ✅ Sección 3 |
| UART para PC | 2 periféricos UART en el MSS | ✅ Sección 3 |
| Reloj de sistema | 50 MHz (pin K1) | ✅ Sección 7 |
| Programador integrado | FlashPro5 integrado | ✅ Sección 5 |
| Acelerómetro ADXL345 | En placa, accesible por bus SPI | ✅ Sección 10 |
| Niveles lógicos | 3.3 V en la totalidad de pines | ✅ Todas las tablas |

**Veredicto: La plataforma seleccionada satisface holgadamente las demandas del proyecto.** La dotación de 6,060 LUTs es suficiente para aceleradores estadísticos compactos. Los 11 bloques multiplicadores de 18×18 atienden el cálculo de varianzas y rutinas ligeras de ML, al tiempo que la eSRAM de 64 KB resulta adecuada para el firmware del filtro de Kalman basado en CMSIS-DSP.

**Nota técnica:** El soporte documental base corresponde al **manual de usuario de la tarjeta Polaris** y no al datasheet exhaustivo del SmartFusion2 M2S005. Si bien dicho manual satisface la asignación de pines y mapeo de periféricos, el análisis de temporización fina o configuraciones profundas del MSS demandará la consulta de las hojas de datos de Microchip.

---

### 1.6 Compatibilidad del entorno y periféricos

| Elemento | Compatible | Observaciones |
|---|---|---|
| Libero SoC v12+ | ✅ | Suite de diseño oficial para la arquitectura SmartFusion2 |
| SoftConsole IDE | ✅ | Entorno Eclipse para el firmware sobre Cortex-M3 |
| CMSIS-DSP | ✅ | Librería matemática estandarizada para núcleos Cortex-M |
| FlashPro5 | ✅ | Circuito de programación embebido en la placa base |
| ADXL345 | ✅ | Sensor montado en placa con pines documentados |
| Analizador lógico | ⚠️ | Dispositivo contemplado en el cronograma sin especificación de modelo ni interfaz |
| Python (numpy, scipy) | ✅ | Herramienta computacional base para el Módulo 2 |

**Consideración de asignación de pines:** La documentación de la plataforma Polaris etiqueta las señales del acelerómetro bajo las nomenclaturas `ADXL_SDA` (pin N4), `ADXL_SCL` (pin P3), `ADXL_SDO` (pin N3) y `ADXL_CS` (pin M3). Aunque la nomenclatura sugiere un bus I²C, la presencia de la línea activa en bajo `ADXL_CS` ratifica la operación en modo SPI. Para efectos de implementación, las equivalencias formales son:
- `ADXL_CS` (M3) → Chip Select (activo bajo)
- `ADXL_SCL` (P3) → SCLK (reloj SPI)
- `ADXL_SDA` (N4) → MOSI/SDI (transmisión hacia el ADXL)
- `ADXL_SDO` (N3) → MISO/SDO (recepción desde el ADXL)

La exposición de estas equivalencias al inicio de la práctica resulta fundamental para evitar errores de conexión conceptual.

---

### 1.7 Dependencias técnicas previas a la ejecución del curso

| Dependencia | Nivel de Criticidad | Acción requerida |
|---|---|---|
| Libero SoC instalado y licenciado | **Crítica** | Instalación obligatoria antes de la Sesión 1. El proceso demanda entre 1 y 2 horas y requiere el registro de licencia Silver gratuita en Microchip |
| SoftConsole IDE operativo | **Crítica** | Despliegue del IDE previo al inicio del programa |
| Controladores FlashPro5 | **Crítica** | Verificación del reconocimiento del programador por el sistema operativo al conectar la placa (Sección 5 del manual) |
| Entorno Python con numpy/scipy/matplotlib | Importante | Preparación del entorno de simulación previo a la Sesión 2 |
| Instrumentación (analizador lógico) | Importante | Confirmación de disponibilidad para la Sesión 6, previendo técnicas sustitutivas como toggling de pines GPIO y osciloscopio |
| Lote de tarjetas Polaris verificado | **Crítica** | Comprobación física individual previa: ciclo de programación, interfaz UART y respuesta del ADXL345 |
| Cables USB Tipo-C | Crítico | Suministro de un cable por estación de trabajo (suministro de energía y enlace de programación) |
| Puertos USB funcionales | Crítico | Verificación de compatibilidad en los equipos de cómputo del laboratorio |

---

### 1.8 Riesgos de diseño y delimitación técnica

**Aspectos formales menores:**
1. El documento fuente en LaTeX presenta el error sintáctico `\ormalsize` en reemplazo de `\normalsize`.
2. Se hace mención a instrumentación genérica de análisis lógico sin explicitar cantidades ni especificaciones de conexión.

**Riesgos técnicos y medidas de contención:**

1. **Procedimiento en MSS Configurator:** La parametrización del subsistema de microcontrolador en Libero SoC exige configurar dominios de reloj, periféricos y generación de drivers. Errores en esta etapa comprometen la operatividad del procesador ARM. **Esta parte está complicada; se está proponiendo esta modificación:** Suministrar un proyecto base pre-configurado en Libero SoC con los subsistemas UART y SPI ya validados, disponible como punto de partida o plantilla de contingencia.

2. **Enrutamiento de líneas SPI a través de la fabric:** Los pines asignados al acelerómetro ADXL345 (N4, P3, N3, M3) residen en la matriz lógica (fabric) y no en el banco dedicado del MSS. Las señales del controlador SPI del microcontrolador deben encaminarse internamente hacia dichos terminales mediante un envoltorio (wrapper) en HDL. **Esta parte está complicada; se está proponiendo esta modificación:** Incorporar explícitamente en el material formativo las instrucciones y el código HDL de enlace para el puenteo de señales entre el MSS y la matriz lógica.

3. **Interconexión AMBA del acelerador en hardware:** El microcontrolador SmartFusion2 utiliza un bus AHB interno para la interconexión con la lógica programable. La integración del bloque IP demanda implementar interfaces esclavas APB o AHB, cuyo diseño desde cero introduce una elevada complejidad protocolar. **Esta parte está complicada; se está proponiendo esta modificación:** Proporcionar una plantilla funcional predefinida del bus APB esclavo o emplear los componentes de generación automática provistos por Libero SoC.

4. **Conversión a representación en punto fijo:** La traslación algorítmica desde Python (precisión flotante) hacia Verilog (representación en punto fijo) suele inducir distorsiones numéricas. Se establece como requisito clarificar el esquema de mapeo de las lecturas del sensor (registros signados de 10 o 13 bits) dentro del formato de punto fijo del acelerador.

---

### 1.9 Propuestas de modificación y simplificación operativa

| Modificación propuesta | Justificación técnica | Impacto en el programa |
|---|---|---|
| **Distribución de proyecto Libero SoC pre-configurado** (UART y SPI operativos, clocks enlazados) | Esta parte está complicada en tiempo real; se propone la plantilla para suprimir demoras en la parametrización inicial del MSS | Focaliza el Módulo 1 en el aprendizaje de la arquitectura y primeras pruebas de validación |
| **Suministro de template de IP con interfaz AMBA APB** para el Módulo 3 | Esta parte está complicada para cubrirse en 4 horas desde cero; se propone el template con el protocolo de bus resuelto | Permite que el participante trabaje directamente sobre el datapath y el aprovechamiento de los bloques DSP |
| **Disponibilidad de un diseño de referencia integral** como contingencia en el Módulo 6 | Esta parte está complicada si surgen fallos acumulados; se propone el sistema de respaldo | Garantiza la consecución de las métricas de benchmarking e inspección de rendimiento |
| **Adelanto de la configuración del SPI del MSS al Módulo 1** | El bus se emplea en el Módulo 4 y su configuración consume tiempo | Libera al Módulo 4 para dedicarse exclusivamente al firmware del controlador en C |
| **Orientación analítica de los perfiles de consumo energético** | La medición experimental requiere hardware de sensado de corriente externo | Previene desfases operativos en la sesión de cierre |
| **Definición de un entregable formal en el Módulo 2** (especificación del acelerador) | Articula conceptualmente los modelos de simulación en Python con el diseño en Verilog | Asegura continuidad algorítmica entre los Módulos 2 y 3 |

---

### 1.10 Mapa de flujo: diseño incremental y activos transferibles

| Día | Módulo | Desarrollado en la sesión | Transferido de la fase previa | Entregable principal |
|---|---|---|---|---|
| 1 | Arquitectura SoC | Proyecto Libero SoC, MSS generado, prueba de LEDs, UART validado | Ninguno (etapa inicial) | Proyecto de hardware en Libero SoC y proyecto de firmware en SoftConsole |
| 2 | Online ML | Simulación en Python del algoritmo, definición de cuantización | Comprensión del modelo de procesamiento del SoC | Script algorítmico y memoria de especificación técnica del acelerador |
| 3 | Aceleración HW | Módulo IP en Verilog sobre bloques DSP, enlace AMBA | Proyecto de hardware del Día 1 + especificación del Día 2 | Módulo IP sintetizable integrado en la topología de bus del sistema |
| 4 | Interfaz ADXL345 | Driver de comunicación SPI en C, adquisición de muestras | Proyecto de Libero SoC con MSS SPI + espacio de trabajo de SoftConsole | Driver operativo con lectura triaxial y transmisión serie vía UART |
| 5 | Filtro Kalman | Implementación del filtro adaptativo en C empleando CMSIS-DSP | Controlador SPI del Día 4 + acelerador del Día 3 | Algoritmo de Kalman en ejecución con parámetros $Q$ y $R$ ajustables |
| 6 | Integración | Sistema completo acoplado, benchmarking y visualización | Integración acumulada de etapas precedentes | Sistema integral operando en la placa Polaris y reporte métrico |

**Condicionantes críticos de ejecución:**
1. Incidencias en la licencia o instalación de Libero SoC/SoftConsole impiden la continuidad del plan.
2. Desajustes en la generación del MSS inhabilitan las actividades de los Módulos 4 y 5.
3. Fallas en la síntesis o mapeo del bus en el IP del Módulo 3 detienen la integración del Módulo 6.
4. Falta de lectura en el driver SPI interrumpe el flujo de datos hacia las fases 5 y 6.

---
---

## PARTE 2: MÓDULO 1 – PLAN DE EJECUCIÓN DETALLADO

### 2.1 Fase de preparación previa a la Sesión 1

#### 2.1.1 Configuración de herramientas de software

**Suite Libero SoC (versión v12 o superior):**
1. Descarga del instalador de Libero SoC v12.x desde el portal de Microchip para la plataforma operativa correspondiente (Windows o Linux).
2. Registro institucional en Microchip para la obtención de la **licencia Silver (sin coste)**, la cual autoriza el trabajo con la familia SmartFusion2 hasta el dispositivo M2S010 (abarcando el silicio M2S005 de la plataforma Polaris).
3. Ejecución del proceso de instalación del software.
4. Vinculación de licencia mediante `Help → License Setup`, enlazando el archivo local o el servidor de licencias asignado.
5. **Comprobación:** Apertura de la herramienta, creación de un proyecto preliminar y verificación de ausencia de restricciones de licencia.

**Entorno de desarrollo SoftConsole:**
1. Descarga del paquete SoftConsole desde el repositorio oficial de Microchip para desarrollo sobre núcleos ARM Cortex-M3.
2. Instalación formal del entorno en los equipos de desarrollo.
3. **Comprobación:** Inicio del entorno y comprobación de la operatividad del compilador.

**Entorno computacional Python (soporte para el Módulo 2):**
1. Instalación de Python 3.8 o superior.
2. Incorporación de librerías científicas requeridas: `pip install numpy scipy matplotlib scikit-learn`.
3. **Comprobación:** Ejecución en terminal del comando de validación: `python -c "import numpy; print(numpy.__version__)"`.

#### 2.1.2 Certificación funcional del hardware

Procedimiento aplicado a **cada unidad Polaris** asignada:

1. Interconexión de la placa al equipo mediante interfaz USB Tipo-C.
2. Validación de reconocimiento del dispositivo FlashPro5 por el sistema operativo:
   - **En Windows:** Acceso al Administrador de Dispositivos, localizando el programador FlashPro5 bajo la categoría de controladores USB y comprobando la habilitación de 4 puertos COM virtuales (Sección 6 del manual).
   - **En Linux:** Ejecución de `lsusb` para verificar la enumeración de Microchip e inspección de salidas `ttyUSB` mediante `dmesg | tail`.
3. Instalación manual de los controladores de programación en caso de no ser detectados automáticamente.
4. Apertura del programador en Libero SoC vía `Tools → FlashPro5` para constatar la detección de la cadena de escaneo JTAG.
5. Registro de los identificadores COM asignados, considerando que **únicamente los tres últimos puertos quedan configurados para comunicaciones UART**, manteniéndose el primer canal reservado para funciones del depurador FlashPro5.

#### 2.1.3 Disposición del material formativo

Organización de la carpeta de trabajo `Modulo1_Material/` con los siguientes elementos:
- Guía de laboratorio detallada en formato digital con diagramas de paso.
- Proyecto base en Libero SoC pre-configurado (propuesto como recurso de respaldo).
- Documento de asignación y correspondencia de puertos serie para cada estación.

#### 2.1.4 Protocolo de validación técnica previa

Se establece como requisito metodológico la ejecución y validación completa del Módulo 1 de manera anticipada sobre una placa Polaris, cubriendo los siguientes hitos:
- Inicialización del proyecto en Libero SoC.
- Generación de componentes del MSS.
- Escritura del módulo HDL para oscilación de diodos emisores de luz.
- Flujo de síntesis, posicionamiento, enrutamiento y programación del silicio.
- Configuración del proyecto de firmware en SoftConsole.
- Codificación y compilación del controlador de transmisión UART en C.
- Comprobación del enlace de comunicación bidireccional hacia la estación de cómputo.

---

### 2.2 Marco conceptual de exposición

#### 2.2.1 Arquitectura del SmartFusion2 (Tiempo expositivo: 30 minutos)

Estructura de distribución funcional del dispositivo:

```
┌─────────────────────────────────────────────────────┐
│            SmartFusion2 M2S005                      │
│                                                     │
│  ┌──────────────────────┐   ┌────────────────────┐  │
│  │  MSS (Microcontroller│   │   FPGA Fabric      │  │
│  │  Subsystem)          │   │                    │  │
│  │                      │   │  6,060 4-LUTs      │  │
│  │  ARM Cortex-M3       │◄─►│  6,060 DFFs        │  │
│  │  64 KB eSRAM         │   │  10 LSRAM (18K)    │  │
│  │  128 KB eNVM         │   │  11 Mult (18×18)   │  │
│  │  2× UART             │AHB│  2× PLL/CCC        │  │
│  │  2× SPI              │APB│                    │  │
│  │  2× I²C              │   │  [IP personalizado]│  │
│  │  1× CAN              │   │  [Acelerador ML]   │  │
│  │  1× USB HS           │   │                    │  │
│  └──────────────────────┘   └────────────────────┘  │
│         │                          │                │
│         ▼                          ▼                │
│    Periféricos MSS           Pines FPGA             │
│    (pines dedicados)         (GPIO, H0, H1)         │
└─────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
    UART → PC              LEDs, Switches,
    SPI → ADXL345          ADXL345, etc.
```

**Principios arquitectónicos a desarrollar:**
- **Coexistencia MSS - Fabric:** El subsistema MSS aloja el núcleo de procesamiento de programa fijo (ARM Cortex-M3 con periféricos dedicados), en tanto que la Fabric provee la matriz lógica programable. Ambas entidades operan de forma síncrona dentro del mismo encapsulation.
- **Jerarquía de buses AMBA:** La transferencia de datos entre el subsistema de procesamiento y la lógica en silicio se gestiona mediante especificaciones AMBA (bus AHB orientado a transacciones de alto ancho de banda y bus APB adaptado al control de periféricos y registros de estado).
- **Ventajas de integración:** La conjunción en un solo silicio optimiza la latencia de interconexión, disminuye el consumo y simplifica el trazado en la placa de circuito impreso en comparación con arquitecturas discretas.

#### 2.2.2 Descripción del entorno de desarrollo (15 minutos)

- **Libero SoC:** Software responsable de la descripción estructural del hardware, síntesis lógica, configuración del procesador integrado y compilación del flujo de bits (bitstream).
- **SoftConsole:** Entorno IDE enfocado en el desarrollo, compilación cruzada y depuración de software en lenguaje C para el procesador ARM Cortex-M3.
- **FlashPro5:** Hardware programador integrado que canaliza la descarga física tanto a las celdas de configuración no volátiles como a la memoria de programa del sistema.

#### 2.2.3 Diagrama del ciclo de diseño (15 minutos)

```
Libero SoC:
  Crear proyecto → Configurar MSS → Escribir HDL → 
  Build Hierarchy → Synthesis → Pin Assignment → 
  Place & Route → Generate Bitstream → Program FPGA

SoftConsole:
  Crear proyecto → Escribir C → Build → 
  Debug/Run → Program ARM Cortex-M3
```

---

### 2.3 Requisitos de software e instrumentación

| Herramienta | Versión | Procedimiento de validación |
|---|---|---|
| Libero SoC | v12.x | Verificación de arranque y estado de licencia Silver |
| SoftConsole | Edición actual | Validación de compilación sobre proyecto vacío |
| Drivers FlashPro5 | Distribución oficial | Comprobación de reconocimiento de puertos COM |
| Terminal serie (PuTTY, Tera Term, minicom) | Compatible | Apertura y enganche sobre el puerto COM virtual |

---

### 2.4 Guía de implementación técnica paso a paso

#### PASO 1: Creación del entorno de trabajo en Libero SoC (30 minutos)

1. Inicialización de Libero SoC.
2. Selección de la secuencia `File → New Project`.
3. Asignación de parámetros en el asistente:
   - **Project Name:** `Polaris_Lab1`
   - **Project Location:** Directorio local excluyendo espacios o caracteres especiales en la ruta (por ejemplo, `C:\Polaris_Lab1` o `/home/user/Polaris_Lab1`).
   - **Project Type:** HDL Design.
   - **Device Family:** SmartFusion2.
   - **Device:** M2S005.
   - **Package:** TQ144 (ajustar conforme a la rotulación física del encapsulated de la placa Polaris).
   - **Speed Grade:** -1.
   - **HDL Language:** Verilog (homologado para articulación con el Módulo 3).
4. Conclusión mediante la opción **Finish**.

**Condición de éxito:** El proyecto se inicializa presentando la jerarquía del flujo de diseño en el panel "Design Flow".

**Punto de control:** Ante anomalías vinculadas al gestor de licencias, se procede a reconfigurar los parámetros en `Help → License Setup`.

---

#### PASO 2: Parametrización del Microcontroller Subsystem (MSS) (45 minutos)

Esta etapa concentra la definición estructural de periféricos y dominios de control.

1. En el panel Design Flow, se ejecuta la acción sobre **"Create MSS Component"** (o navegación mediante `Tools → MSS Configurator`).
2. Se accede a la interfaz del configurador MSS para fijar los parámetros del Cortex-M3.

**Ajustes requeridos en el MSS:**

a) **Generación del reloj del sistema (System Clock):**
   - Se selecciona como referencia la fuente externa de 50 MHz conectada físicamente al pin K1 de la placa (Sección 7 del manual).
   - Se asigna el oscilador principal para alimentar el sistema MSS.
   - Se fija la frecuencia del procesador ARM Cortex-M3 en 50 MHz.

b) **Habilitación de interfaz UART_0:**
   - Navegación hacia la pestaña de periféricos serie y selección de UART.
   - Activación del bloque **MMUART_0**.
   - Definición de la tasa de transferencia en **115200 bps**.
   - **Consideración de enrutamiento:** Dado que la interfaz FlashPro5 se encuentra acoplada a la matriz general de la FPGA, se configuran las líneas TX y RX para su canalización a través de la **FPGA Fabric** hacia los pines asignados en la Tabla 1 del manual:
     - RX0 → Pin FPGA V11
     - TX0 → Pin FPGA W11

c) **Puertos de propósito general (GPIO):**
   - Habilitación de líneas GPIO auxiliares para control de señalización desde software.

d) **Segmentación de memoria:**
   - Confirmación de disponibilidad de los bloques eSRAM (64 KB) y eNVM (128 KB).
   - Selección de la memoria eSRAM como destino de ejecución de depuración.

3. **Compilación y generación del subsistema:**
   - Selección del comando **"Generate MSS Component"**.
   - El proceso sintetiza el bloque estructural en Libero SoC e instrumenta los archivos de cabecera y controladores pertenecientes a la capa HAL (Hardware Abstraction Layer).

**Condición de éxito:** Incorporación del bloque MSS en el esquema de diseño, exponiendo las líneas de bus de reloj, control, enlaces AMBA y señales serie UART.

**Puntos de control:**
- Asegurar la correspondencia del encapsulado y familia del dispositivo.
- Verificar rutas de almacenamiento libres de espacios tipográficos.
- Verificar cobertura de funciones bajo la licencia Silver.

**Archivos generados para integración posterior:**
- Bloque del componente MSS (`.cxz`).
- Controladores HAL generados en C (`mss_uart.h`, `mss_gpio.h`).
- Archivo de definición del subsistema (`.xml` / `.mss`).

---

#### PASO 3: Descripción estructural HDL – Secuenciador de LEDs (45 minutos)

Se procede a la integración de un módulo descriptivo en Verilog para gestionar la secuencia de activación en los terminales de salida.

**Definición del módulo secundario:** `led_blink.v`

```verilog
module led_blink (
    input  wire       clk_50mhz,    // Reloj de 50 MHz (pin K1)
    input  wire       reset_n,      // Reset activo bajo (de un switch)
    output reg  [9:0] leds          // 10 LEDs de usuario
);

    // Contador para generar una señal de ~1 Hz
    // 50,000,000 / 2 = 25,000,000 para medio período
    reg [24:0] counter;
    reg        tick_1hz;

    always @(posedge clk_50mhz or negedge reset_n) begin
        if (!reset_n) begin
            counter  <= 25'd0;
            tick_1hz <= 1'b0;
        end else begin
            if (counter == 25'd24_999_999) begin
                counter  <= 25'd0;
                tick_1hz <= ~tick_1hz;
            end else begin
                counter <= counter + 25'd1;
            end
        end
    end

    // Patrón de parpadeo: shift register
    always @(posedge clk_50mhz or negedge reset_n) begin
        if (!reset_n) begin
            leds <= 10'b0000000001;
        end else if (tick_1hz) begin
            leds <= {leds[8:0], leds[9]}; // Rotación circular
        end
    end

endmodule
```

**Definición del módulo de nivel superior:** `top.v`

```verilog
module top (
    input  wire       CLK_50MHZ,   // Pin K1
    input  wire       SW0,         // Pin D6 (reset)
    output wire [9:0] LED          // LEDs LED0-LED9
);

    led_blink u_led_blink (\
        .clk_50mhz (CLK_50MHZ),\
        .reset_n   (SW0),\
        .leds      (LED)\
    );

endmodule
```

**Incorporación de elementos al entorno:**
1. Selección de `Project → Add Files` integrando los archivos `top.v` y `led_blink.v`.
2. Asignación del archivo raíz haciendo clic derecho sobre `top` y seleccionando **"Set As Root"**.

**Construcción de la jerarquía de diseño:**
1. Ejecución del comando **"Build Hierarchy"** en el panel de flujo.
2. Confirmación de resolución estructural sin inconsistencias.

---

#### PASO 4: Asignación física de terminales (30 minutos)

Con base en la **Tabla 2** (Reloj), **Tabla 3** (Interruptores) y **Tabla 4** (Diodos LED) de la documentación técnica de la tarjeta Polaris:

| Señal lógica (`top.v`) | Terminal físico FPGA | Identificador en circuito |
|---|---|---|
| `CLK_50MHZ` | K1 | CLK_50MHZ |
| `SW0` | D6 | SW0 |
| `LED[0]` | N1 | LED0 |
| `LED[1]` | M2 | LED1 |
| `LED[2]` | M1 | LED2 |
| `LED[3]` | C1 | LED3 |
| `LED[4]` | B1 | LED4 |
| `LED[5]` | B2 | LED5 |
| `LED[6]` | A2 | LED6 |
| `LED[7]` | A7 | LED7 |
| `LED[8]` | B9 | LED8 |
| `LED[9]` | A10 | LED9 |

**Asignación interactiva:**
1. Apertura de la herramienta gráfica en `Design → I/O Editor`.
2. Mapeo individualizado de puertos lógicos hacia terminales de silicio.
3. Fijación del estándar de interfaz eléctrica en **LVCMOS33** ($3.3\text{ V}$) de acuerdo con las especificaciones generales.
4. Almacenamiento de parámetros.

**Asignación mediante archivo de restricciones (PDC):**
Se puede estructurar directamente el fichero de restricciones físicas (`pins.pdc`):

```tcl
# Archivo: pins.pdc
set_io -port_name CLK_50MHZ -pin_name K1
set_io -port_name SW0 -pin_name D6
set_io -port_name LED[0] -pin_name N1
set_io -port_name LED[1] -pin_name M2
set_io -port_name LED[2] -pin_name M1
set_io -port_name LED[3] -pin_name C1
set_io -port_name LED[4] -pin_name B1
set_io -port_name LED[5] -pin_name B2
set_io -port_name LED[6] -pin_name A2
set_io -port_name LED[7] -pin_name A7
set_io -port_name LED[8] -pin_name B9
set_io -port_name LED[9] -pin_name A10
```

---

#### PASO 5: Síntesis lógica y Enrutamiento (Place & Route) (20 minutos)

1. Activación del proceso **"Synthesis"** en el panel central de Libero SoC.
2. Confirmación de conclusión satisfactoria del compilador de síntesis.
3. Ejecución del proceso **"Place and Route"**.
4. Inspección del reporte para constatar convergencia en tiempos y ausencia de conflictos físicos.

**Condiciones de ajuste:**
En caso de advertencias vinculadas a modelos temporales de reloj, se procede a incorporar la restricción de periodo mediante un fichero SDC (`timing.sdc`):
```tcl
# Archivo: timing.sdc
create_clock -name clk_50mhz -period 20.0 [get_ports CLK_50MHZ]
```

---

#### PASO 6: Transferencia y programación de la FPGA (15 minutos)

1. Acoplamiento de la plataforma Polaris mediante la interfaz USB.
2. Apertura del programador mediante `Tools → Program Device` o activación de la tarea **"Program Device"**.
3. Validación de selección del controlador **FlashPro5**.
4. Orden de descarga pulsando **"Program"**.
5. Conclusión del ciclo tras la confirmación de transferencia.

**Comportamiento esperado en hardware:** La barra de LEDs ejecuta una transición circular secuencial activa. El interruptor SW0 en nivel bajo ($0\text{ V}$) sostiene la condición de reinicio (reset); al situarse en nivel alto ($3.3\text{ V}$), el patrón circular inicia su ciclo.

**Diagnóstico ante anomalías de programación:**
- Comprobación del enlace del programador en la Sección 5 del manual.
- Inspección de alimentación eléctrica general en la tarjeta (LED de línea encendido).
- Constatación de la posición del interruptor SW0 en estado habilitado.
- Examen de concordancia de pines asignados en la herramienta I/O Editor.

---

#### PASO 7: Configuración de firmware en SoftConsole y verificación de enlace UART (60 minutos)

1. Ejecución del entorno **SoftConsole**.
2. Selección de la secuencia `File → New → SoftConsole Project`.
3. Establecimiento del tipo de plataforma base: **"Microchip SmartFusion2 MSS"** (o en su defecto, proyecto de base vacía).
4. Parametrización del objetivo:
   - **Target Device:** SmartFusion2 M2S005.
   - **Toolchain:** GCC adaptado a núcleos ARM embebidos.
5. **Importación de la capa HAL del MSS:**
   - Traslado de los ficheros fuente generados por el MSS Configurator hacia la jerarquía del proyecto en SoftConsole.
   - Vinculación de rutas de inclusión de cabeceras en `Project → Properties → C/C++ Build → Settings → Include Paths`.

**Estructura del archivo de aplicación:** `main.c`

```c
#include <stdint.h>
#include "mss_uart.h"  // Driver HAL del UART del MSS

// Instancia del UART
mss_uart_instance_t *const g_uart = &g_mss_uart0;

// Mensaje de prueba
const char *msg = "\r\n[Modulo 1] UART funcional - IEEE CASS UMSA 2026\r\n";

int main(void) {
    // Inicializar UART a 115200 baud, 8N1
    MSS_UART_init(g_uart, MSS_UART_115200_BAUD, 
                  MSS_UART_DATA_8_BITS | MSS_UART_NO_PARITY | MSS_UART_ONE_STOP_BIT);
    
    // Enviar mensaje
    MSS_UART_polled_tx_string(g_uart, (const uint8_t *)msg);
    
    // Bucle principal: eco de caracteres recibidos
    uint8_t rx_byte;
    while (1) {
        if (MSS_UART_get_rx(g_uart, &rx_byte, 1) > 0) {
            // Eco: reenviar lo recibido
            MSS_UART_polled_tx(g_uart, &rx_byte, 1);
        }
    }
    
    return 0;
}
```

**Consideración sobre la API HAL:** Los nombres de las funciones pueden presentar variaciones formales según la revisión de software del generador MSS, verificándose habitualmente bajo las llamadas estándar:
- `MSS_UART_init()`
- `MSS_UART_polled_tx()` o `MSS_UART_polled_tx_string()`
- `MSS_UART_get_rx()` o `MSS_UART_polled_rx()`

6. **Compilación de la aplicación:** Ejecución de `Project → Build All` hasta asegurar cero errores de enlace.
7. **Carga en el procesador Cortex-M3:**
   - Enlace físico de la tarjeta al ordenador.
   - Configuración de la sesión de lanzamiento en `Run → Debug Configurations`.
   - Generación de perfil para SmartFusion2 vinculado al programador FlashPro5.
   - Ejecución de la orden mediante el comando **"Run"** o **"Debug"**.
8. **Validación sobre consola serial en ordenador:**
   - Apertura de una consola serie estándar (PuTTY, Tera Term, minicom).
   - Conexión orientada al puerto COM asignado a UART_0 (seleccionando entre los tres canales superiores generados por el controlador FlashPro5).
   - Configuración de parámetros: **115200 baudios, 8 bits de datos, sin paridad, 1 bit de parada, sin control de flujo**.
   - **Salida esperada:** Presentación de la cadena `[Modulo 1] UART funcional - IEEE CASS UMSA 2026`.
   - **Prueba de eco interactivo:** Envío de caracteres desde el teclado verificando su retorno reflejado en pantalla.

**Diagnóstico de enlace:**
- Conmutación selectiva entre los puertos COM habilitados para determinar la línea activa.
- Constatación de la velocidad de transmisión (115200 baud).
- Comprobación del ruteo entre señales internas de transmisión/recepción y los terminales V11/W11.
- Confirmación de programación finalizada informada por SoftConsole.

---

### 2.5 Matriz de validación y control del Módulo 1

Al término del módulo, se valida el cumplimiento de las siguientes métricas:

| Criterio evaluado | Mecanismo de comprobación |
|---|---|
| Secuencia cíclica en arreglo de LEDs | Inspección visual en hardware |
| Control de reposición mediante SW0 | Retorno de ciclo al conmutar posición |
| Transmisión del mensaje inicial vía UART | Recepción en terminal del ordenador |
| Retorno de eco en comunicación serie | Verificación interactiva de caracteres |
| Síntesis y enrutamiento en Libero SoC | Validación de reportes sin infracciones |
| Compilación de binarios en SoftConsole | Ausencia de advertencias críticas en consola |
| Descarga combinada FPGA y ARM | Ciclo de programación cerrado en FlashPro5 |

---

### 2.6 Registro de activos y entregables modulares

Los elementos consolidados que se trasladan a las etapas posteriores son:

| Activo producido | Utilidad en módulos subsecuentes |
|---|---|
| Directorio del proyecto Libero SoC (`Polaris_Lab1/`) | Entorno de desarrollo para Módulos 3, 4 y 6 |
| Bloque MSS configurado con soporte UART | Plataforma base para Módulos 4 y 5 |
| Módulos de descripción HDL (`top.v`, `led_blink.v`) | Estructuras a adaptar y expandir en el Módulo 3 |
| Archivo de fijación de terminales (`pins.pdc`) | Asignaciones a ampliar en Módulos 3 y 4 |
| Proyecto base de software con HAL | Espacio de desarrollo para firmware en Módulos 4 y 5 |
| Rutina base en C para gestión serie | Núcleo de salida de telemetría en Módulos 4 y 5 |
| Identificación del puerto COM de servicio | Canalización estándar de pruebas en todo el curso |

---

### 2.7 Guía de resolución de incidencias técnicas

| Falla observada | Causa técnica probable | Acción de mitigación |
|---|---|---|
| Restricción de apertura en Libero SoC | Ausencia de asignación de licencia | Ingreso a `Help → License Setup` y vinculación del fichero `.dat` |
| Falla al generar el bloque en MSS Configurator | Silicio no correspondiente o ruta con caracteres de espacio | Confirmar designación M2S005 y reubicar directorio de trabajo |
| Errores durante la síntesis lógica | Inconsistencia de código Verilog | Depuración sintáctica y fijación formal de `top` como entidad raíz |
| Incidencia en Place & Route | Conflictos o ausencia en la definición de pines | Verificación del archivo de restricciones e inspección en I/O Editor |
| Ausencia de detección del FlashPro5 | Falla en controladores o interfaz de cableado | Actualización de drivers y sustitución de cable/puerto USB |
| Proceso de programación interrumpido | Alimentación deficiente o recurso tomado por otra aplicación | Comprobación de tensión en placa y cierre de procesos concurrentes |
| Inactividad total de los LEDs | Interruptor de control SW0 en nivel de reset | Desplazamiento del interruptor SW0 hacia la posición superior (ON) |
| Falta de visualización en terminal serie | Puerto COM no coincidente o desfase de velocidad | Exploración de los canales COM habilitados y confirmación de 115200 bps |
| Salida de texto ilegible en el terminal | Desincronización en el reloj o tasa de baudios errónea | Calibración de tasa a 115200 baudios y revisión de reloj en el MSS |
| Falla de compilación en SoftConsole | Carencia de archivos HAL o rutas de inclusión omitidas | Transferencia de archivos del MSS e inclusión en el panel de compilación |

---

### 2.8 Esquema temporal de ejecución de la Sesión 1

**Actividades previas a la sesión:** Despliegue de Libero SoC, SoftConsole y controladores; certificación individual del parque de tarjetas Polaris.

**Distribución de las 4 horas de trabajo:**
- `0:00–0:30`: Exposición de fundamentos de la arquitectura SmartFusion2.
- `0:30–1:00`: Creación del proyecto base en Libero SoC.
- `1:00–1:45`: Configuración y parametrización del subsistema MSS con soporte UART.
- `1:45–2:30`: Descripción HDL del oscilador en Verilog y mapeo de restricciones de pines.
- `2:30–3:00`: Síntesis, posicionamiento, enrutamiento y programación de la matriz lógica → **Hito alcanzado: Validación del patrón en LEDs**.
- `3:00–3:45`: Estructuración del proyecto de software en SoftConsole y codificación del enlace UART.
- `3:45–4:00`: Compilación, carga en el procesador Cortex-M3 y comprobación del enlace serie → **Hito alcanzado: Comunicación UART bidireccional confirmada**.

**Balance operativo al cierre:** Espacio de trabajo en Libero SoC con MSS operativo, módulo HDL verificado en silicio y proyecto en SoftConsole comunicando telemetría vía UART de forma validada sobre la plataforma Polaris.

---

El plan contempla la continuidad hacia el **Módulo 2: Aprendizaje Automático en Línea (Online ML)**, fase en la que se define el algoritmo matemático a simular en el entorno Python, se genera el documento técnico de requerimientos para el acelerador en hardware y se establecen las reglas de cuantización en punto fijo para su posterior codificación en Verilog durante el Módulo 3.