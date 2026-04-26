# Sistema de Menú Contextual y Ejecución de Acciones

## Principio Fundamental
El sistema está diseñado para ser dinámico y desacoplado. La lógica de qué acciones son posibles está centralizada en una librería de funciones, mientras que la ejecución final recae en los componentes correspondientes, orquestados por un manager global. La UI solo se encarga de solicitar la información y construir los botones.

---
## 1. Sistema de Validación de Acciones (En `BPFL_Utilities`)
Este es el "cerebro" que decide qué acciones se muestran en el menú. Es un proceso de dos fases.

### **Función 1: `GetValidActions` (Validación en el Origen)**
- **Propósito:** Genera una lista de reglas de acción válidas basándose en el ítem y su contenedor de origen.
- **Entradas Clave:** `ItemInfo`, `SlotHandler`, y los booleanos `IsMultiContainer?` y `ValidateMultiUse?`.
- **Lógica Interna Encapsulada:**
    1.  **`MultiUse`:** Si `ValidateMultiUse?` es verdadero, analiza el `MultiUseData` del ítem. Si encuentra una sola acción válida para el contexto, la sugiere. Si encuentra varias, sugiere `Action.OpenMultiUseMenu`.
    2.  **Reglas Genéricas:** Recorre una serie de lógica `if-then` interna para determinar acciones genéricas.
        -   **Regla "Transferir":** Se valida si el ítem/contenedor cumplen sus propiedades/capacidades Y si el booleano `IsMultiContainer?` es verdadero.
        -   **Otras Reglas:** Se validan combinaciones de `ItemProperty` y `ContainerCapability` para conceder acciones como `Split`, `Drop`, `Use`, etc.
    3.  **Filtro Final de Origen:** La lista de acciones candidatas se filtra contra la lista de **Restricciones** de acción del `SourceHandler`.
- **Salida:** Un `Array` de `STR_ActionRules` que pasaron la validación del origen.

### **Función 2: `ValidateTargetActions` (Validación en el Destino)**
- **Propósito:** Filtra la lista de reglas del paso anterior si existe un destino.
- **Lógica:** Si se proporciona un `TargetHandler` válido, recorre la lista de reglas y la filtra comparando las **propiedades del ítem** contra las listas de **permisos y restricciones de ítems** del `TargetHandler`.
- **Salida:** La lista final y completamente validada de `STR_ActionRules`.

---
## 2. Sistema de Construcción de la UI (`WBP_ContextMenu`)
Recibe la lista final de `STR_ActionRules` y construye la interfaz.

-   **Decisión de Tipo de Botón (Simple vs. Numérico):**
    -   La lógica reside en el widget. Para cada regla, comprueba si el `ActionResult` es hijo del tag `Action.Type.Numeric` **Y** si el ítem tiene la propiedad `Item.Property.IsStackable`. Si ambas son ciertas, crea un botón numérico; si no, uno simple.
-   **Decisión de la Etiqueta del Botón:**
    -   Consulta la `DataTable` de presentación `DT_ActionNames`.
    -   Busca la fila por `ActionTag` y recorre su lista de `ContextualLabels`. Si encuentra una que coincida con el `SourceContext`, usa esa etiqueta.
    -   Si no hay coincidencias, usa la `DefaultLabel`.

---
## 3. Sistema de Ejecución de Acciones (Dentro de `InventoryComponent`)
El evento `ExecActions` (llamado por el `BP_UseItemManager`) usa un `Switch on GameplayTag` para ejecutar la lógica de cada acción.

-   **`Action.Transfer`:**
    -   Implementa un flujo seguro de 3 pasos: `Validar -> Quitar -> Añadir`.
    -   **1. Validar:** Llama a `CanAcceptItem` en el `TargetHandler`.
    -   **2. Quitar:** Si es válido, llama a `RemoveAnItem` en el `SourceHandler`.
    -   **3. Añadir:** Si se quitó con éxito, llama a `PickUpItem` en el `TargetHandler`.
    -   **Failsafe:** Si el paso 3 falla, el ítem se suelta en el mundo a los pies del jugador para evitar su pérdida.
-   **`Action.Split`:**
    -   Operación interna en el mismo contenedor.
    -   Llama a `PickUpItem` para crear la nueva pila. Si tiene éxito, llama a `RemoveAnItem` para descontar de la pila original.
-   **`Action.Drop`:**
    -   Llama a `RemoveAnItem` con el booleano `ShouldSpawn` en verdadero.
-   **`Action.Use`:**
    -   Lógica de 2 pasos coordinada por el `UseItemManager`:
        1.  El manager ordena al `SourceHandler` **consumir** el ítem (`RemoveAnItem`).
        2.  Si tiene éxito, el manager ordena al `ActionInitiator` **aplicar el efecto** (a través de una interfaz como `BPI_EffectReceiver`).
-   **`Action.Inspect`:**
    -   Actualmente, solo ejecuta un `Print String` con la información del ítem.