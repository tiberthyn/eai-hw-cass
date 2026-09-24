# Evaluación Integral de Viabilidad y Plan de Ejecución del Programa IEEE CASS UMSA 2026

---

## MÓDULO 1 – PLAN DE EJECUCIÓN DETALLADO

### 1.1 Preparación PREVIA al Día 1 (actividades del instructor antes del curso)

#### 1.1.1 Instalación de software

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

#### 1.1.2 Verificación de hardware

Para **cada placa Polaris** que se utilizará en el curso:

1. Conectar la placa al PC mediante cable USB Tipo-C.
2. Verificar que el sistema operativo detecta el FlashPro5:
   - **Windows:** Abrir el Administrador de Dispositivos → buscar "FlashPro5" o "Microchip" en la sección de dispositivos USB. Deben aparecer 4 puertos COM virtuales (Sección 6 del manual).
   - **Linux:** Ejecutar `lsusb` y buscar Microchip. Ejecutar `dmesg | tail` para observar los puertos ttyUSB creados.
3. En caso de no detectarse, instalar los drivers del FlashPro5 (incluidos con Libero SoC).
4. Abrir Libero SoC → `Tools → FlashPro5` o `Configure Programming` → verificar que el programador aparece listado.
5. Anotar los números de los puertos COM asignados (se requieren para UART). Según el manual (Sección 6), **solo los tres últimos puertos COM se encuentran habilitados para UART**; el primero es reservado para funciones internas del FlashPro5.

#### 1.1.3 Preparación de materiales para estudiantes

Se debe crear una carpeta `Modulo1_Material/` que contenga:
- Una guía paso a paso impresa o en PDF con capturas de pantalla.
- Un proyecto de Libero SoC pre-configurado (opcional pero recomendado como respaldo).
- Un archivo de texto con los números de puerto COM de cada placa.

#### 1.1.4 Verificación personal del instructor

**El instructor DEBE completar la totalidad del Módulo 1 personalmente antes del curso**, en la misma placa Polaris que utilizarán los estudiantes. Esto incluye:
- Crear el proyecto desde cero en Libero SoC.
- Configurar el MSS.
- Crear el diseño HDL de parpadeo de LEDs.
- Sintetizar, Place & Route, programar.
- Crear el proyecto en SoftConsole.
- Escribir y ejecutar el código C de UART.
- Verificar la comunicación con la PC.

---

### 1.2 Conceptos teóricos que el instructor debe dominar y explicar

#### 1.2.1 Arquitectura SmartFusion2 (30 min de teoría)

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

#### 1.2.2 Entorno de desarrollo (15 min)

- **Libero SoC:** Herramienta de diseño de hardware. Aquí se crea la lógica de la FPGA, se configura el MSS, se sintetiza y se genera el bitstream.
- **SoftConsole:** IDE para escribir, compilar y depurar código C para el ARM Cortex-M3.
- **FlashPro5:** Programador integrado en la placa. Programa tanto la FPGA como el ARM.

#### 1.2.3 Flujo de desarrollo (15 min)

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

### 1.3 Herramientas que deben estar instaladas y configuradas

| Herramienta | Versión | Verificación |
|---|---|---|
| Libero SoC | v12.x | Abrir, verificar licencia |
| SoftConsole | Última versión | Abrir, verificar que compila |
| Drivers FlashPro5 | Incluídos con Libero | Conectar placa, verificar puertos COM |
| Terminal serial (PuTTY, Tera Term, minicom) | Cualquiera | Verificar que abre puerto COM |

---

### 1.4 Implementación paso a paso

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

### 1.5 Verificación final del Módulo 1

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

### 1.6 Entregables del Módulo 1 (conservar para Módulos siguientes)

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

### 1.7 Errores esperables y diagnóstico

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

### 1.8 Resumen del Día 1

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
