# Desafío: Base de Datos de Experimentos

Crea una aplicación de escritorio Electron que sirva como base de datos centralizada para gestionar experimentos de ingeniería. La aplicación usa SQLite para almacenamiento local y permite a los usuarios organizar proyectos, registrar ejecuciones de experimentos, almacenar resultados numéricos y recopilar archivos de salida asociados en un solo lugar.

---

## Descripción General

Al ejecutar experimentos (simulaciones, pruebas de hardware, canalizaciones de procesamiento de datos, etc.), los resultados a menudo terminan dispersos en carpetas, unidades USB y adjuntos de correo electrónico. Este desafío te pide construir una herramienta que ponga orden a ese caos: una única aplicación de escritorio donde cada proyecto, cada ejecución y cada archivo resultante convivan juntos.

---

## Modelo de Datos

La aplicación gira en torno a tres entidades principales:

### 1. Proyecto

Representa un esfuerzo de investigación a largo plazo o un tema de investigación.

| Campo        | Tipo    | Descripción                        |
|-------------|---------|------------------------------------|
| `id`        | INTEGER | Clave primaria autoincrementada    |
| `name`      | TEXT    | Nombre breve y legible del proyecto |
| `description` | TEXT  | Resumen libre del proyecto         |
| `created_at`| TEXT    | Marca de tiempo ISO 8601 de creación |

### 2. Ejecución

Una ejecución o ensayo individual perteneciente a un proyecto. Cada ejecución captura un momento de recopilación de datos.

| Campo        | Tipo    | Descripción                              |
|-------------|---------|------------------------------------------|
| `id`        | INTEGER | Clave primaria autoincrementada          |
| `project_id`| INTEGER | Clave foránea al proyecto padre          |
| `date`      | TEXT    | Fecha ISO 8601 de la ejecución           |
| `notes`     | TEXT    | Notas libres opcionales sobre el ensayo  |
| `created_at`| TEXT    | Marca de tiempo ISO 8601 de creación del registro |

### 3. Resultado

Un solo punto de datos capturado durante una ejecución. Una ejecución puede tener muchos resultados (múltiples lecturas de sensores, métricas, mediciones, etc.).

| Campo        | Tipo    | Descripción                              |
|-------------|---------|------------------------------------------|
| `id`        | INTEGER | Clave primaria autoincrementada          |
| `run_id`    | INTEGER | Clave foránea a la ejecución padre       |
| `label`     | TEXT    | Nombre de la métrica (p. ej. "temperatura") |
| `value`     | REAL    | Valor numérico                           |
| `unit`      | TEXT    | Unidad opcional (p. ej. "°C", "ms")         |

### 4. Archivo

Un archivo producido por una ejecución (salida CSV, imágenes, registros, datos binarios, etc.). La aplicación copia el archivo a una carpeta de almacenamiento gestionada y registra la referencia.

| Campo        | Tipo    | Descripción                                    |
|-------------|---------|------------------------------------------------|
| `id`        | INTEGER | Clave primaria autoincrementada                |
| `run_id`    | INTEGER | Clave foránea a la ejecución padre             |
| `filename`  | TEXT    | Nombre original del archivo                    |
| `stored_path`| TEXT   | Ruta dentro de la carpeta de almacenamiento gestionada |
| `size_bytes`| INTEGER | Tamaño del archivo en bytes                    |
| `created_at`| TEXT    | Marca de tiempo ISO 8601 de cuándo se agregó   |

---

## Almacenamiento de Archivos

Cuando se adjunta un archivo a una ejecución, la aplicación debe:

1. Copiar el archivo (no mover) a un directorio de almacenamiento dedicado.
2. Organizar los archivos usando la siguiente estructura:
   ```
   <raiz_almacenamiento>/
     <project_id>/
       <run_id>/
         <nombre_archivo_original>
   ```
3. Si ya existe un archivo con el mismo nombre en el destino, agregar un sufijo numérico para evitar sobrescribir (p. ej. `output_1.csv`, `output_2.csv`).
4. Registrar la ruta almacenada en la base de datos para que el archivo pueda abrirse o mostrarse más tarde.

La raíz de almacenamiento debe ser configurable (por defecto `./experiment_files` relativo al directorio de trabajo de la aplicación).

---

## Requisitos de la Aplicación

### Funcionalidades Principales

1. **Gestión de Proyectos**
   - Crear un nuevo proyecto con nombre y descripción.
   - Listar todos los proyectos con sus nombres y descripciones.
   - Editar el nombre o descripción de un proyecto.
   - Eliminar un proyecto (y eliminar en cascada todas sus ejecuciones, resultados y archivos almacenados).

2. **Gestión de Ejecuciones**
   - Crear una nueva ejecución bajo un proyecto, con fecha y notas opcionales.
   - Listar todas las ejecuciones de un proyecto dado, ordenadas por fecha descendente.
   - Editar la fecha o notas de una ejecución.
   - Eliminar una ejecución (y eliminar en cascada sus resultados y archivos almacenados, y eliminar sus archivos del disco).

3. **Gestión de Resultados**
   - Agregar uno o más resultados a una ejecución, cada uno con una etiqueta, valor numérico y unidad opcional.
   - Ver todos los resultados de una ejecución en una tabla.
   - Editar o eliminar resultados individuales.

4. **Adjuntar Archivos**
   - Adjuntar uno o más archivos a una ejecución seleccionándolos a través de un diálogo de selección de archivos nativo.
   - Copiar los archivos seleccionados al directorio de almacenamiento gestionado (como se describió anteriormente).
   - Ver todos los archivos adjuntos a una ejecución, mostrando nombre del archivo, tamaño y fecha de agregado.
   - Abrir un archivo con la aplicación predeterminada del sistema (p. ej. doble clic para abrir en Excel).
   - Mostrar un archivo en el administrador de archivos del sistema (p. ej. "Mostrar en carpeta").
   - Eliminar una referencia de archivo de la base de datos y eliminar el archivo físico del disco.

5. **Navegación**
   - Una barra lateral o vista de nivel superior que muestre todos los proyectos.
   - Hacer clic en un proyecto muestra sus ejecuciones.
   - Hacer clic en una ejecución muestra sus resultados y archivos adjuntos.

### Directrices de Interfaz de Usuario

- Usa un diseño limpio y responsivo. Un patrón de barra lateral + área de contenido principal funciona bien.
- Los formularios deben tener validación de entrada básica (nombre de proyecto requerido, valores numéricos para resultados, etc.).
- Las tablas deben mostrar los datos claramente con encabezados de columna apropiados.
- Proporciona retroalimentación visual para las acciones: notificaciones de éxito para guardados, diálogos de confirmación para eliminaciones.
- La ventana de la aplicación debe tener un tamaño mínimo razonable (p. ej. 900×600).

---

## Requisitos Técnicos

### Stack

- **Electron** (cualquier versión estable reciente) para el shell de escritorio.
- **SQLite** a través de `better-sqlite3` (sincrónico y simple; es un módulo nativo que normalmente se instala desde binarios precompilados) o `sql.js` (puro JavaScript, basado en WebAssembly, sin compilación nativa). Elige el que prefieras.
- **HTML/CSS/JavaScript** (vanilla o con un framework ligero) para el proceso renderer.
- No se requieren bibliotecas de componentes de UI externas — construye las tuyas con CSS puro o un framework CSS mínimo (p. ej. Pico CSS, Water.css).

### Arquitectura

```
experiments-database/
├── package.json
├── main.js                  # Proceso principal de Electron
├── preload.js              # Puente de contexto para IPC
├── src/
│   ├── renderer/
│   │   ├── index.html       # HTML de la ventana principal
│   │   ├── styles.css       # Estilos de la aplicación
│   │   └── app.js           # Lógica del lado del renderer
│   └── database/
│       ├── schema.js        # Creación de tablas y migraciones
│       └── queries.js       # Sentencias preparadas / asistentes de consultas
├── experiment_files/        # Almacenamiento gestionado de archivos (creado en tiempo de ejecución)
└── README.md
```

### Responsabilidades del Proceso Principal

- Inicializar la base de datos SQLite al iniciar la aplicación y asegurar que las tablas existan.
- Exponer las operaciones de base de datos al renderer a través de IPC (comunicación entre procesos) usando un script preload.
- Manejar operaciones del sistema de archivos: copiar archivos adjuntos, eliminar archivos, listar directorios.
- El proceso renderer **no** debe tener acceso directo a la base de datos o al sistema de archivos. Todas las operaciones pasan a través de canales IPC expuestos por el script preload.

### Ejemplos de Canales IPC

Diseña tus propios canales, pero aquí hay un patrón sugerido:

```
project:create        -> { name, description }
project:list          -> returns [{ id, name, description, created_at }]
project:update        -> { id, name, description }
project:delete        -> { id }

run:create            -> { project_id, date, notes }
run:list              -> { project_id } -> returns [runs]
run:update            -> { id, date, notes }
run:delete            -> { id }

result:create         -> { run_id, label, value, unit }
result:list           -> { run_id } -> returns [results]
result:update         -> { id, label, value, unit }
result:delete         -> { id }

file:attach           -> { run_id, sourcePath } (el renderer envía la ruta desde el diálogo de archivos)
file:list             -> { run_id } -> returns [files]
file:open             -> { id } (abre con la aplicación predeterminada del sistema)
file:reveal           -> { id } (muestra en el administrador de archivos del sistema)
file:delete           -> { id }
```

### Restricciones de Base de Datos

- Las claves foráneas **deben** ser forzadas. Habilita `PRAGMA foreign_keys = ON;` al abrir la base de datos.
- Usa transacciones al crear una ejecución con múltiples resultados o adjuntar múltiples archivos para atomicidad.

---

## Guía de Construcción Paso a Paso

Sigue estos pasos en orden. Cada paso se construye sobre el anterior.

### Paso 1: Andamiaje del Proyecto

1. Crea un nuevo directorio para el proyecto.
2. Inicialízalo con `npm init`.
3. Instala Electron como dependencia de desarrollo y `better-sqlite3` como dependencia de producción.
4. Crea los archivos básicos `main.js`, `preload.js` y `src/renderer/index.html`.
5. Configura `package.json` con el campo `"main": "main.js"` y un script de inicio.
6. Ejecuta `npm start` y verifica que la ventana de Electron se abra con una página en blanco.

### Paso 2: Configuración de la Base de Datos

1. Crea `src/database/schema.js` que exporte una función para inicializar la base de datos.
2. Crea las cuatro tablas (projects, runs, results, files) con tipos y restricciones apropiados.
3. Habilita la restricción de claves foráneas.
4. Conecta esto en `main.js` para que la base de datos se cree/abra cuando la aplicación inicie.
5. Registra en la consola cuando la base de datos se haya inicializado exitosamente.

### Paso 3: Exponer la Base de Datos a través de IPC

1. Crea un archivo `preload.js` usando `contextBridge` para exponer una API segura al renderer.
2. Para cada entidad (projects, runs, results, files), implementa manejadores IPC en el proceso principal.
3. En el preload, expone funciones como `window.api.createProject({ name, description })`, etc.
4. Prueba llamando a estas funciones desde la consola de desarrollador del renderer.

### Paso 4: Interfaz CRUD de Proyectos

1. Construye el diseño principal con una barra lateral (para navegación) y un área de contenido principal.
2. En el área principal, muestra una lista de todos los proyectos.
3. Agrega un botón "Nuevo Proyecto" que abra un formulario en línea o modal para ingresar nombre y descripción.
4. Implementa la edición y eliminación de proyectos (con un diálogo de confirmación antes de eliminar).
5. Verifica que eliminar un proyecto lo remueva de la lista.

### Paso 5: Interfaz de Gestión de Ejecuciones

1. Cuando se seleccione un proyecto en la barra lateral, muestra sus ejecuciones en el área principal.
2. Agrega un botón "Nueva Ejecución" con un formulario para fecha (selector de fecha) y notas opcionales.
3. Implementa la edición y eliminación de ejecuciones.
4. Confirma que las ejecuciones se ordenen por fecha (más reciente primero).

### Paso 6: Interfaz de Resultados

1. Cuando se seleccione una ejecución, muestra sus resultados en una tabla debajo de los detalles de la ejecución.
2. Agrega un formulario para ingresar un resultado: etiqueta (texto), valor (número), unidad (texto, opcional).
3. Permite agregar múltiples resultados en secuencia.
4. Implementa la edición en línea y eliminación de resultados.
5. Muestra un conteo resumido (p. ej. "5 resultados registrados").

### Paso 7: Adjuntar Archivos

1. En la vista de detalles de la ejecución, agrega un botón "Adjuntar Archivo" que abra un selector de archivos nativo.
2. Cuando se seleccionen archivos, envía sus rutas al proceso principal a través de IPC.
3. En el proceso principal, copia cada archivo a `<raiz_almacenamiento>/<project_id>/<run_id>/`.
4. Registra cada archivo en la base de datos.
5. Muestra los archivos adjuntos en una lista con nombre del archivo, tamaño y fecha.
6. Agrega botones "Abrir" y "Eliminar" para cada archivo.
7. Maneja nombres de archivo duplicados agregando un sufijo numérico.

### Paso 8: Pulido

1. Agrega notificaciones de éxito/error para todas las acciones del usuario.
2. Agrega validación de entrada (evita nombres de proyecto vacíos, asegura valores numéricos para resultados).
3. Estiliza la aplicación para un aspecto limpio y profesional.
4. Maneja casos extremos: estados vacíos ("Aún no hay proyectos", "No hay ejecuciones para este proyecto"), truncamiento de texto largo, archivos de gran tamaño.
5. Prueba el flujo completo: crear proyecto → agregar ejecución → agregar resultados → adjuntar archivos → verificar que los archivos existen en el disco → eliminar y verificar la limpieza.

---

## Criterios de Aceptación

Tu entrega está completa cuando:

- [ ] La aplicación se inicia sin errores.
- [ ] Puedes crear, editar y eliminar proyectos.
- [ ] Puedes crear, editar y eliminar ejecuciones dentro de un proyecto.
- [ ] Puedes agregar, editar y eliminar resultados numéricos en una ejecución.
- [ ] Puedes adjuntar archivos a una ejecución, y se copian físicamente al directorio de almacenamiento organizado.
- [ ] Puedes abrir archivos adjuntos con la aplicación predeterminada del sistema.
- [ ] Eliminar un proyecto elimina en cascada todas las ejecuciones, resultados, archivos y archivos almacenados en el disco.
- [ ] Eliminar una ejecución elimina en cascada sus resultados, archivos y archivos almacenados en el disco.
- [ ] La interfaz de usuario es funcional y razonablemente estilizada.

---

## Extensiones Opcionales (Puntos Extra)

Si terminas antes, considera agregar:

- **Búsqueda**: Una barra de búsqueda que filtre proyectos, ejecuciones o resultados por nombre/etiqueta.
- **Exportación**: Exportar las ejecuciones y resultados de un proyecto a un archivo CSV o JSON.
- **Gráficas**: Mostrar valores de resultados a lo largo del tiempo usando una biblioteca de gráficas (p. ej. Chart.js).
- **Etiquetas**: Permitir etiquetar ejecuciones con palabras clave para filtrado entre proyectos.
- **Importación masiva**: Importar resultados desde un archivo CSV a una ejecución.
- **Modo oscuro**: Un interruptor para tema claro/oscuro.
- **Copia de seguridad de base de datos**: Un botón para exportar el archivo completo de la base de datos SQLite.

---

## Recursos

- [Documentación de Electron](https://www.electronjs.org/docs)
- [Documentación de better-sqlite3](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md)
- [Guía de IPC de Electron](https://www.electronjs.org/docs/latest/tutorial/ipc)
- [API de contextBridge](https://www.electronjs.org/docs/latest/api/context-bridge)
