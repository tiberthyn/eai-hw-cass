# Evaluación Integral de Viabilidad y Plan de Ejecución del Programa IEEE CASS UMSA 2026

---

## EVALUACIÓN GENERAL DE VIABILIDAD

### Análisis de coherencia técnica y pedagógica del proyecto

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

### Análisis de factibilidad en seis días de cuatro horas (24 h totales)

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

### Coherencia entre objetivos y tiempo disponible por módulo

Los objetivos resultan mayormente coherentes, con dos excepciones de relevancia:

**Módulo 3 – Condición técnica identificada:** El objetivo establece *"Crear un periférico IP personalizado para acelerar el cálculo de características del ML en paralelo"* e *"Integrar el IP en el bus del sistema AMBA"*. El diseño de un IP con interfaz AMBA APB desde cero, que utilice bloques DSP, que se sintetice correctamente y que se integre en el MSS del SmartFusion2 constituye un trabajo que normalmente requiere días, no horas. Un estudiante con "nociones básicas de VHDL/Verilog" no puede completar esta tarea en 4 horas.

> **Propuesta justificada:** Proporcionar un **template de IP con la interfaz AMBA APB ya implementada** (wrapper de registros, decodificación de direcciones, señales de handshaking). Los estudiantes únicamente deberán modificar el **datapath interno** (la lógica que utiliza los bloques DSP para el cálculo de la varianza). Esta modificación reduce la complejidad en un 70% y mantiene el valor pedagógico asociado a la comprensión de la integración de un acelerador en el bus del sistema.

**Módulo 6 – Condición técnica identificada:** El objetivo establece *"Integrar todos los subsistemas... ejecutar el flujo de datos completo... analizar latencia, ocupación de área y perfiles de energía... comparar cuantitativamente con analizador lógico"*. Si algún módulo anterior presenta problemas (situación probable), este módulo se convierte en una sesión de depuración masiva.

> **Propuesta justificada:** Disponer de un **diseño de referencia pre-integrado y funcional** como respaldo. En caso de que los estudiantes no logren integrar la totalidad del sistema, se puede cargar el diseño de referencia y enfocar la sesión en el benchmarking y análisis. Adicionalmente, se propone simplificar los "perfiles de energía" a una discusión teórica, dado que la medición de consumo real requiere instrumentación específica no contemplada en los recursos disponibles.

---

### Secuencia pedagógica y preparación progresiva del estudiante

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

### Adecuación de la FPGA seleccionada

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

### Compatibilidad de herramientas y periféricos

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

### Dependencias técnicas que deben resolverse ANTES del curso

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

### Riesgos técnicos y conceptos a aclarar

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

### Recomendaciones de simplificación y reorganización

| Cambio propuesto | Razón | Impacto |
|---|---|---|
| **Proporcionar proyecto Libero SoC pre-configurado con MSS** (UART habilitado, SPI habilitado, clocks configurados) | Ahorra 1-2 horas de configuración propensa a errores | Permite que el Módulo 1 se enfoque en conceptos y primeras pruebas |
| **Proporcionar template de IP con interfaz AMBA APB** para el Módulo 3 | El diseño de interfaz AMBA desde cero resulta inviable en 4 h | Los estudiantes se enfocan en el datapath (DSP blocks) |
| **Disponer de diseño de referencia completo** como respaldo para Módulo 6 | Si la integración falla, los estudiantes no quedan sin material de análisis | Garantiza que el Módulo 6 resulte productivo |
| **Mover la configuración del MSS SPI al Módulo 1** | El SPI se necesita en Módulo 4, pero configurarlo requiere tiempo | El Módulo 4 se enfoca únicamente en el driver en C |
| **Simplificar "perfiles de energía" a discusión teórica** | La medición de energía real requiere instrumentación no especificada | Evita frustración en Módulo 6 |
| **Definir entregable explícito del Módulo 2** como especificación del acelerador HW | Conecta conceptualmente el ML en Python con el diseño en Verilog | Mejora la coherencia entre Módulos 2 y 3 |

---

### Visión de conjunto: construcción progresiva y herencia entre módulos

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