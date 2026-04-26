# **Resumen de Arquitectura Final: Sistema de Acciones**
---
## 1. Vistazo Conceptual

El sistema de acciones está diseñado para ser **centralizado, desacoplado y escalable**. En lugar de tener una única función `UseItem` monolítica, el sistema funciona a través de un **evento mundial** que es capturado por un **despachador central (`ActionManager`)**. Este despachador no ejecuta la lógica por sí mismo, sino que la redirige al componente "especialista" apropiado que ya existe en el actor (jugador o PNJ).

El principio fundamental es que la Interfaz de Usuario (UI) es "ciega"; no conoce a los sistemas de juego. Simplemente anuncia una intención al mundo, y el `ActionManager` se encarga de dirigir el tráfico.

---
## 2. Componentes Principales

* **`UI (WBP_Slot)`:** El origen de la interacción. Es responsable de determinar qué acción se solicita.
* **`Evento Global (OnActionRequested)`:** El mensajero universal que comunica la intención desde la UI al resto del juego.
* **`ActionManager` (Singleton):** La "torre de control" o "centralita telefónica". Es un `Object` (o `GameInstanceSubsystem`) que escucha el evento global y redirige las órdenes.
* **`Componentes Especialistas`:** Los componentes persistentes en el actor que contienen la lógica real para cada acción (ej. `InventoryComponent`, `EquipmentComponent`, `FightComponent`).

---
## 3. Flujo de Ejecución Detallado

El proceso completo, desde el clic del jugador hasta la ejecución, sigue estos pasos:

1.  **Interacción del Usuario:** El jugador interactúa con un `WBP_Slot` (doble clic, menú contextual, hotbar, etc.). El slot solo conoce su propio **índice** y su **`Tag de Contexto`** (ej. `UI.Context.Inventory`).

2.  **Determinación de la Acción:** La lógica de la UI determina el **`ActionTag`** que se debe ejecutar. Para acciones por defecto (doble clic, hotbar), sigue esta lógica:
    a. Primero, revisa el `MultiUseData` del ítem para ver si alguna regla de contexto define una acción prioritaria.
    b. Si no, determina la acción por defecto basándose en el `ItemTypeTag` del ítem (ej. `Item.Type.Equipment` resulta en la acción `Action.Equip`).

3.  **Disparo del Evento Global:** La UI emite el evento `OnActionRequested` con un "paquete de datos" ligero y preciso:
    * `RequestingActor` (El personaje del jugador o PNJ).
    * `ContextTag` (El tag del slot, ej. `UI.Context.Inventory`).
    * `SlotIndex` (El índice del slot en su componente de origen).
    * `ActionTag` (La acción específica a realizar, ej. `Action.Equip`).

4.  **Recepción y Dirección (`ActionManager`):**
    * El `ActionManager` es el único que escucha este evento.
    * **Paso A (Interpretar Contexto):** Usa el `ContextTag` para identificar el componente de **origen** en el `RequestingActor`. Si el tag es `UI.Context.Equipment`, sabe que debe buscar el `EquipmentComponent`.
    * **Paso B (Validar):** Comprueba que el `SlotIndex` sea válido para ese componente de origen.

5.  **Despacho al Especialista (`ActionManager`):**
    * El `ActionManager` ahora mira el `ActionTag` para decidir qué componente "especialista" debe hacer el trabajo.
    * Utiliza una lógica interna (`Switch on Gameplay Tag`) para redirigir la orden:
        * Si `ActionTag` es `Action.Equip` o `Action.Unequip` -> El especialista es el **`EquipmentComponent`**.
        * Si `ActionTag` es `Action.Consume` o `Action.Drop` -> El especialista es el **`InventoryComponent`**.
        * Si `ActionTag` es `Action.Throw` -> El especialista es el **`FightComponent`** (o el `InventoryComponent`, según el diseño).

6.  **Ejecución Final:**
    * El `ActionManager` obtiene una referencia al componente especialista y le llama, pasándole los datos necesarios (como el `SlotIndex`).
    * El componente especialista (`EquipmentComponent`, etc.) finalmente ejecuta la lógica detallada y encapsulada para esa acción.

Este flujo garantiza un sistema donde la UI está completamente desacoplada de la lógica del juego, y la lógica de cada acción está contenida en su propio módulo, haciendo que el sistema sea robusto y fácil de expandir en el futuro.