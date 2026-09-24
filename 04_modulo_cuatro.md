# MÓDULO 4: INTERFAZ DE SENSORES (ADXL345)

## 4.1 Preparación PREVIA al Día 4 (actividades del instructor antes del módulo)

### 4.1.1 Verificación del entorno de desarrollo

El Módulo 4 requiere que el entorno de desarrollo se encuentre plenamente operativo con los entregables de los módulos anteriores:

1. **Proyecto Libero SoC del Módulo 3 disponible:** El proyecto debe contener el MSS configurado con SPI_0 habilitado, el IP personalizado del acelerador ML integrado en el bus APB, y las señales SPI ruteadas a los pines del ADXL345.
2. **Proyecto SoftConsole del Módulo 1 disponible:** El proyecto debe contener los drivers HAL del MSS, incluyendo los drivers del SPI generados en el Módulo 3.
3. **Placa Polaris funcional:** Verificar que el ADXL345 responde correctamente a comandos SPI básicos.
4. **Documentación del ADXL345:** Disponer del datasheet del ADXL345 de Analog Devices como referencia técnica.

### 4.1.2 Verificación del ADXL345 en la placa Polaris

Antes del Día 4, el instructor debe verificar que el acelerómetro ADXL345 integrado en la placa Polaris funciona correctamente:

**Procedimiento de verificación:**

1. Conectar la placa Polaris al PC mediante USB Tipo-C.
2. Programar un diseño simple en la FPGA que lea el registro 0x00 (DEVID) del ADXL345.
3. El registro DEVID debe retornar el valor 0xE5 (identificador del ADXL345).
4. Verificar que los ejes X, Y, Z retornan valores coherentes (aproximadamente 0, 0, 256 en reposo con rango ±2g).

**Si el ADXL345 no responde:**
- Verificar que las señales SPI están correctamente ruteadas en el MSS Configurator.
- Verificar que el modo SPI es correcto (CPOL=1, CPHA=1 para ADXL345).
- Verificar que el pin CS (M3) se mantiene en alto cuando no hay comunicación.

### 4.1.3 Preparación de materiales para estudiantes

Se debe crear una carpeta `Modulo4_Material/` que contenga:
- Template del driver SPI `adxl345_driver.h` y `adxl345_driver.c` (proporcionado por el instructor).
- Archivo de configuración `adxl345_config.h` con los registros y constantes del ADXL345.
- Guía paso a paso con capturas de pantalla.
- Script Python `analyze_noise.py` para el análisis de ruido estático.

### 4.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 4 personalmente antes del curso**, verificando:
- La configuración correcta del SPI en el MSS Configurator.
- La lectura de registros del ADXL345 desde el firmware.
- La adquisición de datos en tiempo real a alta velocidad.
- El envío de datos al acelerador HW y por UART.
- El manejo de interrupciones para adquisición continua.

---

## 4.2 Conceptos teóricos que el instructor debe dominar y explicar

### 4.2.1 Arquitectura del ADXL345 (20 min)

El ADXL345 es un acelerómetro digital de 3 ejes con interfaz SPI/I²C, según se describe en la Sección 10 del manual de usuario de la Polaris.

**Características principales:**
- Rango de medición: ±2g, ±4g, ±8g, ±16g (configurable)
- Resolución: 10 bits en modo fixed, 13 bits en modo full resolution
- Frecuencia de muestreo: hasta 3200 Hz
- Interfaz: SPI (modo 0 y modo 3) o I²C
- Consumo: 23-145 μA típico

**Registros principales:**

| Dirección | Nombre | Descripción |
|---|---|---|
| 0x00 | DEVID | ID del dispositivo (0xE5) |
| 0x2D | BW_RATE | Tasa de datos y modo de consumo |
| 0x31 | DATA_FORMAT | Resolución y justificación |
| 0x32-0x37 | DATAX0-DATAZ1 | Datos de aceleración (6 bytes) |
| 0x2C | POWER_CTL | Control de energía |
| 0x38 | INT_ENABLE | Habilitación de interrupciones |

**Modo SPI del ADXL345:**
- Modo 3: CPOL=1 (reloj en alto en reposo), CPHA=1 (datos capturados en flanco descendente)
- Frecuencia máxima: 5 MHz
- Formato: 8 bits, MSB primero
- Lectura: bit 6 del primer byte = 1
- Multi-byte: bit 6 del primer byte = 1 (auto-incremento de dirección)

### 4.2.2 Configuración del SPI en el MSS del SmartFusion2 (20 min)

El SmartFusion2 M2S005 dispone de 2 periféricos SPI en el MSS, según la Sección 3 del manual de usuario.

**Configuración requerida para el ADXL345:**

| Parámetro | Valor | Razón |
|---|---|---|
| Modo | Master | El ARM controla la comunicación |
| CPOL | 1 | Requerido por ADXL345 (modo 3) |
| CPHA | 1 | Requerido por ADXL345 (modo 3) |
| Frecuencia | 1-5 MHz | Máximo del ADXL345 |
| Tamaño de frame | 8 bits | Estándar |
| Bit order | MSB first | Requerido por ADXL345 |
| CS polarity | Active low | Estándar |

**Ruteo de señales SPI a través de la fabric:**

Según la Tabla 5 del manual de usuario de la Polaris, los pines del ADXL345 están en la fabric de la FPGA:

| Señal SPI | Pin FPGA | Nombre en placa |
|---|---|---|
| SPI0_CLK | P3 | ADXL_SCL |
| SPI0_MOSI | N4 | ADXL_SDA |
| SPI0_MISO | N3 | ADXL_SDO |
| SPI0_SS0 | M3 | ADXL_CS |

**Nota técnica:** Los nombres "SDA" y "SCL" corresponden a nomenclatura I²C, pero la placa utiliza SPI. El pin "SDA" es MOSI (datos hacia el ADXL), y "SCL" es el reloj SPI.

**Configuración en el MSS Configurator:**
1. Habilitar SPI_0 como Master.
2. Configurar CPOL=1, CPHA=1 (modo 3).
3. Configurar frecuencia de reloj SPI (divisor del clock del sistema).
4. Ruteare las señales SPI a los pines de la fabric (P3, N4, N3, M3).
5. Generar el MSS para crear los drivers HAL.

### 4.2.3 Adquisición de datos a alta velocidad (15 min)

Para adquirir datos del ADXL345 a alta velocidad (hasta 3200 Hz), se deben considerar las siguientes estrategias:

**Estrategia 1: Polling (sondeo)**
- El ARM lee los registros de datos periódicamente.
- Limitación: overhead de SPI (8 bytes por lectura × 3 ejes).
- Velocidad máxima práctica: ~1000 Hz.

**Estrategia 2: Interrupciones**
- El ADXL345 genera una interrupción cuando hay datos nuevos.
- El ARM lee los datos en el handler de interrupción.
- Velocidad máxima: 3200 Hz (limitada por el ADXL345).

**Estrategia 3: DMA (Direct Memory Access)**
- El DMA transfiere datos del SPI a memoria sin intervención del ARM.
- Velocidad máxima: 3200 Hz.
- Requiere configuración adicional del DMA del MSS.

**Para este curso, se utiliza la Estrategia 2 (interrupciones)** por su balance entre simplicidad y rendimiento.

### 4.2.4 Análisis de ruido estático del sensor (15 min)

El ADXL345 presenta ruido intrínseco incluso en condiciones estáticas. Este ruido debe caracterizarse para configurar correctamente el filtro de Kalman del Módulo 5.

**Fuentes de ruido:**
- Ruido térmico del sensor MEMS
- Ruido de cuantización del ADC interno
- Interferencia electromagnética
- Vibraciones ambientales

**Caracterización del ruido:**
1. Colocar la placa en reposo sobre una superficie estable.
2. Adquirir 1000 muestras de cada eje.
3. Calcular la media y varianza de cada eje.
4. La varianza representa el ruido de medición (matriz R del Kalman).

**Valores esperados (rango ±2g, 10 bits):**
- Media en X, Y: ≈ 0 (±10 cuentas)
- Media en Z: ≈ 256 (1g)
- Desviación estándar: ≈ 3-5 cuentas (ruido típico)

---

## 4.3 Herramientas y configuración requeridas

| Herramienta | Configuración específica | Verificación |
|---|---|---|
| Libero SoC v12+ | Proyecto del Módulo 3 abierto | MSS con SPI_0 habilitado |
| SoftConsole IDE | Proyecto del Módulo 1 abierto | Drivers HAL del SPI disponibles |
| MSS Configurator | SPI_0 configurado como Master, modo 3 | MSS genera sin errores |
| Datasheet ADXL345 | Disponible como referencia | Consultar registros y timing |
| Terminal serial | 115200 baud, 8N1 | Comunicación UART funcional |
| Analizador lógico 24MHz 8CH | Para verificar señales SPI | Conectar a pines SPI |

---

## 4.4 Implementación paso a paso

### PASO 1: Verificar la configuración del SPI en el MSS Configurator (20 min)

1. **Abrir el proyecto de Libero SoC** del Módulo 3.
2. **Abrir el MSS Configurator** haciendo doble clic en "Create MSS Component".
3. **Verificar la configuración del SPI_0:**
   - Ir a la pestaña de periféricos → SPI.
   - Verificar que **SPI_0** está habilitado como Master.
   - Verificar que CPOL=1, CPHA=1 (modo 3).
   - Verificar que la frecuencia de reloj SPI está configurada para generar 1-5 MHz.
4. **Verificar el ruteo de pines:**
   - SPI0_CLK → Pin P3 (ADXL_SCL)
   - SPI0_MOSI → Pin N4 (ADXL_SDA)
   - SPI0_MISO → Pin N3 (ADXL_SDO)
   - SPI0_SS0 → Pin M3 (ADXL_CS)
5. **Generar el MSS** si se realizaron cambios.
6. **Sintetizar y programar** el diseño actualizado.

**Resultado esperado:** El MSS Configurator muestra el SPI_0 configurado correctamente, con las señales ruteadas a los pines del ADXL345.

---

### PASO 2: Crear el driver del ADXL345 en C (60 min)

Se implementa el driver para comunicar con el ADXL345 mediante SPI.

**Archivo a crear:** `adxl345_config.h`

```c
#ifndef ADXL345_CONFIG_H
#define ADXL345_CONFIG_H

// =====================================================================
// REGISTROS DEL ADXL345
// =====================================================================
#define ADXL345_REG_DEVID        0x00  // ID del dispositivo (0xE5)
#define ADXL345_REG_BW_RATE      0x2D  // Tasa de datos y modo de consumo
#define ADXL345_REG_POWER_CTL    0x2C  // Control de energía
#define ADXL345_REG_DATA_FORMAT  0x31  // Resolución y justificación
#define ADXL345_REG_DATAX0       0x32  // Datos X (LSB)
#define ADXL345_REG_DATAX1       0x33  // Datos X (MSB)
#define ADXL345_REG_DATAY0       0x34  // Datos Y (LSB)
#define ADXL345_REG_DATAY1       0x35  // Datos Y (MSB)
#define ADXL345_REG_DATAZ0       0x36  // Datos Z (LSB)
#define ADXL345_REG_DATAZ1       0x37  // Datos Z (MSB)
#define ADXL345_REG_INT_ENABLE   0x2C  // Habilitación de interrupciones
#define ADXL345_REG_INT_MAP      0x2F  // Mapeo de interrupciones
#define ADXL345_REG_INT_SOURCE   0x30  // Fuente de interrupciones

// =====================================================================
// COMANDOS SPI
// =====================================================================
#define ADXL345_SPI_READ         0x80  // Bit 7 = 1 para lectura
#define ADXL345_SPI_MULTI        0x40  // Bit 6 = 1 para multi-byte

// =====================================================================
// CONFIGURACIÓN
// =====================================================================
#define ADXL345_DEVID_EXPECTED   0xE5  // ID esperado del ADXL345

// BW_RATE: 0x0A = 100 Hz, 0x0B = 200 Hz, 0x0C = 400 Hz, 0x0D = 800 Hz
#define ADXL345_BW_RATE_100HZ    0x0A
#define ADXL345_BW_RATE_400HZ    0x0C
#define ADXL345_BW_RATE_3200HZ   0x0F

// DATA_FORMAT: 0x00 = 10 bits fixed, 0x08 = 13 bits full resolution
#define ADXL345_DATA_FORMAT_10BIT  0x00
#define ADXL345_DATA_FORMAT_13BIT  0x08

// POWER_CTL: 0x08 = Measure mode
#define ADXL345_POWER_CTL_MEASURE  0x08

#endif // ADXL345_CONFIG_H
```

**Archivo a crear:** `adxl345_driver.h`

```c
#ifndef ADXL345_DRIVER_H
#define ADXL345_DRIVER_H

#include <stdint.h>
#include "mss_spi.h"

// =====================================================================
// ESTRUCTURA DE DATOS
// =====================================================================
typedef struct {
    int16_t x;  // Aceleración X (10 o 13 bits signed)
    int16_t y;  // Aceleración Y
    int16_t z;  // Aceleración Z
} adxl345_data_t;

// =====================================================================
// FUNCIONES PÚBLICAS
// =====================================================================

/**
 * @brief Inicializa el ADXL345
 * @return 0 si éxito, -1 si error (ID incorrecto)
 */
int adxl345_init(void);

/**
 * @brief Lee los datos de aceleración de los 3 ejes
 * @param data Puntero a estructura donde se almacenan los datos
 * @return 0 si éxito, -1 si error
 */
int adxl345_read_accel(adxl345_data_t *data);

/**
 * @brief Configura la tasa de muestreo
 * @param rate Código de tasa (ej: ADXL345_BW_RATE_100HZ)
 */
void adxl345_set_data_rate(uint8_t rate);

/**
 * @brief Habilita interrupción de datos nuevos
 */
void adxl345_enable_data_ready_interrupt(void);

#endif // ADXL345_DRIVER_H
```

**Archivo a crear:** `adxl345_driver.c`

```c
#include "adxl345_driver.h"
#include "adxl345_config.h"

// =====================================================================
// VARIABLES GLOBALES
// =====================================================================
static mss_spi_instance_t *const g_spi = &g_mss_spi0;

// =====================================================================
// FUNCIONES AUXILIARES
// =====================================================================

/**
 * @brief Escribe un byte en un registro del ADXL345
 */
static void adxl345_write_reg(uint8_t reg, uint8_t value) {
    uint8_t tx_data[2];
    tx_data[0] = reg;  // Dirección del registro (bit 7 = 0 para escritura)
    tx_data[1] = value;
    
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_0);
    MSS_SPI_tx_frame(g_spi, tx_data[0]);
    MSS_SPI_tx_frame(g_spi, tx_data[1]);
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_NOSELECT);
}

/**
 * @brief Lee un byte de un registro del ADXL345
 */
static uint8_t adxl345_read_reg(uint8_t reg) {
    uint8_t tx_data = ADXL345_SPI_READ | reg;  // Bit 7 = 1 para lectura
    uint8_t rx_data;
    
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_0);
    MSS_SPI_tx_frame(g_spi, tx_data);
    rx_data = MSS_SPI_rx_frame(g_spi);
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_NOSELECT);
    
    return rx_data;
}

// =====================================================================
// FUNCIONES PÚBLICAS
// =====================================================================

int adxl345_init(void) {
    uint8_t devid;
    
    // Inicializar SPI (ya configurado en MSS Configurator)
    MSS_SPI_init(g_spi);
    MSS_SPI_configure_master_mode(g_spi);
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_NOSELECT);
    
    // Verificar ID del dispositivo
    devid = adxl345_read_reg(ADXL345_REG_DEVID);
    if (devid != ADXL345_DEVID_EXPECTED) {
        return -1;  // Error: ID incorrecto
    }
    
    // Configurar tasa de datos (100 Hz por defecto)
    adxl345_write_reg(ADXL345_REG_BW_RATE, ADXL345_BW_RATE_100HZ);
    
    // Configurar formato de datos (10 bits fixed)
    adxl345_write_reg(ADXL345_REG_DATA_FORMAT, ADXL345_DATA_FORMAT_10BIT);
    
    // Entrar en modo de medición
    adxl345_write_reg(ADXL345_REG_POWER_CTL, ADXL345_POWER_CTL_MEASURE);
    
    return 0;  // Éxito
}

int adxl345_read_accel(adxl345_data_t *data) {
    uint8_t tx_data = ADXL345_SPI_READ | ADXL345_SPI_MULTI | ADXL345_REG_DATAX0;
    uint8_t rx_data[7];
    
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_0);
    
    // Enviar comando de lectura multi-byte
    MSS_SPI_tx_frame(g_spi, tx_data);
    
    // Leer 6 bytes (X0, X1, Y0, Y1, Z0, Z1)
    for (int i = 0; i < 6; i++) {
        rx_data[i] = MSS_SPI_rx_frame(g_spi);
    }
    
    MSS_SPI_set_slave_select(g_spi, MSS_SPI_SLAVE_NOSELECT);
    
    // Convertir a int16_t (10 bits signed, justificados a la izquierda)
    data->x = (int16_t)((rx_data[1] << 8) | rx_data[0]) >> 6;  // Shift right 6 para 10 bits
    data->y = (int16_t)((rx_data[3] << 8) | rx_data[2]) >> 6;
    data->z = (int16_t)((rx_data[5] << 8) | rx_data[4]) >> 6;
    
    return 0;  // Éxito
}

void adxl345_set_data_rate(uint8_t rate) {
    adxl345_write_reg(ADXL345_REG_BW_RATE, rate);
}

void adxl345_enable_data_ready_interrupt(void) {
    // Habilitar interrupción DATA_READY (bit 7)
    adxl345_write_reg(ADXL345_REG_INT_ENABLE, 0x80);
    
    // Mapear interrupción DATA_READY al pin INT1
    adxl345_write_reg(ADXL345_REG_INT_MAP, 0x80);
}
```

**Nota técnica:** Las funciones `MSS_SPI_tx_frame()` y `MSS_SPI_rx_frame()` pueden variar según la versión del HAL. El instructor debe verificar los nombres exactos en el archivo `mss_spi.h` generado por el MSS Configurator.

**Resultado esperado:** El driver permite inicializar el ADXL345, verificar su ID, y leer los datos de aceleración de los 3 ejes.

---

### PASO 3: Integrar el driver en el firmware principal (45 min)

Se modifica el `main.c` del Módulo 1 para integrar el driver del ADXL345 y enviar los datos al acelerador HW y por UART.

**Archivo a modificar:** `main.c`

```c
#include <stdint.h>
#include <stdio.h>
#include "mss_uart.h"
#include "mss_gpio.h"
#include "adxl345_driver.h"

// =====================================================================
// DIRECCIONES DEL IP PERSONALIZADO (acelerador ML)
// =====================================================================
#define ACCEL_BASE_ADDR        0x40000000  // Dirección base del IP
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

// =====================================================================
// FUNCIONES AUXILIARES
// =====================================================================

/**
 * @brief Envía datos al acelerador HW (registro DATA_IN)
 */
void send_to_accelerator(adxl345_data_t *data) {
    // Empaquetar los 3 ejes en 30 bits (10 bits por eje)
    uint32_t packed_data = ((uint32_t)(data->x & 0x3FF) << 20) |
                           ((uint32_t)(data->y & 0x3FF) << 10) |
                           ((uint32_t)(data->z & 0x3FF));
    
    // Escribir en el registro DATA_IN del acelerador
    volatile uint32_t *reg_data_in = (volatile uint32_t *)ACCEL_REG_DATA_IN;
    *reg_data_in = packed_data;
    
    // Activar el bit START en el registro CONTROL
    volatile uint32_t *reg_control = (volatile uint32_t *)ACCEL_REG_CONTROL;
    *reg_control = 0x01;  // Bit 0 = START
    
    // Esperar a que termine el cálculo (polling del bit DONE)
    volatile uint32_t *reg_status = (volatile uint32_t *)ACCEL_REG_STATUS;
    while (!(*reg_status & 0x02));  // Esperar bit 1 (DONE)
    
    // Limpiar el bit START
    *reg_control = 0x00;
}

/**
 * @brief Lee las características calculadas por el acelerador HW
 */
void read_accelerator_features(void) {
    volatile uint32_t *reg_feat = (volatile uint32_t *)ACCEL_REG_FEAT_0;
    
    for (int i = 0; i < 6; i++) {
        int32_t feat = (int32_t)reg_feat[i];
        // Convertir de Q3.12 a flotante para visualización
        float feat_float = feat / 4096.0f;
        
        char buffer[64];
        snprintf(buffer, sizeof(buffer), "Feat[%d]: %d (%.4f)\r\n", i, feat, feat_float);
        MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
    }
}

// =====================================================================
// FUNCIÓN PRINCIPAL
// =====================================================================
int main(void) {
    // Inicializar UART
    MSS_UART_init(g_uart, MSS_UART_115200_BAUD,
                  MSS_UART_DATA_8_BITS | MSS_UART_NO_PARITY | MSS_UART_ONE_STOP_BIT);
    
    MSS_UART_polled_tx_string(g_uart, (const uint8_t *)"\r\n[Modulo 4] Iniciando sistema...\r\n");
    
    // Inicializar ADXL345
    if (adxl345_init() != 0) {
        MSS_UART_polled_tx_string(g_uart, (const uint8_t *)"ERROR: ADXL345 no responde\r\n");
        while (1);  // Detener ejecución
    }
    
    MSS_UART_polled_tx_string(g_uart, (const uint8_t *)"ADXL345 inicializado correctamente\r\n");
    
    // Bucle principal
    uint32_t sample_count = 0;
    while (1) {
        // Leer datos del acelerómetro
        if (adxl345_read_accel(&accel_data) == 0) {
            sample_count++;
            
            // Enviar datos al acelerador HW
            send_to_accelerator(&accel_data);
            
            // Enviar datos por UART cada 10 muestras
            if (sample_count % 10 == 0) {
                char buffer[128];
                snprintf(buffer, sizeof(buffer),
                         "Muestra %lu: X=%d, Y=%d, Z=%d\r\n",
                         sample_count, accel_data.x, accel_data.y, accel_data.z);
                MSS_UART_polled_tx_string(g_uart, (const uint8_t *)buffer);
                
                // Leer y mostrar características del acelerador
                read_accelerator_features();
            }
        }
        
        // Pequeño delay para no saturar el UART
        for (volatile int i = 0; i < 10000; i++);
    }
    
    return 0;
}
```

**Compilar y programar:**
1. Compilar el proyecto en SoftConsole (`Project → Build All`).
2. Programar el ARM Cortex-M3.
3. Abrir el terminal serial a 115200 baud.

**Resultado esperado:**
- El mensaje "ADXL345 inicializado correctamente" aparece en el terminal.
- Cada 10 muestras, se muestran los valores de X, Y, Z y las 6 características calculadas por el acelerador HW.
- Los valores de X, Y están cerca de 0, y Z está cerca de 256 (1g en rango ±2g, 10 bits).

**Verificación:**
- Comparar los valores leídos del ADXL345 con los esperados (reposo: X≈0, Y≈0, Z≈256).
- Verificar que las características del acelerador HW coinciden con los cálculos de Python (con tolerancia de ±1 LSB).

---

### PASO 4: Análisis de ruido estático (45 min)

Se realiza un análisis del ruido del ADXL345 en condiciones estáticas para caracterizar la matriz R del filtro de Kalman.

**Archivo a crear:** `analyze_noise.py`

```python
import numpy as np
import matplotlib.pyplot as plt
import serial
import time

# =====================================================================
// CONFIGURACIÓN
// =====================================================================
SERIAL_PORT = 'COM3'  # Ajustar según el puerto COM del UART
BAUD_RATE = 115200
NUM_SAMPLES = 1000

# =====================================================================
// ADQUISICIÓN DE DATOS
// =====================================================================
def acquire_data():
    """Adquiere datos del ADXL345 a través del UART"""
    ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
    time.sleep(2)  # Esperar a que el UART se estabilice
    
    data_x = []
    data_y = []
    data_z = []
    
    print(f"Adquiriendo {NUM_SAMPLES} muestras...")
    
    while len(data_x) < NUM_SAMPLES:
        line = ser.readline().decode('utf-8', errors='ignore').strip()
        if 'Muestra' in line:
            # Parsear línea: "Muestra 10: X=5, Y=-3, Z=258"
            parts = line.split(':')
            if len(parts) == 2:
                values = parts[1].strip().split(',')
                x = int(values[0].split('=')[1])
                y = int(values[1].split('=')[1])
                z = int(values[2].split('=')[1])
                data_x.append(x)
                data_y.append(y)
                data_z.append(z)
                if len(data_x) % 100 == 0:
                    print(f"  {len(data_x)} muestras adquiridas")
    
    ser.close()
    return np.array(data_x), np.array(data_y), np.array(data_z)

# =====================================================================
// ANÁLISIS DE RUIDO
// =====================================================================
def analyze_noise(data_x, data_y, data_z):
    """Analiza el ruido estático del sensor"""
    print("\n=== Análisis de Ruido Estático ===")
    
    # Eje X
    mean_x = np.mean(data_x)
    std_x = np.std(data_x)
    var_x = np.var(data_x)
    print(f"Eje X: Media={mean_x:.2f}, Desv.Std={std_x:.2f}, Varianza={var_x:.2f}")
    
    # Eje Y
    mean_y = np.mean(data_y)
    std_y = np.std(data_y)
    var_y = np.var(data_y)
    print(f"Eje Y: Media={mean_y:.2f}, Desv.Std={std_y:.2f}, Varianza={var_y:.2f}")
    
    # Eje Z
    mean_z = np.mean(data_z)
    std_z = np.std(data_z)
    var_z = np.var(data_z)
    print(f"Eje Z: Media={mean_z:.2f}, Desv.Std={std_z:.2f}, Varianza={var_z:.2f}")
    
    # Matriz R para el filtro de Kalman (varianza de cada eje)
    R = np.diag([var_x, var_y, var_z])
    print(f"\nMatriz R (covarianza de medición):\n{R}")
    
    return mean_x, std_x, mean_y, std_y, mean_z, std_z, R

# =====================================================================
// VISUALIZACIÓN
// =====================================================================
def plot_noise(data_x, data_y, data_z, mean_x, mean_y, mean_z, std_x, std_y, std_z):
    """Visualiza el ruido del sensor"""
    fig, axes = plt.subplots(3, 1, figsize=(12, 10))
    
    # Eje X
    axes[0].plot(data_x, alpha=0.6, label='Datos')
    axes[0].axhline(mean_x, color='r', linestyle='--', label=f'Media={mean_x:.2f}')
    axes[0].axhline(mean_x + std_x, color='g', linestyle=':', label=f'±1σ={std_x:.2f}')
    axes[0].axhline(mean_x - std_x, color='g', linestyle=':')
    axes[0].set_title('Eje X - Ruido Estático')
    axes[0].set_ylabel('Aceleración (cuentas)')
    axes[0].legend()
    axes[0].grid(True, alpha=0.3)
    
    # Eje Y
    axes[1].plot(data_y, alpha=0.6, label='Datos')
    axes[1].axhline(mean_y, color='r', linestyle='--', label=f'Media={mean_y:.2f}')
    axes[1].axhline(mean_y + std_y, color='g', linestyle=':', label=f'±1σ={std_y:.2f}')
    axes[1].axhline(mean_y - std_y, color='g', linestyle=':')
    axes[1].set_title('Eje Y - Ruido Estático')
    axes[1].set_ylabel('Aceleración (cuentas)')
    axes[1].legend()
    axes[1].grid(True, alpha=0.3)
    
    # Eje Z
    axes[2].plot(data_z, alpha=0.6, label='Datos')
    axes[2].axhline(mean_z, color='r', linestyle='--', label=f'Media={mean_z:.2f}')
    axes[2].axhline(mean_z + std_z, color='g', linestyle=':', label=f'±1σ={std_z:.2f}')
    axes[2].axhline(mean_z - std_z, color='g', linestyle=':')
    axes[2].set_title('Eje Z - Ruido Estático')
    axes[2].set_xlabel('Muestra')
    axes[2].set_ylabel('Aceleración (cuentas)')
    axes[2].legend()
    axes[2].grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig('adxl345_noise_analysis.png', dpi=150)
    plt.show()

# =====================================================================
// EJECUCIÓN PRINCIPAL
// =====================================================================
if __name__ == "__main__":
    # Adquirir datos
    data_x, data_y, data_z = acquire_data()
    
    # Analizar ruido
    mean_x, std_x, mean_y, std_y, mean_z, std_z, R = analyze_noise(data_x, data_y, data_z)
    
    # Visualizar
    plot_noise(data_x, data_y, data_z, mean_x, mean_y, mean_z, std_x, std_y, std_z)
    
    # Guardar resultados
    np.savez('noise_analysis_results.npz',
             mean_x=mean_x, std_x=std_x,
             mean_y=mean_y, std_y=std_y,
             mean_z=mean_z, std_z=std_z,
             R=R)
    print("\nResultados guardados en 'noise_analysis_results.npz'")
```

**Procedimiento:**
1. Colocar la placa Polaris en reposo sobre una superficie estable.
2. Ejecutar el script `analyze_noise.py`.
3. El script adquiere 1000 muestras y calcula la media y varianza de cada eje.
4. Se generan gráficas del ruido y se guarda la matriz R.

**Resultado esperado:**
- Media en X, Y: ≈ 0 (±10 cuentas)
- Media en Z: ≈ 256 (1g)
- Desviación estándar: ≈ 3-5 cuentas
- Matriz R diagonal con las varianzas de cada eje.

---

### PASO 5: Verificación con analizador lógico (30 min)

Se utiliza el analizador lógico 24MHz 8CH para verificar las señales SPI y medir el tiempo de adquisición.

**Conexiones del analizador lógico:**

| Canal | Señal | Pin FPGA | Descripción |
|---|---|---|---|
| CH1 | SPI_CLK | P3 | Reloj SPI |
| CH2 | SPI_MOSI | N4 | Datos hacia ADXL |
| CH3 | SPI_MISO | N3 | Datos desde ADXL |
| CH4 | SPI_CS | M3 | Chip Select |
| GND | GND | - | Referencia |

**Procedimiento:**
1. Conectar los canales del analizador lógico a los pines SPI.
2. Conectar GND del analizador a GND de la placa.
3. Configurar el analizador a 24 MHz de muestreo.
4. Iniciar la captura.
5. Ejecutar el firmware para generar tráfico SPI.
6. Detener la captura y analizar las señales.

**Verificaciones:**
- **Frecuencia del reloj SPI:** Debe estar entre 1-5 MHz.
- **Modo SPI:** CPOL=1 (reloj en alto en reposo), CPHA=1 (datos capturados en flanco descendente).
- **Tiempo de lectura de 6 bytes:** Debe ser ≈ 48-50 ciclos de reloj SPI.
- **CS activo bajo:** El pin CS debe estar en bajo durante la transferencia.

**Resultado esperado:**
- Las señales SPI muestran el protocolo correcto (modo 3).
- El tiempo de lectura de una muestra (6 bytes) es ≈ 10-20 μs a 5 MHz.
- El throughput máximo es ≈ 50,000-100,000 muestras/segundo (limitado por el software).

---

## 4.5 Verificación final del Módulo 4

Al finalizar el Módulo 4, se debe contar con:

| Criterio | Verificación |
|---|---|
| ADXL345 responde correctamente | ✅ ID 0xE5 leído correctamente |
| Datos de aceleración coherentes | ✅ X≈0, Y≈0, Z≈256 en reposo |
| Datos enviados al acelerador HW | ✅ Registro DATA_IN actualizado |
| Características calculadas por HW | ✅ Coinciden con Python (±1 LSB) |
| Datos enviados por UART | ✅ Visibles en terminal serial |
| Análisis de ruido completado | ✅ Matriz R calculada |
| Señales SPI verificadas con analizador | ✅ Modo 3, frecuencia correcta |

---

## 4.6 Entregables del Módulo 4 (conservar para Módulos siguientes)

| Entregable | Uso futuro |
|---|---|
| Driver `adxl345_driver.c/h` | Se utiliza en Módulos 5 y 6 |
| Configuración `adxl345_config.h` | Referencia de registros |
| Firmware `main.c` actualizado | Base para Módulos 5 y 6 |
| Matriz R del ruido | Se utiliza en Módulo 5 (Kalman) |
| Script `analyze_noise.py` | Referencia para análisis |
| Capturas del analizador lógico | Evidencia de funcionamiento |

---

## 4.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| ADXL345 no responde (ID ≠ 0xE5) | SPI mal configurado o pines incorrectos | Verificar CPOL, CPHA, pines en MSS Configurator |
| Datos de aceleración incorrectos | Formato de datos mal configurado | Verificar DATA_FORMAT (10 bits vs 13 bits) |
| Valores de X, Y, Z siempre en 0 | ADXL no está en modo de medición | Verificar POWER_CTL = 0x08 |
| Acelerador HW no calcula características | Registro DATA_IN mal escrito | Verificar empaquetado de 30 bits |
| UART no muestra datos | Puerto COM incorrecto o baud rate mal | Verificar 115200 baud, probar los 3 últimos puertos COM |
| Analizador lógico no captura señales | Frecuencia de muestreo muy baja | Aumentar a 24 MHz, verificar conexiones |
| Ruido excesivo en mediciones | Interferencia o placa inestable | Aislar la placa, usar superficie estable |

---

## 4.8 Resumen del Día 4

**Antes del Día 4:** Proyecto del Módulo 3 con SPI habilitado. Driver del ADXL345 preparado. Analizador lógico disponible.

**Durante el Día 4 (4 horas):**
- 0:00–0:20: Teoría del ADXL345 y configuración SPI
- 0:20–0:40: Verificar configuración del SPI en MSS Configurator (PASO 1)
- 0:40–1:40: Crear driver del ADXL345 en C (PASO 2)
- 1:40–2:25: Integrar driver en firmware principal (PASO 3)
- 2:25–3:10: Análisis de ruido estático (PASO 4)
- 3:10–3:40: Verificación con analizador lógico (PASO 5)
- 3:40–4:00: Verificación final y resumen

**Al final del Día 4:** Driver SPI funcional, adquisición de datos en tiempo real, envío de datos al acelerador HW y por UART, análisis de ruido completado, señales SPI verificadas con analizador lógico.

---

## 4.9 Relación con los otros módulos

**Relación con el Módulo 1:**
- Se reutiliza el proyecto SoftConsole con UART funcional.
- Se amplía el firmware para integrar el driver del ADXL345.

**Relación con el Módulo 2:**
- Los datos adquiridos del ADXL345 son los mismos que se simularon en Python.
- El análisis de ruido proporciona la matriz R para el filtro de Kalman.

**Relación con el Módulo 3:**
- El SPI habilitado en el MSS Configurator se utiliza para comunicar con el ADXL345.
- Los datos se envían al acelerador HW escribiendo en el registro DATA_IN.
- Las características calculadas por el HW se leen desde los registros FEAT_0 a FEAT_5.

**Relación con el Módulo 5:**
- El filtro de Kalman utilizará los datos del ADXL345 como entrada de medición.
- La matriz R calculada en el análisis de ruido se utilizará para inicializar el filtro.
- Las características del acelerador HW se utilizarán para ajustar dinámicamente las matrices Q y R.

**Relación con el Módulo 6:**
- Se medirá el throughput del sistema completo usando el analizador lógico.
- Se comparará la latencia del acelerador HW vs. cálculo en software.
- Se visualizarán los datos filtrados en la PC.

---
