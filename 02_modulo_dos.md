# MÓDULO 2 – APRENDIZAJE AUTOMÁTICO EN LÍNEA (ONLINE ML)

## 2.1 Preparación PREVIA al Día 2 (actividades del instructor antes del módulo)

### 2.1.1 Instalación y verificación del entorno Python

El entorno de trabajo del Módulo 2 se desarrolla íntegramente en Python, por lo que se requiere la instalación y verificación previa de los siguientes componentes:

**Python 3.8 o superior:**
1. Descargar el instalador desde [python.org](https://www.python.org/downloads/).
2. Durante la instalación, seleccionar la opción **"Add Python to PATH"**.
3. **Verificación:** Ejecutar en terminal `python --version` y confirmar versión ≥ 3.8.

**Paquetes requeridos:**
Ejecutar el siguiente comando en terminal:

```bash
pip install numpy scipy matplotlib scikit-learn pandas
```

**Verificación de paquetes:**

```bash
python -c "import numpy, scipy, matplotlib, sklearn, pandas; print('Todos los paquetes instalados correctamente')"
```

**Versiones recomendadas:**

| Paquete | Versión mínima | Función en el módulo |
|---|---|---|
| numpy | 1.20+ | Operaciones vectoriales y matriciales |
| scipy | 1.7+ | Procesamiento de señales y filtros |
| matplotlib | 3.4+ | Visualización de resultados |
| scikit-learn | 1.0+ | Algoritmos de ML de referencia |
| pandas | 1.3+ | Manejo de datos estructurados |

### 2.1.2 Preparación de materiales para estudiantes

Se debe crear una carpeta `Modulo2_Material/` que contenga:
- Script base de simulación `ml_online_sim.py` (proporcionado por el instructor).
- Archivo de datos de prueba `adxl345_test_data.csv` (opcional, puede generarse en el script).
- Plantilla del documento de especificación del acelerador HW `especificacion_acelerador_hw.md`.
- Guía paso a paso con capturas de pantalla.

### 2.1.3 Verificación personal del instructor

**El instructor DEBE ejecutar la totalidad del Módulo 2 personalmente antes del curso**, verificando:
- La generación de datos sintéticos del ADXL345.
- El cálculo de características estadísticas en ventanas temporales.
- La ejecución del algoritmo de ML en línea.
- La conversión a punto fijo y el análisis de error de cuantización.
- La generación del documento de especificación del acelerador HW.

---

## 2.2 Conceptos teóricos que el instructor debe dominar y explicar

### 2.2.1 Aprendizaje Automático en Línea vs. Entrenamiento Offline (20 min)

Se presenta un diagrama comparativo:

```
┌─────────────────────────────────────────────────────────────┐
│           ENTRENAMIENTO OFFLINE (tradicional)               │
│                                                             │
│   Dataset completo → Entrenamiento → Modelo fijo → Deploy   │
│                                                             │
│   • Requiere dataset completo antes del entrenamiento       │
│   • Modelo no se adapta después del despliegue              │
│   • No maneja cambios en la distribución de datos           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│           APRENDIZAJE EN LÍNEA (Online ML)                  │
│                                                             │
│   Muestra 1 → Update → Muestra 2 → Update → Muestra N       │
│                                                             │
│   • Procesa muestras una a la vez (streaming)               │
│   • Modelo se actualiza continuamente                       │
│   • Se adapta a cambios en tiempo real                      │
│   • Requiere baja complejidad computacional por muestra     │
└─────────────────────────────────────────────────────────────┘
```

**Conceptos clave a explicar:**
- **Online ML:** El modelo se actualiza incrementalmente con cada nueva muestra recibida. No requiere almacenar el dataset completo.
- **Aplicación al proyecto:** El ADXL345 genera un stream continuo de muestras de aceleración. El modelo debe adaptarse en tiempo real a cambios en el patrón de vibración o movimiento.
- **Restricción de sistemas embebidos:** Los algoritmos deben tener complejidad O(1) o O(log n) por muestra, uso de memoria constante y operaciones compatibles con aritmética de punto fijo.

### 2.2.2 Extracción de características estadísticas (20 min)

Para el procesamiento del acelerómetro, se definen las características que se calcularán en ventanas temporales:

```
┌─────────────────────────────────────────────────────────────┐
│          VENTANA TEMPORAL (N muestras)                      │
│                                                             │
│   x[0], x[1], x[2], ..., x[N-1]                            │
│                                                             │
│   Características a calcular:                               │
│   • Media:     μ = (1/N) Σ x[i]                            │
│   • Varianza:  σ² = (1/N) Σ (x[i] - μ)²                   │
│   • RMS:       x_rms = √((1/N) Σ x[i]²)                   │
│   • Magnitud:  |a| = √(ax² + ay² + az²)                    │
└─────────────────────────────────────────────────────────────┘
```

**Justificación técnica:** Estas características son computacionalmente eficientes y capturan información relevante del comportamiento del acelerómetro. La varianza y el RMS son especialmente útiles para detectar cambios en el patrón de vibración, lo cual alimenta el algoritmo de ML.

**Operaciones matemáticas involucradas:**
- Sumas acumulativas (acumuladores)
- Multiplicaciones (para cuadrados y productos)
- Divisiones (por N, constante conocida)
- Raíz cuadrada (para RMS y magnitud)

**Nota crítica:** Las multiplicaciones son las operaciones más costosas y las que se acelerarán en el hardware del Módulo 3 utilizando los 11 bloques DSP 18×18 del SmartFusion2 M2S005.

### 2.2.3 Cuantización de punto fijo (30 min)

La transición de Python (punto flotante) a Verilog (punto fijo) requiere una comprensión precisa del formato de representación numérica.

**Formato Qm.n:**
- **m:** bits para la parte entera (con signo)
- **n:** bits para la parte fraccionaria
- **Total de bits:** m + n + 1 (el bit de signo se cuenta aparte en algunas convenciones)

**Selección del formato para este proyecto:**

| Variable | Rango esperado | Formato Q seleccionado | Bits totales |
|---|---|---|---|
| Muestras ADXL345 (10 bits signed) | ±2g (±512) | Q9.0 (entero signed) | 10 |
| Media μ | ±2g | Q3.12 | 16 |
| Varianza σ² | 0 a 4g² | Q3.12 | 16 |
| Coeficientes ML | ±1.0 | Q1.14 | 16 |
| Productos internos | ±4 | Q3.14 | 18 (compatible con DSP) |
| Acumuladores | Variable | Q7.24 | 32 |

**Justificación del formato Q1.14:**
- El SmartFusion2 M2S005 dispone de 11 multiplicadores de 18×18 bits.
- El formato Q1.14 (1 bit signo + 1 bit entero + 14 bits fracción) = 16 bits.
- Al multiplicar dos operandos Q1.14, el resultado es Q2.28 (32 bits), que cabe en un acumulador de 32 bits.
- Este formato permite representar valores en el rango [-2.0, +1.9999] con resolución de 1/16384 ≈ 6.1×10⁻⁵.

**Conversión de punto flotante a punto fijo:**

```python
def float_to_q(value, integer_bits, fractional_bits):
    """Convierte un valor flotante a formato Qm.n"""
    scale = 2 ** fractional_bits
    q_value = int(round(value * scale))
    # Saturación al rango permitido
    max_val = (2 ** (integer_bits + fractional_bits - 1)) - 1
    min_val = -(2 ** (integer_bits + fractional_bits - 1))
    return max(min(q_value, max_val), min_val)

def q_to_float(q_value, fractional_bits):
    """Convierte un valor Qm.n a punto flotante"""
    return q_value / (2 ** fractional_bits)
```

**Análisis de error de cuantización:**
- Error máximo de redondeo: ±0.5 LSB = ±2⁻¹⁵ ≈ ±3.05×10⁻⁵
- Para una muestra del ADXL345 de 10 bits (±512 en ±2g), el error relativo es despreciable.
- Para coeficientes ML pequeños, el error puede ser significativo y debe evaluarse experimentalmente.

### 2.2.4 Algoritmo de ML en línea seleccionado (20 min)

Para este proyecto se selecciona una **regresión logística incremental con descenso de gradiente estocástico (SGD)**, por las siguientes razones:

1. **Baja complejidad computacional:** O(d) por muestra, donde d es el número de características.
2. **Actualización incremental:** Solo requiere almacenar los coeficientes actuales.
3. **Salida probabilística:** Permite tomar decisiones con umbral configurable.
4. **Compatible con punto fijo:** Las operaciones son multiplicaciones, sumas y una función sigmoide aproximable.

**Ecuaciones del algoritmo:**

```
Predicción:    ŷ = σ(wᵀx + b)
Error:         e = ŷ - y
Actualización: w ← w - η · e · x
               b ← b - η · e
```

Donde:
- **w:** vector de pesos (coeficientes)
- **x:** vector de características (μ, σ², x_rms)
- **b:** bias (sesgo)
- **η:** tasa de aprendizaje (learning rate)
- **σ:** función sigmoide: σ(z) = 1 / (1 + e⁻ᶻ)

**Aproximación de la sigmoide en punto fijo:**
La función sigmoide requiere exponenciales, que son costosas en hardware. Se utiliza una aproximación piecewise-linear:

```
σ(z) ≈ 0           si z < -4
σ(z) ≈ 0.5 + z/8   si -4 ≤ z ≤ 4
σ(z) ≈ 1           si z > 4
```

Esta aproximación es suficiente para el aprendizaje y se implementa fácilmente en hardware con comparadores y shifters.

---

## 2.3 Herramientas que deben estar instaladas y configuradas

| Herramienta | Versión | Verificación |
|---|---|---|
| Python | 3.8+ | `python --version` |
| numpy | 1.20+ | `python -c "import numpy"` |
| scipy | 1.7+ | `python -c "import scipy"` |
| matplotlib | 3.4+ | `python -c "import matplotlib"` |
| scikit-learn | 1.0+ | `python -c "import sklearn"` |
| Editor de código (VS Code, Spyder, Jupyter) | Cualquiera | Verificar ejecución de scripts |

---

## 2.4 Implementación paso a paso

### PASO 1: Generación de datos sintéticos del ADXL345 (30 min)

Se genera un dataset sintético que emula el comportamiento del ADXL345 con tres estados de movimiento: reposo, vibración suave y vibración intensa.

**Archivo a crear:** `ml_online_sim.py`

```python
import numpy as np
import matplotlib.pyplot as plt

# =====================================================================
# PARÁMETROS DEL ADXL345 (según datasheet)
# =====================================================================
FSR_G = 2.0            # Full Scale Range: ±2g
RESOLUTION_BITS = 10   # Resolución de 10 bits en modo fixed
SAMPLE_RATE_HZ = 100   # Frecuencia de muestreo (configurable hasta 3200 Hz)

# =====================================================================
# GENERACIÓN DE DATOS SINTÉTICOS
# =====================================================================
def generate_adxl345_data(n_samples_per_state=500, noise_level=0.05):
    """
    Genera datos sintéticos del ADXL345 con tres estados:
    - Estado 0: Reposo (gravedad en Z, ruido bajo)
    - Estado 1: Vibración suave (oscilaciones pequeñas)
    - Estado 2: Vibración intensa (oscilaciones grandes)
    """
    np.random.seed(42)
    
    data = []
    labels = []
    
    # Estado 0: Reposo
    for _ in range(n_samples_per_state):
        ax = np.random.normal(0, noise_level)
        ay = np.random.normal(0, noise_level)
        az = np.random.normal(1.0, noise_level)  # Gravedad en Z
        data.append([ax, ay, az])
        labels.append(0)
    
    # Estado 1: Vibración suave
    t = np.linspace(0, 5, n_samples_per_state)
    for i in range(n_samples_per_state):
        ax = 0.3 * np.sin(2 * np.pi * 2 * t[i]) + np.random.normal(0, noise_level)
        ay = 0.3 * np.cos(2 * np.pi * 2 * t[i]) + np.random.normal(0, noise_level)
        az = 1.0 + np.random.normal(0, noise_level * 2)
        data.append([ax, ay, az])
        labels.append(1)
    
    # Estado 2: Vibración intensa
    for i in range(n_samples_per_state):
        ax = 1.2 * np.sin(2 * np.pi * 5 * t[i]) + np.random.normal(0, noise_level * 3)
        ay = 1.2 * np.cos(2 * np.pi * 5 * t[i]) + np.random.normal(0, noise_level * 3)
        az = 1.0 + np.random.normal(0, noise_level * 4)
        data.append([ax, ay, az])
        labels.append(2)
    
    # Cuantizar a 10 bits signed (como el ADXL345 real)
    data = np.array(data)
    scale_factor = (2 ** (RESOLUTION_BITS - 1) - 1) / FSR_G
    data_quantized = np.round(data * scale_factor).astype(np.int16)
    
    return data_quantized, np.array(labels)

# Generar datos
X_raw, y_true = generate_adxl345_data()

print(f"Datos generados: {X_raw.shape[0]} muestras, {X_raw.shape[1]} ejes")
print(f"Rango de valores: [{X_raw.min()}, {X_raw.max()}]")
print(f"Distribución de clases: {np.bincount(y_true)}")

# Visualizar datos crudos
fig, axes = plt.subplots(3, 1, figsize=(12, 8))
for i, label in enumerate(['Reposo', 'Vibración Suave', 'Vibración Intensa']):
    mask = (y_true == i)
    idx = np.where(mask)[0]
    axes[i].plot(idx, X_raw[mask, 0], label='AX', alpha=0.7)
    axes[i].plot(idx, X_raw[mask, 1], label='AY', alpha=0.7)
    axes[i].plot(idx, X_raw[mask, 2], label='AZ', alpha=0.7)
    axes[i].set_title(f'Estado {i}: {label}')
    axes[i].set_ylabel('Aceleración (cuentas ADXL)')
    axes[i].legend()
plt.tight_layout()
plt.savefig('adxl345_raw_data.png', dpi=150)
plt.show()
```

**Resultado esperado:**
- Se generan 1500 muestras (500 por estado).
- Los valores están cuantizados a 10 bits signed, en el rango [-512, 511] para ±2g.
- Se visualizan tres gráficas con los tres estados claramente diferenciados.
- Se guarda la imagen `adxl345_raw_data.png`.

**Verificación:**
- El rango de valores debe estar entre -512 y 511.
- La distribución de clases debe ser 500-500-500.
- Las gráficas deben mostrar diferencias claras entre los tres estados.

---

### PASO 2: Extracción de características estadísticas en ventanas (45 min)

Se implementa el cálculo de características en ventanas temporales deslizantes.

**Agregar al script `ml_online_sim.py`:**

```python
# =====================================================================
# EXTRACCIÓN DE CARACTERÍSTICAS EN VENTANAS
# =====================================================================
WINDOW_SIZE = 50  # Tamaño de ventana (0.5 segundos a 100 Hz)

def extract_features(window):
    """
    Extrae características estadísticas de una ventana de muestras.
    
    Entrada: window - array de forma (WINDOW_SIZE, 3) con AX, AY, AZ
    Salida: features - array de 6 características:
        [mean_mag, var_mag, rms_mag, mean_ax, var_ax, rms_ax]
    """
    # Magnitud de la aceleración para cada muestra
    magnitude = np.sqrt(window[:, 0]**2 + window[:, 1]**2 + window[:, 2]**2)
    
    # Características de la magnitud
    mean_mag = np.mean(magnitude)
    var_mag = np.var(magnitude)
    rms_mag = np.sqrt(np.mean(magnitude**2))
    
    # Características del eje X (se pueden extender a Y, Z)
    mean_ax = np.mean(window[:, 0])
    var_ax = np.var(window[:, 0])
    rms_ax = np.sqrt(np.mean(window[:, 0]**2))
    
    return np.array([mean_mag, var_mag, rms_mag, mean_ax, var_ax, rms_ax])

def compute_features_stream(X_raw, window_size=WINDOW_SIZE):
    """
    Calcula características en ventanas deslizantes a lo largo del stream.
    """
    n_samples = X_raw.shape[0]
    features = []
    
    for i in range(0, n_samples - window_size + 1, window_size // 2):  # 50% overlap
        window = X_raw[i:i + window_size]
        feat = extract_features(window)
        features.append(feat)
    
    return np.array(features)

# Calcular características
X_features = compute_features_stream(X_raw)

print(f"Características calculadas: {X_features.shape[0]} ventanas")
print(f"Dimensionalidad: {X_features.shape[1]} características por ventana")
print(f"Ejemplo de características (primera ventana): {X_features[0]}")

# Visualizar características
fig, axes = plt.subplots(2, 3, figsize=(15, 8))
feature_names = ['Media Magnitud', 'Varianza Magnitud', 'RMS Magnitud',
                 'Media AX', 'Varianza AX', 'RMS AX']
for i, (ax, name) in enumerate(zip(axes.flatten(), feature_names)):
    ax.plot(X_features[:, i])
    ax.set_title(name)
    ax.set_xlabel('Ventana')
    ax.set_ylabel('Valor')
    ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('features_extracted.png', dpi=150)
plt.show()
```

**Resultado esperado:**
- Se calculan características para cada ventana de 50 muestras.
- Con 50% de overlap, se generan aproximadamente 29 ventanas.
- Las 6 características deben mostrar patrones diferenciados entre los tres estados.

**Verificación:**
- El número de ventanas debe ser aproximadamente `(1500 - 50) / 25 + 1 = 59`.
- Las características del estado 2 (vibración intensa) deben tener varianza y RMS significativamente mayores.
- Las gráficas deben mostrar tres regiones claramente diferenciadas.

**Operaciones matemáticas identificadas para aceleración HW:**

| Operación | Cantidad por ventana | Aceleración en DSP |
|---|---|---|
| Multiplicaciones (cuadrados) | 3 ejes × 50 muestras + 50 magnitudes = 200 | Sí, en paralelo |
| Sumas acumulativas | 6 características × 50 muestras = 300 | Parcialmente |
| Divisiones por N | 6 (constante, se implementa como shift) | No requiere DSP |
| Raíces cuadradas | 3 (RMS) | Aproximación en HW o lookup table |

---

### PASO 3: Implementación del algoritmo de ML en línea (60 min)

Se implementa la regresión logística incremental con SGD.

**Agregar al script `ml_online_sim.py`:**

```python
# =====================================================================
# ALGORITMO DE ML EN LÍNEA: REGRESIÓN LOGÍSTICA INCREMENTAL
# =====================================================================
class OnlineLogisticRegression:
    """
    Regresión logística con actualización incremental (SGD).
    Implementa aprendizaje en línea: procesa una muestra a la vez.
    """
    def __init__(self, n_features, learning_rate=0.01, n_classes=3):
        self.n_features = n_features
        self.lr = learning_rate
        self.n_classes = n_classes
        
        # Inicialización de pesos (one-vs-rest)
        self.weights = np.zeros((n_classes, n_features))
        self.bias = np.zeros(n_classes)
        
        # Historial para visualización
        self.loss_history = []
        self.accuracy_history = []
    
    def sigmoid(self, z):
        """Función sigmoide numéricamente estable"""
        return np.where(z >= 0,
                        1 / (1 + np.exp(-z)),
                        np.exp(z) / (1 + np.exp(z)))
    
    def predict_proba(self, x):
        """Predice probabilidades para una muestra"""
        z = np.dot(self.weights, x) + self.bias
        return self.sigmoid(z)
    
    def predict(self, x):
        """Predice la clase para una muestra"""
        proba = self.predict_proba(x)
        return np.argmax(proba)
    
    def partial_fit(self, x, y):
        """
        Actualiza el modelo con una sola muestra (aprendizaje en línea).
        """
        # Forward pass
        proba = self.predict_proba(x)
        
        # One-hot encoding de la etiqueta verdadera
        y_onehot = np.zeros(self.n_classes)
        y_onehot[y] = 1.0
        
        # Error
        error = proba - y_onehot
        
        # Actualización de pesos (SGD)
        self.weights -= self.lr * np.outer(error, x)
        self.bias -= self.lr * error
        
        # Cross-entropy loss
        loss = -np.sum(y_onehot * np.log(proba + 1e-10))
        self.loss_history.append(loss)
    
    def evaluate(self, X, y):
        """Evalúa la precisión en un conjunto de datos"""
        predictions = np.array([self.predict(x) for x in X])
        accuracy = np.mean(predictions == y)
        self.accuracy_history.append(accuracy)
        return accuracy

# =====================================================================
# ENTRENAMIENTO EN LÍNEA
# =====================================================================
# Normalizar características (importante para convergencia)
X_mean = np.mean(X_features, axis=0)
X_std = np.std(X_features, axis=0) + 1e-8
X_norm = (X_features - X_mean) / X_std

# Generar etiquetas por ventana (etiqueta del centro de la ventana)
y_windows = []
for i in range(0, len(X_raw) - WINDOW_SIZE + 1, WINDOW_SIZE // 2):
    center_idx = i + WINDOW_SIZE // 2
    y_windows.append(y_true[center_idx])
y_windows = np.array(y_windows)

# Crear modelo
model = OnlineLogisticRegression(n_features=6, learning_rate=0.1, n_classes=3)

# Entrenamiento en línea: procesar una ventana a la vez
print("\n=== Entrenamiento en Línea ===")
for i, (x, y) in enumerate(zip(X_norm, y_windows)):
    model.partial_fit(x, y)
    
    if (i + 1) % 10 == 0:
        acc = model.evaluate(X_norm[:i+1], y_windows[:i+1])
        print(f"Ventana {i+1}: Precisión acumulada = {acc:.4f}")

# Evaluación final
final_accuracy = model.evaluate(X_norm, y_windows)
print(f"\nPrecisión final: {final_accuracy:.4f}")
print(f"Pesos aprendidos:\n{model.weights}")
print(f"Bias aprendido: {model.bias}")

# Visualizar evolución del aprendizaje
fig, axes = plt.subplots(1, 2, figsize=(14, 5))
axes[0].plot(model.loss_history)
axes[0].set_title('Evolución de la Pérdida (Cross-Entropy)')
axes[0].set_xlabel('Iteración (ventana)')
axes[0].set_ylabel('Pérdida')
axes[0].grid(True, alpha=0.3)

# Precisión acumulada
acc_history = []
for i in range(0, len(X_norm), 5):
    acc = model.evaluate(X_norm[:i+1], y_windows[:i+1])
    acc_history.append(acc)
axes[1].plot(range(0, len(X_norm), 5), acc_history)
axes[1].set_title('Evolución de la Precisión')
axes[1].set_xlabel('Iteración (ventana)')
axes[1].set_ylabel('Precisión')
axes[1].grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('online_learning_evolution.png', dpi=150)
plt.show()
```

**Resultado esperado:**
- El modelo procesa una ventana a la vez, actualizando sus pesos incrementalmente.
- La pérdida disminuye progresivamente.
- La precisión acumulada aumenta hasta estabilizarse (esperado > 85%).
- Se visualizan las gráficas de evolución.

**Verificación:**
- La precisión final debe ser superior al 85%.
- La pérdida debe mostrar tendencia decreciente.
- Los pesos aprendidos deben ser valores finitos (no NaN ni Inf).

**Análisis de complejidad por muestra:**
- Predicción: 6 multiplicaciones + 6 sumas + 3 sigmoides = O(d)
- Actualización: 18 multiplicaciones + 18 sumas = O(d × n_classes)
- Total: aproximadamente 30 operaciones aritméticas por ventana
- **Estas operaciones son las que se acelerarán en el Módulo 3.**

---

### PASO 4: Análisis de cuantización de punto fijo (60 min)

Se realiza la conversión del algoritmo a aritmética de punto fijo y se analiza el error introducido.

**Agregar al script `ml_online_sim.py`:**

```python
# =====================================================================
# CUANTIZACIÓN DE PUNTO FIJO
# =====================================================================
class FixedPointQuantizer:
    """
    Clase para manejar la cuantización de punto fijo formato Qm.n
    """
    def __init__(self, integer_bits, fractional_bits):
        self.integer_bits = integer_bits
        self.fractional_bits = fractional_bits
        self.total_bits = integer_bits + fractional_bits
        self.scale = 2 ** fractional_bits
        self.max_val = (2 ** (self.total_bits - 1)) - 1
        self.min_val = -(2 ** (self.total_bits - 1))
    
    def quantize(self, float_value):
        """Convierte flotante a punto fijo con saturación"""
        q_val = int(round(float_value * self.scale))
        return max(min(q_val, self.max_val), self.min_val)
    
    def dequantize(self, q_value):
        """Convierte punto fijo a flotante"""
        return q_value / self.scale

# =====================================================================
# DEFINICIÓN DE FORMATOS DE PUNTO FIJO
# =====================================================================
# Formato para características (salida de la extracción)
# Las características normalizadas están en rango aproximado [-3, +3]
QP_FEAT = FixedPointQuantizer(integer_bits=3, fractional_bits=12)  # Q3.12, 16 bits

# Formato para pesos del modelo (coeficientes ML)
# Los pesos suelen estar en rango [-2, +2]
QP_WEIGHT = FixedPointQuantizer(integer_bits=2, fractional_bits=13)  # Q2.13, 16 bits

# Formato para productos internos (resultado de w^T * x)
# Producto de Q3.12 × Q2.13 = Q5.25, pero se trunca a Q3.14 (18 bits, compatible con DSP)
QP_PROD = FixedPointQuantizer(integer_bits=3, fractional_bits=14)  # Q3.14, 18 bits

# Formato para acumuladores
QP_ACC = FixedPointQuantizer(integer_bits=7, fractional_bits=24)  # Q7.24, 32 bits

print("\n=== Análisis de Cuantización ===")
print(f"Formato características: Q{QP_FEAT.integer_bits}.{QP_FEAT.fractional_bits} "
      f"({QP_FEAT.total_bits} bits), rango [{QP_FEAT.min_val/QP_FEAT.scale:.4f}, {QP_FEAT.max_val/QP_FEAT.scale:.4f}]")
print(f"Formato pesos: Q{QP_WEIGHT.integer_bits}.{QP_WEIGHT.fractional_bits} "
      f"({QP_WEIGHT.total_bits} bits), rango [{QP_WEIGHT.min_val/QP_WEIGHT.scale:.4f}, {QP_WEIGHT.max_val/QP_WEIGHT.scale:.4f}]")
print(f"Formato productos: Q{QP_PROD.integer_bits}.{QP_PROD.fractional_bits} "
      f"({QP_PROD.total_bits} bits, compatible con DSP 18×18)")

# =====================================================================
# IMPLEMENTACIÓN EN PUNTO FIJO
# =====================================================================
class OnlineLogisticRegressionFixedPoint:
    """
    Versión en punto fijo de la regresión logística incremental.
    """
    def __init__(self, n_features, learning_rate_q, n_classes=3):
        self.n_features = n_features
        self.lr_q = learning_rate_q  # learning rate ya cuantizado
        self.n_classes = n_classes
        
        # Pesos en punto fijo (formato Q2.13)
        self.weights_q = np.zeros((n_classes, n_features), dtype=np.int32)
        self.bias_q = np.zeros(n_classes, dtype=np.int32)
    
    def sigmoid_approx_q(self, z_q):
        """
        Aproximación piecewise-linear de la sigmoide en punto fijo.
        Entrada: z_q en formato Q3.14
        Salida: valor en formato Q0.15 (rango [0, 1])
        """
        # Convertir a flotante para la aproximación (en HW se haría con comparadores)
        z_float = z_q / QP_PROD.scale
        
        if z_float < -4:
            return 0
        elif z_float > 4:
            return int(0.999 * (2**15))
        else:
            # Aproximación lineal: 0.5 + z/8
            result_float = 0.5 + z_float / 8.0
            return int(result_float * (2**15))
    
    def predict_q(self, x_q):
        """
        Predice la clase usando aritmética de punto fijo.
        x_q: características en formato Q3.12
        """
        scores_q = np.zeros(self.n_classes, dtype=np.int32)
        
        for c in range(self.n_classes):
            # Producto punto en punto fijo
            # w_q (Q2.13) × x_q (Q3.12) = Q5.25
            # Se requiere shift right por 11 bits para obtener Q3.14
            acc_q = 0
            for i in range(self.n_features):
                product = int(self.weights_q[c, i]) * int(x_q[i])
                acc_q += product
            
            # Shift para ajustar formato: Q5.25 → Q3.14 (shift right 11)
            z_q = (acc_q >> 11) + self.bias_q[c]
            scores_q[c] = z_q
        
        return np.argmax(scores_q)
    
    def partial_fit_q(self, x_q, y):
        """
        Actualiza el modelo con una muestra en punto fijo.
        """
        # Forward pass
        proba_q = np.zeros(self.n_classes, dtype=np.int32)
        scores_q = np.zeros(self.n_classes, dtype=np.int32)
        
        for c in range(self.n_classes):
            acc_q = 0
            for i in range(self.n_features):
                product = int(self.weights_q[c, i]) * int(x_q[i])
                acc_q += product
            z_q = (acc_q >> 11) + self.bias_q[c]
            scores_q[c] = z_q
            proba_q[c] = self.sigmoid_approx_q(z_q)
        
        # Error (proba - y_onehot) en formato Q0.15
        error_q = np.zeros(self.n_classes, dtype=np.int32)
        for c in range(self.n_classes):
            target_q = int(1.0 * (2**15)) if c == y else 0
            error_q[c] = proba_q[c] - target_q
        
        # Actualización de pesos
        # error_q (Q0.15) × x_q (Q3.12) = Q3.27
        # Se requiere shift para ajustar a Q2.13 (shift right 14)
        for c in range(self.n_classes):
            for i in range(self.n_features):
                update = (int(error_q[c]) * int(x_q[i])) >> 14
                update = (update * self.lr_q) >> 15
                self.weights_q[c, i] -= update
            self.bias_q[c] -= (int(error_q[c]) * self.lr_q) >> 15

# =====================================================================
# ENTRENAMIENTO EN PUNTO FIJO
# =====================================================================
# Cuantizar características
X_norm_q = np.zeros_like(X_norm, dtype=np.int32)
for i in range(X_norm.shape[0]):
    for j in range(X_norm.shape[1]):
        X_norm_q[i, j] = QP_FEAT.quantize(X_norm[i, j])

# Cuantizar learning rate (0.1 en Q0.15)
lr_q = int(0.1 * (2**15))

# Crear modelo en punto fijo
model_q = OnlineLogisticRegressionFixedPoint(n_features=6, learning_rate_q=lr_q, n_classes=3)

# Entrenamiento en línea (punto fijo)
print("\n=== Entrenamiento en Punto Fijo ===")
correct_count = 0
for i, (x_q, y) in enumerate(zip(X_norm_q, y_windows)):
    pred = model_q.predict_q(x_q)
    if pred == y:
        correct_count += 1
    model_q.partial_fit_q(x_q, y)
    
    if (i + 1) % 10 == 0:
        print(f"Ventana {i+1}: Precisión acumulada = {correct_count/(i+1):.4f}")

final_accuracy_q = correct_count / len(X_norm_q)
print(f"\nPrecisión final (punto fijo): {final_accuracy_q:.4f}")

# =====================================================================
# COMPARACIÓN: FLOTANTE vs PUNTO FIJO
# =====================================================================
print("\n=== Comparación de Precisión ===")
print(f"Precisión (flotante):   {final_accuracy:.4f}")
print(f"Precisión (punto fijo): {final_accuracy_q:.4f}")
print(f"Diferencia:             {abs(final_accuracy - final_accuracy_q):.4f}")

# Análisis de error de cuantización
print("\n=== Análisis de Error de Cuantización ===")
max_feat_error = 0
max_weight_error = 0
for i in range(X_norm.shape[0]):
    for j in range(X_norm.shape[1]):
        q_val = QP_FEAT.dequantize(X_norm_q[i, j])
        error = abs(X_norm[i, j] - q_val)
        max_feat_error = max(max_feat_error, error)

for c in range(3):
    for i in range(6):
        q_val = QP_WEIGHT.dequantize(int(model.weights[c, i] * QP_WEIGHT.scale))
        error = abs(model.weights[c, i] - q_val / QP_WEIGHT.scale)
        max_weight_error = max(max_weight_error, error)

print(f"Error máximo de cuantización en características: {max_feat_error:.6f}")
print(f"Error máximo de cuantización en pesos: {max_weight_error:.6f}")
print(f"Resolución de características (1 LSB): {1/QP_FEAT.scale:.6f}")
print(f"Resolución de pesos (1 LSB): {1/QP_WEIGHT.scale:.6f}")
```

**Resultado esperado:**
- El modelo en punto fijo alcanza una precisión comparable al modelo en punto flotante (diferencia < 5%).
- Los errores de cuantización son pequeños comparados con la resolución de los formatos.
- Se verifica que los formatos Q3.12 y Q2.13 son adecuados para la aplicación.

**Verificación:**
- La precisión del modelo en punto fijo debe ser superior al 80%.
- La diferencia con el modelo en punto flotante debe ser menor al 5%.
- Los errores de cuantización deben estar dentro de 1 LSB de cada formato.

**Si la precisión en punto fijo es demasiado baja:**
- Aumentar el número de bits fraccionarios (ej: Q3.14 en lugar de Q3.12).
- Reducir el learning rate.
- Aumentar el número de iteraciones de entrenamiento.

---

### PASO 5: Generación del documento de especificación del acelerador HW (45 min)

El entregable más importante del Módulo 2 es el documento de especificación que servirá como entrada para el Módulo 3.

**Archivo a crear:** `especificacion_acelerador_hw.md`

```markdown
# Especificación del Acelerador de Hardware para ML en FPGA

## 1. Descripción General

El acelerador de hardware debe implementar la extracción de características estadísticas
y el producto punto de la regresión logística en punto fijo, aprovechando los 11 bloques
DSP 18×18 del SmartFusion2 M2S005.

## 2. Funcionalidades Requeridas

### 2.1 Módulo de Extracción de Características
- **Entrada:** Stream de muestras del ADXL345 (3 ejes × 10 bits signed)
- **Ventana:** 50 muestras (configurable)
- **Salida:** 6 características en formato Q3.12 (16 bits signed)
  - mean_mag, var_mag, rms_mag
  - mean_ax, var_ax, rms_ax

### 2.2 Módulo de Producto Punto (para ML)
- **Entrada:** Vector de características (6 × Q3.12) y vector de pesos (6 × Q2.13)
- **Operación:** 6 multiplicaciones 16×16 + acumulación
- **Salida:** Score en formato Q3.14 (18 bits, compatible con DSP)

## 3. Interfaz de Hardware

### 3.1 Interfaz con el Bus AMBA APB
- **Dirección base:** 0x4000_0000 (configurable)
- **Registros:**
  - 0x00: Control (start, reset, mode)
  - 0x04: Status (busy, done, error)
  - 0x08: Data In (muestra del ADXL345)
  - 0x0C-0x20: Características (6 × 16 bits)
  - 0x24-0x38: Pesos (6 × 16 bits, reconfigurables)
  - 0x3C: Score de salida (18 bits)

### 3.2 Señales de Control
- clk: 50 MHz (reloj del sistema)
- reset_n: reset activo bajo
- start: inicia el cálculo
- done: indica fin del cálculo
- busy: indica operación en progreso

## 4. Formato de Datos

| Variable | Formato | Bits | Rango |
|---|---|---|---|
| Muestra ADXL345 | Entero signed | 10 | [-512, 511] |
| Característica | Q3.12 | 16 | [-8.0, +7.9998] |
| Peso ML | Q2.13 | 16 | [-4.0, +3.9999] |
| Producto | Q5.25 | 32 | (intermedio) |
| Score | Q3.14 | 18 | [-8.0, +7.9999] |

## 5. Recursos DSP Requeridos

- **Extracción de características:** 3 multiplicadores en paralelo (para los 3 ejes)
- **Producto punto ML:** 6 multiplicadores en paralelo (para las 6 características)
- **Total:** 9 de 11 multiplicadores disponibles (factible)

## 6. Rendimiento Esperado

- **Latencia extracción:** 50 ciclos (una muestra por ciclo) + 10 ciclos (características)
- **Latencia producto punto:** 3 ciclos (pipeline de 6 multiplicaciones en paralelo)
- **Throughput:** ~1 ventana cada 60 ciclos = ~833 ventanas/segundo a 50 MHz

## 7. Restricciones

- Todos los cálculos deben usar aritmética de punto fijo.
- No se permite el uso de divisores (las divisiones por N se implementan como shifts).
- La raíz cuadrada (para RMS) debe aproximarse con lookup table o iteración.
- El diseño debe sintetizar sin errores en Libero SoC v12+.

## 8. Validación

El acelerador debe validarse comparando sus salidas con las del modelo Python en punto fijo.
El error máximo permitido es de 1 LSB en cada característica.
```

**Resultado esperado:**
- Documento completo con todas las especificaciones técnicas.
- Los formatos de punto fijo están claramente definidos.
- La interfaz AMBA APB está especificada.
- Los recursos DSP requeridos están cuantificados.

**Verificación:**
- El documento debe ser autocontenido y comprensible para el diseñador de hardware.
- Todos los parámetros deben estar justificados con los resultados del PASO 4.
- La interfaz debe ser compatible con el MSS Configurator del SmartFusion2.

---

## 2.5 Verificación final del Módulo 2

Al finalizar el Módulo 2, se debe contar con:

| Criterio | Verificación |
|---|---|
| Datos sintéticos generados correctamente | ✅ Rango [-512, 511], 1500 muestras |
| Características extraídas en ventanas | ✅ 6 características por ventana |
| Modelo ML entrenado en línea (flotante) | ✅ Precisión > 85% |
| Modelo ML entrenado en línea (punto fijo) | ✅ Precisión > 80%, diferencia < 5% |
| Errores de cuantización analizados | ✅ Dentro de 1 LSB |
| Documento de especificación HW generado | ✅ Completo y autocontenido |
| Gráficas de resultados guardadas | ✅ PNG en directorio de trabajo |

---

## 2.6 Entregables del Módulo 2 (conservar para Módulos siguientes)

| Entregable | Uso futuro |
|---|---|
| Script `ml_online_sim.py` | Referencia para validación del Módulo 3 |
| Documento `especificacion_acelerador_hw.md` | **Entrada principal del Módulo 3** |
| Formatos de punto fijo definidos (Q3.12, Q2.13, Q3.14) | Se utilizan directamente en Verilog |
| Pesos aprendidos del modelo | Se cargan en los registros del acelerador |
| Datos de prueba `X_norm_q`, `y_windows` | Se usan para validar el HW |
| Gráficas de resultados | Evidencia del funcionamiento |

---

## 2.7 Errores esperables y diagnóstico

| Error | Causa probable | Solución |
|---|---|---|
| `ModuleNotFoundError` | Paquete no instalado | `pip install <paquete>` |
| Precisión del modelo flotante < 80% | Learning rate inadecuado o datos no normalizados | Ajustar learning rate, verificar normalización |
| Precisión del modelo punto fijo muy baja | Formato Q con pocos bits fraccionarios | Aumentar bits fraccionarios (ej: Q3.14) |
| Overflow en cuantización | Rango del formato Q insuficiente | Verificar rango de datos, ajustar integer_bits |
| NaN o Inf en pesos | Learning rate demasiado alto | Reducir learning rate, agregar gradient clipping |
| Diferencia flotante/punto fijo > 10% | Error de cuantización acumulado | Revisar shifts en productos, verificar formatos |
| Especificación HW incompleta | Falta de detalle en interfaz o formatos | Completar todas las secciones del documento |

---

## 2.8 Resumen del Día 2

**Antes del Día 2:** Python y paquetes instalados. Materiales preparados.

**Durante el Día 2 (4 horas):**
- 0:00–0:30: Teoría de Online ML y extracción de características
- 0:30–1:15: Generación de datos sintéticos y extracción de características (PASOS 1-2)
- 1:15–2:15: Implementación del algoritmo ML en línea (PASO 3)
- 2:15–3:15: Análisis de cuantización de punto fijo (PASO 4)
- 3:15–4:00: Generación del documento de especificación HW (PASO 5)

**Al final del Día 2:** Script Python funcional con modelo ML entrenado en línea, análisis de cuantización completado, y documento de especificación del acelerador HW listo para ser utilizado como entrada del Módulo 3.

---

## 2.9 Relación con el Módulo 3

El Módulo 2 proporciona al Módulo 3:

1. **Especificación funcional completa** del acelerador HW (documento `especificacion_acelerador_hw.md`).
2. **Formatos de punto fijo** exactos (Q3.12, Q2.13, Q3.14) que deben implementarse en Verilog.
3. **Interfaz AMBA APB** definida con registros y direcciones.
4. **Datos de prueba** cuantizados para validar el hardware.
5. **Modelo de referencia** en Python para comparar con las salidas del HW.
6. **Justificación de recursos DSP** (9 de 11 multiplicadores utilizados).

El Módulo 3 deberá implementar en Verilog:
- El módulo de extracción de características (media, varianza, RMS) en punto fijo.
- El módulo de producto punto para la inferencia ML.
- La interfaz AMBA APB según la especificación.
- La integración en el bus del sistema del SmartFusion2 M2S005.

---
