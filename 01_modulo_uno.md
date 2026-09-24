# Evaluación Integral de Viabilidad y Plan de Ejecución del Programa IEEE CASS UMSA 2026

---

## PARTE 1: EVALUACIÓN GENERAL DE VIABILIDAD

### 1.1 Análisis de coherencia técnica y pedagógica del proyecto

El proyecto presenta sentido técnico y pedagógico. Se describe un sistema completo de Edge-AI con co-diseño hardware/software que sigue un flujo de datos real y coherente:

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

Se trata de un ejemplo representativo de co-diseño HW/SW: las operaciones matemáticamente intensivas (extracción de características) se descargan a la FPGA, mientras que el control, la lógica adaptativa y el filtrado se ejecutan en el procesador embebido. Desde el punto de vista pedagógico, la propuesta resulta adecuada, ya que los estudiantes pueden observar cómo cada capa del sistema contribuye al resultado final.

**Veredicto: Técnicamente sólido y pedagógicamente valioso.**

---

### 1.2 Análisis de factibilidad en seis días de cuatro horas (24 h totales)

La ejecución resulta extremadamente ajustada, pero factible si se adoptan precauciones específicas. La evaluación por módulo se presenta a continuación:

| Módulo | Tema | Dificultad real | Tiempo asignado | Veredicto |
|--------|------|-----------------|-----------------|-----------|
| 1 | Arquitectura SoC FPGA | Media | 4 h | **Factible**, si las herramientas se encuentran preinstaladas |
| 2 | Online ML en Python | Media | 4 h | **Factible**, corresponde a simulación pura |
| 3 | Aceleración HW en FPGA (Verilog + AMBA) | **Muy alta** | 4 h | **Demasiado ambicioso** tal como se encuentra planteado |
| 4 | Interfaz SPI del ADXL345 | Media-Alta | 4 h | **Factible**, con dependencia del MSS Configurator |
| 5 | Filtro de Kalman Adaptativo | Alta | 4 h | **Factible**, si el driver SPI se encuentra operativo |
| 6 | Integración completa + benchmarking | **Muy alta** | 4 h | **Demasiado ambicioso** si la integración se realiza desde cero |

**Los puntos críticos identificados corresponden al Módulo 3 y al Módulo 6.** Sin modificaciones, el riesgo de no completar el proyecto se considera alto.

---

### 1.3 Coherencia entre objetivos y tiempo disponible por módulo

Los objetivos resultan mayormente coherentes, con dos excepciones de relevancia:

**Módulo 3 – Condición técnica identificada:** El objetivo establece *"Crear un periférico IP personalizado para acelerar el cálculo de características del ML en paralelo"* e *"Integrar el IP en el bus del sistema AMBA"*. El diseño de un IP con interfaz AMBA APB desde cero, que utilice bloques DSP, que se sintetice correctamente y que se integre en el MSS del SmartFusion2 constituye un trabajo que normalmente requiere días, no horas. Un estudiante con "nociones básicas de VHDL/Verilog" no puede completar esta tarea en 4 horas.

> **Propuesta justificada:** Proporcionar un **template de IP con la interfaz AMBA APB ya implementada** (wrapper de registros, decodificación de direcciones, señales de handshaking). Los estudiantes únicamente deberán modificar el **datapath interno** (la lógica que utiliza los bloques DSP para el cálculo de la varianza). Esta modificación reduce la complejidad en un 70% y mantiene el valor pedagógico asociado a la comprensión de la integración de un acelerador en el bus del sistema.

**Módulo 6 – Condición técnica identificada:** El objetivo establece *"Integrar todos los subsistemas... ejecutar el flujo de datos completo... analizar latencia, ocupación de área y perfiles de energía... comparar cuantitativamente con analizador lógico"*. Si algún módulo anterior presenta problemas (situación probable), este módulo se convierte en una sesión de depuración masiva.

> **Propuesta justificada:** Disponer de un **diseño de referencia pre-integrado y funcional** como respaldo. En caso de que los estudiantes no logren integrar la totalidad del sistema, se puede cargar el diseño de referencia y enfocar la sesión en el benchmarking y análisis. Adicionalmente, se propone simplificar los "perfiles de energía" a una discusión teórica, dado que la medición de consumo real requiere instrumentación específica no contemplada en los recursos disponibles.

---

### 1.4 Secuencia pedagógica y preparación progresiva del estudiante

La cadena de dependencias se estructura de la siguiente manera:

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

**Observación relevante:** El Módulo 2 (Python/ML) resulta **conceptualmente necesario** pero **técnicamente desconectado** del flujo de implementación. Lo que se simula en Python no se "transfiere" directamente al hardware. La conexión real consiste en que el estudiante comprende *qué* debe calcular el acelerador del Módulo 3 (varianza, características estadísticas) y *cómo* se cuantiza a punto fijo. Este planteamiento es correcto desde el punto de vista pedagógico, pero debe hacerse explícito: el entregable del Módulo 2 debe consistir en un **documento de especificación** que establezca: "el acelerador HW debe calcular X, con Y bits de precisión, usando Z formato de punto fijo".

**Observación adicional:** El Módulo 4 (driver SPI) podría ubicarse antes del Módulo 3, dado que es independiente del acelerador y proporciona datos reales que motivan la necesidad de la aceleración. No obstante, el orden actual también resulta funcional, ya que el Módulo 3 puede utilizar datos de prueba estáticos.

---

### 1.5 Adecuación de la FPGA seleccionada

**Verificación contra el manual de usuario de la Polaris:**

| Requisito del curso | Recurso en SmartFusion2 M2S005 | Verificado |
|---|---|---|
| Procesador ARM Cortex-M3 | Sí, integrado en el MSS | ✅ Sección 3 del manual |
| Matriz FPGA para acelerador HW | 6,060 4-LUTs, 6,060 DFFs | ✅ Sección 2 |
| Bloques DSP para ML | 11 multiplicadores 18×18 | ✅ Sección 2 |
| Memoria para firmware | 64 KB eSRAM + 128 KB eNVM | ✅ Sección 3 |
| SPI para ADXL345 | 2 SPIs en el MSS | ✅ Sección 3 |
| UART para PC | 2 UARTs en el MSS | ✅ Sección 3 |
| Reloj de sistema | 50 MHz (pin K1) | ✅ Sección 7 |
| Programador integrado | FlashPro5 | ✅ Sección 5 |
| Acelerómetro ADXL345 | Integrado en placa, SPI | ✅ Sección 10 |
| Niveles lógicos | 3.3 V en todos los pines | ✅ Todas las tablas |

**Veredicto: La FPGA resulta adecuada.** Los 6,060 LUTs son modestos pero suficientes para un acelerador de características estadísticas simples. Los 11 multiplicadores 18×18 superan los requisitos para el cálculo de varianzas y operaciones de ML liviano en paralelo. Los 64 KB de eSRAM resultan suficientes para el firmware del filtro de Kalman con CMSIS-DSP.

**Nota técnica:** El documento adjunto corresponde al **manual de usuario de la placa Polaris**, no al datasheet completo del SmartFusion2 M2S005 (documento de Microchip de cientos de páginas). El manual de usuario proporciona información suficiente sobre pines, recursos y periféricos para la ejecución de este proyecto. Para detalles de temporización, configuración profunda del MSS o registros específicos del ARM, se requerirá consultar el datasheet del SmartFusion2 y la documentación de Microchip.

---

### 1.6 Compatibilidad de herramientas y periféricos

| Elemento | Compatible | Observaciones |
|---|---|---|
| Libero SoC v12+ | ✅ | Herramienta oficial de Microchip para SmartFusion2 |
| SoftConsole IDE | ✅ | IDE oficial basado en Eclipse para el Cortex-M3 |
| CMSIS-DSP | ✅ | Librería estándar ARM para Cortex-M |
| FlashPro5 | ✅ | Integrado en la placa, no requiere hardware externo |
| ADXL345 | ✅ | Integrado en la placa, pines documentados |
| Analizador lógico 24MHz 8CH | ✅ | 8 canales (CH1-CH8), 24 MHz, con GND y GND_pwr |
| Python (numpy, scipy) | ✅ | Para Módulo 2, estándar |

---

### 1.7 Dependencias técnicas que deben resolverse ANTES del curso

| Dependencia | Crítica | Acción requerida |
|---|---|---|
| Libero SoC instalado y con licencia | **Crítica** | Instalar antes del Día 1. La instalación puede requerir 1-2 horas y la licencia Silver/Free requiere registro en Microchip |
| SoftConsole IDE instalado | **Crítica** | Instalar antes del Día 1 |
| Drivers del FlashPro5 | **Crítica** | Verificar que el SO reconoce el FlashPro5 al conectar la placa (Sección 5 del manual) |
| Python + numpy/scipy/matplotlib | Importante | Instalar antes del Día 1 (para Módulo 2) |
| Analizador lógico 24MHz 8CH | Importante | Disponer del equipo para Módulo 6. Se cuenta con 8 canales (CH1-CH8), frecuencia de muestreo de 24 MHz, y referencias GND y GND_pwr |
| Placas Polaris verificadas | **Crítica** | Probar cada placa antes del curso: que programe, que el UART funcione, que el ADXL345 responda |
| Cable USB Tipo-C | Crítico | Uno por placa (alimentación + programación) |
| Computadoras con puertos USB funcionales | Crítico | Verificar compatibilidad |

---

### 1.8 Riesgos técnicos y conceptos a aclarar

**Riesgos técnicos identificados:**

1. **MSS Configurator:** La configuración del Microcontroller Subsystem del SmartFusion2 en Libero SoC constituye un proceso complejo que involucra la selección de periféricos, asignación de pines, configuración de clocks y generación de drivers. Si este proceso falla o se ejecuta incorrectamente, el ARM no opera. **Recomendación:** Disponer de un proyecto de Libero SoC pre-configurado con el MSS ya configurado (UART + SPI habilitados) como punto de partida o respaldo.

2. **Ruteo de SPI a través de la fabric:** Los pines del ADXL345 (N4, P3, N3, M3) corresponden a pines de la FPGA fabric, no a pines dedicados del MSS. Esto implica que las señales SPI del MSS deben ruteare a través de la fabric hacia dichos pines. Este procedimiento requiere la creación de un wrapper en HDL que conecte las señales del MSS SPI a los pines de salida de la fabric. **Este paso no resulta trivial y no se menciona explícitamente en el curso.**

3. **Integración AMBA del IP personalizado:** El SmartFusion2 utiliza un bus AHB para la comunicación entre el MSS y la fabric. Para que el ARM acceda al acelerador HW, el IP debe implementar una interfaz APB o AHB slave. Este requisito implica comprender el protocolo AMBA, que presenta complejidad. **Recomendación:** Utilizar el componente "AHB Slave" que Libero SoC puede generar automáticamente, o proporcionar un template.

4. **Cuantización de punto fijo:** La transición de Python (punto flotante) a Verilog (punto fijo) constituye una fuente común de errores. Los estudiantes deben comprender con exactitud cómo mapear los valores del acelerómetro (que son enteros de 10 o 13 bits) al formato de punto fijo del acelerador.

**Aclaración técnica sobre el ADXL345:** El manual de la Polaris nombra los pines del ADXL345 como `ADXL_SDA` (N4), `ADXL_SCL` (P3), `ADXL_SDO` (N3), `ADXL_CS` (M3). Los nombres "SDA" y "SCL" corresponden a nomenclatura I²C, lo cual puede generar confusión. Sin embargo, el ADXL345 soporta ambos protocolos (SPI e I²C), y la presencia del pin `ADXL_CS` confirma que la placa lo tiene configurado para SPI. En modo SPI:
- `ADXL_CS` (M3) → Chip Select (activo bajo)
- `ADXL_SCL` (P3) → SCLK (reloj SPI)
- `ADXL_SDA` (N4) → MOSI/SDI (datos hacia el ADXL)
- `ADXL_SDO` (N3) → MISO/SDO (datos desde el ADXL)

**Esta distinción debe aclararse explícitamente a los estudiantes para evitar confusión.**

---

### 1.9 Recomendaciones de simplificación y reorganización

| Cambio propuesto | Razón | Impacto |
|---|---|---|
| **Proporcionar proyecto Libero SoC pre-configurado con MSS** (UART habilitado, SPI habilitado, clocks configurados) | Ahorra 1-2 horas de configuración propensa a errores | Permite que el Módulo 1 se enfoque en conceptos y primeras pruebas |
| **Proporcionar template de IP con interfaz AMBA APB** para el Módulo 3 | El diseño de interfaz AMBA desde cero resulta inviable en 4 h | Los estudiantes se enfocan en el datapath (DSP blocks) |
| **Disponer de diseño de referencia completo** como respaldo para Módulo 6 | Si la integración falla, los estudiantes no quedan sin material de análisis | Garantiza que el Módulo 6 resulte productivo |
| **Mover la configuración del MSS SPI al Módulo 1** | El SPI se necesita en Módulo 4, pero configurarlo requiere tiempo | El Módulo 4 se enfoca únicamente en el driver en C |
| **Simplificar "perfiles de energía" a discusión teórica** | La medición de energía real requiere instrumentación no especificada | Evita frustración en Módulo 6 |
| **Definir entregable explícito del Módulo 2** como especificación del acelerador HW | Conecta conceptualmente el ML en Python con el diseño en Verilog | Mejora la coherencia entre Módulos 2 y 3 |

---

### 1.10 Visión de conjunto: construcción progresiva y herencia entre módulos

| Día | Módulo | Se construye | Se hereda del anterior | Entregable clave |
|---|---|---|---|---|
| 1 | Arquitectura SoC | Proyecto Libero SoC, MSS configurado, LEDs parpadeando, UART funcional | Nada (día 1) | Proyecto Libero SoC funcional + proyecto SoftConsole con UART |
| 2 | Online ML | Simulación Python del algoritmo, análisis de cuantización | Comprensión de la arquitectura | Script Python + documento de especificación del acelerador HW |
| 3 | Aceleración HW | IP en Verilog con DSP blocks, integrado en AMBA | Proyecto Libero SoC del Día 1 + especificación del Día 2 | IP sintetizable integrado en el bus del sistema |
| 4 | Interfaz ADXL345 | Driver SPI en C, adquisición de datos en tiempo real | Proyecto Libero SoC con MSS SPI + proyecto SoftConsole | Driver funcional que lee los 3 ejes y envía por UART |
| 5 | Filtro Kalman | Implementación en C con CMSIS-DSP, parámetros adaptativos | Driver SPI del Día 4 + IP del Día 3 | Filtro de Kalman funcional con parámetros Q, R adaptables |
| 6 | Integración | Sistema completo, benchmarking, visualización | Todo lo anterior | Sistema funcional en la placa Polaris + informe de rendimiento |

**Puntos críticos que podrían impedir completar el proyecto:**
1. Falla en la instalación de Libero SoC/SoftConsole → bloquea todo el programa
2. Error en la configuración del MSS → bloquea Módulos 4 y 5
3. IP del Módulo 3 no sintetiza o no se integra correctamente → bloquea Módulo 6
4. Driver SPI no funciona → bloquea Módulos 5 y 6

---
---

## PARTE 2: MÓDULO 1 – PLAN DE EJECUCIÓN DETALLADO

### 2.1 Preparación PREVIA al Día 1 (actividades del instructor antes del curso)

#### 2.1.1 Instalación de software

**Libero SoC v12 o superior:**
1. Acceder a [microchip.com/libero](https://www.microchip.com/en-us/products/fpgas-and-plds/fpga-and-soc-design-tools/fpga/libero-software-later-versions) y descargar Libero SoC v12.x para el sistema operativo correspondiente (Windows o Linux).
2. Registrarse en Microchip y obtener una **licencia Silver (gratuita)**. La licencia Silver permite utilizar dispositivos SmartFusion2 hasta el M2S010, que incluye el M2S005 de la Polaris.
3. Instalar Libero SoC. La instalación puede requerir entre 30 y 60 minutos y ocupa varios GB.
4. Configurar la licencia: abrir Libero SoC → `Help → License Setup` → indicar el archivo de licencia o el servidor de licencias.
5. **Verificación:** Abrir Libero SoC y confirmar la ausencia de errores de licencia. Crear un proyecto de prueba y cerrarlo.

**SoftConsole IDE:**
1. Descargar SoftConsole desde la misma página de Microchip. SoftConsole corresponde a un IDE basado en Eclipse para el desarrollo de firmware para el ARM Cortex-M3 del SmartFusion2.
2. Instalar SoftConsole.
3. **Verificación:** Abrir SoftConsole y confirmar que arranca sin errores.

**Python (para Módulo 2, pero se instala en esta etapa):**
1. Instalar Python 3.8+.
2. Instalar paquetes: `pip install numpy scipy matplotlib scikit-learn`.
3. **Verificación:** Ejecutar `python -c "import numpy; print(numpy.__version__)"`.

#### 2.1.2 Verificación de hardware

Para **cada placa Polaris** que se utilizará en el curso:

1. Conectar la placa al PC mediante cable USB Tipo-C.
2. Verificar que el sistema operativo detecta el FlashPro5:
   - **Windows:** Abrir el Administrador de Dispositivos → buscar "FlashPro5" o "Microchip" en la sección de dispositivos USB. Deben aparecer 4 puertos COM virtuales (Sección 6 del manual).
   - **Linux:** Ejecutar `lsusb` y buscar Microchip. Ejecutar `dmesg | tail` para observar los puertos ttyUSB creados.
3. En caso de no detectarse, instalar los drivers del FlashPro5 (incluidos con Libero SoC).
4. Abrir Libero SoC → `Tools → FlashPro5` o `Configure Programming` → verificar que el programador aparece listado.
5. Anotar los números de los puertos COM asignados (se requieren para UART). Según el manual (Sección 6), **solo los tres últimos puertos COM se encuentran habilitados para UART**; el primero es reservado para funciones internas del FlashPro5.

#### 2.1.3 Preparación de materiales para estudiantes

Se debe crear una carpeta `Modulo1_Material/` que contenga:
- Una guía paso a paso impresa o en PDF con capturas de pantalla.
- Un proyecto de Libero SoC pre-configurado (opcional pero recomendado como respaldo).
- Un archivo de texto con los números de puerto COM de cada placa.

#### 2.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 1 personalmente antes del curso**, en la misma placa Polaris que utilizarán los estudiantes. Esto incluye:
- Crear el proyecto desde cero en Libero SoC.
- Configurar el MSS.
- Crear el diseño HDL de parpadeo de LEDs.
- Sintetizar, Place & Route, programar.
- Crear el proyecto en SoftConsole.
- Escribir y ejecutar el código C de UART.
- Verificar la comunicación con la PC.

---

### 2.2 Conceptos teóricos que el instructor debe dominar y explicar

#### 2.2.1 Arquitectura SmartFusion2 (30 min de teoría)

Se explica con un diagrama de bloques:

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

**Conceptos clave a explicar:**
- **MSS vs. Fabric:** El MSS corresponde al "microcontrolador" (ARM + periféricos). La Fabric corresponde a la "FPGA" (lógica programable). Son dos mundos que coexisten en el mismo chip.
- **Buses AMBA:** El MSS y la Fabric se comunican mediante buses AMBA (AHB para datos de alto rendimiento, APB para periféricos de baja velocidad). Cuando el ARM requiere leer un registro del acelerador HW en la FPGA, lo realiza a través de estos buses.
- **Justificación del SmartFusion2:** A diferencia de una FPGA externa + microcontrolador externo, aquí la totalidad se encuentra en un chip. Menor latencia, menor consumo, menor área en PCB.

#### 2.2.2 Entorno de desarrollo (15 min)

- **Libero SoC:** Herramienta de diseño de hardware. Aquí se crea la lógica de la FPGA, se configura el MSS, se sintetiza y se genera el bitstream.
- **SoftConsole:** IDE para escribir, compilar y depurar código C para el ARM Cortex-M3.
- **FlashPro5:** Programador integrado en la placa. Programa tanto la FPGA como el ARM.

#### 2.2.3 Flujo de desarrollo (15 min)

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

### 2.3 Herramientas que deben estar instaladas y configuradas

| Herramienta | Versión | Verificación |
|---|---|---|
| Libero SoC | v12.x | Abrir, verificar licencia |
| SoftConsole | Última versión | Abrir, verificar que compila |
| Drivers FlashPro5 | Incluídos con Libero | Conectar placa, verificar puertos COM |
| Terminal serial (PuTTY, Tera Term, minicom) | Cualquiera | Verificar que abre puerto COM |

---

### 2.4 Implementación paso a paso

#### PASO 1: Crear el proyecto en Libero SoC (30 min)

1. **Abrir Libero SoC.**
2. Ir a `File → New Project`.
3. En el asistente:
   - **Project Name:** `Polaris_Lab1`
   - **Project Location:** Seleccionar un directorio sin espacios en la ruta (ej: `C:\Polaris_Lab1` o `/home/user/Polaris_Lab1`).
   - **Project Type:** HDL Design.
   - **Device Family:** SmartFusion2.
   - **Device:** M2S005.
   - **Package:** TQ144 (verificar que coincida con la placa Polaris; el manual no especifica el paquete exacto, pero el M2S005 en la Polaris utiliza un paquete específico que se puede verificar en la serigrafía del chip).
   - **Speed Grade:** -1 (o el que corresponda).
   - **HDL Language:** Seleccionar Verilog o VHDL (se recomienda Verilog para este curso, ya que el Módulo 3 utiliza Verilog).
4. Presionar **Finish**.

**Resultado esperado:** Se abre el proyecto con el panel "Design Flow" visible a la izquierda.

**Error común:** Si la licencia no se encuentra configurada, Libero mostrará un error al intentar crear el proyecto. Solución: `Help → License Setup`.

---

#### PASO 2: Configurar el MSS (Microcontroller Subsystem) (45 min)

Este corresponde al paso **más crítico** del Módulo 1.

1. En el panel Design Flow, hacer doble clic en **"Create MSS Component"** (o ir a `Tools → MSS Configurator`).
2. Se abre el MSS Configurator. Aquí se configura el ARM Cortex-M3 y sus periféricos.

**Configuración del MSS:**

a) **System Clock:**
   - El reloj de entrada corresponde a 50 MHz (pin K1 de la placa, Sección 7 del manual).
   - Configurar el MSS para utilizar el oscilador externo de 50 MHz como fuente de reloj.
   - El clock del ARM Cortex-M3 puede ser 50 MHz directamente o dividido. Para este curso, se utiliza 50 MHz.

b) **Habilitar UART_0:**
   - Ir a la pestaña de periféricos → UART.
   - Habilitar **MMUART_0**.
   - Configurar baud rate: **115200** (estándar).
   - **Importante:** Las señales TX y RX del UART_0 deben ruteare a pines de la FPGA fabric para que salgan por el FlashPro5. Según la Tabla 1 del manual:
     - RX0 → Pin FPGA V11
     - TX0 → Pin FPGA W11
   - En el MSS Configurator, seleccionar que las señales de UART_0 se conecten a la **FPGA Fabric** (no a pines dedicados del MSS, ya que los pines del FlashPro5 se encuentran en la fabric).

c) **Habilitar GPIO:**
   - Habilitar algunos GPIOs para controlar LEDs desde el ARM si se desea (opcional en este módulo).

d) **Memoria:**
   - Verificar que eSRAM (64 KB) y eNVM (128 KB) se encuentran habilitados.
   - El código C se ejecutará desde eSRAM (más rápido) o eNVM (no volátil). Para desarrollo, se utiliza eSRAM.

3. **Generar el MSS:**
   - Hacer clic en **"Generate MSS Component"**.
   - Esto crea un componente MSS en el proyecto de Libero SoC que contiene el ARM, los periféricos configurados y los drivers HAL (Hardware Abstraction Layer).
   - Se genera una carpeta con archivos de drivers en C que se utilizarán en SoftConsole.

**Resultado esperado:** Un componente MSS aparece en el diseño de Libero SoC, con puertos visibles para UART (TX, RX), clocks, resets y la interfaz AHB/APB hacia la fabric.

**Errores comunes:**
- Si no se selecciona el dispositivo correcto (M2S005), el MSS Configurator no mostrará las opciones correctas.
- Si la ruta del proyecto contiene espacios, la generación puede fallar.
- Si la licencia no es Silver o superior, algunas funciones pueden estar bloqueadas.

**Archivos generados que deben conservarse:**
- El componente MSS (`.cxz` o similar).
- Los archivos de drivers HAL (carpeta `mss_hal/` o similar): `mss_uart.h`, `mss_gpio.h`, etc.
- El archivo de configuración del MSS (`.xml` o `.mss`).

---

#### PASO 3: Crear el diseño HDL – Parpadeo de LEDs (45 min)

Se crea un módulo simple en Verilog que haga parpadear los LEDs de la placa.

**Archivo a crear:** `led_blink.v`

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

**Archivo top-level:** `top.v`

```verilog
module top (
    input  wire       CLK_50MHZ,   // Pin K1
    input  wire       SW0,         // Pin D6 (reset)
    output wire [9:0] LED          // LEDs LED0-LED9
);

    led_blink u_led_blink (
        .clk_50mhz (CLK_50MHZ),
        .reset_n   (SW0),
        .leds      (LED)
    );

endmodule
```

**Agregar los archivos al proyecto:**
1. En Libero SoC, `Project → Add Files` → seleccionar `top.v` y `led_blink.v`.
2. En el Design Flow, hacer clic derecho en `top` → **"Set As Root"**.

**Build Hierarchy:**
1. En el Design Flow, hacer doble clic en **"Build Hierarchy"**.
2. Verificar que no existen errores.

---

#### PASO 4: Asignación de pines (30 min)

Se utiliza la **Tabla 4** (LEDs), **Tabla 3** (Switches) y **Tabla 2** (Reloj) del manual de la Polaris:

| Señal en top.v | Pin FPGA | Nombre en placa |
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

**Procedimiento:**
1. En Libero SoC, ir a `Design → I/O Editor`.
2. Asignar cada señal al pin correspondiente.
3. Configurar el nivel lógico de todos los pines como **LVCMOS33** (3.3 V, como indica el manual en todas las tablas).
4. Guardar.

**Alternativa:** Crear un archivo de constraints (`.pdc` o `.sdc`) manualmente:

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

#### PASO 5: Síntesis y Place & Route (20 min)

1. En el Design Flow, hacer doble clic en **"Synthesis"**.
2. Esperar a que finalice. Verificar que no existen errores (warnings son normales).
3. Hacer doble clic en **"Place and Route"**.
4. Esperar a que finalice. Verificar que no existen errores.

**Errores comunes:**
- Error de pin assignment: algún pin no fue asignado o se asignó a un pin inválido.
- Error de timing: improbable a 1 Hz, pero posible si existen problemas con el clock constraint. Agregar un constraint de reloj:
  ```tcl
  # Archivo: timing.sdc
  create_clock -name clk_50mhz -period 20.0 [get_ports CLK_50MHZ]
  ```

---

#### PASO 6: Programar la FPGA (15 min)

1. Conectar la placa Polaris al PC por USB Tipo-C.
2. En Libero SoC, ir a `Tools → Program Device` o hacer doble clic en **"Program Device"** en el Design Flow.
3. Seleccionar el programador **FlashPro5**.
4. Hacer clic en **"Program"**.
5. Esperar a que finalice (unos segundos).

**Resultado esperado:** Los LEDs de la placa deben mostrar un patrón de rotación circular (un LED encendido que se desplaza de LED0 a LED9 y vuelve a empezar). El switch SW0 en posición inferior (OFF = 0 V) mantiene el reset; al moverlo a la posición superior (ON = 3.3 V), el patrón comienza.

**Si no funciona:**
- Verificar que el FlashPro5 fue detectado (Sección 5 del manual).
- Verificar que la placa se encuentra alimentada (LED de power encendido).
- Verificar que SW0 se encuentra en posición ON (superior).
- Revisar los mensajes de error en la ventana de programación.
- Verificar que los pines fueron asignados correctamente en el I/O Editor.

---

#### PASO 7: Crear proyecto en SoftConsole y comunicación UART (60 min)

1. **Abrir SoftConsole.**
2. `File → New → SoftConsole Project`.
3. Seleccionar el tipo de proyecto: **"Microchip SmartFusion2 MSS"** (o "Empty Project" si no aparece la opción).
4. Configurar:
   - **Target:** SmartFusion2 M2S005.
   - **Toolchain:** GCC for ARM (incluido con SoftConsole).
5. **Importar los drivers HAL del MSS:**
   - Copiar la carpeta de drivers generada por el MSS Configurator (paso 2) al proyecto de SoftConsole.
   - Agregar las rutas de include en las propiedades del proyecto (`Project → Properties → C/C++ Build → Settings → Include Paths`).

**Archivo a crear:** `main.c`

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

**Nota:** Los nombres exactos de las funciones del HAL pueden variar según la versión del MSS Configurator. El instructor debe verificar los nombres reales en los archivos `mss_uart.h` generados. Las funciones típicas son:
- `MSS_UART_init()`
- `MSS_UART_polled_tx()` o `MSS_UART_polled_tx_string()`
- `MSS_UART_get_rx()` o `MSS_UART_polled_rx()`

6. **Compilar:** `Project → Build All`. Verificar que no existen errores de compilación.

7. **Programar el ARM:**
   - Conectar la placa.
   - En SoftConsole, ir a `Run → Debug Configurations`.
   - Crear una nueva configuración de depuración para SmartFusion2.
   - Seleccionar el programador FlashPro5.
   - Hacer clic en **"Debug"** o **"Run"**.

8. **Verificar UART en la PC:**
   - Abrir un terminal serial (PuTTY, Tera Term, minicom, screen).
   - Conectar al puerto COM correspondiente al UART_0 (recordar: según la Sección 6 del manual, el FlashPro5 crea 4 puertos COM; los 3 últimos son UART; identificar cuál corresponde a UART_0 probando cada uno).
   - Configurar: **115200 baud, 8 bits, sin paridad, 1 stop bit, sin control de flujo**.
   - **Resultado esperado:** Observar el mensaje `[Modulo 1] UART funcional - IEEE CASS UMSA 2026` en el terminal.
   - **Prueba de eco:** Escribir caracteres en el terminal y verificar que se reciben de vuelta (eco).

**Si no funciona:**
- Verificar el puerto COM correcto (probar los 3 últimos).
- Verificar que el baud rate coincide (115200).
- Verificar que las señales TX/RX del MSS se encuentran correctamente ruteadas a los pines V11/W11 (a través de la fabric o pines dedicados).
- Verificar que el MSS fue configurado correctamente en el MSS Configurator.
- Revisar si el código C compila sin errores.
- Verificar que el ARM fue programado correctamente (SoftConsole debería mostrar "Download complete" o similar).

---

### 2.5 Verificación final del Módulo 1

Al finalizar el Módulo 1, se debe contar con:

| Criterio | Verificación |
|---|---|
| LEDs parpadean con patrón de rotación | ✅ Visual |
| SW0 funciona como reset | ✅ Mover switch, LEDs se reinician |
| UART envía mensaje al PC | ✅ Visible en terminal serial |
| Eco UART funciona | ✅ Escribir en terminal, ver eco |
| Proyecto Libero SoC sintetiza sin errores | ✅ Verificar en Design Flow |
| Proyecto SoftConsole compila sin errores | ✅ Verificar en consola de build |
| FlashPro5 programa FPGA y ARM | ✅ Sin errores de programación |

---

### 2.6 Entregables del Módulo 1 (conservar para Módulos siguientes)

| Entregable | Uso futuro |
|---|---|
| Proyecto Libero SoC (`Polaris_Lab1/`) | Base para Módulos 3, 4, 6 |
| Configuración del MSS (con UART habilitado) | Se utilizará en Módulos 4 y 5 |
| Archivos HDL (`top.v`, `led_blink.v`) | Se reemplazarán/aumentarán en Módulo 3 |
| Archivo de constraints de pines (`pins.pdc`) | Se ampliará en Módulos 3 y 4 |
| Proyecto SoftConsole con drivers HAL | Base para Módulos 4 y 5 |
| Código `main.c` con UART funcional | Se ampliará en Módulos 4 y 5 |
| Conocimiento de los puertos COM | Necesario para todos los módulos |

---

### 2.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| Libero no abre o muestra error de licencia | Licencia no configurada | `Help → License Setup`, verificar archivo .dat |
| MSS Configurator no genera | Dispositivo incorrecto o ruta con espacios | Verificar M2S005, cambiar ruta del proyecto |
| Síntesis falla | Error de sintaxis en Verilog | Revisar código, verificar que `top` es el root |
| Place & Route falla | Pines no asignados o conflicto de pines | Revisar I/O Editor, verificar tabla de pines |
| FlashPro5 no detectado | Drivers no instalados o cable USB | Reinstalar drivers, probar otro cable/puerto USB |
| Programación falla | Placa no alimentada o FlashPro5 ocupado | Verificar alimentación USB, cerrar otros programas |
| LEDs no encienden | SW0 en posición OFF (reset activo) | Mover SW0 a posición ON (superior) |
| UART no muestra nada | Puerto COM incorrecto o baud rate mal | Probar los 3 últimos puertos COM, verificar 115200 |
| UART muestra caracteres basura | Baud rate incorrecto o clock mal configurado | Verificar 115200 en ambos lados, verificar clock del MSS |
| SoftConsole no compila | Faltan drivers HAL o include paths | Copiar carpeta de drivers del MSS, agregar include paths |

---

### 2.8 Resumen del Día 1

**Antes del Día 1:** Libero SoC, SoftConsole y drivers instalados. Placas verificadas.

**Durante el Día 1 (4 horas):**
- 0:00–0:30: Teoría de arquitectura SmartFusion2
- 0:30–1:00: Crear proyecto en Libero SoC
- 1:00–1:45: Configurar MSS (UART habilitado)
- 1:45–2:30: Escribir Verilog (LED blink), asignar pines
- 2:30–3:00: Síntesis, P&R, programar FPGA → **Hito: LEDs parpadean**
- 3:00–3:45: Crear proyecto SoftConsole, escribir main.c con UART
- 3:45–4:00: Compilar, programar ARM, verificar UART → **Hito: Comunicación serial funcional**

**Al final del Día 1:** Proyecto Libero SoC con MSS configurado + diseño HDL funcional + proyecto SoftConsole con UART operativo. Todo probado en la placa Polaris.
