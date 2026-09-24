# MÓDULO 3: ACELERACIÓN POR HARDWARE EN FPGA

## 3.1 Preparación PREVIA al Día 3 (actividades del instructor antes del módulo)

### 3.1.1 Verificación del entorno Libero SoC

El Módulo 3 requiere que el entorno de Libero SoC se encuentre plenamente operativo, con el proyecto del Módulo 1 como base. Se debe verificar lo siguiente:

1. **Proyecto del Módulo 1 disponible:** El proyecto `Polaris_Lab1` creado en el Día 1 debe estar accesible y funcional. Este proyecto contiene la configuración del MSS con el UART habilitado, la cual se reutilizará y ampliará.
2. **Documento de especificación del Módulo 2:** El archivo `especificacion_acelerador_hw.md` generado en el Día 2 debe estar disponible como referencia técnica para el diseño del IP.
3. **Script Python de referencia:** El archivo `ml_online_sim.py` del Módulo 2 debe estar accesible para generar los vectores de prueba que se utilizarán en el testbench.
4. **Licencia de Libero SoC activa:** Verificar que la licencia Silver permite la síntesis y Place & Route para el dispositivo M2S005.

### 3.1.2 Generación de datos de prueba desde Python

Antes del Día 3, el instructor debe ejecutar el script del Módulo 2 para generar los vectores de prueba cuantizados que se utilizarán en el testbench del acelerador.

**Comando a ejecutar:**

```bash
python ml_online_sim.py --export-testvectors test_vectors.txt
```

Si el script no incluye esta opción, se debe agregar el siguiente fragmento al final del script `ml_online_sim.py`:

```python
# =====================================================================
# EXPORTACIÓN DE VECTORES DE PRUEBA PARA TESTBENCH
# =====================================================================
import sys

if '--export-testvectors' in sys.argv:
    filename = sys.argv[sys.argv.index('--export-testvectors') + 1]
    with open(filename, 'w') as f:
        f.write("// Vectores de prueba generados desde Python\n")
        f.write("// Formato: muestra_ax muestra_ay muestra_az (10 bits signed)\n")
        f.write("// Ventana de 50 muestras\n")
        for i in range(50):
            idx = i * 10  # Tomar muestras espaciadas
            if idx < len(X_raw):
                ax = X_raw[idx, 0]
                ay = X_raw[idx, 1]
                az = X_raw[idx, 2]
                f.write(f"{ax:010b} {ay:010b} {az:010b}\n")
        
        f.write("\n// Características esperadas (Q3.12, 16 bits signed)\n")
        for i in range(6):
            feat_q = QP_FEAT.quantize(X_features[0, i])
            f.write(f"feat[{i}] = {feat_q:016b} ({feat_q})\n")
        
        f.write("\n// Pesos del modelo (Q2.13, 16 bits signed)\n")
        for c in range(3):
            for i in range(6):
                w_q = QP_WEIGHT.quantize(model.weights[c, i])
                f.write(f"w[{c}][{i}] = {w_q:016b} ({w_q})\n")
    
    print(f"Vectores de prueba exportados a {filename}")
```

**Resultado esperado:** Archivo `test_vectors.txt` con 50 muestras del ADXL345 cuantizadas a 10 bits, las 6 características esperadas en formato Q3.12, y los 18 pesos del modelo en formato Q2.13.

### 3.1.3 Preparación del template de IP AMBA APB

Dado que el diseño de una interfaz AMBA APB desde cero resulta inviable en el tiempo disponible (4 horas), el instructor debe preparar un template de IP con la interfaz ya implementada. Los estudiantes únicamente deberán modificar el datapath interno.

**Archivo a crear:** `amba_apb_slave_template.v`

```verilog
//======================================================================
// Template de IP con Interfaz AMBA APB Slave
// SmartFusion2 M2S005 - IEEE CASS UMSA 2026
//======================================================================
// Este módulo implementa la interfaz AMBA APB slave genérica.
// El datapath interno debe ser completado por el estudiante.
//======================================================================

module amba_apb_slave_template #(
    parameter DATA_WIDTH = 32,
    parameter ADDR_WIDTH = 12
)(
    // Señales AMBA APB
    input  wire                    PCLK,       // Reloj APB (50 MHz)
    input  wire                    PRESETn,    // Reset activo bajo
    input  wire [ADDR_WIDTH-1:0]   PADDR,      // Dirección
    input  wire                    PSEL,       // Select
    input  wire                    PENABLE,    // Enable
    input  wire                    PWRITE,     // Write
    input  wire [DATA_WIDTH-1:0]   PWDATA,     // Write data
    output reg  [DATA_WIDTH-1:0]   PRDATA,     // Read data
    output reg                     PREADY,     // Ready (sin wait states)
    
    // Señales de interrupción (opcional)
    output wire                    IRQ
);

    // ====================================================================
    // MAPA DE REGISTROS (según especificación del Módulo 2)
    // ====================================================================
    // 0x00: CONTROL   (RW) - [0] start, [1] reset, [2] mode
    // 0x04: STATUS    (RO) - [0] busy, [1] done, [2] error
    // 0x08: DATA_IN   (WO) - Muestra del ADXL345 (10 bits signed × 3 ejes)
    // 0x0C: FEAT_0    (RO) - Característica 0 (Q3.12)
    // 0x10: FEAT_1    (RO) - Característica 1 (Q3.12)
    // 0x14: FEAT_2    (RO) - Característica 2 (Q3.12)
    // 0x18: FEAT_3    (RO) - Característica 3 (Q3.12)
    // 0x1C: FEAT_4    (RO) - Característica 4 (Q3.12)
    // 0x20: FEAT_5    (RO) - Característica 5 (Q3.12)
    // 0x24: WEIGHT_0  (RW) - Peso [0][0] (Q2.13)
    // 0x28: WEIGHT_1  (RW) - Peso [0][1] (Q2.13)
    // ... (hasta WEIGHT_17 en 0x68)
    // 0x6C: SCORE     (RO) - Score de salida (Q3.14, 18 bits)
    // ====================================================================

    // Registros internos
    reg [31:0] reg_control;
    reg [31:0] reg_status;
    reg [31:0] reg_data_in;
    reg [31:0] reg_feat [0:5];
    reg [31:0] reg_weight [0:17];
    reg [31:0] reg_score;

    // Señales de control del datapath
    wire start_pulse;
    wire reset_pulse;
    wire busy;
    wire done;
    
    assign start_pulse = reg_control[0];
    assign reset_pulse = reg_control[1];
    assign reg_status[0] = busy;
    assign reg_status[1] = done;
    assign IRQ = done;

    // ====================================================================
    // LÓGICA DE ESCRITURA APB
    // ====================================================================
    always @(posedge PCLK or negedge PRESETn) begin
        if (!PRESETn) begin
            reg_control <= 32'd0;
            reg_data_in <= 32'd0;
            // Los pesos se inicializan desde el testbench o desde el ARM
        end else if (PSEL && PENABLE && PWRITE) begin
            case (PADDR[7:2])
                6'h00: reg_control <= PWDATA;
                6'h02: reg_data_in <= PWDATA;
                6'h09: reg_weight[0]  <= PWDATA;
                6'h0A: reg_weight[1]  <= PWDATA;
                6'h0B: reg_weight[2]  <= PWDATA;
                6'h0C: reg_weight[3]  <= PWDATA;
                6'h0D: reg_weight[4]  <= PWDATA;
                6'h0E: reg_weight[5]  <= PWDATA;
                6'h0F: reg_weight[6]  <= PWDATA;
                6'h10: reg_weight[7]  <= PWDATA;
                6'h11: reg_weight[8]  <= PWDATA;
                6'h12: reg_weight[9]  <= PWDATA;
                6'h13: reg_weight[10] <= PWDATA;
                6'h14: reg_weight[11] <= PWDATA;
                6'h15: reg_weight[12] <= PWDATA;
                6'h16: reg_weight[13] <= PWDATA;
                6'h17: reg_weight[14] <= PWDATA;
                6'h18: reg_weight[15] <= PWDATA;
                6'h19: reg_weight[16] <= PWDATA;
                6'h1A: reg_weight[17] <= PWDATA;
                default: ;
            endcase
        end
    end

    // ====================================================================
    // LÓGICA DE LECTURA APB
    // ====================================================================
    always @(posedge PCLK) begin
        if (PSEL && !PENABLE) begin  // Setup phase
            case (PADDR[7:2])
                6'h00: PRDATA <= reg_control;
                6'h01: PRDATA <= reg_status;
                6'h03: PRDATA <= reg_feat[0];
                6'h04: PRDATA <= reg_feat[1];
                6'h05: PRDATA <= reg_feat[2];
                6'h06: PRDATA <= reg_feat[3];
                6'h07: PRDATA <= reg_feat[4];
                6'h08: PRDATA <= reg_feat[5];
                6'h09: PRDATA <= reg_weight[0];
                6'h0A: PRDATA <= reg_weight[1];
                6'h0B: PRDATA <= reg_weight[2];
                6'h0C: PRDATA <= reg_weight[3];
                6'h0D: PRDATA <= reg_weight[4];
                6'h0E: PRDATA <= reg_weight[5];
                6'h0F: PRDATA <= reg_weight[6];
                6'h10: PRDATA <= reg_weight[7];
                6'h11: PRDATA <= reg_weight[8];
                6'h12: PRDATA <= reg_weight[9];
                6'h13: PRDATA <= reg_weight[10];
                6'h14: PRDATA <= reg_weight[11];
                6'h15: PRDATA <= reg_weight[12];
                6'h16: PRDATA <= reg_weight[13];
                6'h17: PRDATA <= reg_weight[14];
                6'h18: PRDATA <= reg_weight[15];
                6'h19: PRDATA <= reg_weight[16];
                6'h1A: PRDATA <= reg_weight[17];
                6'h1B: PRDATA <= reg_score;
                default: PRDATA <= 32'd0;
            endcase
        end
    end

    // PREADY siempre alto (sin wait states)
    always @(posedge PCLK or negedge PRESETn) begin
        if (!PRESETn)
            PREADY <= 1'b1;
        else
            PREADY <= 1'b1;
    end

    // ====================================================================
    // INSTANCIACIÓN DEL DATAPATH (a completar por el estudiante)
    // ====================================================================
    ml_accelerator_datapath u_datapath (
        .clk        (PCLK),
        .reset_n    (PRESETn & ~reset_pulse),
        .start      (start_pulse),
        .data_in    (reg_data_in[29:0]),   // 3 ejes × 10 bits
        .weights    (reg_weight),
        .busy       (busy),
        .done       (done),
        .features   (reg_feat),
        .score      (reg_score)
    );

endmodule
```

Este template debe proporcionarse a los estudiantes al inicio del módulo, junto con una guía que indique claramente qué partes del diseño deben completar.

### 3.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 3 personalmente antes del curso**, verificando:
- La síntesis del datapath en Verilog sin errores.
- La correcta inferencia de los bloques DSP 18×18 (verificar en el reporte de síntesis).
- La simulación del testbench con los vectores de prueba.
- La integración del IP en el MSS Configurator.
- La programación del dispositivo y la comunicación del ARM con el IP a través del bus APB.

---

## 3.2 Conceptos teóricos que el instructor debe dominar y explicar

### 3.2.1 Arquitectura del acelerador de hardware (20 min)

Se presenta el diagrama de bloques del acelerador:

```
┌─────────────────────────────────────────────────────────────────┐
│                ACELERADOR ML EN FPGA                            │
│                (SmartFusion2 M2S005)                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              INTERFAZ AMBA APB SLAVE                      │  │
│  │  PADDR, PSEL, PENABLE, PWRITE, PWDATA, PRDATA, PREADY   │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              REGISTROS INTERNOS                           │  │
│  │  • CONTROL (start, reset, mode)                          │  │
│  │  • STATUS (busy, done, error)                            │  │
│  │  • DATA_IN (muestra ADXL345: 3×10 bits)                  │  │
│  │  • FEATURES (6×16 bits Q3.12)                            │  │
│  │  • WEIGHTS (18×16 bits Q2.13)                            │  │
│  │  • SCORE (18 bits Q3.14)                                 │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                     │
│              ┌────────────┴────────────┐                       │
│              ▼                         ▼                       │
│  ┌───────────────────────┐   ┌───────────────────────────┐    │
│  │  MÓDULO EXTRACCIÓN    │   │  MÓDULO PRODUCTO PUNTO    │    │
│  │  DE CARACTERÍSTICAS   │   │  (ML Inference)           │    │
│  │                       │   │                           │    │
│  │  • Media (suma+shift) │   │  • 6× multiplicaciones    │    │
│  │  • Varianza (MAC)     │   │    16×16 → 32 bits        │    │
│  │  • RMS (MAC+sqrt)     │   │  • Acumulación            │    │
│  │                       │   │  • Shift Q5.25 → Q3.14    │    │
│  │  Usa 3 DSP blocks     │   │                           │    │
│  │  (1 por eje)          │   │  Usa 6 DSP blocks         │    │
│  │                       │   │  (en paralelo)            │    │
│  └───────────┬───────────┘   └─────────────┬─────────────┘    │
│              │                             │                   │
│              └─────────────┬───────────────┘                   │
│                            ▼                                   │
│              ┌─────────────────────────────┐                  │
│              │  CONTROLADOR DE ESTADO      │                  │
│              │  (FSM: IDLE → ACQ → CALC)   │                  │
│              └─────────────────────────────┘                  │
│                                                                 │
│  Total DSP blocks utilizados: 9 de 11 disponibles              │
└─────────────────────────────────────────────────────────────────┘
```

**Conceptos clave a explicar:**
- **Interfaz AMBA APB:** Protocolo de bajo consumo para periféricos. El ARM Cortex-M3 accede a los registros del IP mediante direcciones de memoria mapeadas.
- **Mapeo de memoria:** El IP ocupa una región del espacio de direcciones del ARM. Cada registro tiene una dirección específica (0x00, 0x04, 0x08, etc.).
- **Datapath vs. Control:** El datapath realiza las operaciones aritméticas (multiplicaciones, sumas). El control (FSM) coordina la secuencia de operaciones.
- **Bloques DSP del SmartFusion2:** Multiplicadores hardware de 18×18 bits que realizan la operación en un solo ciclo de reloj. Mucho más eficientes que implementar multiplicaciones en LUTs.

### 3.2.2 Bloques DSP del SmartFusion2 M2S005 (15 min)

El SmartFusion2 M2S005 dispone de 11 bloques multiplicadores dedicados de 18×18 bits, según la Sección 2 del manual de usuario de la Polaris.

**Características de los bloques DSP:**
- Operación: A × B + C (multiplicación con acumulación)
- Ancho de operandos: hasta 18×18 bits signed o unsigned
- Latencia: 1 ciclo de reloj (con pipeline opcional de 2-3 ciclos)
- Throughput: 1 multiplicación por ciclo
- Consumo: significativamente menor que implementar en LUTs

**Inferencia de bloques DSP en Verilog:**
El sintetizador de Libero SoC (Synplify) infiere automáticamente bloques DSP cuando detecta patrones de multiplicación de hasta 18×18 bits. Para garantizar la inferencia, se deben seguir las siguientes prácticas:

```verilog
// Forma correcta: el sintetizador infiere DSP block
reg [17:0] a, b;
reg [35:0] product;
always @(posedge clk) begin
    product <= a * b;  // Inferencia automática de DSP
end

// Forma incorrecta: puede no inferir DSP
wire [35:0] product = a * b;  // Lógica combinacional pura
```

**Atributos para forzar inferencia (si es necesario):**

```verilog
(* syn_dspstyle = "dsp18x18" *)  // Fuerza uso de DSP block
(* syn_black_box *)              // Evita optimización
```

**Verificación en el reporte de síntesis:**
Después de sintetizar, el reporte debe mostrar:
```
DSP Blocks Used: 9 of 11 (81%)
  - 3 blocks for feature extraction (mean, variance, RMS per axis)
  - 6 blocks for dot product (parallel multiplications)
```

### 3.2.3 Protocolo AMBA APB (20 min)

El bus AMBA APB (Advanced Peripheral Bus) es un bus de bajo costo y bajo consumo utilizado para conectar periféricos de baja velocidad.

**Señales del APB:**

| Señal | Dirección | Descripción |
|---|---|---|
| PCLK | Input | Reloj del bus (50 MHz) |
| PRESETn | Input | Reset activo bajo |
| PADDR[11:0] | Input | Dirección (12 bits, 4 KB de espacio) |
| PSEL | Input | Select del slave |
| PENABLE | Input | Enable de la transferencia |
| PWRITE | Input | 1 = escritura, 0 = lectura |
| PWDATA[31:0] | Input | Datos de escritura |
| PRDATA[31:0] | Output | Datos de lectura |
| PREADY | Output | Ready (1 = sin wait states) |

**Ciclo de transferencia APB:**

```
Ciclo de ESCRITURA:
  Fase 1 (Setup):  PSEL=1, PADDR=válido, PWDATA=válido, PWRITE=1, PENABLE=0
  Fase 2 (Access): PSEL=1, PADDR=válido, PWDATA=válido, PWRITE=1, PENABLE=1
                   → Si PREADY=1, la escritura se completa

Ciclo de LECTURA:
  Fase 1 (Setup):  PSEL=1, PADDR=válido, PWRITE=0, PENABLE=0
  Fase 2 (Access): PSEL=1, PADDR=válido, PWRITE=0, PENABLE=1
                   → PRDATA=válido, PREADY=1
```

**Nota importante:** En el SmartFusion2, el MSS Configurator genera automáticamente la lógica de interfaz entre el ARM y la fabric. El estudiante únicamente debe implementar el slave APB según el template proporcionado.

### 3.2.4 Aritmética de punto fijo en hardware (25 min)

Se repasan los formatos de punto fijo definidos en el Módulo 2:

| Variable | Formato | Bits | Rango | Uso |
|---|---|---|---|---|
| Muestra ADXL345 | Entero signed | 10 | [-512, 511] | Entrada del sensor |
| Característica | Q3.12 | 16 | [-8.0, +7.9998] | Salida de extracción |
| Peso ML | Q2.13 | 16 | [-4.0, +3.9999] | Coeficientes del modelo |
| Producto | Q5.25 | 32 | (intermedio) | Resultado de multiplicación |
| Score | Q3.14 | 18 | [-8.0, +7.9999] | Salida del producto punto |

**Operaciones clave:**

1. **Multiplicación Q3.12 × Q2.13:**
   ```
   Operando A: 16 bits signed (Q3.12)
   Operando B: 16 bits signed (Q2.13)
   Resultado:  32 bits signed (Q5.25)
   
   En Verilog:
   wire signed [15:0] a_q3_12;
   wire signed [15:0] b_q2_13;
   wire signed [31:0] product_q5_25;
   
   assign product_q5_25 = a_q3_12 * b_q2_13;
   ```

2. **Ajuste de formato Q5.25 → Q3.14:**
   ```
   Se requiere shift right por 11 bits (25 - 14 = 11)
   
   wire signed [31:0] product_q5_25;
   wire signed [17:0] result_q3_14;
   
   assign result_q3_14 = product_q5_25[28:11];  // Truncamiento
   // O con redondeo:
   assign result_q3_14 = (product_q5_25 + 32'd1024) >>> 11;
   ```

3. **Acumulación de productos (producto punto):**
   ```
   6 productos Q5.25 se suman en un acumulador de 32 bits
   
   reg signed [31:0] accumulator;
   always @(posedge clk) begin
       if (reset) accumulator <= 32'd0;
       else if (enable) accumulator <= accumulator + product_q5_25;
   end
   ```

4. **División por N (para media y varianza):**
   ```
   En lugar de división, se usa shift right
   Para N = 50: shift right por 6 (aproximación: 50 ≈ 64)
   
   wire signed [31:0] sum;
   wire signed [15:0] mean_q3_12;
   
   assign mean_q3_12 = sum[25:10];  // Shift right 10 + ajuste de formato
   ```

**Consideración crítica:** La aproximación de N=50 como 64 (shift right 6) introduce un error de ~28%. Para mayor precisión, se puede usar una constante de multiplicación:
```
mean = sum * (2^20 / 50) >> 20
     = sum * 20971 >> 20
```
Esta multiplicación por constante también se puede implementar con DSP blocks.

---

## 3.3 Herramientas y configuración requeridas

| Herramienta | Configuración específica | Verificación |
|---|---|---|
| Libero SoC v12+ | Proyecto del Módulo 1 abierto | Diseño sintetiza sin errores |
| MSS Configurator | SPI habilitado (para Módulo 4), IP personalizado agregado | MSS genera sin errores |
| ModelSim o simulador de Libero | Para testbench del IP | Simulación ejecuta correctamente |
| Documento `especificacion_acelerador_hw.md` | Del Módulo 2 | Disponible como referencia |
| Archivo `test_vectors.txt` | Generado desde Python | Contiene 50 muestras + características esperadas |
| Template `amba_apb_slave_template.v` | Proporcionado por el instructor | Sintaxis verificada |

---

## 3.4 Implementación paso a paso

### PASO 1: Crear el módulo datapath del acelerador (60 min)

Se implementa el módulo `ml_accelerator_datapath` que realiza la extracción de características y el producto punto.

**Archivo a crear:** `ml_accelerator_datapath.v`

```verilog
//======================================================================
// Datapath del Acelerador ML
// IEEE CASS UMSA 2026 - SmartFusion2 M2S005
//======================================================================
// Funcionalidad:
// 1. Recibe muestras del ADXL345 (3 ejes × 10 bits)
// 2. Acumula en ventana de 50 muestras
// 3. Calcula 6 características (media, varianza, RMS × 2)
// 4. Calcula producto punto con pesos (6 multiplicaciones en paralelo)
// 5. Genera score de salida
//======================================================================

module ml_accelerator_datapath (
    input  wire         clk,
    input  wire         reset_n,
    input  wire         start,
    input  wire [29:0]  data_in,      // {ax[9:0], ay[9:0], az[9:0]}
    input  wire [31:0]  weights [0:17], // 18 pesos Q2.13
    
    output reg          busy,
    output reg          done,
    output reg  [31:0]  features [0:5], // 6 características Q3.12
    output reg  [31:0]  score           // Score Q3.14
);

    // ====================================================================
    // MÁQUINA DE ESTADOS
    // ====================================================================
    localparam [2:0] S_IDLE    = 3'd0,
                     S_ACQ     = 3'd1,  // Adquisición de 50 muestras
                     S_CALC    = 3'd2,  // Cálculo de características
                     S_DOTPROD = 3'd3,  // Producto punto
                     S_DONE    = 3'd4;
    
    reg [2:0] state, next_state;
    reg [5:0]  sample_count;  // Contador de muestras (0-49)
    
    // ====================================================================
    // ACUMULADORES PARA EXTRACCIÓN DE CARACTERÍSTICAS
    // ====================================================================
    // Para cada eje (X, Y, Z) y para la magnitud:
    // - Suma de muestras (para media)
    // - Suma de cuadrados (para varianza y RMS)
    
    // Eje X
    reg signed [31:0] sum_x;
    reg signed [31:0] sum_sq_x;
    
    // Eje Y
    reg signed [31:0] sum_y;
    reg signed [31:0] sum_sq_y;
    
    // Eje Z
    reg signed [31:0] sum_z;
    reg signed [31:0] sum_sq_z;
    
    // Magnitud (sqrt(ax^2 + ay^2 + az^2))
    reg signed [31:0] sum_mag;
    reg signed [31:0] sum_sq_mag;
    
    // Extracción de los 3 ejes desde data_in
    wire signed [9:0] ax = data_in[29:20];
    wire signed [9:0] ay = data_in[19:10];
    wire signed [9:0] az = data_in[9:0];
    
    // ====================================================================
    // MULTIPLICACIONES (inferencia de DSP blocks)
    // ====================================================================
    // Cada multiplicación de 10×10 bits se infiere como DSP block
    
    wire signed [19:0] ax_sq = ax * ax;
    wire signed [19:0] ay_sq = ay * ay;
    wire signed [19:0] az_sq = az * az;
    
    // Magnitud al cuadrado: ax^2 + ay^2 + az^2
    wire signed [21:0] mag_sq = ax_sq + ay_sq + az_sq;
    
    // Aproximación de magnitud (sqrt) usando aproximación lineal
    // Para simplificar, se usa mag ≈ (|ax| + |ay| + |az|) * 0.875
    // En hardware: shift y suma
    wire signed [9:0] abs_ax = ax[9] ? -ax : ax;
    wire signed [9:0] abs_ay = ay[9] ? -ay : ay;
    wire signed [9:0] abs_az = az[9] ? -az : az;
    
    wire signed [11:0] mag_approx = (abs_ax + abs_ay + abs_az) - 
                                     ((abs_ax + abs_ay + abs_az) >>> 3);
    
    wire signed [23:0] mag_approx_sq = mag_approx * mag_approx;
    
    // ====================================================================
    // ACUMULACIÓN (durante S_ACQ)
    // ====================================================================
    always @(posedge clk or negedge reset_n) begin
        if (!reset_n) begin
            sum_x      <= 32'd0;
            sum_sq_x   <= 32'd0;
            sum_y      <= 32'd0;
            sum_sq_y   <= 32'd0;
            sum_z      <= 32'd0;
            sum_sq_z   <= 32'd0;
            sum_mag    <= 32'd0;
            sum_sq_mag <= 32'd0;
        end else if (state == S_ACQ) begin
            sum_x      <= sum_x      + {{22{ax[9]}}, ax};
            sum_sq_x   <= sum_sq_x   + {{12{ax_sq[19]}}, ax_sq};
            sum_y      <= sum_y      + {{22{ay[9]}}, ay};
            sum_sq_y   <= sum_sq_y   + {{12{ay_sq[19]}}, ay_sq};
            sum_z      <= sum_z      + {{22{az[9]}}, az};
            sum_sq_z   <= sum_sq_z   + {{12{az_sq[19]}}, az_sq};
            sum_mag    <= sum_mag    + {{20{mag_approx[11]}}, mag_approx};
            sum_sq_mag <= sum_sq_mag + {{8{mag_approx_sq[23]}}, mag_approx_sq};
        end else if (state == S_IDLE) begin
            // Resetear acumuladores
            sum_x      <= 32'd0;
            sum_sq_x   <= 32'd0;
            sum_y      <= 32'd0;
            sum_sq_y   <= 32'd0;
            sum_z      <= 32'd0;
            sum_sq_z   <= 32'd0;
            sum_mag    <= 32'd0;
            sum_sq_mag <= 32'd0;
        end
    end
    
    // ====================================================================
    // CÁLCULO DE CARACTERÍSTICAS (durante S_CALC)
    // ====================================================================
    // Media = suma / 50 ≈ suma * 20971 >> 20
    // Varianza = (suma_sq / 50) - media^2
    // RMS = sqrt(suma_sq / 50) ≈ (suma_sq / 50) >> 1 (aproximación)
    
    // Constante para división por 50: 2^20 / 50 = 20971
    localparam signed [17:0] DIV_50_CONST = 18'd20971;
    
    // Multiplicaciones para dividir por 50 (usando DSP blocks)
    wire signed [49:0] mean_x_full = sum_x * DIV_50_CONST;
    wire signed [49:0] mean_y_full = sum_y * DIV_50_CONST;
    wire signed [49:0] mean_z_full = sum_z * DIV_50_CONST;
    wire signed [49:0] mean_mag_full = sum_mag * DIV_50_CONST;
    
    wire signed [15:0] mean_x   = mean_x_full[35:20];    // Q3.12
    wire signed [15:0] mean_y   = mean_y_full[35:20];
    wire signed [15:0] mean_z   = mean_z_full[35:20];
    wire signed [15:0] mean_mag = mean_mag_full[35:20];
    
    // Varianza del eje X
    wire signed [49:0] var_x_full = (sum_sq_x * DIV_50_CONST) - (mean_x * mean_x);
    wire signed [15:0] var_x = var_x_full[35:20];
    
    // RMS del eje X (aproximación: sqrt(var + mean^2) ≈ var >> 1 + mean)
    wire signed [15:0] rms_x = (var_x >>> 1) + mean_x;
    
    // ====================================================================
    // PRODUCTO PUNTO (durante S_DOTPROD)
    // ====================================================================
    // 6 multiplicaciones en paralelo: features[i] * weights[i]
    // features: Q3.12 (16 bits), weights: Q2.13 (16 bits)
    // producto: Q5.25 (32 bits)
    
    reg [2:0] dotprod_idx;
    reg signed [31:0] dotprod_acc;
    reg signed [31:0] current_product;
    
    // Multiplicador compartido (time-multiplexed) o 6 en paralelo
    // Para simplicidad, se usan 6 multiplicadores en paralelo
    
    wire signed [15:0] feat_0 = features[0][15:0];
    wire signed [15:0] feat_1 = features[1][15:0];
    wire signed [15:0] feat_2 = features[2][15:0];
    wire signed [15:0] feat_3 = features[3][15:0];
    wire signed [15:0] feat_4 = features[4][15:0];
    wire signed [15:0] feat_5 = features[5][15:0];
    
    wire signed [15:0] w0 = weights[0][15:0];
    wire signed [15:0] w1 = weights[1][15:0];
    wire signed [15:0] w2 = weights[2][15:0];
    wire signed [15:0] w3 = weights[3][15:0];
    wire signed [15:0] w4 = weights[4][15:0];
    wire signed [15:0] w5 = weights[5][15:0];
    
    // 6 multiplicaciones en paralelo (6 DSP blocks)
    wire signed [31:0] prod0 = feat_0 * w0;
    wire signed [31:0] prod1 = feat_1 * w1;
    wire signed [31:0] prod2 = feat_2 * w2;
    wire signed [31:0] prod3 = feat_3 * w3;
    wire signed [31:0] prod4 = feat_4 * w4;
    wire signed [31:0] prod5 = feat_5 * w5;
    
    // Suma de productos
    wire signed [34:0] dotprod_sum = {{3{prod0[31]}}, prod0} +
                                      {{3{prod1[31]}}, prod1} +
                                      {{3{prod2[31]}}, prod2} +
                                      {{3{prod3[31]}}, prod3} +
                                      {{3{prod4[31]}}, prod4} +
                                      {{3{prod5[31]}}, prod5};
    
    // Ajuste de formato Q5.25 → Q3.14 (shift right 11)
    wire signed [17:0] score_q3_14 = dotprod_sum[28:11];
    
    // ====================================================================
    // MÁQUINA DE ESTADOS: LÓGICA DE TRANSICIÓN
    // ====================================================================
    always @(posedge clk or negedge reset_n) begin
        if (!reset_n) begin
            state <= S_IDLE;
            sample_count <= 6'd0;
            busy <= 1'b0;
            done <= 1'b0;
        end else begin
            done <= 1'b0;
            
            case (state)
                S_IDLE: begin
                    busy <= 1'b0;
                    if (start) begin
                        state <= S_ACQ;
                        busy <= 1'b1;
                        sample_count <= 6'd0;
                    end
                end
                
                S_ACQ: begin
                    busy <= 1'b1;
                    if (sample_count == 6'd49) begin
                        state <= S_CALC;
                        sample_count <= 6'd0;
                    end else begin
                        sample_count <= sample_count + 6'd1;
                    end
                end
                
                S_CALC: begin
                    busy <= 1'b1;
                    // Calcular características y almacenar
                    features[0] <= {{16{mean_mag[15]}}, mean_mag};
                    features[1] <= {{16{var_x[15]}}, var_x};
                    features[2] <= {{16{rms_x[15]}}, rms_x};
                    features[3] <= {{16{mean_x[15]}}, mean_x};
                    features[4] <= 32'd0; // var_ax (extender si se requiere)
                    features[5] <= 32'd0; // rms_ax (extender si se requiere)
                    
                    state <= S_DOTPROD;
                end
                
                S_DOTPROD: begin
                    busy <= 1'b1;
                    // Score calculado en un ciclo (6 multiplicaciones en paralelo)
                    score <= {{14{score_q3_14[17]}}, score_q3_14};
                    state <= S_DONE;
                end
                
                S_DONE: begin
                    busy <= 1'b0;
                    done <= 1'b1;
                    state <= S_IDLE;
                end
                
                default: state <= S_IDLE;
            endcase
        end
    end

endmodule
```

**Verificación de inferencia de DSP blocks:**
Después de sintetizar este módulo, el reporte de Libero SoC debe indicar el uso de 9 bloques DSP:
- 3 bloques para `ax*ax`, `ay*ay`, `az*az`
- 1 bloque para `mag_approx * mag_approx`
- 4 bloques para las divisiones por 50 (media de magnitud, X, Y, Z)
- 1 bloque para `mean_x * mean_x` (varianza)
- 6 bloques para el producto punto (prod0 a prod5)

**Nota:** Si el total supera los 11 bloques disponibles, se debe time-multiplexar el producto punto (usar 1 o 2 DSP blocks y calcular las 6 multiplicaciones en 3-6 ciclos).

---

### PASO 2: Integrar el datapath con el template AMBA APB (20 min)

Se verifica que el template `amba_apb_slave_template.v` instancia correctamente el datapath.

**Verificación del top-level:**

```verilog
module top (
    input  wire       CLK_50MHZ,
    input  wire       SW0,
    output wire [9:0] LED,
    output wire       UART_TX,
    input  wire       UART_RX
);

    // Señales del MSS (generadas por el MSS Configurator)
    wire        mss_ready;
    wire [11:0] mss_apb_paddr;
    wire        mss_apb_psel;
    wire        mss_apb_penable;
    wire        mss_apb_pwrite;
    wire [31:0] mss_apb_pwdata;
    wire [31:0] mss_apb_prdata;
    wire        mss_apb_pready;
    
    // Instancia del MSS (generada automáticamente)
    mss_component u_mss (
        // ... conexiones del MSS ...
        .APB_PADDR(mss_apb_paddr),
        .APB_PSEL(mss_apb_psel),
        // ... etc ...
    );
    
    // Instancia del acelerador ML
    amba_apb_slave_template u_ml_accel (
        .PCLK    (CLK_50MHZ),
        .PRESETn (SW0),
        .PADDR   (mss_apb_paddr),
        .PSEL    (mss_apb_psel),
        .PENABLE (mss_apb_penable),
        .PWRITE  (mss_apb_pwrite),
        .PWDATA  (mss_apb_pwdata),
        .PRDATA  (mss_apb_prdata),
        .PREADY  (mss_apb_pready),
        .IRQ     ()
    );

endmodule
```

**Nota:** Las señales exactas del MSS dependen de la configuración del MSS Configurator. El instructor debe verificar los nombres de puertos en el componente MSS generado.

---

### PASO 3: Configurar el MSS Configurator para integrar el IP (40 min)

1. **Abrir el MSS Configurator** desde el proyecto de Libero SoC.
2. **Habilitar el SPI** (necesario para el Módulo 4):
   - Ir a la pestaña de periféricos → SPI.
   - Habilitar **SPI_0**.
   - Configurar como Master, modo 0 (CPOL=0, CPHA=0), frecuencia máxima 10 MHz.
   - Ruteare las señales SPI a los pines del ADXL345:
     - SPI0_CLK → Pin P3 (ADXL_SCL)
     - SPI0_MOSI → Pin N4 (ADXL_SDA)
     - SPI0_MISO → Pin N3 (ADXL_SDO)
     - SPI0_SS0 → Pin M3 (ADXL_CS)
3. **Agregar el IP personalizado:**
   - En el MSS Configurator, buscar la opción "Add Custom IP" o "User IP".
   - Seleccionar el módulo `amba_apb_slave_template`.
   - Configurar la dirección base: **0x4000_0000** (o la que asigna el MSS Configurator automáticamente).
   - Configurar el tamaño: **4 KB** (12 bits de dirección).
   - Conectar las señales APB del MSS al IP.
4. **Generar el MSS:**
   - Hacer clic en "Generate MSS Component".
   - Verificar que no hay errores.
   - El MSS Configurator genera automáticamente:
     - Los archivos de drivers HAL para el SPI.
     - Las macros de C para acceder al IP personalizado (direcciones de registros).
     - La lógica de interfaz entre el ARM y la fabric.

**Resultado esperado:** El MSS Configurator muestra el ARM Cortex-M3 conectado al bus AHB, el SPI_0 ruteado a los pines del ADXL345, y el IP personalizado conectado al bus APB.

**Archivos generados que deben conservarse:**
- `mss_spi.h` y `mss_spi.c` (drivers del SPI).
- `mss_accel_custom.h` (macros para acceder al IP personalizado).
- El componente MSS actualizado.

---

### PASO 4: Escribir el testbench del acelerador (45 min)

Se crea un testbench para verificar el funcionamiento del datapath antes de sintetizar.

**Archivo a crear:** `tb_ml_accelerator.v`

```verilog
`timescale 1ns / 1ps

module tb_ml_accelerator;

    // Señales de prueba
    reg         clk;
    reg         reset_n;
    reg         start;
    reg  [29:0] data_in;
    reg  [31:0] weights [0:17];
    
    wire        busy;
    wire        done;
    wire [31:0] features [0:5];
    wire [31:0] score;
    
    // Instancia del datapath
    ml_accelerator_datapath uut (
        .clk       (clk),
        .reset_n   (reset_n),
        .start     (start),
        .data_in   (data_in),
        .weights   (weights),
        .busy      (busy),
        .done      (done),
        .features  (features),
        .score     (score)
    );
    
    // Generación de reloj 50 MHz
    initial begin
        clk = 0;
        forever #10 clk = ~clk;  // Período de 20 ns = 50 MHz
    end
    
    // Vectores de prueba (cargados desde archivo o definidos aquí)
    reg [9:0] test_data [0:49];  // 50 muestras de prueba
    
    // Tarea para cargar datos de prueba
    task load_test_data;
        integer i;
        begin
            // Datos de ejemplo: reposo (gravedad en Z)
            for (i = 0; i < 50; i = i + 1) begin
                test_data[i] = 30'b0;  // ax = 0
                test_data[i] = 30'b0;  // ay = 0
                test_data[i][9:0] = 10'd256;  // az = 256 (≈ 1g)
            end
        end
    endtask
    
    // Tarea para cargar pesos
    task load_weights;
        begin
            // Pesos de ejemplo (Q2.13)
            weights[0]  = 32'd100;   // w[0][0]
            weights[1]  = 32'd200;   // w[0][1]
            weights[2]  = 32'd-150;  // w[0][2]
            weights[3]  = 32'd50;    // w[0][3]
            weights[4]  = 32'd0;     // w[0][4]
            weights[5]  = 32'd0;     // w[0][5]
            // ... completar los 18 pesos ...
        end
    endtask
    
    // Test principal
    integer i;
    initial begin
        // Inicialización
        reset_n = 0;
        start = 0;
        data_in = 30'd0;
        
        load_test_data;
        load_weights;
        
        // Reset
        #100;
        reset_n = 1;
        #50;
        
        // Enviar 50 muestras
        for (i = 0; i < 50; i = i + 1) begin
            @(posedge clk);
            data_in = {test_data[i], test_data[i], test_data[i]};
            if (i == 0) start = 1;
            else start = 0;
        end
        
        // Esperar cálculo
        wait(done);
        
        // Verificar resultados
        #100;
        $display("=== Resultados del Acelerador ===");
        $display("Features[0] (mean_mag): %d", features[0]);
        $display("Features[1] (var_x):    %d", features[1]);
        $display("Features[2] (rms_x):    %d", features[2]);
        $display("Score:                  %d", score);
        
        // Comparar con valores esperados (calculados en Python)
        // $display("Error: %d", features[0] - expected_feat0);
        
        $finish;
    end
    
    // Timeout de seguridad
    initial begin
        #100000;
        $display("ERROR: Timeout en la simulación");
        $finish;
    end

endmodule
```

**Ejecución del testbench:**
1. En Libero SoC, ir a `Tools → Simulation → Run Simulation`.
2. Seleccionar el testbench `tb_ml_accelerator`.
3. Ejecutar la simulación.
4. Verificar en la consola que los valores de `features` y `score` coinciden con los calculados en Python (con tolerancia de ±1 LSB).

**Resultado esperado:**
- La simulación completa en aproximadamente 100 ciclos de reloj (50 de adquisición + 1 de cálculo + 1 de producto punto).
- Los valores de `features` coinciden con los del script Python con error ≤ 1 LSB.
- El valor de `score` coincide con el producto punto calculado en Python.

**Si los valores no coinciden:**
- Verificar los shifts de formato (Q3.12, Q2.13, Q3.14).
- Verificar la extensión de signo en las sumas.
- Comparar paso a paso con los cálculos de Python.

---

### PASO 5: Síntesis, Place & Route y programación (30 min)

1. **Build Hierarchy:**
   - En el Design Flow, hacer doble clic en "Build Hierarchy".
   - Verificar que no hay errores.

2. **Síntesis:**
   - Hacer doble clic en "Synthesis".
   - Esperar a que finalice.
   - **Verificar en el reporte:**
     - DSP Blocks Used: 9 de 11 (o el número que corresponda).
     - LUTs utilizados: debe ser < 6060.
     - Frecuencia máxima: debe ser ≥ 50 MHz.

3. **Place & Route:**
   - Hacer doble clic en "Place and Route".
   - Esperar a que finalice.
   - **Verificar en el reporte:**
     - Timing: todos los paths deben cumplir con el reloj de 50 MHz (slack ≥ 0).
     - Recursos: verificar que no se exceden los límites del dispositivo.

4. **Programación:**
   - Conectar la placa Polaris al PC.
   - Hacer doble clic en "Program Device".
   - Seleccionar FlashPro5.
   - Programar la FPGA y el ARM.

**Resultado esperado:**
- La programación se completa sin errores.
- El sistema está listo para ser probado desde el firmware del ARM (Módulo 4).

---

## 3.5 Verificación final del Módulo 3

Al finalizar el Módulo 3, se debe contar con:

| Criterio | Verificación |
|---|---|
| Datapath sintetiza sin errores | ✅ Verificar en reporte de síntesis |
| 9 bloques DSP utilizados | ✅ Verificar en reporte de síntesis |
| Testbench pasa con valores correctos | ✅ Comparar con Python |
| IP integrado en MSS Configurator | ✅ MSS genera sin errores |
| SPI habilitado para Módulo 4 | ✅ Verificar en MSS Configurator |
| Place & Route cumple timing 50 MHz | ✅ Verificar slack ≥ 0 |
| Programación exitosa | ✅ FlashPro5 sin errores |

---

## 3.6 Entregables del Módulo 3 (conservar para Módulos siguientes)

| Entregable | Uso futuro |
|---|---|
| Módulo `ml_accelerator_datapath.v` | Parte del diseño final |
| Template `amba_apb_slave_template.v` | Integrado en el top-level |
| MSS Configurator con SPI + IP personalizado | Base para Módulos 4, 5, 6 |
| Drivers HAL del SPI (`mss_spi.h`) | Se utilizan en Módulo 4 |
| Macros de acceso al IP (`mss_accel_custom.h`) | Se utilizan en Módulos 4, 5, 6 |
| Testbench y resultados de simulación | Evidencia de funcionamiento |
| Reporte de síntesis (uso de DSP blocks) | Se analiza en Módulo 6 |

---

## 3.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| Síntesis reporta > 11 DSP blocks | Demasiadas multiplicaciones en paralelo | Time-multiplexar el producto punto |
| Timing violation a 50 MHz | Ruta crítica en sumas de 32 bits | Agregar pipeline stages o relajar timing |
| Testbench da valores incorrectos | Error en shifts de formato | Revisar Q3.12, Q2.13, Q3.14 |
| MSS Configurator no genera | IP mal configurado o direcciones conflictivas | Verificar dirección base y tamaño |
| ARM no puede acceder al IP | Dirección incorrecta en el firmware | Verificar macro `mss_accel_custom.h` |
| SPI no funciona con ADXL345 | Pines mal ruteados o modo SPI incorrecto | Verificar CPOL, CPHA, pines en MSS Configurator |
| Place & Route falla | Recursos insuficientes | Reducir uso de LUTs o cambiar dispositivo |

---

## 3.8 Resumen del Día 3

**Antes del Día 3:** Template AMBA APB preparado. Vectores de prueba generados desde Python. Proyecto del Módulo 1 disponible.

**Durante el Día 3 (4 horas):**
- 0:00–0:30: Teoría de bloques DSP, AMBA APB, punto fijo en hardware
- 0:30–1:30: Implementar datapath en Verilog (PASO 1)
- 1:30–1:50: Integrar con template AMBA APB (PASO 2)
- 1:50–2:30: Configurar MSS Configurator con SPI + IP (PASO 3)
- 2:30–3:15: Escribir y ejecutar testbench (PASO 4)
- 3:15–3:45: Síntesis, Place & Route (PASO 5)
- 3:45–4:00: Programación y verificación final

**Al final del Día 3:** IP personalizado sintetizado e integrado en el MSS, con 9 bloques DSP utilizados, testbench verificado, SPI habilitado para el Módulo 4, y diseño programado en la placa Polaris.

---

## 3.9 Relación con los otros módulos

**Relación con el Módulo 1:**
- Se reutiliza el proyecto `Polaris_Lab1` con el MSS configurado.
- Se amplía el MSS agregando el SPI y el IP personalizado.

**Relación con el Módulo 2:**
- Se implementa exactamente lo especificado en `especificacion_acelerador_hw.md`.
- Se utilizan los formatos de punto fijo Q3.12, Q2.13, Q3.14 definidos en Python.
- Los vectores de prueba se generan desde el script `ml_online_sim.py`.

**Relación con el Módulo 4:**
- El SPI habilitado en el MSS Configurator se utilizará para comunicar con el ADXL345.
- El driver SPI en C enviará las muestras al acelerador escribiendo en el registro `DATA_IN` (0x08).
- El driver leerá las características desde los registros `FEAT_0` a `FEAT_5` (0x0C a 0x20).

**Relación con el Módulo 5:**
- El filtro de Kalman utilizará las características calculadas por el acelerador para ajustar las matrices Q y R.
- Los pesos del modelo ML se cargarán en los registros `WEIGHT_0` a `WEIGHT_17` (0x24 a 0x68).
- El score de salida (registro `SCORE` en 0x6C) se utilizará para la inferencia ML.

**Relación con el Módulo 6:**
- Se analizará el uso de recursos DSP (9 de 11 bloques) y se comparará con una implementación puramente en software.
- Se medirá la latencia del acelerador usando el analizador lógico 24MHz 8CH.
- Se evaluará el throughput del sistema completo.

---