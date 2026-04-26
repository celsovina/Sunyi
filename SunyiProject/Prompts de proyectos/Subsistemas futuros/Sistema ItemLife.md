---
# **Sistema "ItemLife" (Vida Útil de Ítems)**
**Objetivo**: Implementar mecánicas de desgaste para herramientas/armas y pudrición para consumibles, permitiendo transformaciones de ítems y variaciones de "vitalidad" por instancia (ej. por rareza o calidad de crafteo).
---
## 1. **Nuevos Enums Sugeridos para ItemLife**
*(Ubicación: `Content/GameDataTypes/Data/Enums/ItemLife/` o similar)*
- **ENUM_DegradationTrigger** (Valores en Inglés, ej: `EDT_OnUse`, `EDT_OnHitDealt`, `EDT_OnHitReceived`, `EDT_OverTime`): Define cómo se activa la degradación primaria de un ítem.
- **ENUM_ItemCondition** (Valores en Inglés, ej: `EIC_Pristine`, `EIC_Worn`, `EIC_Damaged`, `EIC_Broken`, `EIC_Fresh`, `EIC_Stale`, `EIC_Rotten`): Podría usarse para estados visuales o lógicos intermedios. (Opcional, las transformaciones de ItemID también pueden cubrir esto).

---
## 2. **Modificaciones y Nuevas Estructuras Sugeridas para ItemLife**
*(Ubicación: `Content/GameDataTypes/Data/Structs/ItemLife/` o similar, y modificaciones a `STR_ItemInfo` y `STR_Slot`)*

1.  **STR_ItemInfo (Modificaciones para Configuración de ItemLife)**:
    * Se añadirían campos o una sub-estructura (ej. `LifeSystemConfigData`) para definir las *reglas base* de la vida útil. Todas las variables internas en Inglés.
    * **`HasPrimaryDegradation? (Boolean)`**: ¿El ítem tiene un sistema de "usos" o "durabilidad" que se consume por eventos?
    * **`MaxPrimaryStateValue (Int32)`**: Valor máximo de usos/durabilidad base (ej. 100 para Espada, 2 para Pan).
    * **`PrimaryDegradationTrigger (ENUM_DegradationTrigger)`**: Cómo se consume el estado primario.
    * **`IntermediateStateChanges (Array of STR_StateChangeRule)`**: Para transformaciones progresivas (ej. Espada -> Espada Abollada).
        * **`STR_StateChangeRule`** (Nueva Struct):
            * `PrimaryStateThreshold (Int32)`: Umbral del estado primario para la transformación.
            * `TransformTo_NewItemID (Name)`: El `ItemID` al que se transforma.
    * **`DepletedResultItems (Array of STR_ResultItemQuantity)`**: Ítems generados cuando el estado primario llega a 0 (ej. partes de espada rota).
        * **`STR_ResultItemQuantity`** (Nueva Struct): `ResultItemID (Name)`, `ResultQuantity (Int32)`.
    * **`DoesSpoil? (Boolean)`**: ¿El ítem se pudre con el tiempo?
    * **`SpoilageTime_Seconds (Float)`**: Tiempo total (segundos de juego) para pudrirse completamente.
    * **`SpoiledResultItems (Array of STR_ResultItemQuantity)`**: Ítems generados al pudrirse (ej. Carne Cruda -> Carne Podrida).
    * **`BaseRarityMultiplier (Float)`**: (Opcional) Multiplicador base si la rareza afecta directamente la vida útil máxima definida aquí. (Alternativamente, la rareza de la instancia modifica la durabilidad al crearse la instancia).

2.  **STR_Slot (Modificación Mínima Confirmada)**:
    * **ItemID (Name)** (Existente)
    * **Quantity (Int32)** (Existente)
    * **`ItemInstanceUID (Text)`**: (Añadido) Identificador único para la instancia del ítem si este tiene estado dinámico. Se genera cuando un ítem con potencial de "vida útil" se crea o entra por primera vez a un inventario gestionado. Vacío si el ítem no tiene estado individual. **CRÍTICO: NO SE AÑADIRÁN MÁS CAMPOS DE ESTADO DIRECTAMENTE A `STR_Slot`**.

3.  **STR_TrackedItemState (Nueva Struct para el "Cuaderno" del Guardián de Estados)**:
    * **InstanceUID (Text)**: Vincula esta entrada al `ItemInstanceUID` en `STR_Slot`.
    * **CurrentPrimaryStateValue (Int32)**: Durabilidad o usos actuales de esta instancia.
    * **InstanceMaxPrimaryStateValue (Int32)**: Durabilidad o usos máximos para *esta instancia específica* (después de aplicar modificadores de rareza/calidad a `MaxPrimaryStateValue` de `STR_ItemInfo`).
    * **SpoilProcessStartTime (Float)**: `GetGameTimeInSeconds()` cuando el ítem entró en estado "fresco" o se reinició su contador de pudrición. Usado para calcular el progreso de pudrición.
    * **IsBrokenOrFullyUsed? (Boolean)**: Flag que indica si el estado primario se ha agotado.

---
## 3. **Nuevos/Modificados Blueprints y Componentes para ItemLife**

1.  **`BP_ItemStateGuardianComponent` (Nuevo Componente de Actor)**:
    * Se añade a actores que "poseen" ítems y gestionan activamente su vida útil (ej. `BP_ThirdPersonCharacter`, NPCs relevantes).
    * **Variables**:
        * `TrackedItemInstanceStates (Array of STR_TrackedItemState)`: El "cuaderno" que almacena el estado actual de todos los ítems con `ItemInstanceUID` que posee el actor.
    * **Funciones Principales**:
        * `RegisterNewStatefulItem(InSlot STR_Slot, InItemInfo STR_ItemInfo, InitialRarity ENUM_Rarity) returns ItemInstanceUID (Text)`:
            * Genera un nuevo `ItemInstanceUID`, lo asigna a `InSlot.ItemInstanceUID` (esta función necesitaría modificar el slot o devolver el UID para que el llamador lo asigne).
            * Crea una nueva entrada `STR_TrackedItemState`.
            * Calcula `InstanceMaxPrimaryStateValue` basado en `InItemInfo.MaxPrimaryStateValue` y `InitialRarity`.
            * Inicializa `CurrentPrimaryStateValue = InstanceMaxPrimaryStateValue`.
            * Inicializa `SpoilProcessStartTime` si `InItemInfo.DoesSpoil?` es `true`.
            * Añade la entrada a `TrackedItemInstanceStates`.
        * `GetItemState(ItemInstanceUID Text) returns (STR_TrackedItemState FoundState, Boolean Success?)`.
        * `UpdateItemPrimaryState(ItemInstanceUID Text, NewStateValue Int32)`.
        * `ProcessItemUsage(ItemInstanceUID Text, UsageType ENUM_DegradationTrigger, UsageAmount Int32)`: Reduce `CurrentPrimaryStateValue`, verifica umbrales de `IntermediateStateChanges` y `DepletedResultItems` de `STR_ItemInfo`, y gestiona las transformaciones de `ItemID` en el `STR_Slot` correspondiente (notificando al `InventoryComponent` dueño del slot).
        * `TickSpoilageForAllItems(DeltaSeconds Float)`: Itera `TrackedItemInstanceStates`, actualiza el progreso de pudrición basado en `SpoilProcessStartTime` y `SpoilageTime_Seconds` de `STR_ItemInfo`. Gestiona la transformación a `SpoiledResultItems` si un ítem se pudre (notificando al `InventoryComponent`).
            * **Mecánica de Spoilage para Stacks**: Si un ítem en stack se pudre, reduce la `Quantity` del `STR_Slot` en 1, genera el `SpoiledResultItem`, y reinicia el `SpoilProcessStartTime` para el stack restante (si `Quantity > 0`).

2.  **InventoryComponent (Modificaciones para Interactuar con ItemLife)**:
    * **Al Añadir un Ítem Nuevo que Puede Tener Estado**:
        * Llama a `BP_ItemStateGuardianComponent.RegisterNewStatefulItem` de su actor propietario para obtener/asignar el `ItemInstanceUID` al `STR_Slot` y registrar su estado inicial.
    * **Al Mover un Ítem con `ItemInstanceUID` a un Contenedor "Pasivo" (sin Guardián propio, como `BP_Bag` o un cofre simple que solo almacena estado "congelado")**:
        * El `InventoryComponent` del contenedor pasivo debe tener su propio array: `FrozenItemInstanceStates (Array of STR_TrackedItemState)`.
        * Cuando se le añade un `STR_Slot` con un `ItemInstanceUID` y su `STR_TrackedItemState` correspondiente (proveniente del Guardián del actor que lo dropeó):
            * Almacena el `STR_Slot` en `InventoryContent`.
            * Almacena el `STR_TrackedItemState` en `FrozenItemInstanceStates`.
    * **Al Retirar un Ítem con `ItemInstanceUID` de un Contenedor Pasivo**:
        * Se recuperan tanto el `STR_Slot` como su `STR_TrackedItemState` asociado desde `FrozenItemInstanceStates`.
        * Esta información se usa para registrar/actualizar el ítem en el `BP_ItemStateGuardianComponent` del nuevo propietario.
    * **Lógica de `UseItem`**: Si el ítem tiene `HasPrimaryDegradation?`, debe notificar al `BP_ItemStateGuardianComponent` del propietario para que llame a `ProcessItemUsage`.

3.  **BP_Bag (Interacción con ItemLife)**:
    * **NO tendrá su propio `BP_ItemStateGuardianComponent`** (porque los ítems dentro no se degradan activamente, solo la bolsa desaparece por tiempo).
    * Sus funciones `SlotHandler` y `MultiSlotHandler` (de `BPI_ItemHandler`) necesitarán aceptar no solo los `STR_Slot`(s) sino también los `STR_TrackedItemState`(s) correspondientes si los ítems dropeados tenían estado.
    * El `InventoryComponent` de `BP_Bag` almacenará estos estados en su array `FrozenItemInstanceStates`.
    * Al recoger de la bolsa, se recupera el `STR_Slot` y su `STR_TrackedItemState` congelado para pasarlo al Guardián del jugador.

---
## 4. **Flujo Conceptual General del ItemLife**
1.  **Definición**: `STR_ItemInfo` define *si* un tipo de ítem se degrada/pudre, *cómo* lo hace (usos, tiempo), cuáles son sus *valores base máximos*, y en qué se *transforma*.
2.  **Instanciación con Estado**: Cuando un ítem con potencial de vida útil entra en posesión de un actor con `BP_ItemStateGuardianComponent` (Jugador/NPC), se le asigna un `ItemInstanceUID` (en `STR_Slot`) y se crea un registro `STR_TrackedItemState` en el "Guardián" de ese actor. La durabilidad/usos máximos iniciales de esta instancia se calculan (`Base * ModificadorDeRareza`).
3.  **Degradación Activa**: El "Guardián" del actor actualiza `CurrentPrimaryStateValue` (durabilidad/usos) cuando el actor usa el ítem, y `SpoilProcessStartTime` para la pudrición pasiva. Gestiona las transformaciones (`IntermediateStateChanges`, `DepletedResultItems`, `SpoiledResultItems`) modificando el `ItemID` en el `STR_Slot` del `InventoryComponent` y añadiendo/quitando ítems según las reglas en `STR_ItemInfo`.
4.  **Transferencia y Almacenamiento Pasivo**: Al mover un ítem con estado a un contenedor sin "Guardián" activo (como `BP_Bag` o un cofre simple), su `STR_Slot` (con `ItemInstanceUID`) y su `STR_TrackedItemState` actual ("congelado") se almacenan en el `InventoryComponent` de ese contenedor. El estado no cambia mientras esté allí.
5.  **Reactivación**: Al mover el ítem de un contenedor pasivo de vuelta a un actor con "Guardián", el estado "congelado" se usa para inicializar el seguimiento activo en el nuevo "Guardián".
---

## **Plan de Desarrollo del Sistema "ItemLife" (Pasos Propuestos)**
* **Paso IL.0: Preparación de Tipos de Datos Base**
    * IL.0.1: Definir y crear `ENUM_DegradationTrigger`.
    * IL.0.2: Definir y crear `STR_ResultItemQuantity`.
    * IL.0.3: Definir y crear `STR_StateChangeRule`.
* **Paso IL.1: Configuración de ItemLife en `STR_ItemInfo`**
    * IL.1.1: Añadir campos/sub-struct de configuración de ItemLife a `STR_ItemInfo`.
    * IL.1.2: Actualizar `DT_ItemInfo` con datos de ItemLife para ítems de prueba.
* **Paso IL.2: Modificación de `STR_Slot` y Creación de `STR_TrackedItemState`**
    * IL.2.1: Añadir `ItemInstanceUID (Text)` a `STR_Slot`.
    * IL.2.2: Definir y crear `STR_TrackedItemState`.
* **Paso IL.3: Creación y Configuración Básica de `BP_ItemStateGuardianComponent`**
    * IL.3.1: Crear `BP_ItemStateGuardianComponent`.
    * IL.3.2: Añadir `TrackedItemInstanceStates` array.
    * IL.3.3: Implementar `RegisterNewStatefulItem`.
    * IL.3.4: Implementar `GetItemState` y `RemoveItemState`.
    * IL.3.5: Añadirlo a `BP_ThirdPersonCharacter` (y base de NPCs).
* **Paso IL.4: Integración de `InventoryComponent` (Actores Activos)**
    * IL.4.1: Modificar `InventoryComponent` para llamar a `RegisterNewStatefulItem` al añadir ítems.
    * IL.4.2: Modificar `InventoryComponent` para llamar a `RemoveItemState` al eliminar ítems.
* **Paso IL.5: Implementación de Degradación Primaria (Durabilidad/Usos)**
    * IL.5.1: Implementar `ProcessItemUsage` en `BP_ItemStateGuardianComponent`.
    * IL.5.2: Modificar sistemas de juego para llamar a `ProcessItemUsage`.
* **Paso IL.6: Implementación de Pudrición (Spoilage)**
    * IL.6.1: Implementar `TickSpoilageForAllItems` en `BP_ItemStateGuardianComponent`.
    * IL.6.2: Decidir cómo/dónde se llama `TickSpoilageForAllItems`.
* **Paso IL.7: Manejo de Estado en Contenedores Pasivos (`BP_Bag`, Cofres Simples)**
    * IL.7.1: Modificar `InventoryComponent` para añadir `FrozenItemInstanceStates`.
    * IL.7.2: Modificar `BP_Bag` y lógica de transferencia para manejar `FrozenItemInstanceStates`.
* **Paso IL.8: Implementación de Modificadores de Rareza (Opcional/Extensión)**
* **Paso IL.9: Integración con UI y Pruebas**
---