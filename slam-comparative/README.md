# Desafío: Comparación de algoritmos SLAM

Ejecuta **dos (o más) algoritmos SLAM** sobre la misma grabación (ROS 2 bag) y compara su desempeño: cuál construye el mejor mapa, cuál estima mejor la posición del robot, y cuál es más eficiente.

---

## Resumen

SLAM significa *Simultaneous Localization and Mapping* (Localización y Mapeo Simultáneo). Es el problema donde un robot **construye un mapa de un lugar desconocido mientras, al mismo tiempo, descubre dónde está dentro de ese mapa**. Los algoritmos SLAM son el "cerebro" de los robots que se mueven solos: aspiradoras, autos autónomos, drones, etc.

En este desafío tendrás:

1. Una **grabación (bag)** de los sensores de un robot: una **cámara Intel RealSense** (RGB + profundidad). El bag es **solo visual**: no hay otro sensor que entregue la posición, así que la posición del robot la debe **estimar el propio SLAM visual (V-SLAM)**. Además se grabó la **posición exacta (ground truth)** del robot con un sistema de captura de movimiento OptiTrack.
2. Tú eliges **qué algoritmos SLAM instalar y ejecutar** (los instructores te sugerirán algunos).
3. Ejecutas cada algoritmo sobre la **misma grabación**, capturas sus resultados (mapa y trayectoria del robot).
4. **Comparas los resultados** de forma cuantitativa (números) y cualitativa (visualmente por qué uno es mejor que otro).

No hay una única respuesta correcta. Se trata de experimentar, medir y explicar qué viste.

---

## Materiales Entregados

| Archivo | Descripción |
|---------|-------------|
| [robot_recording.bag (descargar ~3 GB)](https://tecmx-my.sharepoint.com/:u:/g/personal/a01649586_tec_mx/IQAObN_rbBxeT7re-22Y0ugBAZZoXmlwdGkat4xbLE200F8?e=BW03qk) | Grabación ROS 2 con los datos de la cámara RealSense (RGB + profundidad) y la posición ground truth (OptiTrack). |

### Tópicos en el bag

Los nombres exactos pueden variar ligeramente. **Revisa los tópicos reales con `ros2 bag info`** (Parte 2). Este bag contiene los tópicos típicos de una cámara Intel RealSense (cámara e IMU, sin otros sensores externos) más la referencia OptiTrack:

| Tópico | Tipo | Descripción |
|--------|------|-------------|
| `/camera/color/image_raw` | `sensor_msgs/msg/Image` | Imagen RGB de la cámara. |
| `/camera/color/camera_info` | `sensor_msgs/msg/CameraInfo` | Calibración de la cámara RGB. |
| `/camera/depth/image_rect_raw` | `sensor_msgs/msg/Image` | Imagen de profundidad (registrada con la RGB). |
| `/camera/depth/camera_info` | `sensor_msgs/msg/CameraInfo` | Calibración de la cámara de profundidad. |
| `/camera/imu` *(si está habilitado)* | `sensor_msgs/msg/Imu` | IMU de la RealSense (aceleración + giroscopio). |
| `/tf` y `/tf_static` | `tf2_msgs/msg/TFMessage` | Relaciones entre frames (`camera_link`, `camera_color_optical_frame`, etc.). |
| `/optitrack/rigid_body` | `geometry_msgs/msg/PoseStamped` | **Posición exacta (ground truth)** del robot medida por OptiTrack. No debe alimentar al SLAM: sirve para **evaluar** las trayectorias estimadas. |

> **El SLAM debe ser visual (V-SLAM):** el bag solo trae la cámara RGB-D (y la IMU si está habilitada). No hay odometría externa; el SLAM estima la posición por sí mismo a partir de la cámara.

---

## Background — Qué necesitas saber

### ¿Qué es un algoritmo SLAM?

Un algoritmo SLAM recibe los datos del robot (las imágenes de la cámara y, si existe, la IMU) **en tiempo real** y va construyendo el mapa mientras estima dónde está el robot con respecto a ese mapa. Los algoritmos se diferencian en:

- **Qué sensores usan:** cámara monocular, cámara RGB-D (la RealSense), o RGB-D + IMU (visual-inertial).
- **Qué tipo de mapa generan:** nube de puntos 3D, "mapa de bits" 2D (*occupancy grid*) derivado de la cámara, o un grafo de posiciones.
- **Cómo resuelven el problema matemático:** filtros de partículas, optimización de grafos, características visuales (features), etc. *(No necesitas entender los detalles matemáticos para este desafío, pero sí te ayuda a explicar por qué los resultados difieren.)*

### ¿Qué es la odometría (visual) y por qué SLAM la "corrige"?

La **odometría visual** es el movimiento que el algoritmo deduce comparando imágenes consecutivas: "entre este cuadro y el siguiente la cámara se movió 3 cm para adelante y giró 2°". Con el tiempo se **acumulan errores** (*drift*): si la textura de la pared engaña o hay poca luz, el algoritmo "cree" que avanzó 10 cm cuando en realidad avanzó 8. SLAM usa el reconocimiento de **lugares ya vistos** (bucles) y la optimización del grafo para corregir ese error acumulado. A eso se le llama **cierre de bucle (*loop closure*)**.

Cuando un algoritmo SLAM funciona bien, al final del recorrido la trayectoria estimada **vuelve a coincidir** con la trayectoria real. Cuando funciona mal, la trayectoria "se desvía" y el mapa queda deformado.

### ¿Cómo se compara un algoritmo SLAM?

Típicamente se analiza:

- **Calidad del mapa:** ¿se ve igual al lugar real? ¿las paredes están rectas? ¿se ven dobles paredes o zonas borrosas?
- **Error de la trayectoria:** ¿qué tan lejos está la posición estimada por SLAM de la posición real (ground truth de OptiTrack)?
- **Cierre de bucle:** ¿el algoritmo detecta cuándo el robot vuelve a un lugar conocido?
- **Recursos:** ¿cuánto CPU/RAM consume? ¿va en tiempo real o más lento?

---

## Paso a Paso

### Parte 1 — Preparar el ambiente

Necesitas **ROS 2** instalado (se recomienda ROS 2 Humble sobre Ubuntu 22.04). Si nunca has instalado ROS, la documentación oficial es la mejor referencia:

- [Instalación de ROS 2 Humble](https://docs.ros.org/en/humble/Installation.html)

Los comandos de este desafío asumen que tu terminal ya "sabe" dónde está ROS 2. Cada vez que abras una terminal nueva, carga el ambiente:

```bash
source /opt/ros/humble/setup.bash
```

> Tip: puedes agregar esa línea al final de tu `~/.bashrc` para no escribirla a mano cada vez.

**Primeros pasos en ROS 2 (recomendado si es tu primera vez):**

- [Tutoriales oficiales de ROS 2 Humble](https://docs.ros.org/en/humble/Tutorials.html) — lee al menos *Understanding Nodes*, *Understanding Topics* y *Recording and Playing Back Data*.
- [¿Qué es un bag? (Playback de datos)](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)

### Parte 2 — Inspeccionar el bag

Antes de hacer nada, mira qué contiene el bag:

```bash
ros2 bag info robot_recording.bag
```

Este comando te muestra los **tópicos**, los **tipos de mensaje** y la **duración** de la grabación. Anota:

1. Nombres de los tópicos de sensores (cámara, IMU) y de la referencia OptiTrack.
2. Nombres de los tópicos de transformaciones (TF).
3. Duración total de la grabación.

> Parte del desafío es que estos nombres los descubras tú con este comando.

### Parte 3 — Elegir e instalar algoritmos SLAM

Como el bag es solo visual, debes comparar **al menos 2 algoritmos V-SLAM** de estrategias distintas (p. ej. dos enfoques distintos sobre el mismo sensor RGB-D). Acá van los más usados para ROS 2:

| Algoritmo | Sensor | Qué genera | Instalación (Humble) |
|-----------|--------|-----------|----------------------|
| [RTAB-Map](https://github.com/introlab/rtabmap) | RGB-D / cámara | Nube de puntos 3D + mapa 2D | `sudo apt install ros-humble-rtabmap-ros` |
| [ORB-SLAM3](https://github.com/UZ-SLAMLab/ORB_SLAM3) | Cámara | Mapa de puntos 3D | Compilación desde código (más avanzado) |

**Recomendación para empezar:**

- Instala **RTAB-Map** con un solo comando de `apt`; es la opción más rápida para tener resultados y funciona directo con la cámara RGB-D de la RealSense.

```bash
sudo apt install ros-humble-rtabmap-ros
```

Verifica que quedó instalado:

```bash
ros2 pkg list | grep rtabmap
```

> Si quieres una alternativa visual sin el mapa 2D y estás dispuesto a compilar desde código, **ORB-SLAM3** es la siguiente opción. Es más complicado de instalar que RTAB-Map.

### Parte 4 — Ejecutar SLAM sobre el bag

El flujo general es:

1. **Terminal A:** reproducir el bag (los datos "se transmiten" en ROS como si el robot estuviera vivo).
2. **Terminal B:** ejecutar el algoritmo SLAM, que escucha esos datos y construye el mapa.
3. **Terminal C (opcional):** abrir RViz2 para visualizar el mapa en vivo.

Un detalle importante: si el bag fue grabado en una simulación o tiene su propio reloj, hay que avisarle a ROS que use el **tiempo de la grabación** en vez del tiempo real:

```bash
# Terminal A — reproducir el bag
ros2 bag play robot_recording.bag

# Terminal B — SLAM con RTAB-Map (usa *sim time* = tiempo del bag)
ros2 launch rtabmap_launch rtabmap.launch.py use_sim_time:=true camera:=camera rgbd_sync:=true

# Terminal C — visualización
rviz2
```

En RViz2, agrega el mapa desde el panel *Add → By topic* (los tópicos `/map` normalmente). Deberías ver cómo el mapa va creciendo conforme avanza el bag.

Una vez que el bag termina, **tu mapa está listo**. Guárdalo:

```bash
ros2 run nav2_map_server map_saver_cli -f mapa_rtabmap
```

Repite el proceso con tu segundo algoritmo (ORB-SLAM3, etc.) y guarda cada mapa con su nombre correspondiente.

> **Importante:** usa la **misma grabación** y, en lo posible, los **mismos frames/paquetes de datos** para ambos algoritmos. Si no, la comparación no es justa.

### Parte 5 — Comparar los resultados

Ahora viene la parte de ingeniería: ¿cómo saber cuál algoritmo es mejor?

#### 5.1 Comparación visual de mapas

Abre los mapas generados (son archivos `.pgm` + `.yaml`) en cualquier visor de imágenes o en RViz. Busca:

- **Paredes dobles o borrosas** (señal de mal mapeo).
- **Mapas deformados** (las esquinas deberían ser de 90°, los corredores rectos).
- **Regiones vacías** donde la cámara no "vio" nada (huecos de profundidad o falta de textura).
- **Consistencia al cerrar bucles:** si el recorrido vuelve al punto de inicio, el mapa debería "cerrarse" bien.

#### 5.2 Comparación cuantitativa de trayectorias

Compara la trayectoria estimada por SLAM contra la **ground truth de OptiTrack** (`/optitrack/rigid_body`). La referencia **no debe alimentar al SLAM** — solo se usa al final para **evaluar** las trayectorias.

Extrae ambas trayectorias del bag (p. ej. con la API `rosbag2_py` o escuchando los tópicos durante el replay) y calcula las métricas de la tabla siguiente con un script propio o la librería que prefieras.

**Métricas que reportarás:**

| Métrica | Qué significa | Cómo se interpreta |
|---------|---------------|---------------------|
| **APE (Absolute Pose Error)** | Qué tan lejos está la trayectoria estimada de la referencia en cada instante | **Más bajo = mejor** |
| **RPE (Relative Pose Error)** | Qué tan consistente es el movimiento *entre* dos instantes cercanos (mide la deriva local) | **Más bajo = mejor** |
| **RMSE** | Raíz del error cuadrático medio (penaliza errores grandes) | **Más bajo = mejor** |
| **Cierre de bucle** | ¿El error final del recorrido vuelve a cerca de cero? | Observa la gráfica del APE: si el error crece y no baja, no hay buen cierre de bucle |

> **Ojo:** la trayectoria de OptiTrack y la estimada por SLAM suelen estar en **marcos de coordenadas distintos** (SLAM publica en el frame `map`). Alinea las trayectorias geométricamente antes de calcular métricas y documenta el método en tu reporte.

#### 5.3 Otros datos de interés

- **Consumo de CPU/RAM:** mide cuánta memoria usó cada proceso (p. ej. `htop`) mientras corría cada algoritmo.
- **Tiempo real vs. tiempo de cómputo:** si el algoritmo termina el bag *antes* de que este termine de reproducirse, va en tiempo real. Si se atrasa, no.
- **Nube de puntos 3D (si usas RGB-D):** compara visualmente las nubes generadas por RTAB-Map/ORB-SLAM3.

### Parte 6 — Reporte

Escribe un reporte breve (PDF o Markdown) que incluya:

1. **Elección de algoritmos:** ¿qué algoritmos comparaste y por qué?
2. **Setup:** cómo instalaste y ejecutaste cada uno (y qué tópicos encontraste en el bag).
3. **Mapas:** capturas de los mapas generados, lado a lado.
4. **Tabla de métricas:** valores de APE/RPE/RMSE para cada algoritmo.
5. **Análisis:** ¿cuál algoritmo dio el mejor mapa? ¿el más preciso en posición? ¿el más liviano? ¿El mismo algoritmo puede ganar en una categoría y perder en otra? ¿Por qué crees que pasó?
6. **Discusión:** ¿qué limitaciones tuvo tu comparación? (p. ej. la alineación de marcos entre SLAM y OptiTrack, el ambiente era simple/complejo, etc.)

---

## Estructura de Proyecto Sugerida

```
slam-comparative/
├── README.md
├── maps/
│   ├── mapa_rtabmap.pgm
│   ├── mapa_rtabmap.yaml
│   └── (mapas o nubes del segundo algoritmo)
├── results/
│   ├── trayectoria_rtabmap.csv   # trayectoria estimada vs. OptiTrack
│   ├── trayectoria_orb_slam3.csv
│   ├── metricas_rtabmap.csv
│   ├── metricas_orb_slam3.csv
│   └── comparativa.csv           # tabla resumen
└── report/
    └── report.md                 # tu análisis
```

Es una sugerencia — organízate como prefieras.

---

## Lista de Aceptación

Para aprobar este desafío, el proyecto debe cumplir **todos** los siguientes criterios:

- [ ] **Inspección del bag:** Se identifican correctamente tópicos, tipos, QoS y duración del bag.
- [ ] **≥2 algoritmos de SLAM ejecutados:** Ambos algoritmos corrieron sobre el **mismo** bag y generaron sus mapas.
- [ ] **Comparación visual:** Hay capturas de los mapas lado a lado con una breve descripción de diferencias de calidad.
- [ ] **Comparación cuantitativa:** Hay tabla con APE/RPE/RMSE calculados para cada algoritmo y una interpretación de resultados.
- [ ] **Reporte completo:** El reporte documenta el setup (instalación, lanzamientos, tópicos, parámetros), es reproducible y tiene estructura clara.
- [ ] **Discusión crítica:** Se reconocen limitaciones de la comparación (p. ej. la alineación de marcos entre SLAM y OptiTrack, ambiente, configuración de parámetros).

---

## Tips

- **No omitas la Parte 2.** Inspeccionar el contenido del bag es la diferencia entre una ejecución limpia y horas de depuración buscando un tópico que no existe.
- **Prueba primero visualizar el bag** con RViz2 (`ros2 bag play` + RViz) antes de meter SLAM. Así sabes si los datos se ven bien.
- **Usa `use_sim_time:=true`** si el bag viene de simulación; si no, las transformaciones de tiempo se desfasarán.
- **Guarda cada mapa con su nombre** (`mapa_rtabmap`, `mapa_orb_slam3`). Es muy fácil sobrescribir el mapa del primer algoritmo con el segundo y arruinar la comparación.
- Si un algoritmo "explota" (consume demasiada RAM o se atrasa), anótalo: **eso también es un resultado válido** y muy interesante de discutir.
- No configures demasiados parámetros al inicio; usa la configuración por defecto y solo ajusta lo necesario. La comparación entre algoritmos con configuración estándar es más justa y simple de reproducir.

---

## Recursos

- [Documentación ROS 2 Humble](https://docs.ros.org/en/humble/)
- [Tutoriales oficiales ROS 2 (comienza aquí si eres nuevo)](https://docs.ros.org/en/humble/Tutorials.html)
- [Grabar y reproducir bags](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)
- [RTAB-Map — GitHub](https://github.com/introlab/rtabmap) y [RTAB-Map en ROS 2](https://github.com/introlab/rtabmap_ros)
- [ORB-SLAM3 — GitHub](https://github.com/UZ-SLAMLab/ORB_SLAM3)
- [Rosbag2 (Python API del bag)](https://docs.ros.org/en/humble/p/rosbag2_py/)
