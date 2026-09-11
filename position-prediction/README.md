# Desafío: Predicción de Posición de Drones

Dada una secuencia de posiciones registradas de un dron, estima las siguientes **n** posiciones a partir de las **k** observaciones anteriores. Los datos proporcionados provienen de un vuelo sintético de un dron con una tasa de muestreo constante. Tu tarea es construir un pipeline de predicción que ingiera una ventana deslizante de posiciones pasadas y pronostique la trayectoria futura del dron.

---

## Resumen

Los drones dependen de la estimación a bordo para mantener un vuelo estable. En la práctica, los datos de los sensores son ruidosos, tienen retrasos y, a veces, están incompletos. Un enfoque de ingeniería común es combinar un modelo de movimiento con observaciones ruidosas para producir una estimación suave y precisa del estado del vehículo — y proyectar ese estado hacia adelante en el tiempo.

Este desafío te entrega una grabación en CSV de la posición de un dron a lo largo del tiempo. Sabes **lo que pasó** — tu trabajo es descubrir **lo que pasará después**, usando solo una ventana de observaciones pasadas.

No existe un único enfoque correcto. Puedes usar un estimador clásico, un modelo de aprendizaje automático, o algo intermedio. El camino sugerido usa un **filtro de Kalman con un modelo de movimiento uniformemente acelerado**, que es muy adecuado para este problema y accesible para principiantes. Pero si te sientes seguro explorando otros métodos, adelante.

---

## Materiales Proporcionados

| Archivo | Descripción |
|------|-------------|
| `flight_data.csv` | Grabación en serie temporal de la posición 3-D del dron durante el vuelo. |

### Formato del CSV

El archivo contiene las siguientes columnas:

| Columna | Unidad | Descripción |
|--------|------|-------------|
| `t` | s | Marca de tiempo (segundos desde el inicio de la grabación). Período de muestreo constante. |
| `x` | m | Posición Norte en el marco local NED. |
| `y` | m | Posición Este en el marco local NED. |
| `z` | m | Posición Abajo en el marco local NED (positiva hacia abajo). |

Ejemplo (primeras filas):

```
t,x,y,z
0.0,0.006,0.01,0.98
0.02,-0.021,-0.012,0.993
0.04,0.015,0.025,1.006
0.06,0.019,-0.013,0.998
```

> **Nota:** Las posiciones contienen ruido de sensor realista. Esto es intencional — un dron real nunca reporta posiciones perfectas. Tu método debe ser robusto frente a este ruido.

---

## Declaración del Problema

Dado:
- Una ventana de **k** observaciones de posición consecutivas del CSV: `(t_{i-k}, x_{i-k}), ..., (t_i, x_i)` para cada eje.
- Un horizonte de predicción de **n** pasos hacia el futuro.

Produce:
- Las posiciones **(x, y, z)** predichas para los siguientes **n** pasos de tiempo: `(t_{i+1}, x_{i+1}), ..., (t_{i+n}, x_{i+n})`.

El horizonte de predicción **n** no es fijo — tú lo eliges. Un punto de partida razonable es **n = 10** (0,2 s, es decir, diez pasos a la tasa de muestreo de 50 Hz). También debes evaluar cómo se degradan tus predicciones a medida que **n** aumenta.

---

## Enfoque Sugerido: Filtro de Kalman con Aceleración Constante

El método recomendado modela al dron como una masa puntual que sufre **movimiento uniformemente acelerado** — la aceleración se mantiene aproximadamente constante entre pasos de muestreo, y sus cambios lentos y suaves se modelan como ruido de proceso. Esto encaja bien con los drones porque, entre correcciones de control, el movimiento de un dron está gobernado aproximadamente por aceleración constante.

### ¿Qué es un filtro de Kalman?

En su esencia, un filtro de Kalman es un estimador recursivo que combina **dos fuentes de información** para reconstruir el estado oculto de un sistema (aquí: posición, velocidad y aceleración del dron):

- **La predicción del modelo** — dónde esperamos que esté el dron según la física del movimiento y el estado anterior.
- **La medición** — lo que dice el sensor, que siempre trae algo de ruido.

Ambas fuentes son imperfectas, así que el filtro las pondera con la **ganancia de Kalman** `K`: un número que, en cada paso, decide cuánto confiar en la observación frente a la predicción. Si el sensor es confiable (ruido pequeño), `K` es grande y el filtro sigue a las mediciones; si la medición es ruidosa, `K` es pequeño y el filtro se apoya más en el modelo.

El filtro hace un ciclo de dos pasos en cada instante de tiempo:

1. **Predecir** — propaga el estado y la incertidumbre (`P`) hacia adelante usando el modelo de movimiento.
2. **Actualizar** — cuando llega una nueva observación, corrige el estado mezclándola con la predicción según `K`.

La ventaja clave sobre un simple suavizado es que el filtro mantiene una **estimación de su propia incertidumbre** (`P`): sabe cuánto puede confiar en su estado actual. Y como la predicción de futuro es simplemente repetir el paso de predicción sin actualizaciones, esa misma incertidumbre te dice automáticamente **cuánto se degrada el pronóstico** a medida que el horizonte `n` crece.

### ¿Por qué este modelo?

Un modelo dinámico completo del dron (empuje, torque, aerodinámica, respuesta del motor) es complejo y requiere parámetros que no tienes. Un modelo de movimiento uniformemente acelerado es un punto intermedio: más expresivo que el de velocidad constante, más simple que la dinámica completa, y maneja de forma natural los giros y cambios de velocidad mediante su estado de aceleración.

### Vector de Estado

Para cada eje de forma independiente, el estado en el tiempo `t` es:

```
s(t) = [ position(t) ]
       [ velocity(t) ]
       [ acceleration(t) ]
```

Así, para movimiento 3-D ejecutas tres filtros independientes (uno por eje) o los apilas en un vector de estado de 9 dimensiones.

### Paso de Predicción (Propagación del Estado)

Entre observaciones, el estado evoluciona según las ecuaciones cinemáticas para aceleración constante:

```
position(t + Δt) = position(t) + velocity(t) · Δt + 0.5 · acceleration(t) · Δt²
velocity(t + Δt) = velocity(t) + acceleration(t) · Δt
acceleration(t + Δt) = acceleration(t)
```

En forma matricial, la matriz de transición de estado **F** para un solo eje es:

```
F = [ 1   Δt   0.5·Δt² ]
    [ 0    1      Δt     ]
    [ 0    0       1     ]
```

Y el estado predicho es: `s_predicted = F · s`

### Ruido de Proceso

La aceleración real no es perfectamente constante — el controlador del dron hace correcciones, el viento lo empuja, etc. Esta incertidumbre se modela añadiendo **ruido de proceso** al componente de aceleración. Un enfoque común es usar un modelo de ruido blanco de aceleración constante por tramos (también llamado modelo de "ruido blanco discreto"), donde la covarianza del ruido de proceso **Q** es:

```
Q = q · [ Δt⁵/20   Δt⁴/8   Δt³/6 ]
        [ Δt⁴/8    Δt³/3   Δt²/2  ]
        [ Δt³/6    Δt²/2    Δt     ]
```

Donde `q` es la **densidad espectral** del ruido de aceleración — un parámetro de ajuste que deberás elegir. Un `q` más grande significa que esperas más variación de aceleración (filtro más receptivo pero más ruidoso). Un `q` más pequeño significa que confías más en el modelo de aceleración constante (más suave pero más lento para reaccionar).

### Modelo de Medición

La medición es la posición observada (del CSV):

```
z(t) = [ position(t) ]
```

La matriz de medición **H** selecciona la posición del estado:

```
H = [ 1  0  0 ]
```

La covarianza del ruido de medición **R** representa el ruido del sensor en las lecturas de posición. Puedes estimarla a partir de los datos (ver sección siguiente) o tratarla como un parámetro de ajuste.

### Ecuaciones del Filtro de Kalman

En cada paso de tiempo, el filtro realiza dos pasos:

**1. Predecir:**
```
s_predicted = F · s_previous
P_predicted = F · P_previous · Fᵀ + Q
```

**2. Actualizar (cuando llega una nueva observación):**
```
K = P_predicted · Hᵀ · (H · P_predicted · Hᵀ + R)⁻¹    (ganancia de Kalman)
s_updated = s_predicted + K · (z_measured - H · s_predicted)
P_updated = (I - K · H) · P_predicted
```

Donde **P** es la matriz de covarianza del estado (incertidumbre en la estimación del estado) y **K** es la ganancia de Kalman (cuánto confiar en la observación frente a la predicción).

### Predicción Multi-Paso

Para predecir **n** pasos adelante sin nuevas observaciones, simplemente aplica el paso de predicción **n** veces en sucesión, propagando el estado y la covarianza hacia adelante. Cada paso ensancha la incertidumbre (P crece) porque estás extrapolando sin corrección.

---

## Guía Paso a Paso

### Paso 1: Cargar e Inspeccionar los Datos

1. Carga `flight_data.csv` en un dataframe de Python o en un array de numpy.
2. Imprime estadísticas básicas: número de muestras, período de muestreo (comprueba si es constante), rangos de posición.
3. Grafica x, y, z vs. tiempo para entender la trayectoria visualmente.
4. Calcula diferencias finitas para estimar velocidad y aceleración — esto te ayudará a inicializar el filtro y elegir los parámetros de ruido.

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

df = pd.read_csv("flight_data.csv")
dt = df["t"].diff().median()  # período de muestreo
print(f"Período de muestreo: {dt:.4f} s  ({1/dt:.1f} Hz)")
print(f"Muestras: {len(df)}")

# Estimación rápida de velocidad (diferencias centrales)
vx = np.gradient(df["x"].values, df["t"].values)
vy = np.gradient(df["y"].values, df["t"].values)
vz = np.gradient(df["z"].values, df["t"].values)

fig, axes = plt.subplots(3, 1, figsize=(10, 6), sharex=True)
for ax, coord, vel in zip(axes, ["x", "y", "z"], [vx, vy, vz]):
    ax.plot(df["t"], df[coord], label=coord, alpha=0.7)
    ax.set_ylabel(f"{coord} [m]")
    ax.legend()
axes[-1].set_xlabel("Tiempo [s]")
plt.tight_layout()
plt.savefig("output/trajectory_overview.png", dpi=150)
plt.show()
```

### Paso 2: Estimar los Parámetros de Ruido

Antes de ejecutar el filtro, necesitas valores razonables para **R** (ruido de medición) y **q** (densidad espectral del ruido de proceso).

**Ruido de medición (R):**
- Si el dron permanece en un punto estacionario durante un tramo, la varianza de la posición en ese segmento es una buena estimación de la varianza del ruido del sensor.
- Alternativamente, calcula la varianza de la velocidad por diferencias finitas en un segmento donde el dron esté casi estacionario.

**Ruido de proceso (q):**
- Empieza con un valor pequeño (p. ej. `0.1` a `1.0`) y ajústalo observando la salida del filtro.
- Si el filtro es demasiado lento para rastrear giros, aumenta `q`. Si es demasiado ruidoso, disminuye `q`.

### Paso 3: Implementar el Filtro de Kalman

Escribe una clase o un conjunto de funciones que implemente el ciclo predecir/actualizar para un solo eje:

```python
class KalmanFilterCA:
    """Filtro de Kalman con modelo de aceleración constante (un solo eje)."""

    def __init__(self, dt, R, q):
        self.dt = dt
        self.R = R  # varianza del ruido de medición

        # Estado: [posición, velocidad, aceleración]
        self.x = np.zeros(3)

        # Matriz de transición de estado
        self.F = np.array([
            [1, dt, 0.5 * dt**2],
            [0,  1,       dt    ],
            [0,  0,        1    ]
        ])

        # Matriz de medición
        self.H = np.array([[1, 0, 0]])

        # Covarianza del ruido de proceso (modelo de ruido constante por tramos)
        self.Q = q * np.array([
            [dt**5/20, dt**4/8, dt**3/6],
            [dt**4/8,  dt**3/3, dt**2/2],
            [dt**3/6,  dt**2/2, dt     ]
        ])

        # Covarianza del estado
        self.P = np.eye(3) * 10.0  # gran incertidumbre inicial

    def predict(self):
        self.x = self.F @ self.x
        self.P = self.F @ self.P @ self.F.T + self.Q

    def update(self, z):
        z_pred = self.H @ self.x
        S = self.H @ self.P @ self.H.T + self.R
        K = self.P @ self.H.T @ np.linalg.inv(S)
        self.x = self.x + K @ (z - z_pred)
        self.P = (np.eye(3) - K @ self.H) @ self.P

    def get_state(self):
        return self.x.copy()
```

### Paso 4: Ejecutar el Filtro sobre la Trayectoria Completa

1. Instancia tres filtros (uno por eje).
2. Introduce cada observación de posición a través de predecir → actualizar.
3. Almacena el estado filtrado (posición, velocidad, aceleración) en cada paso de tiempo.
4. Grafica la posición filtrada frente a las mediciones en bruto para verificar que el filtro está suavizando el ruido correctamente.

### Paso 5: Predecir Posiciones Futuras

Después de procesar todas (o un subconjunto de) las observaciones, usa el estado actual para predecir **n** pasos adelante:

```python
def predict_future(filter_obj, n_steps):
    """Predice n pasos hacia el futuro sin nuevas observaciones."""
    predictions = []
    for _ in range(n_steps):
        filter_obj.predict()
        predictions.append(filter_obj.x[0])  # posición
    return np.array(predictions)
```

- Grafica las posiciones predichas junto con la verdad de referencia (que tienes en el CSV) para evaluar la precisión.
- Prueba diferentes valores de **n** (5, 10, 20, 50 pasos) y observa cómo se degrada la predicción.

### Paso 6: Evaluar tus Predicciones

Cuantifica el error de predicción:

| Métrica | Descripción |
|--------|-------------|
| **MAE** | Error absoluto medio por eje |
| **RMSE** | Raíz del error cuadrático medio por eje |
| **Max Error** | Error en el peor caso sobre el horizonte de predicción |

También evalúa cómo crece el error con el horizonte de predicción calculando estas métricas para n = 1, 2, 5, 10, 20, 50.

### Paso 7: Ajuste y Análisis

Experimenta con lo siguiente y documenta tus hallazgos:

- **Varía `q` (ruido de proceso):** ¿Cómo responde el filtro? Grafica filtrado vs. bruto para al menos 3 valores.
- **Varía `R` (ruido de medición):** ¿Qué pasa si sobreestimas o subestimas el ruido del sensor?
- **Varía `k` (tamaño de la ventana):** ¿Cuántas observaciones pasadas necesita el filtro antes de que las predicciones se estabilicen?
- **Diferentes patrones de movimiento:** ¿Se desempeña el filtro igualmente bien durante el vuelo estacionario, el vuelo en línea recta y los giros agresivos?

---

## Enfoques Alternativos

Si quieres explorar más allá del filtro de Kalman, considera:

| Método | Descripción | Dificultad |
|--------|-------------|------------|
| **Filtro de Kalman Extendido (EKF)** | Extensión no lineal — útil si añades arrastre u otra dinámica no lineal. | Media |
| **Filtro de Kalman sin Centrado (UKF)** | Mejor que el EKF para modelos muy no lineales; usa puntos sigma. | Media |
| **Filtro de Kalman de Velocidad Constante** | Modelo más simple (sin estado de aceleración). Buena línea base para comparar. | Fácil |
| **Regresión Polinomial** | Ajusta un polinomio a las últimas k posiciones y extrapola. Simple pero sin estimación de incertidumbre. | Fácil |
| **LSTM / RNN** | Entrena una red neuronal recurrente sobre la trayectoria. Necesita suficientes datos para evitar sobreajuste. | Difícil |
| **Regresión por Procesos Gaussianos** | No paramétrica, proporciona incertidumbre de forma natural. Puede ser lenta para conjuntos de datos grandes. | Difícil |
| **Interpolación + Extrapolación** | Interpolación por splines cúbicos sobre la ventana, extrapolada hacia adelante. Rápido y aproximado. | Fácil |

Eres libre de usar cualquier método o combinación de métodos. Si usas algo distinto al filtro de Kalman, explica por qué lo elegiste y cómo se compara.

---

## Estructura de Proyecto Sugerida

```
position-prediction/
├── README.md
├── flight_data.csv               # Datos proporcionados
├── requirements.txt              # Dependencias de Python
├── src/
│   ├── kalman_filter.py          # Implementación del filtro de Kalman
│   ├── predict.py                # Pipeline de predicción
│   ├── evaluate.py               # Métricas de error y análisis
│   └── utils.py                  # Carga de datos, helpers de visualización
├── output/
│   ├── trajectory_overview.png   # Gráfico de datos en bruto
│   ├── filtered_trajectory.png   # Filtrado vs. bruto
│   ├── prediction_comparison.png # Predicho vs. real
│   ├── error_vs_horizon.png      # Crecimiento del error con n
│   └── results.json              # Métricas de evaluación
└── report/
    └── report.md                 # Tu análisis y hallazgos
```

---

## Entregables

1. **Código** — Un pipeline funcional que lea el CSV, ejecute el filtro (o tu método elegido) y genere predicciones.
2. **Evaluación** — Métricas cuantitativas (MAE, RMSE, error máximo) para al menos tres horizontes de predicción.
3. **Visualizaciones** — Como mínimo:
   - Vista general de la trayectoria en bruto.
   - Trayectoria filtrada superpuesta sobre los datos en bruto.
   - Posiciones predichas vs. verdad de referencia para una o más ventanas de prueba.
   - Gráfico de error vs. horizonte de predicción.
4. **Informe** — Un documento breve (Markdown, PDF o notebook de Jupyter) que cubra:
   - Qué método elegiste y por qué.
   - Cómo determinaste los parámetros de ruido.
   - Resultados e interpretación.
   - Discusión de limitaciones y posibles mejoras.

---

## Criterios de Aprobación

El desafío se considera completado solo si se cumplen **todos** los siguientes criterios:

- [ ] Los datos se cargan correctamente y se incluye un resumen estadístico (número de muestras, período de muestreo, rangos de posición).
- [ ] Se genera al menos un gráfico de la trayectoria en bruto (posición vs. tiempo).
- [ ] Se implementa un filtro de Kalman (o método alternativo justificado) con un modelo de estado correcto.
- [ ] Se calculan métricas cuantitativas de error (MAE, RMSE) para cada horizonte de predicción.
- [ ] El error de predicción crece de forma gradual y razonable con el horizonte (no explota ni es constante).
- [ ] La elección de los parámetros de ruido (proceso y medición) y del estado inicial está justificada.
- [ ] Se incluyen al menos 3 gráficos: trayectoria en bruto, trayectoria filtrada vs. bruto, y posiciones predichas vs. reales.
- [ ] Se entrega un informe (Markdown, PDF o Jupyter) que cubra: método elegido y justificación, determinación de parámetros, resultados, y limitaciones/mejoras.

---

## Consejos

- **Empieza simple.** Primero haz funcionar el filtro de Kalman de velocidad constante y luego añade el estado de aceleración. Un filtro de aceleración constante malo es peor que uno de velocidad constante bueno.
- **Visualiza con frecuencia y desde el principio.** Grafica después de cada paso. Si la trayectoria filtrada no se ve bien, algo está mal con tu modelo o tus parámetros — arréglalo antes de continuar.
- **El período de muestreo importa.** Asegúrate de que `dt` sea correcto. Si el CSV tiene marcas de tiempo no uniformes, usa el `Δt` real entre muestras en lugar de asumir una tasa fija.
- **La inicialización es crítica.** El filtro necesita un estado inicial razonable (posición, velocidad, aceleración). Usa los primeros datos para estimarlos. Un mal estado inicial hace que el filtro diverja en las primeras observaciones.
- **Estabilidad numérica.** Si la matriz de covarianza `P` se vuelve no simétrica o desarrolla valores propios negativos, usa la forma de Joseph para el paso de actualización o impón la simetría: `P = 0.5 * (P + P.T)`.
- **El ajuste es iterativo.** No existe una fórmula para el `q` y `R` perfectos. Ajústalos observando los residuos (diferencia entre posiciones predichas y observadas). Los filtros bien ajustados producen residuos que parecen ruido blanco.

---

## Recursos

- [Kalman Filter Wikipedia](https://en.wikipedia.org/wiki/Kalman_filter) — Útil para entender las matemáticas.
- [Kalman Filter Tutorial (blog)](https://www.kalmanfilter.net/) — Explicaciones visuales interactivas.
- [pykalman](https://github.com/pykalman/pykalman) — Librería de filtros de Kalman en Python (útil para validación, pero implementa el tuyo primero).
- [Designing Kalman Filters — AIAA Book](https://arc.aiaa.org/doi/10.2514/6.2019-1999) — Orientación práctica sobre la selección de parámetros de ruido.
- [Understanding the Kalman Gain](https://www.kalmanfilter.net/kalmanGain.html) — Intuición sobre cómo la ganancia equilibra la predicción frente a la medición.
