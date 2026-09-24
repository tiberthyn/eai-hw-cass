# MÓDULO 5: FILTRADO ADAPTATIVO DE KALMAN

## 5.1 Preparación PREVIA al Día 5 (actividades del instructor antes del módulo)

### 5.1.1 Verificación del entorno de desarrollo

El Módulo 5 requiere que el entorno de desarrollo se encuentre plenamente operativo con los entregables de los módulos anteriores:

1. **Proyecto Libero SoC del Módulo 3 disponible:** El proyecto debe contener el MSS configurado con SPI_0 habilitado, el IP personalizado del acelerador ML integrado en el bus APB, y las señales SPI ruteadas a los pines del ADXL345.
2. **Proyecto SoftConsole del Módulo 4 disponible:** El proyecto debe contener el driver del ADXL345 funcional, el UART operativo, y la comunicación con el acelerador HW verificada.
3. **Matriz R del ruido estático disponible:** El archivo `noise_analysis_results.npz` generado en el Módulo 4 debe estar accesible para inicializar el filtro de Kalman.
4. **Pesos del modelo ML disponibles:** Los pesos cuantizados del modelo entrenado en el Módulo 2 deben estar accesibles para cargar en el acelerador HW.
5. **CMSIS-DSP disponible:** La librería CMSIS-DSP debe estar instalada y configurada en el entorno de SoftConsole.

### 5.1.2 Instalación y configuración de CMSIS-DSP

CMSIS-DSP (Cortex Microcontroller Software Interface Standard - Digital Signal Processing) es una librería optimizada de funciones matemáticas para procesadores ARM Cortex-M.

**Procedimiento de instalación:**

1. **Descargar CMSIS:**
   - Acceder al repositorio oficial: [github.com/ARM-software/CMSIS_5](https://github.com/ARM-software/CMSIS_5)
   - Descargar la versión 5.9.0 o superior.
   - Alternativamente, CMSIS-DSP puede estar incluido con SoftConsole en la carpeta de instalación.

2. **Estructura de CMSIS-DSP:**

```
CMSIS_5/
├── CMSIS/
│   ├── Core/          (definiciones del core ARM)
│   ├── DSP/
│   │   ├── Include/   (archivos .h: arm_math.h, etc.)
│   │   └── Source/    (archivos .c organizados por categoría)
│   │       ├── BasicMathFunctions/
│   │       ├── MatrixFunctions/
│   │       ├── FilteringFunctions/
│   │       └── ...
│   └── Include/       (arm_common_tables.h, etc.)
```

3. **Configuración en SoftConsole:**
   - Copiar la carpeta `CMSIS_5/CMSIS/DSP/Include` al proyecto.
   - Agregar la ruta de include en `Project → Properties → C/C++ Build → Settings → Include Paths`.
   - Compilar los archivos fuente de CMSIS-DSP necesarios (solo los utilizados):
     - `MatrixFunctions/`: `arm_mat_init_f32.c`, `arm_mat_mult_f32.c`, `arm_mat_add_f32.c`, `arm_mat_sub_f32.c`, `arm_mat_inverse_f32.c`, `arm_mat_trans_f32.c`
     - `BasicMathFunctions/`: `arm_scale_f32.c`, `arm_dot_prod_f32.c`
     - `CommonTables/`: `arm_common_tables.c`

4. **Verificación:**
   - Crear un archivo de prueba `test_cmsis.c`:

```c
#include "arm_math.h"
#include <stdio.h>

int main(void) {
    arm_matrix_instance_f32 A;
    float32_t pData[4] = {1.0f, 2.0f, 3.0f, 4.0f};
    arm_mat_init_f32(&A, 2, 2, pData);
    
    printf("CMSIS-DSP inicializado correctamente\r\n");
    printf("Matriz A[0][0] = %f\r\n", pData[0]);
    return 0;
}
```

   - Compilar y verificar que no existen errores.

### 5.1.3 Preparación de materiales para estudiantes

Se debe crear una carpeta `Modulo5_Material/` que contenga:
- Template del filtro de Kalman `kalman_filter.h` y `kalman_filter.c`.
- Archivo de configuración `kalman_config.h` con las matrices iniciales.
- Guía paso a paso con capturas de pantalla.
- Script Python `kalman_reference.py` para generar la referencia de comparación.
- Archivo con los pesos del modelo ML cuantizados `ml_weights_q.h`.

### 5.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 5 personalmente antes del curso**, verificando:
- La inicialización correcta del filtro de Kalman con la matriz R del Módulo 4.
- La estimación de inclinación a partir de los datos del ADXL345.
- La adaptación dinámica de las matrices Q y R según la clasificación del modelo ML.
- El envío de datos filtrados por UART para visualización.
- La comparación entre datos crudos y datos filtrados.

---

## 5.2 Conceptos teóricos que el instructor debe dominar y explicar

### 5.2.1 Teoría del Filtro de Kalman (30 min)

El Filtro de Kalman es un algoritmo recursivo óptimo para la estimación de estado en sistemas lineales con ruido gaussiano. Opera en dos fases: **predicción** y **corrección**.

**Modelo del sistema:**

```
Ecuación de estado:    x[k] = A · x[k-1] + B · u[k] + w[k]
Ecuación de medición:  z[k] = H · x[k] + v[k]

Donde:
  x[k] : estado estimado (variable a estimar)
  u[k] : entrada de control (opcional)
  z[k] : medición del sensor
  w[k] : ruido de proceso (covarianza Q)
  v[k] : ruido de medición (covarianza R)
  A    : matriz de transición de estado
  B    : matriz de control
  H    : matriz de observación
```

**Fase de Predicción (Time Update):**

```
x̂⁻[k] = A · x̂[k-1] + B · u[k]        (predicción del estado)
P⁻[k] = A · P[k-1] · Aᵀ + Q           (predicción de la covarianza)
```

**Fase de Corrección (Measurement Update):**

```
K[k]   = P⁻[k] · Hᵀ · (H · P⁻[k] · Hᵀ + R)⁻¹   (ganancia de Kalman)
x̂[k]   = x̂⁻[k] + K[k] · (z[k] - H · x̂⁻[k])    (actualización del estado)
P[k]   = (I - K[k] · H) · P⁻[k]                  (actualización de la covarianza)
```

**Diagrama de bloques del Filtro de Kalman:**

```
                    ┌─────────────────────────────────────────┐
                    │         FILTRO DE KALMAN                │
                    │                                         │
u[k] ──►┌───────┐   │  ┌──────────┐      ┌──────────────┐    │
        │ Modelo│   │  │PREDICCIÓN│      │  CORRECCIÓN  │    │
        │Estado │───┼─►│          │      │              │    │
        └───────┘   │  │ x⁻ = Ax  │      │ K = PHᵀ(HPHᵀ+R)⁻¹│
                    │  │ P⁻ = APAᵀ+Q   │ x = x⁻ + K(z-Hx⁻)  │
        Q[k] ───────┼─►│          │───┬─►│ P = (I-KH)P⁻     │
                    │  └──────────┘   │  └────────┬─────────┘
                    │                 │           │
        R[k] ─────────────────────────┼───────────┤
                    │                 │           │
                    └─────────────────┼───────────┼───────────┘
                                      │           │
                                      │           ▼
                                      │      x̂[k] (estado estimado)
                                      │           │
                                      │           ▼
                                      │      Salida filtrada
                                      │
z[k] ──────────────────────────────────┴──────────┘
(medición del sensor)
```

**Interpretación física:**
- **Q (covarianza de proceso):** Representa la incertidumbre del modelo. Un Q alto indica que el modelo es poco confiable, por lo que el filtro confía más en las mediciones.
- **R (covarianza de medición):** Representa el ruido del sensor. Un R alto indica que las mediciones son ruidosas, por lo que el filtro confía más en el modelo.
- **K (ganancia de Kalman):** Balance dinámico entre el modelo y las mediciones. Si K ≈ 1, confía en las mediciones; si K ≈ 0, confía en el modelo.

### 5.2.2 Estimación de inclinación mediante gravedad (20 min)

Para este proyecto, se estima el ángulo de inclinación de la placa utilizando la componente Z de la aceleración medida por el ADXL345.

**Modelo físico:**

```
En reposo, el acelerómetro mide la gravedad:
  az = g · cos(θ)

Donde:
  θ : ángulo de inclinación respecto al eje vertical
  g : aceleración de la gravedad (≈ 1g = 256 cuentas en ±2g, 10 bits)

Despejando θ:
  θ = arccos(az / g)

Para ángulos pequeños (θ < 15°):
  cos(θ) ≈ 1 - θ²/2
  θ ≈ √(2 · (1 - az/g))  (aproximación)
```

**Modelo de espacio de estado para el filtro:**

```
Estado a estimar: x = [θ, ω]ᵀ
  θ : ángulo de inclinación (rad)
  ω : velocidad angular (rad/s)

Ecuación de estado (modelo de movimiento uniforme):
  θ[k] = θ[k-1] + ω[k-1] · Δt
  ω[k] = ω[k-1]

En forma matricial:
  x[k] = [1  Δt] · x[k-1] + w[k]
         [0   1]

Matriz A = [1  Δt]
           [0   1]

Ecuación de medición (observación del ángulo):
  z[k] = [1  0] · x[k] + v[k]

Matriz H = [1  0]
```

**Cálculo de la medición z[k] a partir del ADXL345:**

```c
// Lectura del ADXL345 (10 bits signed, ±2g)
int16_t az = accel_data.z;  // ≈ 256 en reposo (1g)

// Conversión a unidades de g
float az_g = az / 256.0f;

// Saturación para evitar NaN en arccos
if (az_g > 1.0f) az_g = 1.0f;
if (az_g < -1.0f) az_g = -1.0f;

// Cálculo del ángulo (en radianes)
float z_meas = acosf(az_g);
```

### 5.2.3 Filtrado adaptativo con Machine Learning (25 min)

La innovación de este proyecto consiste en adaptar dinámicamente las matrices Q y R del filtro de Kalman según la clasificación del modelo de ML en línea implementado en el Módulo 2 y acelerado en el Módulo 3.

**Estrategia de adaptación:**

| Clase ML | Estado detectado | Acción sobre R | Acción sobre Q |
|---|---|---|---|
| 0 (Reposo) | Vibración mínima | R bajo (confiar en medición) | Q bajo (confiar en modelo) |
| 1 (Vibración suave) | Vibración moderada | R medio | Q medio |
| 2 (Vibración intensa) | Vibración extrema | R alto (desconfiar de medición) | Q alto (suavizar más) |

**Justificación técnica:**
- En **reposo**, el ruido del sensor es bajo, por lo que las mediciones son confiables. El filtro debe responder rápidamente a cambios reales.
- En **vibración intensa**, las mediciones están contaminadas por vibraciones que no corresponden a cambios reales de inclinación. El filtro debe ignorar estas perturbaciones y confiar más en el modelo predictivo.

**Implementación de la adaptación:**

```c
// Factores de escala para Q y R según la clase ML
const float R_scale[3] = {1.0f, 5.0f, 20.0f};   // Escala de R
const float Q_scale[3] = {1.0f, 2.0f, 5.0f};    // Escala de Q

// Matrices base (calculadas en el análisis de ruido del Módulo 4)
float R_base = 0.001f;   // Varianza del ruido del sensor (ejemplo)
float Q_base = 0.0001f;  // Varianza del ruido de proceso (ejemplo)

// Actualización adaptativa
void kalman_adapt_matrices(ml_class_t ml_class) {
    R_current = R_base * R_scale[ml_class];
    Q_current = Q_base * Q_scale[ml_class];
}
```

**Diagrama de co-diseño HW/SW:**

```
┌──────────────────────────────────────────────────────────────┐
│                    SISTEMA ADAPTATIVO                        │
│                                                              │
│   ┌──────────┐    ┌─────────────────┐    ┌──────────────┐  │
│   │ ADXL345  │───►│ ACCELERADOR HW  │───►│ ARM Cortex-M3│  │
│   │ (SPI)    │    │ (Módulo 3)      │    │              │  │
│   │          │    │                 │    │ 1. Clasifica  │  │
│   │ az, ay,  │    │ Características │    │    con ML     │  │
│   │ ax       │    │ (var, rms, etc) │    │              │  │
│   └──────────┘    └─────────────────┘    │ 2. Adapta   │  │
│                                          │    Q y R      │  │
│   ┌──────────┐                           │              │  │
│   │  UART    │◄──────────────────────────│ 3. Ejecuta   │  │
│   │  → PC    │                           │    Kalman     │  │
│   │          │                           │              │  │
│   │ Datos    │                           │ 4. Envía     │  │
│   │filtrados │                           │    resultado  │  │
│   └──────────┘                           └──────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.2.4 CMSIS-DSP para operaciones matriciales (15 min)

CMSIS-DSP proporciona funciones optimizadas para operaciones matriciales en ARM Cortex-M. Las funciones relevantes para el Filtro de Kalman son:

| Función | Descripción | Uso en Kalman |
|---|---|---|
| `arm_mat_init_f32` | Inicializa una instancia de matriz | Todas las matrices |
| `arm_mat_mult_f32` | Multiplicación de matrices | A·P·Aᵀ, H·P·Hᵀ, P·Hᵀ |
| `arm_mat_add_f32` | Suma de matrices | A·P·Aᵀ + Q |
| `arm_mat_sub_f32` | Resta de matrices | I - K·H |
| `arm_mat_inverse_f32` | Inversión de matriz | (H·P·Hᵀ + R)⁻¹ |
| `arm_mat_trans_f32` | Transposición de matriz | Aᵀ, Hᵀ |
| `arm_mat_scale_f32` | Escalado de matriz | K·(z - H·x⁻) |

**Estructura de datos de CMSIS-DSP:**

```c
typedef struct {
    uint16_t numRows;     // Número de filas
    uint16_t numCols;     // Número de columnas
    float32_t *pData;     // Puntero a los datos (array en orden fila)
} arm_matrix_instance_f32;
```

**Ejemplo de uso:**

```c
#include "arm_math.h"

// Declarar matrices
float32_t A_data[4] = {1.0f, 0.01f, 0.0f, 1.0f};  // 2x2
float32_t x_data[2] = {0.0f, 0.0f};                // 2x1
arm_matrix_instance_f32 A, x;

// Inicializar
arm_mat_init_f32(&A, 2, 2, A_data);
arm_mat_init_f32(&x, 2, 1, x_data);

// Multiplicar: result = A * x
float32_t result_data[2];
arm_matrix_instance_f32 result;
arm_mat_init_f32(&result, 2, 1, result_data);
arm_mat_mult_f32(&A, &x, &result);
```

**Consideraciones de rendimiento:**
- Las funciones de CMSIS-DSP están optimizadas con instrucciones SIMD y FPU (si está disponible).
- El SmartFusion2 M2S005 tiene un Cortex-M3 sin FPU, por lo que las operaciones en punto flotante se realizan en software.
- Para mayor eficiencia, se pueden usar versiones en punto fijo (`arm_mat_mult_q31`), pero para este curso se utiliza la versión en punto flotante por simplicidad.

---

## 5.3 Herramientas y configuración requeridas

| Herramienta | Configuración específica | Verificación |
|---|---|---|
| Libero SoC v12+ | Proyecto del Módulo 3 abierto | MSS con SPI_0 + IP personalizado |
| SoftConsole IDE | Proyecto del Módulo 4 abierto | Driver ADXL345 + UART funcional |
| CMSIS-DSP | Instalado y configurado | Compila sin errores |
| Matriz R del Módulo 4 | Archivo `noise_analysis_results.npz` | Valores de varianza disponibles |
| Pesos del modelo ML | Archivo `ml_weights_q.h` | Pesos cuantizados disponibles |
| Terminal serial | 115200 baud, 8N1 | Comunicación UART funcional |

---

## 5.4 Implementación paso a paso

### PASO 1: Crear la estructura del filtro de Kalman (45 min)

Se implementa el filtro de Kalman en C utilizando CMSIS-DSP para las operaciones matriciales.

**Archivo a crear:** `kalman_config.h`

```c
#ifndef KALMAN_CONFIG_H
#define KALMAN_CONFIG_H

// =====================================================================
// PARÁMETROS DEL FILTRO DE KALMAN
// =====================================================================

// Dimensión del estado (θ, ω)
#define KALMAN_STATE_DIM     2

// Dimensión de la medición (θ medido)
#define KALMAN_MEAS_DIM      1

// Período de muestreo (segundos)
#define KALMAN_DT            0.01f   // 100 Hz (coincide con ADXL345)

// Matriz de transición de estado A = [1 DT; 0 1]
#define KALMAN_A_DATA        {1.0f, KALMAN_DT, 0.0f, 1.0f}

// Matriz de observación H = [1 0]
#define KALMAN_H_DATA        {1.0f, 0.0f}

// Covarianza inicial del estado P[0]
#define KALMAN_P0_DATA       {0.1f, 0.0f, 0.0f, 0.1f}

// Covarianza base del ruido de proceso Q (se adapta con ML)
#define KALMAN_Q_BASE        0.0001f

// Covarianza base del ruido de medición R (se adapta con ML)
// Este valor se obtiene del análisis de ruido del Módulo 4
#define KALMAN_R_BASE        0.001f

// Factores de escala para adaptación según clase ML
#define KALMAN_R_SCALE_0     1.0f    // Reposo: confiar en medición
#define KALMAN_R_SCALE_1     5.0f    // Vibración suave
#define KALMAN_R_SCALE_2     20.0f   // Vibración intensa: desconfiar

#define KALMAN_Q_SCALE_0     1.0f    // Reposo
#define KALMAN_Q_SCALE_1     2.0f    // Vibración suave
#define KALMAN_Q_SCALE_2     5.0f    // Vibración intensa

// Conversión ADXL345 a unidades de g (10 bits, ±2g)
#define ADXL_SCALE_TO_G      (1.0f / 256.0f)

#endif // KALMAN_CONFIG_H
```

**Archivo a crear:** `kalman_filter.h`

```c
#ifndef KALMAN_FILTER_H
#define KALMAN_FILTER_H

#include <stdint.h>
#include "arm_math.h"
#include "kalman_config.h"

// =====================================================================
// TIPO DE CLASE ML (del Módulo 2)
// =====================================================================
typedef enum {
    ML_CLASS_REST = 0,       // Estado 0: Reposo
    ML_CLASS_MILD_VIB = 1,   // Estado 1: Vibración suave
    ML_CLASS_INTENSE_VIB = 2 // Estado 2: Vibración intensa
} ml_class_t;

// =====================================================================
// ESTRUCTURA DEL FILTRO DE KALMAN
// =====================================================================
typedef struct {
    // Estado estimado x = [θ, ω]ᵀ
    arm_matrix_instance_f32 x;
    float32_t x_data[KALMAN_STATE_DIM];
    
    // Covarianza del estado P (2x2)
    arm_matrix_instance_f32 P;
    float32_t P_data[KALMAN_STATE_DIM * KALMAN_STATE_DIM];
    
    // Matrices del sistema
    arm_matrix_instance_f32 A;  // Transición (2x2)
    float32_t A_data[4];
    
    arm_matrix_instance_f32 H;  // Observación (1x2)
    float32_t H_data[2];
    
    // Covarianzas adaptativas
    arm_matrix_instance_f32 Q;  // Proceso (2x2)
    float32_t Q_data[4];
    
    arm_matrix_instance_f32 R;  // Medición (1x1)
    float32_t R_data[1];
    
    // Matrices temporales para cálculos
    float32_t temp_data[4];     // 2x2
    arm_matrix_instance_f32 temp;
    
    float32_t temp2_data[4];    // 2x2
    arm_matrix_instance_f32 temp2;
    
    float32_t temp3_data[2];    // 2x1
    arm_matrix_instance_f32 temp3;
    
    float32_t temp4_data[2];    // 2x1
    arm_matrix_instance_f32 temp4;
    
    float32_t S_data[1];        // 1x1 (escalar)
    arm_matrix_instance_f32 S;
    
    float32_t K_data[2];        // 2x1 (ganancia de Kalman)
    arm_matrix_instance_f32 K;
    
    // Estado actual
    ml_class_t current_class;
    float32_t current_R;
    float32_t current_Q;
    
    // Inicializado
    uint8_t initialized;
} kalman_filter_t;

// =====================================================================
// FUNCIONES PÚBLICAS
// =====================================================================

/**
 * @brief Inicializa el filtro de Kalman
 * @param kf Puntero a la estructura del filtro
 * @param R_base Covarianza base de medición (del análisis de ruido)
 */
void kalman_init(kalman_filter_t *kf, float32_t R_base);

/**
 * @brief Actualiza las matrices Q y R según la clase ML
 * @param kf Puntero a la estructura del filtro
 * @param ml_class Clase detectada por el modelo ML
 */
void kalman_adapt(kalman_filter_t *kf, ml_class_t ml_class);

/**
 * @brief Ejecuta un ciclo del filtro de Kalman
 * @param kf Puntero a la estructura del filtro
 * @param z Medición del ángulo (en radianes)
 * @return Ángulo estimado θ (en radianes)
 */
float32_t kalman_update(kalman_filter_t *kf, float32_t z);

/**
 * @brief Obtiene el estado estimado completo
 * @param kf Puntero a la estructura del filtro
 * @param theta Puntero para almacenar el ángulo estimado
 * @param omega Puntero para almacenar la velocidad angular estimada
 */
void kalman_get_state(kalman_filter_t *kf, float32_t *theta, float32_t *omega);

/**
 * @brief Obtiene la ganancia de Kalman actual (indicador de confianza)
 * @param kf Puntero a la estructura del filtro
 * @return Ganancia K[0] (entre 0 y 1)
 */
float32_t kalman_get_gain(kalman_filter_t *kf);

#endif // KALMAN_FILTER_H
```

**Archivo a crear:** `kalman_filter.c`

```c
#include "kalman_filter.h"
#include <math.h>

// =====================================================================
// INICIALIZACIÓN
// =====================================================================
void kalman_init(kalman_filter_t *kf, float32_t R_base) {
    // Inicializar estado x = [0, 0]ᵀ
    kf->x_data[0] = 0.0f;
    kf->x_data[1] = 0.0f;
    arm_mat_init_f32(&kf->x, KALMAN_STATE_DIM, 1, kf->x_data);
    
    // Inicializar covarianza P
    float32_t P0[KALMAN_STATE_DIM * KALMAN_STATE_DIM] = KALMAN_P0_DATA;
    for (int i = 0; i < 4; i++) kf->P_data[i] = P0[i];
    arm_mat_init_f32(&kf->P, KALMAN_STATE_DIM, KALMAN_STATE_DIM, kf->P_data);
    
    // Inicializar matriz A
    float32_t A_init[KALMAN_STATE_DIM * KALMAN_STATE_DIM] = KALMAN_A_DATA;
    for (int i = 0; i < 4; i++) kf->A_data[i] = A_init[i];
    arm_mat_init_f32(&kf->A, KALMAN_STATE_DIM, KALMAN_STATE_DIM, kf->A_data);
    
    // Inicializar matriz H
    float32_t H_init[KALMAN_MEAS_DIM * KALMAN_STATE_DIM] = KALMAN_H_DATA;
    for (int i = 0; i < 2; i++) kf->H_data[i] = H_init[i];
    arm_mat_init_f32(&kf->H, KALMAN_MEAS_DIM, KALMAN_STATE_DIM, kf->H_data);
    
    // Inicializar Q y R con valores base
    kf->current_Q = KALMAN_Q_BASE;
    kf->current_R = R_base;
    
    kf->Q_data[0] = kf->current_Q; kf->Q_data[1] = 0.0f;
    kf->Q_data[2] = 0.0f;          kf->Q_data[3] = kf->current_Q;
    arm_mat_init_f32(&kf->Q, KALMAN_STATE_DIM, KALMAN_STATE_DIM, kf->Q_data);
    
    kf->R_data[0] = kf->current_R;
    arm_mat_init_f32(&kf->R, KALMAN_MEAS_DIM, KALMAN_MEAS_DIM, kf->R_data);
    
    // Inicializar matrices temporales
    arm_mat_init_f32(&kf->temp, 2, 2, kf->temp_data);
    arm_mat_init_f32(&kf->temp2, 2, 2, kf->temp2_data);
    arm_mat_init_f32(&kf->temp3, 2, 1, kf->temp3_data);
    arm_mat_init_f32(&kf->temp4, 2, 1, kf->temp4_data);
    arm_mat_init_f32(&kf->S, 1, 1, kf->S_data);
    arm_mat_init_f32(&kf->K, KALMAN_STATE_DIM, KALMAN_MEAS_DIM, kf->K_data);
    
    kf->current_class = ML_CLASS_REST;
    kf->initialized = 1;
}

// =====================================================================
// ADAPTACIÓN DE MATRICES Q Y R
// =====================================================================
void kalman_adapt(kalman_filter_t *kf, ml_class_t ml_class) {
    if (!kf->initialized) return;
    
    kf->current_class = ml_class;
    
    // Seleccionar factores de escala según la clase ML
    float32_t R_scale, Q_scale;
    switch (ml_class) {
        case ML_CLASS_REST:
            R_scale = KALMAN_R_SCALE_0;
            Q_scale = KALMAN_Q_SCALE_0;
            break;
        case ML_CLASS_MILD_VIB:
            R_scale = KALMAN_R_SCALE_1;
            Q_scale = KALMAN_Q_SCALE_1;
            break;
        case ML_CLASS_INTENSE_VIB:
            R_scale = KALMAN_R_SCALE_2;
            Q_scale = KALMAN_Q_SCALE_2;
            break;
        default:
            R_scale = 1.0f;
            Q_scale = 1.0f;
    }
    
    // Actualizar R
    kf->current_R = KALMAN_R_BASE * R_scale;
    kf->R_data[0] = kf->current_R;
    
    // Actualizar Q (matriz diagonal)
    kf->current_Q = KALMAN_Q_BASE * Q_scale;
    kf->Q_data[0] = kf->current_Q;
    kf->Q_data[1] = 0.0f;
    kf->Q_data[2] = 0.0f;
    kf->Q_data[3] = kf->current_Q;
}

// =====================================================================
// ACTUALIZACIÓN DEL FILTRO (PREDICCIÓN + CORRECCIÓN)
// =====================================================================
float32_t kalman_update(kalman_filter_t *kf, float32_t z) {
    if (!kf->initialized) return 0.0f;
    
    // =================================================================
    // FASE DE PREDICCIÓN
    // =================================================================
    
    // 1. x⁻ = A · x
    arm_mat_mult_f32(&kf->A, &kf->x, &kf->temp3);
    for (int i = 0; i < 2; i++) kf->x_data[i] = kf->temp3_data[i];
    
    // 2. P⁻ = A · P · Aᵀ + Q
    //    temp = A · P
    arm_mat_mult_f32(&kf->A, &kf->P, &kf->temp);
    
    //    temp2 = temp · Aᵀ
    arm_matrix_instance_f32 AT;
    float32_t AT_data[4];
    arm_mat_init_f32(&AT, 2, 2, AT_data);
    arm_mat_trans_f32(&kf->A, &AT);
    arm_mat_mult_f32(&kf->temp, &AT, &kf->temp2);
    
    //    P = temp2 + Q
    arm_mat_add_f32(&kf->temp2, &kf->Q, &kf->P);
    
    // =================================================================
    // FASE DE CORRECCIÓN
    // =================================================================
    
    // 3. S = H · P · Hᵀ + R  (escalar 1x1)
    //    temp3 = P · Hᵀ  (2x1)
    arm_matrix_instance_f32 HT;
    float32_t HT_data[2];
    arm_mat_init_f32(&HT, KALMAN_STATE_DIM, KALMAN_MEAS_DIM, HT_data);
    arm_mat_trans_f32(&kf->H, &HT);
    arm_mat_mult_f32(&kf->P, &HT, &kf->temp3);
    
    //    S = H · temp3 + R  (1x1)
    float32_t HPT_data[1];
    arm_matrix_instance_f32 HPT;
    arm_mat_init_f32(&HPT, 1, 1, HPT_data);
    arm_mat_mult_f32(&kf->H, &kf->temp3, &HPT);
    kf->S_data[0] = HPT_data[0] + kf->R_data[0];
    
    // 4. K = P · Hᵀ · S⁻¹  (2x1)
    float32_t S_inv = 1.0f / kf->S_data[0];
    kf->K_data[0] = kf->temp3_data[0] * S_inv;
    kf->K_data[1] = kf->temp3_data[1] * S_inv;
    
    // 5. x = x⁻ + K · (z - H · x⁻)
    //    innovation = z - H · x
    float32_t Hx = kf->H_data[0] * kf->x_data[0] + kf->H_data[1] * kf->x_data[1];
    float32_t innovation = z - Hx;
    
    kf->x_data[0] = kf->x_data[0] + kf->K_data[0] * innovation;
    kf->x_data[1] = kf->x_data[1] + kf->K_data[1] * innovation;
    
    // 6. P = (I - K · H) · P
    //    temp = K · H  (2x2)
    kf->temp_data[0] = 1.0f - kf->K_data[0] * kf->H_data[0];
    kf->temp_data[1] = 0.0f - kf->K_data[0] * kf->H_data[1];
    kf->temp_data[2] = 0.0f - kf->K_data[1] * kf->H_data[0];
    kf->temp_data[3] = 1.0f - kf->K_data[1] * kf->H_data[1];
    
    //    P = temp · P
    arm_mat_mult_f32(&kf->temp, &kf->P, &kf->temp2);
    for (int i = 0; i < 4; i++) kf->P_data[i] = kf->temp2_data[i];
    
    return kf->x_data[0];  // Retornar θ estimado
}

// =====================================================================
// FUNCIONES AUXILIARES
// =====================================================================
void kalman_get_state(kalman_filter_t *kf, float32_t *theta, float32_t *omega) {
    if (!kf->initialized) {
        *theta = 0.0f;
        *omega = 0.0f;
        return;
    }
    *theta = kf->x_data[0];
    *omega = kf->x_data[1];
}

float32_t kalman_get_gain(kalman_filter_t *kf) {
    if (!kf->initialized) return 0.0f;
    return kf->K_data[0];
}
```

**Resultado esperado:** El filtro de Kalman se inicializa correctamente, puede adaptarse según la clase ML, y ejecuta los ciclos de predicción y corrección.

**Verificación:**
- Compilar el código sin errores ni warnings.
- Verificar que las funciones de CMSIS-DSP se enlazan correctamente.
- Verificar que no existen violaciones de memoria (los arrays temporales tienen el tamaño correcto).

---

### PASO 2: Crear el módulo de clasificación ML en el firmware (30 min)

Se implementa la inferencia del modelo ML en el firmware del ARM, utilizando los pesos cargados desde el acelerador HW.

**Archivo a crear:** `ml_weights_q.h`

```c
#ifndef ML_WEIGHTS_Q_H
#define ML_WEIGHTS_Q_H

#include <stdint.h>

// =====================================================================
// PESOS DEL MODELO ML CUANTIZADOS (del Módulo 2)
// Formato: Q2.13 (16 bits signed)
// =====================================================================

// Número de características
#define ML_N_FEATURES  6

// Número de clases
#define ML_N_CLASSES   3

// Pesos del modelo (3 clases × 6 características)
// Estos valores deben obtenerse del script Python del Módulo 2
static const int16_t ml_weights_q[ML_N_CLASSES][ML_N_FEATURES] = {
    // Clase 0: Reposo
    { 1200, -800,  600,  400, -200,  300},
    // Clase 1: Vibración suave
    {-500,  1500, -900, -300,  700, -400},
    // Clase 2: Vibración intensa
    {-700, -700,  300, -100, -500,  100}
};

// Bias del modelo (3 clases)
static const int16_t ml_bias_q[ML_N_CLASSES] = {500, -300, -200};

// Media y desviación estándar para normalización (Q3.12)
static const int16_t ml_mean_q[ML_N_FEATURES] = {4100, 120, 4200, 50, 80, 4150};
static const int16_t ml_std_q[ML_N_FEATURES]  = {1800, 350, 1900, 200, 280, 1850};

#endif // ML_WEIGHTS_Q_H
```

**Archivo a crear:** `ml_classifier.h`

```c
#ifndef ML_CLASSIFIER_H
#define ML_CLASSIFIER_H

#include <stdint.h>
#include "ml_weights_q.h"
#include "kalman_filter.h"

/**
 * @brief Clasifica una muestra de características usando el modelo ML
 * @param features Array de 6 características (formato Q3.12)
 * @return Clase ML detectada (0, 1, o 2)
 */
ml_class_t ml_classify(const int32_t features[ML_N_FEATURES]);

/**
 * @brief Convierte una clase ML a cadena de texto (para UART)
 * @param ml_class Clase ML
 * @return Cadena de texto descriptiva
 */
const char* ml_class_to_string(ml_class_t ml_class);

#endif // ML_CLASSIFIER_H
```

**Archivo a crear:** `ml_classifier.c`

```c
#include "ml_classifier.h"

// =====================================================================
// APROXIMACIÓN DE SIGMOIDE EN PUNTO FIJO
// =====================================================================
static int32_t sigmoid_approx_q(int32_t z_q) {
    // z_q está en formato Q3.14 (18 bits signed)
    // Convertir a flotante para la aproximación
    float z = z_q / 16384.0f;
    
    if (z < -4.0f) return 0;
    if (z > 4.0f) return 32767;  // ≈ 1.0 en Q0.15
    
    // Aproximación lineal: 0.5 + z/8
    float result = 0.5f + z / 8.0f;
    return (int32_t)(result * 32767.0f);
}

// =====================================================================
// CLASIFICACIÓN ML
// =====================================================================
ml_class_t ml_classify(const int32_t features[ML_N_FEATURES]) {
    int32_t scores[ML_N_CLASSES];
    int32_t max_score = -2147483648;
    ml_class_t max_class = ML_CLASS_REST;
    
    // Para cada clase, calcular el score = wᵀx + b
    for (int c = 0; c < ML_N_CLASSES; c++) {
        int64_t acc = 0;
        
        for (int i = 0; i < ML_N_FEATURES; i++) {
            // Normalizar característica: (feat - mean) / std
            // En punto fijo: ((feat - mean) * 32768) / std
            int32_t normalized = ((features[i] - ml_mean_q[i]) * 32768) / ml_std_q[i];
            
            // Producto: weight * normalized
            // Q2.13 × Q3.12 = Q5.25
            int64_t product = (int64_t)ml_weights_q[c][i] * (int64_t)normalized;
            acc += product;
        }
        
        // Ajustar formato: Q5.25 → Q3.14 (shift right 11)
        int32_t z_q = (int32_t)(acc >> 11) + ml_bias_q[c];
        
        // Aplicar sigmoide
        scores[c] = sigmoid_approx_q(z_q);
        
        // Buscar la clase con mayor score
        if (scores[c] > max_score) {
            max_score = scores[c];
            max_class = (ml_class_t)c;
        }
    }
    
    return max_class;
}

// =====================================================================
// UTILIDADES
// =====================================================================
const char* ml_class_to_string(ml_class_t ml_class) {
    switch (ml_class) {
        case ML_CLASS_REST:        return "REPOSO";
        case ML_CLASS_MILD_VIB:    return "VIB_SUAVE";
        case ML_CLASS_INTENSE_VIB: return "VIB_INTENSA";
        default:                   return "DESCONOCIDO";
    }
}
```

---

### PASO 3: Integrar todo en el firmware principal (60 min)

Se modifica el `main.c` del Módulo 4 para integrar el filtro de Kalman adaptativo.

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

// =====================================================================
// DIRECCIONES DEL IP PERSONALIZADO (acelerador ML)
// =====================================================================
#define ACCEL_BASE_ADDR        0x40000000
#define ACCEL_REG_CONTROL      (ACCEL_BASE_ADDR + 0x00)
#define ACCEL_REG_STATUS       (ACCEL_BASE_ADDR + 0x04)
#define ACCEL_REG_DATA_IN      (ACCEL_BASE_ADDR + 0x08)
#define ACCEL_REG_FEAT_0       (ACCEL_BASE_ADDR + 0x0C)
#define ACCEL_REG_SCORE        (ACCEL_BASE_ADDR + 0x6C)

// =====================================================================
// VARIABLES GLOBALES
// =====================================================================
mss_uart_instance_t *const g_uart = &g_mss_uart0;
adxl345_data_t accel_data;
kalman_filter_t kf;

// Buffer circular para ventana de características (50 muestras)
#define WINDOW_SIZE 50
static int32_t window_buffer[WINDOW_SIZE][3];  // [ax, ay, az]
static uint32_t window_idx = 0;
static uint32_t window_count = 0;

// =====================================================================
// FUNCIONES AUXILIARES
// =====================================================================

/**
 * @brief Envía una muestra al acelerador HW
 */
void send_to_accelerator(adxl345_data_t *data) {
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
}

/**
 * @brief Lee las características del acelerador HW
 */
void read_accelerator_features(int32_t features[6]) {
    volatile uint32_t *reg_feat = (volatile uint32_t *)ACCEL_REG_FEAT_0;
    for (int i = 0; i < 6; i++) {
        features[i] = (int32_t)reg_feat[i];
    }
}

/**
 * @brief Calcula el ángulo de inclinación a partir de az
 */
float32_t compute_tilt_angle(int16_t az) {
    float az_g = az * ADXL_SCALE_TO_G;
    
    // Saturación
    if (az_g > 1.0f) az_g = 1.0f;
    if (az_g < -1.0f) az_g = -1.0f;
    
    return acosf(az_g);  // Ángulo en radianes
}

/**
 * @brief Agrega una muestra a la ventana deslizante
 */
void add_to_window(adxl345_data_t *data) {
    window_buffer[window_idx][0] = data->x;
    window_buffer[window_idx][1] = data->y;
    window_buffer[window_idx][2] = data->z;
    window_idx = (window_idx + 1) % WINDOW_SIZE;
    if (window_count < WINDOW_SIZE) window_count++;
}

// =====================================================================
// FUNCIÓN PRINCIPAL
// =====================================================================
int main(void) {
    // Inicializar UART
    MSS_UART_init(g_uart, MSS_UART_115200_BAUD,
                  MSS_UART_DATA_8_BITS | MSS_UART_NO_PARITY | MSS_UART_ONE_STOP_BIT);
    
    MSS_UART_polled_tx_string(g_uart, 
        (const uint8_t *)"\r\n[Modulo 5] Filtro de Kalman Adaptativo - IEEE CASS UMSA 2026\r\n");
    
    // Inicializar ADXL345
    if (adxl345_init() != 0) {
        MSS_UART_polled_tx_string(g_uart, 
            (const uint8_t *)"ERROR: ADXL345 no responde\r\n");
        while (1);
    }
    MSS_UART_polled_tx_string(g_uart, 
        (const uint8_t *)"ADXL345 inicializado\r\n");
    
    // Inicializar filtro de Kalman
    // R_base obtenido del análisis de ruido del Módulo 4
    // Ejemplo: varianza típica ≈ 9 (desv.std ≈ 3 cuentas)
    // En unidades de g: (3/256)² ≈ 0.000137
    float32_t R_base = 0.000137f;
    kalman_init(&kf, R_base);
    MSS_UART_polled_tx_string(g_uart, 
        (const uint8_t *)"Filtro de Kalman inicializado\r\n");
    
    // Bucle principal
    uint32_t sample_count = 0;
    while (1) {
        // Leer datos del acelerómetro
        if (adxl345_read_accel(&accel_data) == 0) {
            sample_count++;
            
            // 1. Enviar datos al acelerador HW
            send_to_accelerator(&accel_data);
            
            // 2. Agregar a la ventana deslizante
            add_to_window(&accel_data);
            
            // 3. Cada WINDOW_SIZE muestras, calcular características y clasificar
            if (window_count >= WINDOW_SIZE && (sample_count % WINDOW_SIZE == 0)) {
                // Leer características del acelerador HW
                int32_t features[6];
                read_accelerator_features(features);
                
                // Clasificar con el modelo ML
                ml_class_t ml_class = ml_classify(features);
                
                // Adaptar matrices Q y R del Kalman
                kalman_adapt(&kf, ml_class);
                
                // Enviar información de clasificación por UART
                char buffer[128];
                snprintf(buffer, sizeof(buffer),
                         "[ML] Clase: %s\r\n", ml_class_to_string(ml_class));
                MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
            }
            
            // 4. Calcular ángulo de inclinación (medición)
            float32_t z_meas = compute_tilt_angle(accel_data.z);
            
            // 5. Ejecutar ciclo del filtro de Kalman
            float32_t theta_est = kalman_update(&kf, z_meas);
            
            // 6. Obtener estado completo
            float32_t theta, omega;
            kalman_get_state(&kf, &theta, &omega);
            float32_t gain = kalman_get_gain(&kf);
            
            // 7. Enviar datos filtrados por UART cada 10 muestras
            if (sample_count % 10 == 0) {
                char buffer[256];
                snprintf(buffer, sizeof(buffer),
                         "M%lu: az=%d, z_meas=%.4f, theta=%.4f, omega=%.4f, K=%.4f, R=%.6f\r\n",
                         sample_count, accel_data.z, z_meas, theta, omega, gain, kf.current_R);
                MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
            }
        }
        
        // Delay para mantener frecuencia de muestreo ≈ 100 Hz
        for (volatile int i = 0; i < 5000; i++);
    }
    
    return 0;
}
```

**Compilar y programar:**
1. Compilar el proyecto en SoftConsole (`Project → Build All`).
2. Verificar que no existen errores de compilación ni enlazado.
3. Programar el ARM Cortex-M3.
4. Abrir el terminal serial a 115200 baud.

**Resultado esperado:**
- El sistema inicializa correctamente el ADXL345 y el filtro de Kalman.
- Cada 50 muestras, se clasifica el estado del sistema (reposo, vibración suave, vibración intensa).
- Las matrices Q y R se adaptan según la clasificación.
- Cada 10 muestras, se envían por UART los datos crudos y filtrados.
- El ángulo estimado θ es más suave que la medición cruda z_meas.

**Verificación:**
- En reposo: el filtro converge rápidamente a un valor estable.
- Bajo vibración suave: el filtro suaviza las oscilaciones.
- Bajo vibración intensa: el filtro ignora las perturbaciones y mantiene la estimación previa.

---

### PASO 4: Generar referencia de comparación en Python (30 min)

Se crea un script Python que replica el filtro de Kalman para comparar con la implementación en el ARM.

**Archivo a crear:** `kalman_reference.py`

```python
import numpy as np
import matplotlib.pyplot as plt
import serial
import time
import re

# =====================================================================
// CONFIGURACIÓN
// =====================================================================
SERIAL_PORT = 'COM3'  # Ajustar según el puerto COM
BAUD_RATE = 115200
NUM_SAMPLES = 500

# Parámetros del filtro (deben coincidir con kalman_config.h)
DT = 0.01
A = np.array([[1, DT], [0, 1]])
H = np.array([[1, 0]])
Q_base = 0.0001
R_base = 0.000137

# Factores de escala
R_scale = {0: 1.0, 1: 5.0, 2: 20.0}
Q_scale = {0: 1.0, 1: 2.0, 2: 5.0}

# =====================================================================
// FILTRO DE KALMAN EN PYTHON (referencia)
// =====================================================================
class KalmanFilter:
    def __init__(self, R_base):
        self.x = np.zeros((2, 1))
        self.P = np.eye(2) * 0.1
        self.A = A
        self.H = H
        self.Q = np.eye(2) * Q_base
        self.R = np.array([[R_base]])
        self.current_class = 0
    
    def adapt(self, ml_class):
        self.current_class = ml_class
        self.R = np.array([[R_base * R_scale[ml_class]]])
        self.Q = np.eye(2) * (Q_base * Q_scale[ml_class])
    
    def update(self, z):
        # Predicción
        self.x = self.A @ self.x
        self.P = self.A @ self.P @ self.A.T + self.Q
        
        # Corrección
        S = self.H @ self.P @ self.H.T + self.R
        K = self.P @ self.H.T @ np.linalg.inv(S)
        y = z - self.H @ self.x
        self.x = self.x + K @ y
        self.P = (np.eye(2) - K @ self.H) @ self.P
        
        return self.x[0, 0]

# =====================================================================
// ADQUISICIÓN DE DATOS DESDE UART
// =====================================================================
def acquire_data():
    ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
    time.sleep(2)
    
    data = []
    ml_classes = []
    
    print(f"Adquiriendo {NUM_SAMPLES} muestras...")
    
    pattern_meas = re.compile(r'M(\d+): az=(-?\d+), z_meas=([-\d.]+), theta=([-\d.]+), omega=([-\d.]+), K=([-\d.]+), R=([-\d.]+)')
    pattern_ml = re.compile(r'\[ML\] Clase: (\w+)')
    
    current_class = 0
    class_map = {'REPOSO': 0, 'VIB_SUAVE': 1, 'VIB_INTENSA': 2}
    
    while len(data) < NUM_SAMPLES:
        line = ser.readline().decode('utf-8', errors='ignore').strip()
        
        ml_match = pattern_ml.search(line)
        if ml_match:
            class_name = ml_match.group(1)
            current_class = class_map.get(class_name, 0)
            continue
        
        meas_match = pattern_meas.search(line)
        if meas_match:
            sample = int(meas_match.group(1))
            az = int(meas_match.group(2))
            z_meas = float(meas_match.group(3))
            theta_hw = float(meas_match.group(4))
            omega_hw = float(meas_match.group(5))
            K_hw = float(meas_match.group(6))
            R_hw = float(meas_match.group(7))
            
            data.append({
                'sample': sample,
                'az': az,
                'z_meas': z_meas,
                'theta_hw': theta_hw,
                'omega_hw': omega_hw,
                'K_hw': K_hw,
                'R_hw': R_hw,
                'ml_class': current_class
            })
            
            if len(data) % 50 == 0:
                print(f"  {len(data)} muestras adquiridas")
    
    ser.close()
    return data

# =====================================================================
// COMPARACIÓN: HARDWARE vs PYTHON
// =====================================================================
def compare_implementations(data):
    kf_ref = KalmanFilter(R_base)
    
    errors_theta = []
    errors_omega = []
    
    for d in data:
        # Adaptar según la clase ML detectada en HW
        kf_ref.adapt(d['ml_class'])
        
        # Ejecutar filtro de referencia
        theta_ref = kf_ref.update(d['z_meas'])
        omega_ref = kf_ref.x[1, 0]
        
        # Calcular error
        err_theta = abs(d['theta_hw'] - theta_ref)
        err_omega = abs(d['omega_hw'] - omega_ref)
        errors_theta.append(err_theta)
        errors_omega.append(err_omega)
        
        # Almacenar referencia
        d['theta_ref'] = theta_ref
        d['omega_ref'] = omega_ref
    
    print("\n=== Comparación HW vs Python ===")
    print(f"Error medio en θ: {np.mean(errors_theta):.6f} rad")
    print(f"Error máximo en θ: {np.max(errors_theta):.6f} rad")
    print(f"Error medio en ω: {np.mean(errors_omega):.6f} rad/s")
    print(f"Error máximo en ω: {np.max(errors_omega):.6f} rad/s")
    
    return errors_theta, errors_omega

# =====================================================================
// VISUALIZACIÓN
// =====================================================================
def plot_results(data, errors_theta):
    samples = [d['sample'] for d in data]
    z_meas = [d['z_meas'] for d in data]
    theta_hw = [d['theta_hw'] for d in data]
    theta_ref = [d['theta_ref'] for d in data]
    ml_classes = [d['ml_class'] for d in data]
    R_values = [d['R_hw'] for d in data]
    K_values = [d['K_hw'] for d in data]
    
    fig, axes = plt.subplots(4, 1, figsize=(14, 12))
    
    # 1. Medición vs Estimación
    axes[0].plot(samples, z_meas, 'b-', alpha=0.5, label='Medición cruda (z)')
    axes[0].plot(samples, theta_hw, 'r-', linewidth=2, label='θ estimado (HW)')
    axes[0].plot(samples, theta_ref, 'g--', linewidth=1.5, label='θ estimado (Python ref)')
    axes[0].set_title('Estimación de Inclinación θ')
    axes[0].set_ylabel('Ángulo (rad)')
    axes[0].legend()
    axes[0].grid(True, alpha=0.3)
    
    # 2. Clase ML detectada
    axes[1].step(samples, ml_classes, 'k-', where='post')
    axes[1].set_title('Clasificación ML (adaptativa)')
    axes[1].set_ylabel('Clase')
    axes[1].set_yticks([0, 1, 2])
    axes[1].set_yticklabels(['Reposo', 'Vib. Suave', 'Vib. Intensa'])
    axes[1].grid(True, alpha=0.3)
    
    # 3. Matriz R adaptativa
    axes[2].plot(samples, R_values, 'm-')
    axes[2].set_title('Covarianza R Adaptativa')
    axes[2].set_ylabel('R')
    axes[2].grid(True, alpha=0.3)
    
    # 4. Error entre HW y Python
    axes[3].plot(samples[:len(errors_theta)], errors_theta, 'c-')
    axes[3].set_title('Error entre Implementación HW y Referencia Python')
    axes[3].set_xlabel('Muestra')
    axes[3].set_ylabel('Error |θ_hw - θ_ref| (rad)')
    axes[3].grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig('kalman_adaptive_results.png', dpi=150)
    plt.show()

# =====================================================================
// EJECUCIÓN PRINCIPAL
// =====================================================================
if __name__ == "__main__":
    data = acquire_data()
    errors_theta, errors_omega = compare_implementations(data)
    plot_results(data, errors_theta)
    
    print("\nResultados guardados en 'kalman_adaptive_results.png'")
```

**Procedimiento:**
1. Ejecutar el firmware en la placa Polaris.
2. Ejecutar el script `kalman_reference.py` en la PC.
3. El script adquiere los datos enviados por UART y los compara con la referencia Python.

**Resultado esperado:**
- El error entre la implementación HW y la referencia Python es pequeño (< 0.01 rad).
- Las gráficas muestran claramente la adaptación del filtro según la clase ML.
- En reposo, R es bajo y el filtro responde rápidamente.
- En vibración intensa, R es alto y el filtro suaviza más.

---

### PASO 5: Pruebas con diferentes condiciones de movimiento (30 min)

Se realizan pruebas sistemáticas para verificar el comportamiento adaptativo del filtro.

**Prueba 1: Reposo estático**
1. Colocar la placa en reposo sobre una superficie estable.
2. Observar que la clase ML detectada es 0 (Reposo).
3. Verificar que R se mantiene bajo.
4. El filtro converge rápidamente al ángulo real.

**Prueba 2: Vibración suave**
1. Sujetar la placa con la mano y moverla suavemente.
2. Observar que la clase ML cambia a 1 (Vibración suave).
3. Verificar que R aumenta (×5).
4. El filtro suaviza las oscilaciones pero responde a cambios de inclinación.

**Prueba 3: Vibración intensa**
1. Agitar la placa vigorosamente.
2. Observar que la clase ML cambia a 2 (Vibración intensa).
3. Verificar que R aumenta significativamente (×20).
4. El filtro ignora las perturbaciones y mantiene la estimación previa.

**Prueba 4: Transición entre estados**
1. Comenzar en reposo, luego agitar, luego volver a reposo.
2. Observar que el filtro se adapta dinámicamente a cada estado.
3. Verificar que la transición es suave y sin discontinuidades.

**Resultados esperados:**

| Condición | Clase ML | R actual | K (ganancia) | Comportamiento |
|---|---|---|---|---|
| Reposo | 0 | Bajo | Alto (≈0.8) | Responde rápido |
| Vib. suave | 1 | Medio | Medio (≈0.4) | Suaviza moderadamente |
| Vib. intensa | 2 | Alto | Bajo (≈0.1) | Ignora perturbaciones |

---

## 5.5 Verificación final del Módulo 5

Al finalizar el Módulo 5, se debe contar con:

| Criterio | Verificación |
|---|---|
| CMSIS-DSP instalado y funcional | ✅ Compila sin errores |
| Filtro de Kalman inicializado | ✅ Matrices A, H, P, Q, R configuradas |
| Estimación de inclinación funcional | ✅ θ calculado a partir de az |
| Clasificación ML integrada | ✅ Clase detectada cada 50 muestras |
| Adaptación de Q y R funcional | ✅ Matrices cambian según clase ML |
| Datos filtrados enviados por UART | ✅ Visibles en terminal serial |
| Comparación con referencia Python | ✅ Error < 0.01 rad |
| Pruebas en diferentes condiciones | ✅ Reposo, vibración suave, intensa |

---

## 5.6 Entregables del Módulo 5 (conservar para Módulos siguientes)

| Entregable | Uso futuro |
|---|---|
| `kalman_filter.h/c` | Parte del firmware final |
| `kalman_config.h` | Configuración del filtro |
| `ml_classifier.h/c` | Clasificación ML en firmware |
| `ml_weights_q.h` | Pesos del modelo cuantizados |
| `main.c` actualizado | Firmware completo |
| `kalman_reference.py` | Script de validación |
| Gráficas de resultados | Evidencia de funcionamiento |
| Matrices Q y R adaptativas | Se analizan en Módulo 6 |

---

## 5.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| Error de compilación en CMSIS-DSP | Faltan archivos fuente o include paths | Verificar configuración del proyecto |
| NaN en el ángulo estimado | az fuera de rango [-256, 256] | Verificar saturación en `compute_tilt_angle` |
| Filtro diverge (valores crecientes) | Q o R mal configurados | Verificar valores base y factores de escala |
| Error de enlazado (undefined reference) | Funciones CMSIS-DSP no compiladas | Agregar archivos .c necesarios al proyecto |
| Clasificación ML siempre en la misma clase | Pesos mal cargados o características incorrectas | Verificar `ml_weights_q.h` y acelerador HW |
| UART no muestra datos | Puerto COM incorrecto | Probar los 3 últimos puertos COM |
| Error grande entre HW y Python | Diferencia en formatos de punto fijo | Verificar shifts y formatos Q |
| Filtro no se adapta | `kalman_adapt` no se llama | Verificar que se ejecuta cada 50 muestras |

---

## 5.8 Resumen del Día 5

**Antes del Día 5:** CMSIS-DSP instalado. Matriz R del Módulo 4 disponible. Pesos ML del Módulo 2 disponibles.

**Durante el Día 5 (4 horas):**
- 0:00–0:30: Teoría del Filtro de Kalman y estimación de inclinación
- 0:30–1:15: Implementar filtro de Kalman en C con CMSIS-DSP (PASO 1)
- 1:15–1:45: Implementar clasificación ML en firmware (PASO 2)
- 1:45–2:45: Integrar todo en firmware principal (PASO 3)
- 2:45–3:15: Generar referencia Python y comparar (PASO 4)
- 3:15–3:45: Pruebas con diferentes condiciones (PASO 5)
- 3:45–4:00: Verificación final y resumen

**Al final del Día 5:** Filtro de Kalman adaptativo funcional, con matrices Q y R reconfiguradas dinámicamente según la clasificación del modelo ML, datos filtrados enviados por UART, y validación contra referencia Python.

---

## 5.9 Relación con los otros módulos

**Relación con el Módulo 1:**
- Se reutiliza el UART para enviar los datos filtrados a la PC.
- Se reutiliza el proyecto SoftConsole como base del firmware.

**Relación con el Módulo 2:**
- Se utilizan los pesos del modelo ML entrenado en Python.
- Los formatos de punto fijo (Q3.12, Q2.13, Q3.14) son los mismos.
- La clasificación ML es la misma que se simuló en Python.

**Relación con el Módulo 3:**
- El acelerador HW calcula las características que alimentan al clasificador ML.
- Las características se leen desde los registros del IP personalizado.

**Relación con el Módulo 4:**
- El driver del ADXL345 proporciona los datos crudos (az) para el filtro.
- La matriz R base se obtuvo del análisis de ruido del Módulo 4.

**Relación con el Módulo 6:**
- El filtro de Kalman adaptativo es el componente central del sistema final.
- Se analizará el rendimiento del filtro (latencia, uso de CPU).
- Se comparará la calidad del filtrado con y sin adaptación ML.
- Se visualizarán los datos filtrados en la PC como producto final.

---

