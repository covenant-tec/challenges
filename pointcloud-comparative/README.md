# Desafío: Nube de Puntos Comparativa — Profundidad Monocular vs. Estéreo

Genera una nube de puntos 3D a partir de una única imagen 2D mediante estimación de profundidad monocular y compárala con la nube de puntos de referencia ("ground truth") producida por una cámara estéreo Intel RealSense D435i. El objetivo es evaluar qué tan bien puede una sola imagen RGB recuperar la geometría 3D en comparación con un sensor de profundidad estéreo dedicado.

---

## Resumen

Las cámaras estéreo como la RealSense D435i usan dos sensores infrarrojos y un proyector IR para calcular mapas de profundidad densos en tiempo real. Una cámara RGB estándar no tiene ese hardware, y sin embargo los modelos modernos de aprendizaje profundo pueden inferir un mapa de profundidad a partir de una sola imagen. Este desafío te pide:

1. Extraer fotogramas de color y de profundidad de un archivo bag de ROS 2 proporcionado, grabado con una RealSense D435i.
2. Construir una nube de puntos "de referencia" a partir de los datos de profundidad estéreo.
3. Producir una nube de puntos monocular estimando la profundidad solo a partir de la imagen en color (cualquier método que elijas).
4. Comparar las dos nubes de puntos de forma cuantitativa y discutir los resultados.

Este es un desafío abierto: no hay una única forma "correcta" de realizar la estimación de profundidad monocular. La metodología de comparación y el análisis son donde ocurre la verdadera ingeniería.

---

## Materiales Proporcionados

| Archivo | Descripción |
|------|-------------|
| [realsense_recording.zip (descargar ~3 GB)](https://tecmx-my.sharepoint.com/:u:/g/personal/a01649586_tec_mx/IQAObN_rbBxeT7re-22Y0ugBAZZoXmlwdGkat4xbLE200F8?e=BW03qk) | Archivo comprimido del bag de ROS 2 grabado con una Intel RealSense D435i. Al descomprimirse produce el bag completo en formato directorio (`metadata.yaml` + archivo de base de datos del bag) con imágenes en color, mapas de profundidad e intrínsecos de la cámara. |

### Tópicos del Bag

| Tópico | Tipo | Descripción |
|-------|------|-------------|
| `/camera/color/image_raw` | `sensor_msgs/msg/Image` | Fotogramas de color RGB (8 bits, BGR o RGB). |
| `/camera/color/camera_info` | `sensor_msgs/msg/CameraInfo` | Parámetros intrínsecos de la cámara de color. |
| `/camera/depth/image_rect_raw` | `sensor_msgs/msg/Image` | Fotogramas de profundidad rectificados y alineados (16 bits, milímetros). |
| `/camera/depth/camera_info` | `sensor_msgs/msg/CameraInfo` | Parámetros intrínsecos de la cámara de profundidad. |

> **Nota:** La imagen de profundidad ya está rectificada y alineada con el fotograma de color por el pipeline de RealSense. No necesitas realizar alineación adicional.

---

## Contexto (Background)

### ¿Qué es una Nube de Puntos?

Una nube de puntos es una colección de puntos 3D `(X, Y, Z)` que describen la geometría de la superficie de una escena. Para una cámara calibrada, cada píxel `(u, v)` en una imagen con una profundidad conocida `d` puede ser retro-proyectado al espacio 3D usando la matriz intrínseca `K` de la cámara:

```
X = (u - cx) * d / fx
Y = (v - cy) * d / fy
Z = d
```

Donde `fx`, `fy` son las distancias focales y `cx`, `cy` es el punto principal, todos presentes en el mensaje `CameraInfo`.

### Profundidad Estéreo vs. Monocular

| Aspecto | Estéreo (RealSense D435i) | Monocular (Imagen Única) |
|--------|--------------------------|--------------------------|
| Hardware | Dos sensores IR + proyector IR | Una sola cámara RGB |
| Rango de profundidad | ~0.2 m – 10 m (configurable) | Dependiente del modelo, a menudo sin acotar |
| Escala métrica | Absoluta (milímetros) | A menudo requiere recuperación de escala |
| Densa / Dispersa | Densa (por píxel) | Densa (dependiente del modelo) |
| Precisión | Alta (sub-cm a 2 m) | Varía mucho según la escena y el modelo |

La estimación de profundidad monocular es un **problema mal planteado (ill-posed)** — una sola imagen es una proyección 2D de una escena 3D, por lo que infinitas reconstrucciones 3D son consistentes con ella. Los modelos modernos de aprendizaje profundo aprenden a predecir profundidad *plausible*, pero el resultado suele ser solo a un factor de escala y puede contener distorsiones en regiones sin textura o reflectantes.

---

## Desglose de Tareas

### Parte 1 — Configuración del Entorno

Elige tu lenguaje y herramientas de trabajo. Se recomienda Python con ROS 2 y Open3D, pero eres libre de usar cualquier stack con el que te sientas cómodo.

> **Usuarios de Windows:** ROS 2 Humble no se ejecuta de forma nativa en Windows sin soporte oficial. Se recomienda instalar **WSL 2** con Ubuntu 22.04 y trabajar dentro de la terminal de WSL (los comandos de este desafío y el bag se gestionan desde ahí). Guías oficiales: https://learn.microsoft.com/windows/wsl/install

**Paquetes de Python recomendados:**

```
pip install opencv-python numpy open3d matplotlib
```

Si quieres trabajar con el bag de ROS 2 directamente en Python, instala las herramientas CLI de ROS 2 y la API de bag para Python:

```bash
# Instalar ROS 2 Humble (si no está ya presente)
# Sigue: https://docs.ros.org/en/humble/Installation.html

# Instalar la API de bag para Python
sudo apt install ros-humble-rosbag2-py
```

> Si nunca has usado ROS antes, empieza con la documentación oficial de **ROS 2 Humble**: https://docs.ros.org/en/humble/Tutorials.html. Los tutoriales "Understanding Nodes" y "Recording and Playing Back Data" son directamente relevantes para este desafío.

**Inspección rápida del bag de ROS 2 (sin necesidad de código):**

```bash
# Descomprimir el bag (contiene metadata.yaml y la base de datos del bag)
unzip realsense_recording.zip

# Listar tópicos y tipos de mensaje disponibles en el bag
ros2 bag info realsense_recording

# Reproducir el bag (visualízalo en RViz2 si lo deseas)
ros2 bag play realsense_recording
```

### Parte 2 — Extraer Fotogramas del Bag

Necesitas extraer al menos **un** fotograma de color y su fotograma de profundidad correspondiente del bag. Puedes usar todo el bag o un solo timestamp: la comparación es por fotograma.

**Enfoque A — Python (sin ROS):**

Si prefieres evitar ROS por completo, puedes leer el bag con la librería `rosbags`, que funciona sin una instalación de ROS activa:

```bash
pip install rosbags
```

**Enfoque B — API de Python de ROS 2:**

Usa `rosbag2_py` para iterar sobre los mensajes grabados y extraer los fotogramas que necesites.

**Enfoque C — Herramientas CLI:**

Convierte el bag a archivos de imagen individuales usando `ros2 bag` y herramientas externas, y luego léelos con OpenCV.

Sea cual sea el enfoque que elijas, guarda o mantén en memoria:

1. La **imagen de color** (como un array de numpy o equivalente).
2. La **imagen de profundidad** (como un array de numpy de 16 bits, valores en milímetros).
3. Los **intrínsecos de la cámara** (`fx`, `fy`, `cx`, `cy`) de `/camera/color/camera_info`.

### Parte 3 — Construir la Nube de Puntos de Referencia

Usando la imagen de profundidad y los intrínsecos de color, retro-proyecta cada píxel `(u, v)` con una profundidad válida a un punto 3D. Colorea cada punto con su valor RGB correspondiente.

**Pseudocódigo:**

```
para cada píxel (u, v):
    d = depth_image[v, u]
    si d == 0:          # no hay lectura de profundidad válida
        saltar
    X = (u - cx) * d / fx
    Y = (v - cy) * d / fy
    Z = d
    color = color_image[v, u]
    añadir punto (X, Y, Z) con color a la nube de puntos
```

Si usas Open3D, la función `create_from_depth_image` maneja esto directamente. Asegúrate de pasar los parámetros intrínsecos correctos.

Guárdala como tu **nube de puntos de referencia** (p. ej. `ground_truth.ply` o `ground_truth.pcd`).

### Parte 4 — Generar la Nube de Puntos Monocular

Este es el núcleo del desafío. Usando **solo** la imagen de color (sin imagen de profundidad, sin estéreo), produce una estimación de profundidad y conviértela en una nube de puntos.

**Reglas:**

- Puedes usar cualquier método de estimación de profundidad monocular (aprendizaje profundo, CV clásica, etc.).
- Puedes usar cualquier modelo pre-entrenado o entrenar el tuyo propio.
- Debes explicar en tu informe qué método elegiste y por qué.

**Métodos populares (lista no exhaustiva):**

| Método | Tipo | Notas |
|--------|------|-------|
| [MiDaS](https://github.com/isl-org/MiDaS) | Aprendizaje profundo | Buena generalización, profundidad relativa |
| [DPT (Dense Prediction Transformer)](https://github.com/isl-org/DPT) | Aprendizaje profundo | Alta calidad, mayor cómputo |
| [Depth Anything](https://github.com/LiheYoung/Depth-Anything) | Aprendizaje profundo | Estado del arte, zero-shot |
| [AdaBins](https://github.com/shariqfarooq123/AdaBins) | Aprendizaje profundo | Buen rendimiento en interiores |
| Structure from Motion (OpenCV) | CV clásica | De disperso a denso, requiere movimiento de cámara |

**Consideración importante — recuperación de escala:**

La mayoría de los modelos monoculares producen profundidad **relativa** o **a-factor de escala**. Necesitarás alinear la profundidad monocular a la escala métrica de la referencia antes de comparar. Enfoques comunes:

1. **Escalado por mediana:** Calcula la profundidad mediana de ambas nubes y escala la profundidad monocular para que coincida.
2. **Alineación por mínimos cuadrados:** Ajusta una transformación lineal `d_mono = a * d_gt + b` sobre puntos muestreados.
3. **Alineación ICP:** Usa Iterative Closest Point para registrar la nube monocular con la de referencia.

Documenta qué método de alineación usaste.

### Parte 5 — Comparar las Nubes de Puntos

Evalúa cuantitativa y cualitativamente la nube de puntos monocular frente a la de referencia.

#### Métricas Cuantitativas

Calcula al menos **tres** de las siguientes métricas:

| Métrica | Fórmula / Descripción | Qué te indica |
|--------|-----------------------|-------------------|
| **MAE** (Error Absoluto Medio) | `(1/N) * Σ \|d_mono - d_gt\|` | Error de profundidad promedio |
| **RMSE** (Error Cuadrático Medio de la Raíz) | `sqrt((1/N) * Σ (d_mono - d_gt)²)` | Penaliza errores grandes |
| **Error Mediano** | `median(\|d_mono - d_gt\|)` | Robusto ante valores atípicos |
| **δ < 1.25** | `% de puntos donde max(d_mono/d_gt, d_gt/d_mono) < 1.25` | Porcentaje de puntos "precisos" (común en la literatura de profundidad) |
| **Distancia nube a nube** | Distancia promedio al vecino más cercano entre las dos nubes de puntos | Similitud geométrica |
| **Distancia de Hausdorff** | Máximo de todas las distancias mínimas entre las dos nubes | Desviación en el peor caso |
| **Cobertura** | Fracción de puntos monoculares que caen dentro de un umbral de cualquier punto de la referencia | Completitud |

#### Evaluación Cualitativa

- Visualiza ambas nubes de puntos lado a lado (Open3D, RViz2, MeshLab o similar).
- Identifica regiones específicas donde la estimación monocular tiene éxito o falla (p. ej. bordes, paredes sin textura, vidrio, objetos distantes).

### Parte 6 — Informe

Escribe un informe breve (PDF, Markdown o Jupyter notebook) que incluya:

1. **Descripción del método:** ¿Qué enfoque de estimación de profundidad monocular usaste y por qué?
2. **Detalles de implementación:** Librerías, versiones de modelos, hardware usado, tiempo de ejecución.
3. **Procedimiento de alineación:** ¿Cómo recuperaste la escala métrica?
4. **Resultados:** Tabla de métricas cuantitativas, visualizaciones lado a lado.
5. **Análisis:** ¿Dónde tiene éxito la estimación monocular? ¿Dónde falla? ¿Por qué?
6. **Discusión:** ¿Bajo qué condiciones podría un enfoque monocular ser una alternativa viable al estéreo? ¿Cuáles son las limitaciones?

---

## Estructura de Proyecto Sugerida

```
pointcloud-comparative/
├── README.md
├── requirements.txt           # Dependencias de Python
├── src/
│   ├── extract_frames.py      # Lee el rosbag y extrae color + profundidad
│   ├── build_pointcloud.py    # Retro-proyecta la profundidad a una nube de puntos coloreada
│   ├── monocular_depth.py     # Tu pipeline de estimación de profundidad monocular
│   ├── compare.py             # Calcula métricas entre dos nubes de puntos
│   └── visualise.py           # Utilidades de visualización
├── output/
│   ├── ground_truth.ply       # Nube de puntos de la profundidad de RealSense
│   ├── monocular.ply          # Nube de puntos de la estimación monocular
│   └── metrics.json           # Resultados de evaluación
└── report/
    └── report.md              # Tu informe de análisis
```

Esto es una sugerencia: organiza tu proyecto como mejor te convenga.

---

## Lista de Aceptación

Para aprobar este desafío, el proyecto debe cumplir **todos** los siguientes criterios:

- [ ] **Extracción de fotogramas:** Se extrae correctamente al menos un fotograma de color, su correspondiente de profundidad y los intrínsecos de la cámara del bag de ROS 2.
- [ ] **Nube de puntos de referencia:** Se construye una nube de puntos 3D a partir de la profundidad estéreo usando los intrínsecos correctos, y se verifica visualmente que la geometría sea coherente con la escena.
- [ ] **Estimación de profundidad monocular:** Se genera un mapa de profundidad a partir de **solo** la imagen de color, utilizando cualquier método (aprendizaje profundo, CV clásico, etc.), y se convierte en una nube de puntos.
- [ ] **Alineación de escala:** La profundidad monocular se alinea a la escala métrica de la referencia, documentando el método de alineación empleado.
- [ ] **Comparación cuantitativa:** Se calculan al menos **tres** métricas cuantitativas (MAE, RMSE, Error Mediano, δ < 1.25, distancia nube a nube, Hausdorff o Cobertura) y se reportan en el informe.
- [ ] **Análisis cualitativo:** Se visualizan las dos nubes de puntos (lado a lado o superpuestas) y se identifican al menos dos regiones donde la estimación monocular tiene éxito o falla.
- [ ] **Informe completo:** El informe incluye descripción del método, detalles de implementación, procedimiento de alineación, resultados cuantitativos, análisis cualitativo y discusión.

---

## Consejos

- Empieza extrayendo y visualizando primero la nube de puntos de referencia. Asegúrate de que se vea correcta antes de continuar.
- Verifica los intrínsecos: valores incorrectos de `fx`, `fy`, `cx`, `cy` distorsionarán toda la reconstrucción.
- La mayoría de los modelos monoculares esperan entrada RGB en un rango de normalización específico (p. ej. `[0, 1]` o normalización de ImageNet). Lee la documentación del modelo con atención.
- Las imágenes de profundidad de la RealSense están en **milímetros** como enteros sin signo de 16 bits. Un valor de `0` significa ausencia de respuesta: trátalos como inválidos.
- Si tu nube de puntos monocular se ve deformada, es posible que la matriz intrínseca no coincida con la resolución del mapa de profundidad. Asegura la consistencia.
- Para nubes de puntos grandes, el downsample por vóxeles de Open3D puede acelerar la comparación sin afectar significativamente la precisión.

---

## Recursos

- [Documentación de ROS 2 Humble](https://docs.ros.org/en/humble/)
- [ROS 2 Bag Working with Bags](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)
- [Librería de Python rosbags](https://github.com/MetroRobo/rosbags) (lee bags sin ROS)
- [Documentación de Nubes de Puntos de Open3D](http://www.open3d.org/docs/release/tutorial/geometry/pointcloud.html)
- [Especificaciones de Intel RealSense D435i](https://www.intelrealsense.com/depth-camera-d435i/)
- [Estimación de profundidad MiDaS](https://github.com/isl-org/MiDaS)
- [Depth Anything](https://github.com/LiheYoung/Depth-Anything)
- [Calibración de cámara OpenCV](https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html)
