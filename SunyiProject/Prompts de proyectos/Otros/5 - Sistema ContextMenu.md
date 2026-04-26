# Resumen de Contexto: Sistema de Inventario y Acciones RPG-Survival (UE 5.3 Blueprints)

## Sistema de Datos
- **`STR_ItemInfo`:** Estructura de datos estática que define una plantilla de ítem. Contiene `PropertyTags` (características inherentes como `IsStackable`).
- **`STR_ActionRules`:** Estructura que define una regla de acción simple:
    - `RequiredItemProperty` (`GameplayTag`)
    - `RequiredContainerCapability` (`GameplayTag`)
    - `ActionResult` (`GameplayTag`)
- **`STR_ActionNames`:** Estructura para la `DataTable` de presentación. Contiene el nombre por defecto y las variaciones contextuales.
    - `DefaultLabel` (`Text`)
    - `ContextualLabels` (`Array` de `STR_ContextualLabel`)
- **`STR_ContextualLabel`:** Estructura anidada que define una variación de nombre.
    - `SourceContext` (`GameplayTag`)
    - `Label` (`Text`)

---
## Sistema Unificado de Ejecución de Acciones (`UseItem`)
- **`STR_UseItemPayload`:** Estructura de datos que viaja por el sistema. Contiene:
    - `ActionInitiator` (Actor)
    - `SourceContainerOwner` (Actor)
    - `DestinationHandler` (`BPI_SlotHandler`)
    - `ActionTag` (`GameplayTag`)
    - `Quantity` (Integer)
- **`BP_UseItemManager`:** Manager singleton que orquesta todas las acciones. Recibe el `Payload` y delega la ejecución al componente correcto. Para transferencias, coordina la comunicación entre el origen y el destino.
- **`BPI_SlotHandler`:** Interfaz que implementan todos los componentes contenedores para ser "manejables" y proveer sus datos (capacidades, restricciones, etc.).

---
## Sistema de Menú Contextual Dinámico
El objetivo es generar una lista de acciones válidas sin `DataTables` de lógica.

### **Proceso de Validación (Dos Fases en `BPFL_Utilities`)**

1.  **Función 1: `GetValidSourceRules` (Validar Origen)**
    * **Entradas:** `ItemInfo`, `SourceHandler`.
    * **Lógica:**
        1.  La lógica está **encapsulada** en la función. No depende de `GameInstance` o `DataTables`.
        2.  Contiene una serie de comprobaciones `if-then` que actúan como un "libro de reglas" interno. Cada regla comprueba una combinación de `ItemProperty` y `ContainerCapability` para conceder un `ActionResult`.
        3.  El resultado es una lista de `STR_ActionRules` que son válidas para el ítem y las capacidades del origen.
        4.  Esta lista se filtra contra las **Restricciones de acción** del `SourceHandler`. El filtro de **Permisos** se descartó en favor de un control más directo a través de las Capacidades.
    * **Salida:** Un `Array` de `STR_ActionRules` que pasaron la validación del origen.

2.  **Función 2: `ValidateTargetActions` (Validar Destino)**
    * **Entradas:** La lista de reglas del paso anterior, el `TargetHandler` y el `ItemInfo`.
    * **Lógica:**
        1.  Si no hay `TargetHandler`, devuelve la lista sin cambios.
        2.  Si hay destino, recorre la lista de reglas. Por cada regla, comprueba si el **ítem** es aceptado por el destino, comparando las **propiedades del ítem** contra las listas de **Permisos** (`AllowedItems`) y **Restricciones** (`ExcludedItems`) de **ítems** del `TargetHandler`.
    * **Salida:** La lista final y completamente validada de `STR_ActionRules`.

### **Construcción de la UI (`WBP_ContextMenu`)**
1.  El widget recibe la **lista final de reglas válidas** (`Array<STR_ActionRules>`).
2.  Recorre esta lista para crear los botones.
3.  **Decisión del Tipo de Botón (Simple vs. Numérico):**
    * La lógica reside en el widget. Para cada regla, realiza la ¿El ítem tiene la propiedad `Item.Property.IsStackable`?
    * Si es cierta, crea un `WBP_NumericButton`; si no, un `WBP_ActionButton`.
4.  **Decisión de la Etiqueta del Botón:**
    * Se usa una `DataTable` (`DT_ActionNames`) **solo para presentación**.
    * La UI busca la fila por `ActionTag`. Recorre el array `ContextualLabels` de la fila. Si encuentra una entrada cuyo `SourceContext` coincide con el del contenedor actual, usa esa etiqueta.
    * Si no encuentra ninguna coincidencia, usa la `DefaultLabel`.

### **Sistema `MultiUse`**
-   Se maneja de forma independiente.
-   El menú contextual solo mostrará una acción genérica (`Action.OpenMultiUseMenu`) si el ítem es `MultiUse`.
-   Al hacer clic, esta acción abrirá una **GUI dedicada (ej. un menú radial)**, encapsulando su propia lógica y presentación.

---









# Resumen de Contexto: Sistema de Inventario y Acciones RPG-Survival (UE 5.3 Blueprints)

## Sistema de Datos Clave
- **`STR_ItemInfo`:** Estructura de datos estática que define una plantilla de ítem. Contiene `PropertyTags` (características inherentes como `IsStackable`) y `MultiUseBehavior` (para acciones especiales). **Esta estructura no se modifica.**
- **`STR_ActionRules`:** Estructura que define una regla de acción genérica. Esta estructura vive como un "libro de reglas" local dentro de la función de validación.
    - `RequiredItemProperty` (`GameplayTag`)
    - `RequiredContainerCapability` (`GameplayTag`)
    - `ActionResult` (`GameplayTag`)
- **`STR_ActionNames`:** Estructura para la `DataTable` de presentación (`DT_ActionNames`).
    - `DefaultLabel` (`Text`)
    - `ContextualLabels` (`Array` de `STR_ContextualLabel`)
- **`STR_ContextualLabel`:** Estructura anidada que define una variación de nombre.
    - `SourceContext` (`GameplayTag`)
    - `Label` (`Text`)

---
## Sistema Unificado de Ejecución de Acciones (`UseItem`)
- **`STR_UseItemPayload`:** Estructura de datos que viaja por el sistema. Contiene:
    - `ActionInitiator` (Actor)
    - `SourceContainerOwner` (Actor)
    - `DestinationHandler` (`BPI_SlotHandler`)
    - `ActionTag` (`GameplayTag`)
    - `Quantity` (Integer)
- **`BP_UseItemManager`:** Manager singleton que orquesta todas las acciones. Recibe el `Payload` y delega la ejecución al componente correcto.

---
## Sistema de Validación de Acciones (El "Motor de Reglas")
La lógica completa vive en una librería de funciones (`BPFL_Utilities`) para ser totalmente autónoma. El proceso se divide en dos funciones.

### **Función 1: `GetValidSourceActions` (Validar Origen)**
Esta función determina qué acciones son posibles basándose en el ítem y su contenedor de origen.
- **Entradas:** `ItemInfo`, `SourceHandler`, `ValidateMultiUse?` (Bool).
- **Lógica Interna (Encapsulada):**
    1.  **Validación de `MultiUse`:** Si el booleano de entrada lo permite, la función primero analiza el `MultiUseData` del ítem.
        -   Encuentra todas las acciones `MultiUse` válidas para el contexto actual.
        -   Si hay **una sola** acción, la añade como candidata.
        -   Si hay **más de una**, añade la acción genérica `Action.OpenMultiUseMenu`. Si el flag `AcceptRandom?` es verdadero, también añade `Action.ExecuteRandomMultiUse`.
    2.  **Validación de Acciones Genéricas:** La función recorre una lista interna de `STR_ActionRules`. Por cada regla, comprueba si el ítem y el contenedor cumplen con la `RequiredItemProperty` y la `RequiredContainerCapability`. Las acciones de las reglas que pasan se añaden a la lista de candidatos.
    3.  **Filtro Final de Origen:** La lista combinada de acciones (candidatas de `MultiUse` + genéricas) se filtra una última vez contra la lista de **Restricciones** de acción del `SourceHandler`.
- **Salida:** Un `Array` de `STR_ActionRules` que pasaron la validación del origen.

### **Función 2: `ValidateTargetActions` (Validar Destino)**
- **Entradas:** La lista de reglas del paso anterior, el `TargetHandler` y el `ItemInfo`.
- **Lógica:**
    1.  Si no hay `TargetHandler`, devuelve la lista sin cambios.
    2.  Si hay destino, recorre la lista de reglas. Por cada regla, comprueba si el **ítem** es aceptado por el destino, comparando las **propiedades del ítem** contra las listas de **Permisos** (`AllowedItems`) y **Restricciones** (`ExcludedItems`) de **ítems** del `TargetHandler`.
- **Salida:** La lista final y completamente validada de `STR_ActionRules`.

---
### **Construcción de la Interfaz de Usuario (`WBP_ContextMenu`)**
1.  El widget recibe la **lista final de reglas válidas** (`Array<STR_ActionRules>`).
2.  Recorre esta lista para crear los botones.
3.  **Decisión del Tipo de Botón (Simple vs. Numérico):**
    -   La lógica reside en el widget. Para cada regla, se comprueba si el `ActionResult` de la regla tiene el tag padre `Action.Type.Numeric` **Y** si el ítem es `IsStackable`. Si ambas son ciertas, crea un botón numérico.
4.  **Decisión de la Etiqueta del Botón:**
    -   Se usa la `DataTable` `DT_ActionNames` **solo para presentación**.
    -   La UI busca la fila por `ActionTag`. Recorre el array `ContextualLabels`. Si encuentra una entrada cuyo `SourceContext` coincide con el del contenedor actual, usa esa etiqueta.
    -   Si no encuentra ninguna coincidencia, usa la `DefaultLabel`.

### **Sistema `MultiUse`**
-   Se maneja de forma inteligente. Si hay más de una acción `MultiUse` disponible, el menú contextual solo mostrará un botón genérico ("Multi-acción").
-   Al hacer clic, esta acción abrirá una **GUI dedicada (ej. un menú radial)**, encapsulando su propia lógica y presentación.