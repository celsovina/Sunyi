# Resumen de Arquitectura Final: Sistema de Uso de Ítems (RPG-Survival UE5)

## 1. Principios de Diseño

* **Desacoplado**: La Interfaz de Usuario (UI), los Managers y los Componentes de Lógica no se conocen directamente entre sí. La comunicación se realiza a través de un sistema de eventos e interfaces.
* **Control Centralizado**: Un único manager singleton (`BP_UseItemManager`) actúa como "director de orquesta", recibiendo todas las peticiones de uso y enrutándolas, pero sin contener la lógica de ejecución final.
* **Lógica Especializada**: La lógica final de cada acción (consumir, equipar) reside dentro del componente que contiene el ítem (`InventoryComponent`, `HotbarComponent`, etc.), haciéndolos autosuficientes.
* **Universal**: El sistema funciona de forma idéntica para el Jugador y para los PNJs, ya que el emisor de la petición (UI o IA) es irrelevante para el manager.

## 2. Componentes Clave

* **`UI (WBP_Slot, WBP_ContextMenu)`**: El origen de la interacción del jugador. Su única responsabilidad es empaquetar los datos necesarios en la estructura `STR_UseItemPayload` y disparar el evento global.
* **`GI_Sunyi (GameInstance)`**:
    * Actúa como el contenedor de los singletons. En su evento `Init`, construye una instancia única del `BP_UseItemManager` y la mantiene en una variable.
    * Contiene el `Event Dispatcher` global `OnUseRequested`.
* **`BP_UseItemManager` (Singleton, `Object` class)**:
    * Se vincula al evento `OnUseRequested` del `GameInstance` al ser creado.
    * Es el cerebro del sistema. Recibe la petición, determina la acción final a realizar y encuentra el componente que debe ejecutarla.
    * Delega la orden final a través de una interfaz genérica.
* **Componentes Contenedores (`InventoryComponent`, `HotbarComponent`, `ChestComponent`, etc.)**:
    * Son los componentes que físicamente contienen un array de ítems.
    * Se identifican de forma única mediante un `Component Tag` (tipo `Name`) que se asigna dinámicamente en tiempo de ejecución a partir de un `GameplayTag` público.
    * Implementan la interfaz `BPI_UseItemHandler` para recibir y ejecutar las órdenes del `UseItemManager`.

## 3. Interfaces y Estructuras Clave

* **`STR_UseItemPayload` (Structure)**: El paquete de datos que viaja desde el emisor hasta el ejecutor final. Contiene:
    * `Owner` (Actor): El actor que posee el componente (jugador o PNJ).
    * `ContextTag` (GameplayTag): Identifica de forma única al componente de origen (ej. `UI.Context.Inventory`).
    * `SlotData` (STR_Slot): La estructura completa del slot con el ítem.
    * `SlotSourceIndex` (Integer): El índice del slot en su contenedor.
    * `ActionTag` (GameplayTag): La acción específica a realizar. Puede ir vacío para acciones implícitas.
* **`BPI_UseItemHandler` (Interface)**: El contrato genérico para la ejecución.
    * Contiene **una sola función**: `UseItem(Payload)`.

## 4. Flujo de Ejecución Completo

1.  **Disparo (UI)**: El jugador hace doble clic en un `WBP_Slot`. El widget reúne los datos en un `STR_UseItemPayload`, dejando el `ActionTag` vacío.
2.  **Evento Global**: El widget llama al `Event Dispatcher` `OnUseRequested` en el `GameInstance`, pasando el `Payload`.
3.  **Recepción (Manager)**: El evento `HandleUseRequest` en la instancia única del `BP_UseItemManager` se activa.
4.  **Orquestación (`HandleUseRequest`)**:
    * **a. Determinar Acción**: Llama a su función interna `DetermineActionTag`.
        * Esta función revisa si el `ActionTag` del `Payload` ya es válido (acción explícita). Si es así, lo devuelve.
        * Si no, revisa las reglas de `MultiUseData` del ítem. Si encuentra una regla que coincida con el `ContextTag` del `Payload`, devuelve la acción correspondiente (la primera si hay varias, o una aleatoria si el flag `AcceptRandom?` es verdadero).
        * Si no hay reglas `MultiUse`, determina una acción por defecto basándose en el `ItemTypeTag` del ítem (ej. `Item.Type.Consumable` -> `Action.Consume`).
        * Si no se encuentra ninguna acción válida, devuelve un tag inválido.
    * **b. Encontrar Componente**: Llama a su función interna `FindTargetComponent`.
        * Esta función toma el `ContextTag` del `Payload`, lo convierte a `Name` (`GetTagName`), y usa `GetComponentsByTag` en el `Owner` para encontrar el componente contenedor exacto. Devuelve una referencia a este componente.
    * **c. Delegar Orden**: El evento `HandleUseRequest` valida que tanto la acción como el componente sean válidos. Luego:
        * Actualiza el `Payload` con el `ActionTag` final usando `Set Members`.
        * Llama a la función de interfaz `UseItem` en el componente encontrado, pasándole el `Payload` final.
5.  **Ejecución Final (Componente)**:
    * El componente (`InventoryComponent`, etc.) recibe la llamada a `UseItem`.
    * Dentro de esta función, utiliza un `Switch on Gameplay Tag` con el `ActionTag` del `Payload` para dirigir la ejecución a su lógica interna específica (la función de descontar ítem, la de equipar, etc.).
