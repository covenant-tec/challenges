# Desafío: Soporte de Inclinación Manual (Gimbal de Pitch) para Cámara Intel RealSense D435i

Diseña y fabrica un soporte para la cámara Intel RealSense D435i que permita ajustar **manualmente** el ángulo de inclinación (pitch), inspirándote en modelos 3D ya existentes. Sin motores: el ángulo se ajusta a mano y se fija con un tornillo, para poder hacer pruebas a diferentes ángulos de forma rápida. Por ahora solo te importa el **pitch** — el soporte debe quedar fijo en roll y yaw.

---

## Resumen

Cuando montas una cámara de profundidad en un dron (p. ej. un frame Holybro X650), muchas veces necesitas cambiar el ángulo al que apunta: mirando al frente, mirando hacia el suelo, o cualquier punto intermedio. Los soportes comerciales y los de Thingiverse/Printables suelen tener un ángulo **fijo** (15°, 30°, 45°...). Este desafío consiste en diseñar tu propio soporte donde ese ángulo se pueda cambiar a mano, en segundos, sin herramientas complejas.

Trabajas con **diseño mecánico + impresión 3D**, no con programación. El punto de partida es un modelo gratuito que fija la cámara al sistema de raíles del dron:

- [Angled Rail Mount for Intel Realsense Camera (S500/S550/X500) — Printables](https://www.printables.com/model/1374162-angled-rail-mount-for-intel-realsense-camera-s500s)

Ese modelo está pensado para el mismo sistema de raíles de 60 mm (tubos de 10 mm) que usa tu dron. La diferencia: tú le agregarás **un eje de giro ajustable** en lugar de usar un ángulo fijo.

---

## Materiales proporcionados

| Material | Descripción |
|----------|-------------|
| Cámara Intel RealSense D435i | Cámara de profundidad estéreo. La misma que se usa en el desafío `pointcloud-comparative`. |
| Frame Holybro X650 | Dron con sistema de raíles de 60 mm (tubos de 10 mm) en el que se monta el soporte. |
| Modelo de referencia (STL) | Montura de ángulo fijo para la misma cámara y el mismo raíl. Descárgalo y estúdialo antes de diseñar. |
| Impresora 3D (FDM/FFF) | Disponible para imprimir tu diseño. |
| Software CAD | Fusion 360, Onshape (gratuito en la nube) o FreeCAD. Cualquiera sirve. |

### Datos de montaje de la cámara (D435/D435i)

| Dato | Valor |
|------|-------|
| Rosca trípode (base) | 1/4-20 (estándar de cámaras) |
| Orificios de montaje traseros | 2 × M3 (roscados) |
| Dimensiones aproximadas | ~90 × 25 × 25 mm |

> Las medidas exactas (distancia entre orificios, profundidad de rosca, posicionamiento) están en el **datasheet** de la cámara. Aprende a leerlo y úsalo como referencia — es la fuente oficial.

---

## Conceptos que debes conocer

- **Gimbal**: montaje que permite orientar un dispositivo en el espacio. Un gimbal completo tiene 3 ejes; el tuyo solo tendrá 1.
- **Pitch / Roll / Yaw**: los tres giros posibles de un objeto.
  - **Pitch**: inclinación hacia arriba/abajo (apuntar al suelo o al frente).
  - **Roll**: rotación lateral (balanceo).
  - **Yaw**: rotación horizontal (girar a izquierda/derecha).
- **Grados de libertad (DOF)**: número de movimientos independientes que permite tu mecanismo. El tuyo debe tener **1 DOF** (solo pitch).
- **STL**: formato de archivo estándar para imprimir en 3D (malla de triángulos, sin colores ni unidades de "fábrica").
- **Slicer**: programa que convierte el STL en instrucciones para la impresora (código G).
- **Tolerancia de impresión**: las piezas impresas no quedan exactamente de la medida del modelo; hay que dejar juego (~0.2–0.4 mm) en agujeros y ejes.

---

## Objetivo y requisitos del diseño

Diseña un soporte que:

1. **Se monte al raíl de 60 mm del X650** (tubos de 10 mm), igual que el modelo de referencia.
2. **Sujete la cámara D435i con seguridad**, usando los orificios M3 traseros o la rosca 1/4-20 de la base. La cámara no debe moverse ni zafarse con la vibración.
3. **Permita ajustar el ángulo de pitch manualmente**, fijándolo con un tornillo (giro manual, sin llaves si es posible; un tornillo de mariposa o de aleta es ideal).
4. **Mantenga fijos roll y yaw**. El eje de giro debe estar bien alineado (el pivote debe ser el único movimiento posible, y debe quedar apretado).
5. **Cubra un rango de inclinación útil** para pruebas: se sugiere de **0° (hacia el frente) a 90° (vertical hacia el suelo)**, con marcas graduadas cada 5° o 10° impresas en el mecanismo.
6. **No bloquee la lente, ni el cable USB, ni las ventilas** de la cámara (la D435i se calienta operando; necesita disipar).

No hay una sola forma correcta de lograrlo. El punto técnico principal es decidir **cómo se ajusta y se fija** el ángulo (ver "Mecanismos posibles").

---

## Plan de trabajo sugerido

No es una receta obligatoria, pero este orden te ahorra iteraciones.

### Paso 1 — Estudia el modelo de referencia
Descarga el STL del modelo de Printables y ábrelo en tu slicer o visor STL. Identifica: ¿cómo se agarra al raíl? ¿dónde van los tornillos de la cámara? ¿qué espesor tienen las paredes?

### Paso 2 — Mide la cámara y el raíl
Con el datasheet de la RealSense y un calibre (vernier), anota:
- Distancia entre los dos orificios M3 traseros y su profundidad.
- Dimensiones exteriores de la cámara y zona por donde sale el cable USB-C.
- Diámetro de los tubos del raíl y la distancia entre ambos.

Estas medidas serán las cotas de tu modelo. Si no encuentras el calibre, el datasheet tiene el dibujo acotado de la cámara.

### Paso 3 — Entiende el software CAD
Elige una herramienta y haz un tutorial introductorio antes de diseñar (todos tardan ~30 min):
- Fusion 360: gratis para estudiantes.
- Onshape: funciona en el navegador, no instala nada.
- FreeCAD: open source y gratuito.

Busca en la sección de Recursos los enlaces oficiales de aprendizaje.

### Paso 4 — Diseña la versión simple primero
Empieza por una pieza que cumpla **solo el punto de partida**: la sujeción al raíl y la sujeción de la cámara, sin el mecanismo de giro. Prueba en CAD que la cámara "entre" en tu soporte (una simple caja con los agujeros M3 alineados es suficiente para validar).

### Paso 5 — Agrega el eje de pitch
Divide tu diseño en **2 piezas**:
- **Base**: se agarra al raíl.
- **Placa de la cámara**: sujeta la D435i.

Únelas con una **horquilla y un pivote** (un eje) cuya línea quede perpendicular a la dirección de vuelo. Añade el mecanismo de fijación de tu elección (Paso 6).

### Paso 6 — Decide cómo fijar el ángulo
Elige un mecanismo (ver tabla abajo), añádelo al modelo e incluye las **marcas graduadas** de ángulo.

### Paso 7 — Dimensiona para imprimir
Consejos de impresión del autor del modelo de referencia:
- Imprime la base **con los agujeros del raíl perpendiculares a la cama** (quedan circulares y calzan bien).
- Usa **soportes tipo "tree" (auto)** solo debajo de las zonas en voladizo.
- Revisa qué significa *layer height*, *infill* y *supports* en tu slicer antes de imprimir.

### Paso 8 — Ensambla y prueba
1. Imprime tus piezas y ensámblalas con la tornillería M3 (consigue tuercas y tornillos con cabeza adecuada).
2. Usa un **inclinómetro** (app de tu celular funciona bien) para verificar que el ángulo marcado corresponde al real.
3. Monta la cámara, conecta el cable USB y verifica que puedas girar todo el rango sin obstrucciones.
4. Prueba que el ángulo **no se mueva solo** al agitar el soporte (simula vibración).

### Paso 9 — Documenta y entrega
Fotos del ensamble, tus archivos CAD, el ángulo real medido en cada marca y un texto corto explicando las decisiones de diseño.

---

## Mecanismos posibles para fijar el ángulo

| Mecanismo | Cómo funciona | Ventajas | Desventajas |
|-----------|---------------|----------|-------------|
| **Tornillo de fricción lateral** | Un tornillo atraviesa un lado de la horquilla y presiona contra el pivote o la placa. | Simple, pocas piezas, ajuste continuo. | Con vibración fuerte puede aflojarse; requiere apriete suficiente. |
| **Arco ranurado + tornillo de mariposa** | La placa tiene un arco con ranura; un tornillo de mariposa lo aprieta contra la base. | Muy firme, fácil de graduar el ángulo en el arco. | Un poco más de piezas; la ranura exige más precisión de impresión. |
| **Agujeros discretos + pasador** | Varios agujeros a ángulos fijos (0°, 15°, 30°...) y un pasador que los traba. | Extremadamente rígido, repetición exacta. | Ángulos predeterminados, no continuos. |

Elige uno, justifica tu elección en el reporte y menciona qué harías diferente en una segunda iteración.

---

## Estructura de proyecto sugerida

```
realsense-gimbal/
├── README.md                  # Este documento
├── CAD/
│   ├── base.f3d (o .step)     # Pieza que se monta al raíl
│   ├── placa_camara.f3d       # Pieza que sujeta la cámara
│   └── ensamble.f3d           # Ensamble completo
├── STL/
│   ├── base.stl
│   ├── placa_camara.stl
│   └── (archivos .3mf del slicer)
├── fotos/
│   └── (fotos del ensamble y de pruebas)
└── reporte.md                 # Decisiones de diseño, medidas y resultados
```

Esta es una sugerencia: organízalo como prefieras, pero **entrega** los archivos fuente del CAD, no solo el STL.

---

## Entregables

1. **Archivos CAD nativos** (p. ej. `.f3d`, `.step`) y **STL** de todas las piezas.
2. **Lista de hardware** usada (tornillos, tuercas, pernos M3, etc.).
3. **Piezas impresas ensambladas** en el raíl con la cámara montada.
4. **Verificación de ángulos**: tabla con el ángulo marcado vs. el ángulo medido con inclinómetro.
5. **Reporte corto** (Markdown, PDF o documento) que explique:
   - Qué mecanismo elegiste y por qué.
   - Rendimiento del soporte (¿se mueve con vibración? ¿es fácil ajustar?).
   - Qué cambiarías en la próxima versión.

---

## Criterios de aceptación

Tu propuesta se considera **aprobada** cuando se cumple todo lo siguiente:

- [ ] El soporte se monta en el raíl de 60 mm del X650 y queda firme (no se desliza ni suelta).
- [ ] La cámara D435i calza en el soporte usando sus tornillos de montaje (M3) o la rosca 1/4-20, y queda sujeta sin moverse.
- [ ] El ángulo de pitch se ajusta **a mano** (sin herramientas especiales) y se fija con un tornillo.
- [ ] El ángulo elegido **no cambia solo** al agitar o vibrar el soporte.
- [ ] Roll y yaw permanecen fijos durante el ajuste y el uso.
- [ ] El rango de inclinación cubre el pedido en los requisitos (de 0° a 90°).
- [ ] Las marcas graduadas impresas corresponden al ángulo real medido con inclinómetro (error ≤ 5°).
- [ ] El cable USB y las lentes de la cámara quedan accesibles y sin obstrucciones; las ventilas no están tapadas.
- [ ] Se entregan los archivos CAD nativos y STL de todas las piezas.
- [ ] El ensamble quedó documentado con fotos y una lista del hardware usado.
- [ ] El reporte explica la elección del mecanismo de fijación y qué cambiarías en la próxima versión.

---

## Consejos

- **Empieza con una pieza simple.** Un soporte básico bien impreso vale más que un diseño complejo que no cierra. Perfecciona después.
- **Mide con calibre, no adivines.** El error de 0.5 mm en un agujero M3 se nota a la hora de ensamblar.
- **Usa el modelo de referencia como inspiración o remix.** Printables permite hacer "remixes" con crédito al autor; es legítimo y te ahorra validar el agarre al raíl.
- **Deja tolerancia.** Los agujeros y ejes impresos necesitan juego: agrega ~0.2–0.4 mm sobre la medida nominal.
- **La primera impresión casi nunca sale perfecta.** Espera usar 2–3 iteraciones. Es parte del desafío.
- **Verifica el ángulo con una app de inclinómetro**; tu ojo no es exacto a 5°.
- **Aprieta el tornillo con un dedo o con un destornillador cualquiera.** Si necesitas una llave especial, reconsidera el diseño: las pruebas deben ser rápidas.
- **La cámara genera calor**: deja circular aire (agujeros de ventilación en la placa) y nunca tapes las lentes.

---

## Recursos

### Cámara
- [Intel RealSense D435i — Página oficial](https://www.intelrealsense.com/depth-camera-d435i/)
- [Datasheet D400 Series (dimensiones y montaje)](https://www.intelrealsense.com/wp-content/uploads/2024/10/Intel-RealSense-D400-Series-Datasheet-October-2024.pdf) — secciones de *Mechanical Dimensions* y *Mounting Guidance*.
- [Guía de montaje de cámaras RealSense (documentación)](https://dev.intelrealsense.com/)

### Modelo de referencia
- [Angled Rail Mount for Intel Realsense Camera (S500/S550/X500)](https://www.printables.com/model/1374162-angled-rail-mount-for-intel-realsense-camera-s500s) — autor: 0xkryo.

### CAD (elige uno, no todos)
- [Fusion 360 — Aprender (Autodesk)](https://learn.fusion360.autodesk.com/) — gratis para estudiantes.
- [Onshape — Learning Center](https://learn.onshape.com/) — funciona en el navegador, gratis para educación.
- [FreeCAD — Documentación](https://wiki.freecad.org/) — open source.

### Impresión 3D
- [Prusa Knowledge Base — Diseño de piezas para impresión 3D](https://help.prusa3d.com/) — tolerancias, soportes, orientación de impresión.
- [Conceptos de slicing (Prusa)](https://help.prusa3d.com/tag/slicing) — layer height, infill, supports.

### Conceptos de mecánica
- [Gimbal — Wikipedia](https://es.wikipedia.org/wiki/Gimbal)
- [Movimiento de pitch, roll y yaw — Wikipedia](https://es.wikipedia.org/wiki/Convenci%C3%B3n_de_ejes_aerodin%C3%A1micos)