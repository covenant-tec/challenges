# Desafíos de Investigación — Evasión de Obstáculos Dinámicos en Ambientes de Construcción

## La historia

Son las 7 de la mañana. El sol castiga la obra y una grúa de 40 metros gira sin aviso cargando planchas de acero. Abajo, un obrero camina rápido entre andamios mientras una retroexcavadora levanta material. Polvo por todas partes, cables sueltos, sombras duras y cambiantes.

En medio de todo eso, un dron despega.

Su trabajo: inspeccionar el sitio, mapear el avance de la obra, detectar amenazas. Pero nadie le dio un plano del caos. Está solo, con cámaras y sensores ruidosos, y debe decidir en **fracciones de segundo** si una grúa que se acerca lo va a golpear... o si tiene tiempo de pasar por debajo. Ahí no hay margen para el error: cada décima de segundo de predicción es un metro de decisión.

Ese es el problema que estamos resolviendo: **la evasión de obstáculos dinámicos en ambientes de construcción**. Es uno de los retos abiertos más difíciles de los drones autónomos.

## Los Desafíos

Para esquivar un obstáculo en movimiento, un dron necesita resolver una cadena completa de problemas:

| Nº | Pregunta que el dron debe responder | Pipeline |
|----|--------------------------------------|----------|
| 1 | ¿Dónde estoy y cómo se ve el lugar? | **Percepción + SLAM** |
| 2 | ¿Qué hay alrededor en 3D, y a qué distancia? | **Visión 3D** |
| 3 | ¿Hacia dónde se mueve y dónde va a estar? | **Predicción** |
| 4 | ¿Con qué ojo y en qué ángulo lo veo? | **Hardware de sensado** |
| 5 | ¿Y cómo registro todo esto para que otro pueda repetirlo? | **Gestión de datos** |

Cada desafío de este repositorio es una de esas piezas. Son **independientes**: puedes tomar cualquiera, en el orden que quieras, sin depender de los demás. Pero si entiendes cómo encaja tu pieza en la cadena, verás el mismo desafío con otros ojos.

### 1. Comparación de algoritmos SLAM — *¿Dónde estoy y cómo se ve esto?*

[slam-comparative/](slam-comparative/)

Antes de esquivar nada, el dron tiene que saber **dónde está y cómo es el entorno**. Eso es SLAM: construir el mapa del lugar desconocido mientras te localizas dentro de él, a la vez. En este desafío ejecutarás **dos o más algoritmos de SLAM visual** sobre la misma grabación de una cámara RealSense, y los expondrás con métricas reales (APE, RPE, RMSE) usando una referencia de captura de movimiento OptiTrack. Ganará el que construya el mejor mapa sin perderse.

> **Tu pieza del rompecabezas:** sin un buen mapa y una buena localización, un dron no puede planear hacia dónde moverse para huir.

---

### 2. Nube de Puntos Comparativa — *Ver el mundo en 3D sin hardware caro*

[pointcloud-comparative/](pointcloud-comparative/)

El dron necesita saber que esa grúa está a **2 metros**, no que "aparece en una foto". Las cámaras estéreo (como la RealSense) lo logran con hardware dedicado; los modelos de aprendizaje profundo intentan **inferir la profundidad a partir de una sola imagen RGB**. Este desafío te pide construir la nube de puntos 3D de una escena con ambos métodos y compararlos punto a punto: ¿puede una cámara barata y liviana reemplazar a un sensor caro? Tu análisis decide.

> **Tu pieza del rompecabezas:** detectar obstáculos es percibir geometría en 3D. Si una sola cámara alcanza, el dron carga menos peso — y evade más rápido.

---

### 3. Predicción de Posición de Drones — *Anticiparte al golpe*

[position-prediction/](position-prediction/)

Esquivar no es solo ver: es **calcular dónde estará la grúa** cuando tú llegues. Con los datos ruidosos de un vuelo real frente a ti, construirás un **filtro de Kalman** que suavice las mediciones y proyecte la trayectoria del dron hacia el futuro. Verás cómo el error crece con el horizonte de predicción, y aprenderás a confiar en la incertidumbre del propio filtro. Es la diferencia entre reaccionar y anticipar.

> **Tu pieza del rompecabezas:** la predicción es el corazón de la evasión. Saber dónde estará el obstáculo en el futuro es lo que convierte un choque en un 30 cm de margen.

---

### 4. Soporte de Inclinación (Gimbal) para RealSense — *El hardware que sostiene al 'ojo'*

[realsense-gimbal/](realsense-gimbal/)

Ningún algoritmo sirve si el ojo está mal montado. La RealSense es el ojo del dron, y para ver hacia el frente y hacia el suelo necesita un soporte que puedas **ajustar a mano** en pleno campo. Sin motores, sin llaves: un pitch que se fija con un tornillo. Diseñarás la pieza en CAD partiendo de un modelo existente, la imprimirás en 3D y la validarás con un inclinómetro en el frame real de tu dron. Es el desafío donde la ingeniería mecánica se encuentra con la de software.

> **Tu pieza del rompecabezas:** la evasión depende de apuntar el sensor al lugar correcto en el momento correcto. Sin el gimbal, el dron vuela a ciegas.

---

### 5. Base de Datos de Experimentos — *La ciencia necesita orden*

[experiments-database/](experiments-database/)

Aquí está la verdad incómoda de la investigación: los resultados viven dispersos en 4 carpetas, 2 USB y 17 correos. Cuando le preguntes a la ciencia "¿qué pasó en la prueba de ayer?", nadie podrá responderte. Este desafío te pide construir una **aplicación de escritorio (Electron + SQLite)** que centralice proyectos, ejecuciones, resultados numéricos y archivos adjuntos, con todo el flujo CRUD e IPC de una app real. Es la herramienta que la investigación necesitará para siempre.

> **Tu pieza del rompecabezas:** un experimento que no se puede repetir no sirve para la ciencia. Ordena los datos y la evasión de obstáculos se vuelve *ciencia real*, no anécdotas.

---

## Cómo empezar

1. **Elige tu desafío.** Lee los README de cada carpeta y elige el que más te motive. Ninguno requiere los demás.
2. **Te recomiendo empezar** por `position-prediction/` si te gusta el control y los estimadores, por `pointcloud-comparative/` si te atrae la visión por computadora, por `realsense-gimbal/` si eres de CAD e impresión 3D, y por `experiments-database/` si quieres construir software de escritorio real.
3. **Trabaja un semestre.** Cada desafío está pensado para desarrollarse, documentarse y pulirse durante todo el semestre. Prioriza criterios de aceptación claros, entrega algo que funcione de verdad, y documenta cada decisión.
4. **Presenta y comparte.** Al final, muestra qué encontraste: las métricas, los mapas, las decisiones de diseño y lo que le harías a la próxima versión.

- Todo el contenido del repositorio está en español, incluyendo este README.
- Cada carpeta contiene su propio `README.md` completo: prerequisitos, datos, guía paso a paso, criterios de aceptación y recursos.
- Cualquier herramienta es válida: Python, ROS 2, CAD, impresión 3D... elige tu stack y justifica tu elección.

---

## Haz un fork y comparte tu progreso

Este repositorio es el punto de partida de todos. Para que sepamos **quién está trabajando en qué** y **cómo va su avance**, cada quien debe trabajar sobre su propia copia:

1. **Haz fork** de este repositorio a tu cuenta de GitHub (botón *Fork*, arriba a la derecha).
2. **Clona tu fork** y trabaja ahí: `git clone https://github.com/TU_USUARIO/challenges.git`
3. El **historial de commits** en tu fork es tu bitácora pública de progreso: nos permite verificar tu avance semana a semana.

### Sube tu progreso constantemente

- **Haz commits pequeños y frecuentes.** Cada prueba que funcione, cada gráfica, cada archivo CAD terminado y cada corrección de un error son un `git add` + `git commit` con un mensaje claro de lo que hiciste. Un commit por semana no basta; veinte commits significativos por semana representan un avance real.
- **Sube (push) al menos una vez por semana** a tu fork en GitHub. Un trabajo sin commits es un trabajo invisible.
- **Un buen mensaje de commit explica el porqué**, no solo el qué:
  - **Evita:** `cambios` / `update` / `avance 3`
  - **Preferente:** `Agrega filtro de Kalman con aceleración constante y gráfica de error vs. horizonte`
- **No rompas el flujo de los demás:** edita solo dentro de la carpeta de tu desafío y de tu trabajo; tu fork es tuyo, pero mantén el repo limpio para quien venga después.
- Al terminar el semestre, **comparte la URL de tu fork**. Ahí revisaremos tu trabajo: commits, código e informe.

> **Regla de oro:** si no está en GitHub, no ocurrió. Un buen portafolio se construye sobre un historial de commits que cuente tu semestre completo. Publica tu progreso: esos commits forman parte de tu portafolio profesional.

---

## Contacto

¿Tienes dudas, quieres validar tu enfoque o compartir un resultado? Escríbeme:

- **Nombre:** Kevin Martinez
- **Correo:** [A01649586@tec.mx](mailto:A01649586@tec.mx)
- **Teléfono:** [33 2319 1926](tel:+523323191926)

Este repositorio sigue en desarrollo. Si algo no te queda claro o consideras que algún desafío puede mejorarse, compártelo: las buenas propuestas se incorporan a la investigación.
