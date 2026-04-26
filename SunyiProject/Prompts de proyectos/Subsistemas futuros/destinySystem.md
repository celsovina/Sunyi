# **Resumen del Sistema de Destinos (`Destiny System`)**
---
## **1. Vistazo Conceptual**

El "Sistema de Destinos" es una arquitectura de progresión basada en hitos que, aunque se diseñó pensando en la evolución de ítems, es lo suficientemente genérica para aplicarse a otras entidades como compañeros, asentamientos o facciones.

* **Foco y Distinción:** Es un sistema **centrado en la entidad** (el ítem, el compañero, etc.), no en el personaje. Su propósito es contar la "historia" o el "viaje" de esa entidad, a diferencia del `QuestSystem`, que cuenta la historia del personaje. Ambos sistemas pueden enlazarse.

* **Descubrimiento:** Un destino asignado a una entidad no es conocido por el jugador por defecto. Debe ser "descubierto" a través de acciones de investigación en el juego (usar una estación de investigación, leer un libro, hablar con un PNJ). Esto se rastrea mediante un booleano en los datos de instancia de la entidad.

* **Activación:** Los destinos se asignan a las instancias a través de los `EmpiricalDiscoveryPaths` definidos en el `STR_ItemInfo` del ítem base, o como recompensa de una misión. Un gestor global (`DestinyManager`) escucha los eventos del juego y, cuando se cumplen las condiciones de activación de un destino, lo asigna a la instancia de ítem correcta.

* **Reglas de Prioridad:** Para evitar ambigüedades si múltiples ítems candidatos existen, el gestor sigue un orden de prioridad para asignar el destino:
    1.  El ítem directamente involucrado en el evento contextual.
    2.  Si no hay contexto, el ítem compatible que esté equipado.
    3.  Si sigue habiendo ambigüedad, se le presenta al jugador una UI para que elija.

---
## **2. Estructuras y Enums Requeridos**

### **Enums de Soporte**

* **`ENUM_MilestoneCompletionLogic`**:
    * `MCL_Sequential`: Los hitos deben completarse en orden.
    * `MCL_All`: Todos los hitos deben completarse, en cualquier orden.
    * `MCL_Any`: Basta con completar un solo hito de la lista.

* **`ENUM_DestinyRepeatability`**:
    * `DRE_OncePerGame`: Único para toda la partida.
    * `DRE_OncePerInstance`: Cada instancia puede completarlo una vez.
    * `DRE_Conditional`: Repetible si se cumplen las `ReactivationConditions`.

* **`ENUM_MilestoneType`**:
    * `MIL_CheckStat`: Comprueba un stat numérico.
    * `MIL_CheckCharacterTag`: Comprueba un tag en el personaje/dueño.
    * `MIL_CheckInventoryByID`: Comprueba el inventario por `ItemID` y cantidad.
    * `MIL_CheckInventoryByProperty`: Comprueba el inventario por `PropertyTags` y cantidad.
    * `MIL_CheckWorldState`: Comprueba un tag de estado global del mundo.

### **Estructuras Principales**

* **`STR_Milestone` (Define un Hito Atómico)**:
    * `MilestoneDesc` (`FText`): Descripción para el diseñador.
    * `ConditionType` (`ENUM_MilestoneType`): Define el tipo de condición.
    * `MilestoneTag` (`FGameplayTagContainer`): Contenedor de tags a verificar (para `MIL_CheckCharacterTag` o `MIL_CheckInventoryByProperty`).
    * `NameID` (`FName`): El ID a verificar (para `MIL_CheckInventoryByID`).
    * `ComparisonValue` (`float`): El valor numérico objetivo.
    * `ComparisonMethod` (`ENUM_ComparisonMethod`): El operador de comparación.
    * `MilestoneRewards` (`TArray<STR_StatData>`): Lista opcional de "mini-recompensas" al completar este hito.

* **`STR_Destiny` (Define un Destino Completo - Fila de `DT_Destinies`)**:
    * `DisplayName` (`FText`): Nombre visible para el jugador.
    * `Description` (`FText`): Descripción visible para el jugador.
    * `LinkedQuestID` (`FName`): ID opcional de la misión que se enlaza al descubrir este destino.
    * `OriginatingTriggers` (`FGameplayTagContainer`): Enlace inverso a los `EmpiricalDiscovery` que pueden iniciarlo.
    * `CompletionLogic` (`ENUM_MilestoneCompletionLogic`): Define cómo se deben completar los hitos.
    * `Milestones` (`TArray<STR_Milestone>`): La lista de hitos que componen el destino.
    * `Repeatability` (`ENUM_DestinyRepeatability`): Regla de repetibilidad.
    * `ReactivationConditions` (`TArray<STR_FXCondition>`): Condiciones para que un destino repetible vuelva a estar disponible.
    * `UseMilItems?` (`bool`): Si `true`, consume los ítems de los hitos de inventario al completar el destino.
    * `FinalRewards` (`TArray<STR_StatData>`): La gran recompensa al completar el destino.