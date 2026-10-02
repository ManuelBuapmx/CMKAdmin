# CMKAdmin · Panel de administración de CheckMyKnowledge

**Propiedad intelectual exclusiva de PRISMAL MESH.** Todos los derechos reservados. Marca: `PRISMAL MESH` (dos palabras, mayúsculas, separadas por un espacio).

Panel web para que el profesor administre **preguntas, alumnos y resultados** de CheckMyKnowledge, la app Android de exámenes de opción múltiple con reglas anti-trampa.

Este repositorio contiene **un solo archivo**: `admin.html` (HTML + CSS + JS puro, sin librerías ni build). Se abre en un navegador; **no va dentro del APK**. La app del alumno vive en otro repositorio (`checkmyknowledge`).

## Contenido del repo

| Archivo | Qué es |
|---|---|
| `admin.html` | El panel completo: login, preguntas, alumnos, importar examen por script y resultados. |
| `README.md` | Este documento. |

## Cómo usarlo

1. Descarga o abre `admin.html` en un navegador (Chrome, Edge, Opera, etc.). No requiere instalación ni compilación.
2. Inicia sesión con el **correo y contraseña del profesor** (usuario de Supabase Auth).
3. Navega por las pestañas: **Preguntas · Alumnos · Importar · Resultados**.

Si lo publicas en algún hosting, recuerda que el panel es una página estática: toda la seguridad depende de las políticas RLS de Supabase (ver más abajo), no de ocultar el archivo.

## Backend (Supabase)

El panel habla directo con la API REST de Supabase usando la clave `anon` (pública por diseño). Tras iniciar sesión usa el `access_token` del profesor, y las políticas RLS "admin puede…" le permiten leer y escribir.

Tablas que usa:

| Tabla | Uso en el panel |
|---|---|
| `materias` | Selectores de materia (el profesor ve también las inactivas). Solo lectura desde el panel. |
| `preguntas` | Alta, edición, borrado e importación. |
| `alumnos` | Lista autorizada para entrar al examen (`matricula`, `nombre`, `grupo`, `activo`). |
| `resultados` | Consulta y borrado de exámenes contestados. |

### Políticas RLS necesarias para el usuario admin

- `materias`: SELECT.
- `preguntas`: SELECT, INSERT, UPDATE, DELETE.
- `alumnos`: SELECT, INSERT, UPDATE, DELETE. La importación por CSV usa *upsert*, que necesita INSERT **y** UPDATE.
- `resultados`: SELECT, DELETE.

Además, la columna `alumnos.activo` debe tener `default true`, porque la importación por CSV no la envía. Verificado el 2/oct/2026: ya la tiene, y las políticas de admin (limitadas al correo del profesor) existen para `alumnos`, `preguntas`, `materias` y `resultados`.

**Alumno de prueba** (cargado en Supabase y usado como ejemplo en el panel): `123456 · ROBLES GONZÁLEZ JOSÉ MANUEL · 10B`.

El rol `anon` **no** debe tener lectura sobre `alumnos`: la app del alumno solo usa las funciones RPC `validar_matricula` y `calificar_examen`.

## Pestañas

### Preguntas
- Lista con orden, texto, etiqueta de sección, materia, respuesta correcta y estado.
- Filtro por materia.
- Alta y edición con: materia (obligatoria), sección (opcional), contexto (opcional: link de YouTube o texto de apoyo), texto, 4 opciones con la correcta marcada, orden y casilla "Activa".
- Borrado individual con confirmación.
- Una pregunta **sin materia** se marca como "⚠ Sin materia" y no aparece en la app del alumno.

### Alumnos
- Lista con búsqueda (por matrícula o nombre, sin distinguir acentos) y filtro por grupo.
- **Nuevo alumno**: matrícula, nombre, grupo y casilla "Activo". La matrícula no se puede editar después.
- **Activar / Desactivar**: impide la entrada sin borrar historial.
- **Borrar**: elimina al alumno; sus resultados anteriores se conservan.
- **Importar CSV**: columnas `matricula,nombre,grupo`, con o sin encabezado (si no hay encabezado se asume ese orden). Separador coma, punto y coma o tabulador. Guarda el Excel como **CSV UTF-8** para conservar acentos.
  - Vista previa por fila: *Nuevo*, *Actualiza* o el error (falta matrícula, nombre o grupo; matrícula repetida en el archivo).
  - Si la matrícula ya existe, se actualizan nombre y grupo; **`activo` no se toca**.
- Todas las escrituras sobre `alumnos` verifican que la base realmente aplicó el cambio; si el RLS lo bloquea, el panel avisa en lugar de fingir éxito.

### Importar (examen por script)
Se pega un script al estilo **Google Apps Script para Forms** y se ejecuta en el navegador para generar la vista previa.

Llamadas soportadas:

| Llamada | Efecto |
|---|---|
| `FormApp.create('Título')` / `exam.setTitle(...)` | El título es la **sección** de sus preguntas. Si el script crea varios exámenes, se importan todos, en orden. |
| `exam.addMultipleChoiceItem()` / `addCheckboxItem()` | Crea una pregunta real. |
| `item.setTitle(...)` | Texto de la pregunta. |
| `item.createChoice(valor, esCorrecta)` + `item.setChoices([...])` | Opciones y respuesta correcta. |
| `item.setChoiceValues([...])` + `item.setCorrectAnswer('B')` o índice | Forma alternativa de definir opciones y correcta. |
| `setRequired`, `setPoints`, `setHelpText`, `setDescription`, `setIsQuiz`, `setConfirmationMessage`, `setShuffleQuestions` | Se aceptan y se ignoran. |
| `addTextItem`, `addParagraphTextItem`, `addListItem` | Se aceptan y se ignoran (los datos del alumno ya los pide la app). |

Comportamiento:
- Si el script define funciones, **se ejecutan todas sin argumentos** al final.
- Cada pregunta se valida: título, al menos 2 opciones y una correcta marcada. Con errores, el botón **Confirmar examen** queda bloqueado.
- En la vista previa se puede **editar** (sección, contexto, texto, opciones, orden, activa) o **quitar** cada pregunta.
- Se elige la **materia destino** (por defecto, la primera).
- Casilla **Reemplazar el examen activo de esta materia**: desactiva solo las preguntas activas de esa materia antes de insertar. Si no se marca, el orden se corre para quedar después de las existentes.

### Resultados
Tabla con alumno, matrícula, grupo, materia, puntaje y fecha. Los resultados anteriores a la matrícula muestran "—". Se pueden borrar de uno en uno.

## Limitaciones conocidas

- **Sesión:** solo se guarda el `access_token` en `localStorage` (clave `cmk_session`); no hay renovación. Al vencer, el panel cierra la sesión y pide entrar de nuevo.
- **Importador:** el script se ejecuta con `new Function` y tiene acceso al `localStorage` del panel (incluido el token). **Pega solo scripts propios.**
- **Importador:** no soporta secciones por encabezado, videos ni saltos de página de Forms; las funciones auxiliares que reciben parámetros (p. ej. `addQuestion(exam, ...)`) se llaman sin argumentos y fallan.
- **Reemplazo no atómico:** al importar con "Reemplazar", primero se desactivan las preguntas y luego se insertan las nuevas. Si el insert falla, la materia queda sin preguntas activas.
- **Sin gestión de materias** ni de secciones desde el panel: las materias se administran en Supabase y la sección es solo un texto libre en `preguntas.seccion`.
- **Borrado de preguntas y resultados:** no muestra error si el RLS lo bloquea (a diferencia de `alumnos`).
- El panel carga la fuente **Inter** desde Google Fonts; sin internet usa la fuente del sistema, sin afectar el funcionamiento.
- El `<img class="logo">` de la barra superior aún no tiene `src` (pendiente el SVG de PRISMAL MESH).

## Reglas para quien modifique el código (persona o IA)

1. **Comenta el código.** Los comentarios explican el *porqué*, no solo el qué.
2. **Entrega el archivo completo** (`admin.html` entero), no fragmentos.
3. **Actualiza este README** en cada cambio relevante.
4. **Nunca se maneja la respuesta correcta en la app del alumno.** Aquí sí se ve (es el panel del profesor); la calificación la hace `calificar_examen` en Supabase.
5. La clave `anon` de Supabase es pública por diseño; la seguridad real está en las políticas RLS.
6. **No dar a `anon` permisos de lectura sobre `alumnos`.**
7. Las escrituras sobre `alumnos` deben seguir pasando por `escribirAlumnos()` para detectar cambios bloqueados por RLS.
8. **Identidad de marca:** todo comentario, documento o variable que nombre a la empresa usa `PRISMAL MESH`. Los archivos nuevos llevan en el encabezado: `Propiedad intelectual de PRISMAL MESH. Todos los derechos reservados.`

## Relación con la app del alumno

- Repo de la app: `checkmyknowledge` (Android, WebView + `index.html`, anti-trampa en `MainActivity.java`).
- Lo que el profesor guarda aquí (materias, secciones, contexto, alumnos) es lo que la app lee y valida. Si cambias columnas o reglas en Supabase, revisa ambos repos.

## Historial de cambios

- **1/oct/2026** — README inicial del repo. El panel incluye pestañas Preguntas, Alumnos (con importación CSV), Importar y Resultados (con matrícula y grupo), y la identidad visual PRISMAL MESH (tema oscuro, fuente Inter).
- **2/oct/2026** — `admin.html`: ejemplo del CSV con el alumno de prueba real (`123456 · ROBLES GONZÁLEZ JOSÉ MANUEL · 10B`) y comentario que lo explica. Se corrigió el archivo publicado, que había quedado con marcadores de conflicto de merge sin resolver. El mismo alumno se cargó en la tabla `alumnos` de Supabase.
