# Análisis: desactivar nodos y herramientas 3D del motor y del editor (enfoque 2D)

**Fecha:** 13 de abril de 2026  
**Árbol analizado:** Godot (rama de desarrollo local, `b:\godot\editores\v4\godot`)  
**Objetivo:** Evaluar qué habría que cambiar para **compilar un editor orientado a 2D** eliminando (o no registrando) la capa 3D de escena, física/navegación 3D asociada y el editor 3D, con **optimización** (tiempo de compilación, tamaño binario, menos código muerto) para quien **solo desarrolla en 2D**.

---

## 1. Resumen ejecutivo

| Enfoque | Viabilidad | Nota |
|--------|------------|------|
| **`disable_3d=yes` en SCons** | Solo **plantillas de exportación** (`template_debug` / `template_release`) | El **editor** lo rechaza explícitamente en `SConstruct`. |
| **Flags de preprocesador `_3D_DISABLED` / `PHYSICS_3D_DISABLED` / …** | Bien integrados en **runtime** (`scene/`, `servers/`, tests) | Hay **cientos** de referencias; el **editor** casi no usa `_3D_DISABLED`. |
| **“Editor 2D only” de verdad** | Requiere **trabajo de ingeniería grande** | Hay que envolver o excluir **~114 archivos** solo bajo `editor/scene/3d/`, más plugins, importadores 3D, menús del editor, etc. |

**Conclusión:** Hoy Godot ya permite un binario **de juego** sin API 3D (`disable_3d` en plantillas). Un **editor** sin 3D **no está soportado** por la misma palanca de compilación; sería un **fork o conjunto de parches** que extienda `disable_3d` al `target=editor` y condicione gran parte de `editor/`.

---

## 2. Estado actual en el código

### 2.1. Restricción de compilación (editor vs plantilla)

En `SConstruct`, si `target=editor` y se activa `disable_3d`, la configuración **termina con error**. Las opciones `disable_3d`, `disable_physics_3d`, etc., están en la lista de opciones **no permitidas para el editor** (solo plantillas de exportación).

Efecto colateral documentado en el mismo archivo: con `disable_3d` se define `_3D_DISABLED` y se fuerzan `disable_navigation_3d`, `disable_physics_3d` y `disable_xr`.

### 2.2. Runtime: registro de tipos 3D

En `scene/register_scene_types.cpp`, bloques grandes están bajo `#ifndef _3D_DISABLED`: includes de `scene/3d/`, recursos 3D, nodos XR (si no están desactivados), registro `GDREGISTER_*` de toda la jerarquía 3D, física 3D y navegación 3D cuando aplica.

Esto **sí** reduce superficie de API y compilación del **motor** cuando se construye una **plantilla** con `disable_3d=yes`.

### 2.3. Servidores y render

Aparecen guardas `_3D_DISABLED` en piezas como `servers/rendering/renderer_viewport.cpp` y `servers/register_server_types.cpp` (registro condicional de subsistemas 3D/navegación/física según flags).

La **pipeline de renderizado 3D** (Forward+, clustering, etc.) sigue siendo una parte importante del árbol; desactivar “nodos 3D” **no** implica automáticamente un motor sin **código de render 3D** a menos que se definan **opciones adicionales** (p. ej. renderers, módulos de malla, compresión de mallas) y se audite `servers/rendering/`.

### 2.4. Módulos condicionados por `disable_3d`

Varios `modules/*/config.py` deshabilitan el módulo si `env["disable_3d"]` (p. ej. CSG, FBX, glTF, GridMap, meshoptimizer, según el árbol). Cualquier build sin 3D debe revisar **dependencias** entre módulos y `doc_classes`.

### 2.5. Editor: casi sin ` _3D_DISABLED`

`editor/register_editor_types.cpp` **incluye y registra** de forma incondicional clases y plugins 3D (`EditorNode3DGizmo`, `Camera3DEditorPlugin`, `MeshInstance3DEditorPlugin`, importadores `ResourceImporterOBJ` / `ResourceImporterScene`, `Texture3DEditorPlugin`, etc.). No hay bloques `#ifndef _3D_DISABLED` equivalentes a los de `scene/register_scene_types.cpp`.

Solo un **puñado** de archivos bajo `editor/` mencionan `_3D_DISABLED` (p. ej. partes de `editor_node.cpp`, `game_view_plugin.cpp`, `project_manager.cpp`, `mesh_library_editor_plugin.cpp`), frente a la base completa del editor 3D.

**Dato de alcance:** del orden de **114 archivos** bajo `editor/scene/3d/` (solo esa carpeta), sin contar importadores, shaders del editor, docks compartidos, etc.

### 2.6. Perfil de build en el editor (UI)

`editor/settings/editor_build_profile.cpp` lista identificadores como `"disable_3d"` para **alinear la UI del editor** con las capacidades del binario exportado; eso **no sustituye** a compilar el editor sin 3D, sino a **mostrar/ocultar** opciones coherentes con la plantilla.

---

## 3. Cambios necesarios para un “editor 2D only” (visión por capas)

### 3.1. Sistema de build (SCons)

1. **Permitir `disable_3d=yes` con `target=editor`**  
   - Eliminar o acotar la comprobación que aborta si el editor usa `disable_3d` / ciertas opciones relacionadas.  
   - Revisar **dependencias**: tests del editor, herramientas de documentación, recuperación de proyectos 3D.

2. **Propagación de defines**  
   - Garantizar `_3D_DISABLED` (y derivados) en **toda** la unidad de compilación del editor, igual que en plantillas.  
   - Validar **Mono/GDExtension** si aplica: APIs 3D inexistentes pueden romper generación de glue o bindings.

### 3.2. Registro y UI del editor

1. **`editor/register_editor_types.cpp`**  
   - Envolver includes y `GDREGISTER_*` / `EditorPlugins::add_by_type<...>()` para todo lo 3D bajo `#ifndef _3D_DISABLED`.  
   - Incluye plugins en `editor/scene/3d/`, gizmos 3D, importadores 3D, editores de texturas 3D, etc.

2. **`EditorNode` y flujos globales**  
   - Menús “Nodo raíz” (2D vs 3D), vistas predeterminadas, atajos, **Game View**, selección de viewport 3D, recuperación de escenas con nodos 3D.  
   - Cualquier `if` que asuma existencia de clases 3D debe tener rama segura o desaparecer.

3. **Project Manager / plantillas**  
   - Plantillas de proyecto y mini-ejemplos que crean escena 3D por defecto.

4. **Inspector y dock de nodos**  
   - Ocultar categorías 3D; evitar referencias a tipos no registrados.

### 3.3. Importación y assets

- Importadores que requieren mallas/escenas 3D (`ResourceImporterScene`, OBJ, etc.): desactivar o compilar solo stubs.  
- Flujos GLTF/FBX/CSG: ya ligados a `disable_3d` en módulos; el **editor** debe dejar de registrarlos si el módulo no existe.

### 3.4. Servidores y thirdparty

- Auditar **`servers/`** y **`drivers/`** para código 3D que siga enlazándose sin nodos 3D (p. ej. partes de **RenderingDevice**, sombras, GI).  
- Decidir si el objetivo es “**sin nodos 3D**” o “**sin render 3D**” (más agresivo; impacta shaders, tests y compatibilidad).

### 3.5. Tests y CI

- Tests bajo `tests/` que instancian nodos 3D o escenas mixtas: excluirlos o usar `#ifndef _3D_DISABLED`.  
- Ajustar pipelines que asumen editor completo.

### 3.6. Documentación y traducciones

- Referencias en docs generadas, lista de clases, y cadenas del editor que asumen siempre disponible el 3D.

---

## 4. Alternativas sin recompilar el motor

Para **desarrollo 2D** sin fork:

- **Perfil de funciones del editor** y **feature tags**: ocultar nodos o reducir ruido en la UI sin tocar C++.  
- **Viewport** con 3D desactivado en el proyecto / escena.  
- **Optimización real** (FPS/tiempo de apertura) en editor stock suele venir más de **renderer**, **caché**, y **hardware** que de eliminar clases 3D del binario.

Estas opciones **no** reducen tamaño del ejecutable ni tiempo de link como un build `disable_3d` completo.

---

## 5. Riesgos y coste

| Riesgo | Descripción |
|--------|-------------|
| **Mantenimiento** | Cada versión upstream añade nodos/plugins 3D; hay que re-aplicar guardas. |
| **Proyectos ajenos** | Abrir un `.godot` con escenas 3D en un editor sin 3D exige degradación elegante o rechazo explícito. |
| **Extensiones** | Plugins de AssetLib que asumen `Node3D` fallarán al cargar. |
| **Rendimiento** | El beneficio marginal tras eliminar solo **registro de nodos** puede ser menor que el esperado si el **render 3D** sigue en el binario. |

**Orden de magnitud:** no es un cambio de “unas decenas de líneas”; es un **proyecto multi-semana** con revisión continua de `editor/`, `scene/` y `servers/`, más QA.

---

## 6. Recomendaciones

1. **Si el objetivo es binario de juego 2D:** usar ya **`disable_3d=yes`** en **plantillas** y medir tamaño/tiempos; el editor puede seguir siendo el estándar.  
2. **Si el objetivo es obligatoriamente editor sin 3D:** planificar un **fork** con la lista de la sección 3, empezando por **levantar la restricción** en `SConstruct` y **`editor/register_editor_types.cpp`**.  
3. **Antes de invertir en fork:** validar si **perfiles del editor** + **viewport 2D** cubren la “experiencia 2D” deseada.

---

*Documento generado como análisis técnico del árbol local; no forma parte de la documentación oficial de Godot.*
