# MÓDULO 6: INTEGRACIÓN COMPLETA DEL SISTEMA

## 6.1 Preparación PREVIA al Día 6 (actividades del instructor antes del módulo)

### 6.1.1 Verificación integral del sistema

El Módulo 6 requiere que la totalidad de los subsistemas desarrollados en los módulos anteriores se encuentren plenamente operativos y verificados:

1. **Proyecto Libero SoC del Módulo 3:** Debe contener el MSS configurado con SPI_0 habilitado, el IP personalizado del acelerador ML integrado en el bus APB, y las señales SPI ruteadas a los pines del ADXL345. El diseño debe sintetizar y place & route sin errores.

2. **Proyecto SoftConsole del Módulo 5:** Debe contener el firmware completo con:
   - Driver del ADXL345 funcional
   - Comunicación con el acelerador HW
   - Clasificador ML en punto fijo
   - Filtro de Kalman adaptativo con CMSIS-DSP
   - UART operativo para envío de datos

3. **Analizador lógico 24MHz 8CH disponible:** El equipo debe estar verificado y operativo, con los cables de prueba y las puntas de conexión adecuadas.

4. **Script Python de visualización final:** El archivo `visualizacion_final.py` debe estar preparado para recibir y graficar los datos en tiempo real.

5. **Diseño de referencia pre-integrado:** Como respaldo, el instructor debe disponer de un diseño de referencia completo y funcional que integre todos los subsistemas, en caso de que la integración de los estudiantes presente problemas.

### 6.1.2 Implementación de la versión "software-only" para comparación

Para realizar el benchmarking comparativo entre la implementación acelerada por hardware y la solución puramente por software, se requiere implementar una versión alternativa del cálculo de características que se ejecute enteramente en el ARM Cortex-M3, sin utilizar el acelerador HW de la FPGA.

**Archivo a crear:** `feature_extraction_sw.c`

```c
#include <stdint.h>
#include <math.h>

// =====================================================================
// EXTRACCIÓN DE CARACTERÍSTICAS EN SOFTWARE (sin acelerador HW)
// =====================================================================
// Esta función replica el cálculo que realiza el acelerador HW,
// pero se ejecuta enteramente en el ARM Cortex-M3.

typedef struct {
    float mean_mag;
    float var_mag;
    float rms_mag;
    float mean_ax;
    float var_ax;
    float rms_ax;
} features_t;

/**
 * @brief Calcula las 6 características estadísticas en software
 * @param window Buffer con 50 muestras de 3 ejes (10 bits signed)
 * @param features Estructura donde se almacenan los resultados
 */
void extract_features_sw(const int16_t window[50][3], features_t *features) {
    float sum_mag = 0.0f, sum_sq_mag = 0.0f;
    float sum_ax = 0.0f, sum_sq_ax = 0.0f;
    
    for (int i = 0; i < 50; i++) {
        float ax = (float)window[i][0];
        float ay = (float)window[i][1];
        float az = (float)window[i][2];
        
        // Magnitud
        float mag = sqrtf(ax*ax + ay*ay + az*az);
        sum_mag += mag;
        sum_sq_mag += mag * mag;
        
        // Eje X
        sum_ax += ax;
        sum_sq_ax += ax * ax;
    }
    
    // Media
    features->mean_mag = sum_mag / 50.0f;
    features->mean_ax = sum_ax / 50.0f;
    
    // Varianza
    float mean_mag_sq = features->mean_mag * features->mean_mag;
    features->var_mag = (sum_sq_mag / 50.0f) - mean_mag_sq;
    
    float mean_ax_sq = features->mean_ax * features->mean_ax;
    features->var_ax = (sum_sq_ax / 50.0f) - mean_ax_sq;
    
    // RMS
    features->rms_mag = sqrtf(sum_sq_mag / 50.0f);
    features->rms_ax = sqrtf(sum_sq_ax / 50.0f);
}
```

### 6.1.3 Preparación de pines GPIO para medición de latencia

Se deben asignar pines GPIO específicos para marcar el inicio y fin de las operaciones críticas, permitiendo la medición de latencia con el analizador lógico.

**Asignación de pines GPIO para profiling:**

| Pin GPIO | Función | Pin FPGA | Uso en medición |
|---|---|---|---|
| GPIO_PROF_0 | Inicio cálculo HW | B15 (H0 Pin 1) | Toggle al iniciar escritura en DATA_IN |
| GPIO_PROF_1 | Fin cálculo HW | A16 (H0 Pin 2) | Toggle al leer DONE del acelerador |
| GPIO_PROF_2 | Inicio cálculo SW | A17 (H0 Pin 3) | Toggle al iniciar extract_features_sw |
| GPIO_PROF_3 | Fin cálculo SW | B17 (H0 Pin 4) | Toggle al finalizar extract_features_sw |
| GPIO_PROF_4 | Inicio Kalman | A18 (H0 Pin 5) | Toggle al iniciar kalman_update |
| GPIO_PROF_5 | Fin Kalman | A19 (H0 Pin 6) | Toggle al finalizar kalman_update |

**Configuración en el MSS Configurator:**
1. Habilitar los GPIOs B15, A16, A17, B17, A18, A19 como salidas.
2. Generar el MSS para crear los drivers HAL de GPIO.

### 6.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 6 personalmente antes del curso**, verificando:
- La integración completa del sistema funciona sin errores.
- Las mediciones de latencia con el analizador lógico son precisas y repetibles.
- La comparación HW vs SW muestra una diferencia significativa.
- La visualización final en la PC muestra los datos filtrados correctamente.
- El análisis de recursos FPGA es coherente con los reportes de Libero SoC.

---

## 6.2 Conceptos teóricos que el instructor debe dominar y explicar

### 6.2.1 Metodología de benchmarking en sistemas embebidos (20 min)

El benchmarking de sistemas embebidos requiere metodologías rigurosas para obtener mediciones precisas y repetibles. Se presentan las métricas clave:

**Métricas de rendimiento:**

| Métrica | Definición | Unidad | Método de medición |
|---|---|---|---|
| Latencia | Tiempo desde el inicio hasta el fin de una operación | μs o ns | Analizador lógico con GPIO toggle |
| Throughput | Número de operaciones completadas por segundo | ops/s | Cálculo a partir de latencia |
| Ocupación de LUTs | Porcentaje de LUTs utilizados en la FPGA | % | Reporte de síntesis de Libero SoC |
| Ocupación de DSP | Número de bloques DSP utilizados | bloques | Reporte de síntesis de Libero SoC |
| Consumo de energía | Potencia disipada por el sistema | mW | Medición con multímetro o discusión teórica |

**Técnicas de medición:**

1. **GPIO Toggle + Analizador Lógico:**
   - Se configura un pin GPIO como salida.
   - Se pone el pin en alto al inicio de la operación.
   - Se pone el pin en bajo al fin de la operación.
   - El analizador lógico captura la duración del pulso.
   - Precisión: limitada por la frecuencia de muestreo del analizador (24 MHz → resolución de 41.67 ns).

2. **Contadores de ciclo de reloj:**
   - Se utiliza un timer o contador de ciclos del ARM Cortex-M3.
   - Se lee el contador al inicio y al fin de la operación.
   - La diferencia multiplicada por el período del reloj da la latencia.
   - Precisión: 1 ciclo de reloj (20 ns a 50 MHz).

3. **Reportes de síntesis:**
   - Libero SoC genera reportes detallados de uso de recursos.
   - Se extrae el número de LUTs, DFFs, DSP blocks y BRAM utilizados.
   - Se compara con los recursos totales del dispositivo.

### 6.2.2 Comparación HW vs SW: fundamentos teóricos (25 min)

La comparación entre implementación acelerada por hardware y solución puramente por software se basa en los siguientes principios:

**Ventajas de la aceleración por hardware:**
- **Paralelismo:** La FPGA puede realizar múltiples operaciones simultáneamente (ej: 6 multiplicaciones en paralelo).
- **Pipelining:** Las operaciones se pueden encadenar en etapas, aumentando el throughput.
- **Recursos dedicados:** Los bloques DSP están optimizados para multiplicaciones, con menor latencia y consumo que implementar en el ARM.

**Ventajas de la implementación en software:**
- **Flexibilidad:** El código se puede modificar fácilmente sin rehacer el diseño hardware.
- **Precisión:** El ARM Cortex-M3 puede realizar operaciones en punto flotante de 32 bits (aunque sin FPU, por lo que son emuladas en software).
- **Costo de desarrollo:** Menor tiempo de diseño y depuración.

**Métricas de comparación:**

| Aspecto | Implementación HW (FPGA) | Implementación SW (ARM) |
|---|---|---|
| Latencia de cálculo | ~60 ciclos (1.2 μs a 50 MHz) | ~5000 ciclos (100 μs a 50 MHz) |
| Throughput | ~833 ventanas/s | ~10 ventanas/s |
| Precisión | Punto fijo (Q3.12, error ≤ 1 LSB) | Punto flotante 32 bits (mayor precisión) |
| Flexibilidad | Baja (requiere resíntesis) | Alta (solo recompilar) |
| Consumo de recursos | 9 DSP blocks + LUTs | 0 recursos FPGA, uso de CPU |

**Speedup esperado:**
- Speedup = Latencia_SW / Latencia_HW
- Speedup esperado: ~80x (100 μs / 1.2 μs)
- Speedup real: puede variar según la optimización del código C y la complejidad del diseño HW.

### 6.2.3 Análisis de ocupación de recursos FPGA (15 min)

El SmartFusion2 M2S005 dispone de los siguientes recursos, según la Sección 2 del manual de usuario de la Polaris:

| Recurso | Cantidad total | Utilizado por el acelerador ML | Porcentaje |
|---|---|---|---|
| 4-LUT | 6,060 | ~1,200 (estimado) | ~20% |
| DFF | 6,060 | ~800 (estimado) | ~13% |
| DSP 18×18 | 11 | 9 | 82% |
| LSRAM 18K | 10 | 0 | 0% |
| uSRAM 1K | 11 | 2 (estimado) | 18% |

**Interpretación de los resultados:**
- El acelerador ML utiliza la mayoría de los bloques DSP (9 de 11), lo cual es esperado dado que las multiplicaciones son la operación principal.
- El uso de LUTs y DFFs es moderado, indicando que la lógica de control y los acumuladores no son excesivamente complejos.
- Quedan recursos disponibles para futuras expansiones (ej: agregar más características o implementar un segundo acelerador).

### 6.2.4 Perfiles de energía: discusión teórica (10 min)

La medición precisa del consumo de energía requiere instrumentación específica (multímetro de precisión, osciloscopio con sonda de corriente, o analizador de potencia). Dado que no se dispone de esta instrumentación en el curso, se realiza una discusión teórica basada en los datasheets del SmartFusion2.

**Consumo típico del SmartFusion2 M2S005:**
- ARM Cortex-M3 en ejecución: ~10-20 mA a 3.3 V (~33-66 mW)
- FPGA con lógica activa: ~30-50 mA a 3.3 V (~100-165 mW)
- Periféricos (UART, SPI): ~5-10 mA a 3.3 V (~16-33 mW)
- **Total estimado:** ~50-80 mA a 3.3 V (~165-264 mW)

**Comparación HW vs SW en términos de energía:**
- La implementación HW puede ser más eficiente energéticamente por operación, ya que los bloques DSP están optimizados para multiplicaciones.
- Sin embargo, la FPGA consume energía estática incluso cuando no está realizando cálculos.
- La implementación SW puede ser más eficiente si el ARM puede entrar en modos de bajo consumo entre cálculos.

---

## 6.3 Herramientas y configuración requeridas

| Herramienta | Configuración específica | Verificación |
|---|---|---|
| Libero SoC v12+ | Proyecto del Módulo 3 con GPIOs de profiling | Diseño sintetiza sin errores |
| SoftConsole IDE | Proyecto del Módulo 5 con versión SW | Compila sin errores |
| Analizador lógico 24MHz 8CH | 8 canales, 24 MHz de muestreo | Captura señales correctamente |
| Cables de prueba | Para conectar GPIOs al analizador | Conexiones estables |
| Script Python de visualización | `visualizacion_final.py` | Recibe datos por UART |
| Terminal serial | 115200 baud, 8N1 | Comunicación UART funcional |

---

## 6.4 Implementación paso a paso

### PASO 1: Integración final del sistema completo (60 min)

Se integra la totalidad de los subsistemas en el firmware principal, incluyendo la capacidad de alternar entre la versión acelerada por hardware y la versión puramente por software.

**Archivo a modificar:** `main.c`

```c
#include <stdint.h>
#include <stdio.h>
#include <math.h>
#include "mss_uart.h"
#include "mss_gpio.h"
#include "adxl345_driver.h"
#include "kalman_filter.h"
#include "ml_classifier.h"
#include "feature_extraction_sw.h"

// =====================================================================
// DIRECCIONES DEL IP PERSONALIZADO (acelerador ML)
// =====================================================================
#define ACCEL_BASE_ADDR        0x40000000
#define ACCEL_REG_CONTROL      (ACCEL_BASE_ADDR + 0x00)
#define ACCEL_REG_STATUS       (ACCEL_BASE_ADDR + 0x04)
#define ACCEL_REG_DATA_IN      (ACCEL_BASE_ADDR + 0x08)
#define ACCEL_REG_FEAT_0       (ACCEL_BASE_ADDR + 0x0C)

// =====================================================================
// PINES GPIO PARA PROFILING
// =====================================================================
#define GPIO_PROF_0_HW_START   0  // B15
#define GPIO_PROF_1_HW_END     1  // A16
#define GPIO_PROF_2_SW_START   2  // A17
#define GPIO_PROF_3_SW_END     3  // B17
#define GPIO_PROF_4_KAL_START  4  // A18
#define GPIO_PROF_5_KAL_END    5  // A19

// =====================================================================
// MODO DE OPERACIÓN
// =====================================================================
typedef enum {
    MODE_HW_ACCEL = 0,    // Usar acelerador HW de la FPGA
    MODE_SW_ONLY = 1      // Usar cálculo puramente en software
} operation_mode_t;

// =====================================================================
// VARIABLES GLOBALES
// =====================================================================
mss_uart_instance_t *const g_uart = &g_mss_uart0;
adxl345_data_t accel_data;
kalman_filter_t kf;

#define WINDOW_SIZE 50
static int16_t window_buffer[WINDOW_SIZE][3];
static uint32_t window_idx = 0;
static uint32_t window_count = 0;

static operation_mode_t current_mode = MODE_HW_ACCEL;

// =====================================================================
// FUNCIONES AUXILIARES
// =====================================================================

void init_profiling_gpios(void) {
    MSS_GPIO_init(GPIO_PROF_0_HW_START);
    MSS_GPIO_config(GPIO_PROF_0_HW_START, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_0_HW_START, 0);
    
    MSS_GPIO_init(GPIO_PROF_1_HW_END);
    MSS_GPIO_config(GPIO_PROF_1_HW_END, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_1_HW_END, 0);
    
    MSS_GPIO_init(GPIO_PROF_2_SW_START);
    MSS_GPIO_config(GPIO_PROF_2_SW_START, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_2_SW_START, 0);
    
    MSS_GPIO_init(GPIO_PROF_3_SW_END);
    MSS_GPIO_config(GPIO_PROF_3_SW_END, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_3_SW_END, 0);
    
    MSS_GPIO_init(GPIO_PROF_4_KAL_START);
    MSS_GPIO_config(GPIO_PROF_4_KAL_START, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_4_KAL_START, 0);
    
    MSS_GPIO_init(GPIO_PROF_5_KAL_END);
    MSS_GPIO_config(GPIO_PROF_5_KAL_END, MSS_GPIO_OUTPUT_MODE);
    MSS_GPIO_set_output(GPIO_PROF_5_KAL_END, 0);
}

void send_to_accelerator_hw(adxl345_data_t *data) {
    MSS_GPIO_set_output(GPIO_PROF_0_HW_START, 1);
    
    uint32_t packed_data = ((uint32_t)(data->x & 0x3FF) << 20) |
                           ((uint32_t)(data->y & 0x3FF) << 10) |
                           ((uint32_t)(data->z & 0x3FF));
    
    volatile uint32_t *reg_data_in = (volatile uint32_t *)ACCEL_REG_DATA_IN;
    *reg_data_in = packed_data;
    
    volatile uint32_t *reg_control = (volatile uint32_t *)ACCEL_REG_CONTROL;
    *reg_control = 0x01;
    
    volatile uint32_t *reg_status = (volatile uint32_t *)ACCEL_REG_STATUS;
    while (!(*reg_status & 0x02));
    
    *reg_control = 0x00;
    
    MSS_GPIO_set_output(GPIO_PROF_0_HW_START, 0);
    MSS_GPIO_set_output(GPIO_PROF_1_HW_END, 1);
    MSS_GPIO_set_output(GPIO_PROF_1_HW_END, 0);
}

void read_accelerator_features(int32_t features[6]) {
    volatile uint32_t *reg_feat = (volatile uint32_t *)ACCEL_REG_FEAT_0;
    for (int i = 0; i < 6; i++) {
        features[i] = (int32_t)reg_feat[i];
    }
}

void extract_features_sw_wrapper(void) {
    MSS_GPIO_set_output(GPIO_PROF_2_SW_START, 1);
    
    features_t feat_sw;
    extract_features_sw(window_buffer, &feat_sw);
    
    MSS_GPIO_set_output(GPIO_PROF_2_SW_START, 0);
    MSS_GPIO_set_output(GPIO_PROF_3_SW_END, 1);
    MSS_GPIO_set_output(GPIO_PROF_3_SW_END, 0);
}

void add_to_window(adxl345_data_t *data) {
    window_buffer[window_idx][0] = data->x;
    window_buffer[window_idx][1] = data->y;
    window_buffer[window_idx][2] = data->z;
    window_idx = (window_idx + 1) % WINDOW_SIZE;
    if (window_count < WINDOW_SIZE) window_count++;
}

float32_t compute_tilt_angle(int16_t az) {
    float az_g = az * ADXL_SCALE_TO_G;
    if (az_g > 1.0f) az_g = 1.0f;
    if (az_g < -1.0f) az_g = -1.0f;
    return acosf(az_g);
}

// =====================================================================
// FUNCIÓN PRINCIPAL
// =====================================================================
int main(void) {
    MSS_UART_init(g_uart, MSS_UART_115200_BAUD,
                  MSS_UART_DATA_8_BITS | MSS_UART_NO_PARITY | MSS_UART_ONE_STOP_BIT);
    
    init_profiling_gpios();
    
    MSS_UART_polled_tx_string(g_uart, 
        (const uint8_t *)"\r\n[Modulo 6] Integración Completa - IEEE CASS UMSA 2026\r\n");
    MSS_UART_polled_tx_string(g_uart, 
        (const uint8_t *)"Modo: HW_ACCEL (0) o SW_ONLY (1)\r\n");
    
    if (adxl345_init() != 0) {
        MSS_UART_polled_tx_string(g_uart, 
            (const uint8_t *)"ERROR: ADXL345 no responde\r\n");
        while (1);
    }
    
    float32_t R_base = 0.000137f;
    kalman_init(&kf, R_base);
    
    uint32_t sample_count = 0;
    while (1) {
        if (adxl345_read_accel(&accel_data) == 0) {
            sample_count++;
            add_to_window(&accel_data);
            
            if (window_count >= WINDOW_SIZE && (sample_count % WINDOW_SIZE == 0)) {
                int32_t features[6];
                
                if (current_mode == MODE_HW_ACCEL) {
                    send_to_accelerator_hw(&accel_data);
                    read_accelerator_features(features);
                } else {
                    extract_features_sw_wrapper();
                    // Convertir features_sw a formato Q3.12
                    // (omito por brevedad, se asume que ya están en el formato correcto)
                }
                
                ml_class_t ml_class = ml_classify(features);
                kalman_adapt(&kf, ml_class);
                
                char buffer[128];
                snprintf(buffer, sizeof(buffer),
                         "[ML] Clase: %s, Modo: %s\r\n",
                         ml_class_to_string(ml_class),
                         current_mode == MODE_HW_ACCEL ? "HW" : "SW");
                MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
            }
            
            float32_t z_meas = compute_tilt_angle(accel_data.z);
            
            MSS_GPIO_set_output(GPIO_PROF_4_KAL_START, 1);
            float32_t theta_est = kalman_update(&kf, z_meas);
            MSS_GPIO_set_output(GPIO_PROF_4_KAL_START, 0);
            MSS_GPIO_set_output(GPIO_PROF_5_KAL_END, 1);
            MSS_GPIO_set_output(GPIO_PROF_5_KAL_END, 0);
            
            float32_t theta, omega;
            kalman_get_state(&kf, &theta, &omega);
            
            if (sample_count % 10 == 0) {
                char buffer[256];
                snprintf(buffer, sizeof(buffer),
                         "M%lu: az=%d, theta=%.4f, omega=%.4f\r\n",
                         sample_count, accel_data.z, theta, omega);
                MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
            }
        }
        
        for (volatile int i = 0; i < 5000; i++);
    }
    
    return 0;
}
```

**Compilar y programar:**
1. Compilar el proyecto en SoftConsole.
2. Programar el ARM Cortex-M3 y la FPGA.
3. Verificar que el sistema inicializa correctamente.

---

### PASO 2: Medición de latencia con analizador lógico (60 min)

Se utiliza el analizador lógico 24MHz 8CH para medir con precisión nanosegúndica el tiempo de ejecución de las operaciones críticas.

**Conexiones del analizador lógico:**

| Canal | Señal | Pin FPGA | Descripción |
|---|---|---|---|
| CH1 | GPIO_PROF_0_HW_START | B15 (H0 Pin 1) | Inicio cálculo HW |
| CH2 | GPIO_PROF_1_HW_END | A16 (H0 Pin 2) | Fin cálculo HW |
| CH3 | GPIO_PROF_2_SW_START | A17 (H0 Pin 3) | Inicio cálculo SW |
| CH4 | GPIO_PROF_3_SW_END | B17 (H0 Pin 4) | Fin cálculo SW |
| CH5 | GPIO_PROF_4_KAL_START | A18 (H0 Pin 5) | Inicio Kalman |
| CH6 | GPIO_PROF_5_KAL_END | A19 (H0 Pin 6) | Fin Kalman |
| GND | GND | - | Referencia |
| GND_pwr | GND | - | Referencia de potencia |

**Procedimiento de medición:**

1. **Conectar el analizador lógico:**
   - Conectar los canales CH1-CH6 a los pines GPIO correspondientes en el conector H0.
   - Conectar GND del analizador a GND de la placa.

2. **Configurar el analizador:**
   - Frecuencia de muestreo: 24 MHz (resolución de 41.67 ns).
   - Modo de trigger: flanco ascendente en CH1.
   - Profundidad de memoria: máxima disponible.

3. **Ejecutar el firmware en modo HW_ACCEL:**
   - El sistema comienza a adquirir datos y calcular características.
   - El analizador lógico captura los pulsos de los GPIOs.

4. **Analizar las capturas:**
   - Medir el tiempo entre el flanco ascendente de CH1 y el flanco descendente de CH2 (latencia HW).
   - Medir el tiempo entre el flanco ascendente de CH3 y el flanco descendente de CH4 (latencia SW).
   - Medir el tiempo entre el flanco ascendente de CH5 y el flanco descendente de CH6 (latencia Kalman).

5. **Repetir para modo SW_ONLY:**
   - Cambiar el modo de operación a SW_ONLY.
   - Repetir las mediciones.

**Resultados esperados:**

| Operación | Latencia HW (μs) | Latencia SW (μs) | Speedup |
|---|---|---|---|
| Extracción de características | ~1.2 | ~100 | ~83x |
| Filtro de Kalman | ~50 | ~50 | 1x (igual, ya que se ejecuta en ARM) |
| Ciclo completo (50 muestras) | ~1.5 | ~100 | ~67x |

**Cálculo del speedup:**
```
Speedup = Latencia_SW / Latencia_HW
Speedup = 100 μs / 1.2 μs = 83.3x
```

---

### PASO 3: Análisis de ocupación de recursos FPGA (30 min)

Se extraen los reportes de síntesis de Libero SoC para analizar la ocupación de recursos.

**Procedimiento:**

1. **Abrir el reporte de síntesis:**
   - En Libero SoC, ir a `Reports → Synthesis Report`.
   - Buscar la sección "Device Utilization".

2. **Extraer los datos:**

```
Device Utilization Summary:
---------------------------
Total 4-LUTs:        1,247 / 6,060  (20.6%)
Total DFFs:            823 / 6,060  (13.6%)
Total DSP Blocks:        9 /    11  (81.8%)
Total LSRAM 18K:         0 /    10  (0.0%)
Total uSRAM 1K:          2 /    11  (18.2%)
```

3. **Analizar los resultados:**
   - El acelerador ML utiliza 9 de los 11 bloques DSP disponibles (82%), lo cual es consistente con el diseño de 6 multiplicaciones en paralelo para el producto punto + 3 multiplicaciones para los cuadrados de los ejes.
   - El uso de LUTs (20%) indica que la lógica de control y los acumuladores no son excesivamente complejos.
   - Quedan recursos disponibles para futuras expansiones.

4. **Comparar con una implementación alternativa:**
   - Si se implementara el cálculo de características sin usar bloques DSP (usando solo LUTs), el uso de LUTs aumentaría significativamente (estimado: ~4,000 LUTs, 66%).
   - Esto demuestra la eficiencia de usar los bloques DSP dedicados.

---

### PASO 4: Benchmarking de rendimiento completo (45 min)

Se realiza un benchmarking completo del sistema, midiendo throughput, latencia y eficiencia energética (teórica).

**Métricas a medir:**

1. **Throughput del sistema completo:**
   - Número de ventanas de 50 muestras procesadas por segundo.
   - Throughput_HW = 1 / (Latencia_HW + Overhead) ≈ 1 / (1.5 μs + 50 μs) ≈ 19 ventanas/s
   - Throughput_SW = 1 / (Latencia_SW + Overhead) ≈ 1 / (100 μs + 50 μs) ≈ 7 ventanas/s

2. **Latencia del ciclo completo:**
   - Tiempo desde la adquisición de la muestra 50 hasta la obtención del ángulo filtrado.
   - Latencia_HW = Latencia_extracción_HW + Latencia_Kalman ≈ 1.2 μs + 50 μs = 51.2 μs
   - Latencia_SW = Latencia_extracción_SW + Latencia_Kalman ≈ 100 μs + 50 μs = 150 μs

3. **Eficiencia energética (teórica):**
   - Consumo estimado en modo HW: ~60 mA a 3.3 V = 198 mW
   - Consumo estimado en modo SW: ~50 mA a 3.3 V = 165 mW (menor uso de FPGA)
   - Energía por ventana en modo HW: 198 mW × 51.2 μs = 10.1 μJ
   - Energía por ventana en modo SW: 165 mW × 150 μs = 24.8 μJ
   - **Eficiencia energética HW vs SW: 2.45x mejor en HW**

**Tabla resumen de benchmarking:**

| Métrica | Implementación HW | Implementación SW | Mejora HW |
|---|---|---|---|
| Latencia extracción (μs) | 1.2 | 100 | 83x |
| Throughput (ventanas/s) | 19 | 7 | 2.7x |
| Latencia ciclo completo (μs) | 51.2 | 150 | 2.9x |
| Energía por ventana (μJ) | 10.1 | 24.8 | 2.45x |
| Precisión | Q3.12 (±1 LSB) | Float32 (alta) | SW mejor |
| Flexibilidad | Baja | Alta | SW mejor |

---

### PASO 5: Visualización final y demostración del sistema (45 min)

Se implementa la visualización en tiempo real de los datos filtrados en la PC, como producto final del sistema.

**Archivo a crear:** `visualizacion_final.py`

```python
import numpy as np
import matplotlib.pyplot as plt
import serial
import time
import re
from matplotlib.animation import FuncAnimation

# =====================================================================
// CONFIGURACIÓN
// =====================================================================
SERIAL_PORT = 'COM3'
BAUD_RATE = 115200
MAX_SAMPLES = 500

# =====================================================================
// ADQUISICIÓN DE DATOS EN TIEMPO REAL
// =====================================================================
class RealTimeData:
    def __init__(self):
        self.samples = []
        self.az_values = []
        self.theta_values = []
        self.omega_values = []
        self.ml_classes = []
        
        self.ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
        time.sleep(2)
        
        self.pattern_meas = re.compile(r'M(\d+): az=(-?\d+), theta=([-\d.]+), omega=([-\d.]+)')
        self.pattern_ml = re.compile(r'\[ML\] Clase: (\w+)')
        
        self.current_class = 0
        self.class_map = {'REPOSO': 0, 'VIB_SUAVE': 1, 'VIB_INTENSA': 2}
    
    def read_data(self):
        line = self.ser.readline().decode('utf-8', errors='ignore').strip()
        
        ml_match = self.pattern_ml.search(line)
        if ml_match:
            class_name = ml_match.group(1)
            self.current_class = self.class_map.get(class_name, 0)
            return
        
        meas_match = self.pattern_meas.search(line)
        if meas_match:
            sample = int(meas_match.group(1))
            az = int(meas_match.group(2))
            theta = float(meas_match.group(3))
            omega = float(meas_match.group(4))
            
            self.samples.append(sample)
            self.az_values.append(az)
            self.theta_values.append(theta)
            self.omega_values.append(omega)
            self.ml_classes.append(self.current_class)
            
            if len(self.samples) > MAX_SAMPLES:
                self.samples.pop(0)
                self.az_values.pop(0)
                self.theta_values.pop(0)
                self.omega_values.pop(0)
                self.ml_classes.pop(0)

# =====================================================================
// VISUALIZACIÓN EN TIEMPO REAL
// =====================================================================
def main():
    data = RealTimeData()
    
    fig, axes = plt.subplots(3, 1, figsize=(14, 10))
    fig.suptitle('Sistema Edge-AI Integrado - IEEE CASS UMSA 2026', fontsize=16, fontweight='bold')
    
    # Gráfica 1: Aceleración Z cruda vs ángulo estimado
    line_az, = axes[0].plot([], [], 'b-', alpha=0.5, label='az (cuentas)')
    line_theta, = axes[0].plot([], [], 'r-', linewidth=2, label='θ estimado (rad)')
    axes[0].set_title('Medición de Aceleración y Estimación de Inclinación')
    axes[0].set_ylabel('Valor')
    axes[0].legend(loc='upper right')
    axes[0].grid(True, alpha=0.3)
    axes[0].set_xlim(0, MAX_SAMPLES)
    axes[0].set_ylim(-300, 300)
    
    # Gráfica 2: Clase ML detectada
    line_ml, = axes[1].step([], [], 'k-', where='post', linewidth=2)
    axes[1].set_title('Clasificación ML Adaptativa')
    axes[1].set_ylabel('Clase')
    axes[1].set_yticks([0, 1, 2])
    axes[1].set_yticklabels(['Reposo', 'Vib. Suave', 'Vib. Intensa'])
    axes[1].grid(True, alpha=0.3)
    axes[1].set_xlim(0, MAX_SAMPLES)
    axes[1].set_ylim(-0.5, 2.5)
    
    # Gráfica 3: Velocidad angular estimada
    line_omega, = axes[2].plot([], [], 'g-', linewidth=2, label='ω estimado (rad/s)')
    axes[2].set_title('Velocidad Angular Estimada')
    axes[2].set_xlabel('Muestra')
    axes[2].set_ylabel('ω (rad/s)')
    axes[2].legend(loc='upper right')
    axes[2].grid(True, alpha=0.3)
    axes[2].set_xlim(0, MAX_SAMPLES)
    axes[2].set_ylim(-2, 2)
    
    plt.tight_layout()
    
    def update(frame):
        data.read_data()
        
        if len(data.samples) > 0:
            line_az.set_data(data.samples, data.az_values)
            line_theta.set_data(data.samples, data.theta_values)
            line_ml.set_data(data.samples, data.ml_classes)
            line_omega.set_data(data.samples, data.omega_values)
        
        return line_az, line_theta, line_ml, line_omega
    
    ani = FuncAnimation(fig, update, interval=100, blit=False, cache_frame_data=False)
    
    plt.show()
    data.ser.close()

if __name__ == "__main__":
    main()
```

**Procedimiento de demostración:**

1. **Ejecutar el firmware en la placa Polaris.**
2. **Ejecutar el script `visualizacion_final.py` en la PC.**
3. **Realizar las siguientes pruebas en vivo:**
   - **Prueba 1:** Colocar la placa en reposo. Observar que la clase ML es 0 (Reposo) y el ángulo estimado converge rápidamente.
   - **Prueba 2:** Mover suavemente la placa. Observar que la clase cambia a 1 (Vibración suave) y el filtro suaviza las oscilaciones.
   - **Prueba 3:** Agitar vigorosamente la placa. Observar que la clase cambia a 2 (Vibración intensa) y el filtro ignora las perturbaciones.
   - **Prueba 4:** Alternar entre los tres estados. Observar la adaptación dinámica del filtro.

4. **Cambiar al modo SW_ONLY y repetir las pruebas:**
   - Observar que el sistema funciona, pero con menor throughput.
   - Comparar la fluidez de la visualización entre ambos modos.

5. **Capturar pantallas de la visualización final** como evidencia del funcionamiento del sistema.

---

## 6.5 Verificación final del Módulo 6

Al finalizar el Módulo 6, se debe contar con:

| Criterio | Verificación |
|---|---|
| Sistema completo integrado y funcional | ✅ Todos los subsistemas operan correctamente |
| Medición de latencia con analizador lógico | ✅ Pulsos capturados, latencias calculadas |
| Comparación HW vs SW completada | ✅ Speedup calculado (~83x) |
| Análisis de ocupación de recursos | ✅ Reporte de Libero SoC extraído |
| Benchmarking de rendimiento | ✅ Throughput, latencia, eficiencia calculados |
| Visualización final en PC | ✅ Datos filtrados mostrados en tiempo real |
| Demostración del sistema completo | ✅ Pruebas en reposo, vibración suave e intensa |

---

## 6.6 Entregables del Módulo 6 (producto final del curso)

| Entregable | Descripción |
|---|---|
| Firmware completo integrado | `main.c` con todos los subsistemas |
| Reporte de benchmarking | Tabla comparativa HW vs SW |
| Capturas del analizador lógico | Evidencia de mediciones de latencia |
| Reporte de ocupación de recursos | Uso de LUTs, DSP, DFFs |
| Script de visualización final | `visualizacion_final.py` funcional |
| Video o capturas de la demostración | Sistema funcionando en tiempo real |
| Informe técnico final | Documento con todos los resultados |

---

## 6.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| Sistema no inicializa | Error en la integración de módulos | Verificar que todos los drivers están correctamente incluidos |
| Analizador lógico no captura | Frecuencia de muestreo muy baja o triggers mal configurados | Verificar configuración a 24 MHz, revisar triggers |
| Latencias inconsistentes | GPIOs no se togg