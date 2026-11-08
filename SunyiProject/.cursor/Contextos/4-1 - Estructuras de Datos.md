# Sistema de Interacción Contextual — Estructuras de Datos

> Nombre técnico en Unreal (carpeta y assets): **`ContextActionSystem`**.

---

## Descripción del sistema

Este documento formaliza la **declaración de datos y estructuras** acordadas para el Sistema de Interacción Contextual, continuando desde `4-0 - Sistema de Acciones.md`. Reemplaza y detalla el boceto provisional de estructuras que quedó en ese documento (que era una primera aproximación, ya superada en varios campos).

El foco de este documento es exclusivamente estructural: qué datos existen, su forma, y cómo se relacionan entre sí y con las estructuras ya congeladas de `STR_ItemInfo`. No repite los principios generales ni el flujo de Resolución/Ejecución, que ya están documentados en `4-0`.

---

## Principios de diseño

- **Una acción = una fila.** Toda la información de una acción (requisitos, etiqueta, ejecución) vive en un único lugar, no repartida entre varias tablas.
- **Sin tabla de "acción default" separada.** La correspondencia propiedad→acción implícita no se guarda por duplicado: se deriva directamente del campo `RequiredItemProperty` de la fila correspondiente en la tabla principal.
- **`ItemTypeTags` no participa en la resolución de acciones.** Esa responsabilidad es exclusiva de `PropertyTags`, específicamente de la familia `Item.Property.Use.*`. `ItemTypeTags` queda reservado a categorización/filtros en otras partes del sistema (esto ya es así en los datos reales: ej. `Item.Type.Consumable`, `Item.Type.Resource`, `Item.Type.Head`).
- **Disciplina de contenido (no estructural) que sostiene la resolución implícita sin ambigüedad:**
  - Cada ítem debe tener como máximo un tag de la familia `Item.Property.Use.*` en su `PropertyTags`.
  - Cada tag de esa familia debe ser el `RequiredItemProperty` de una única fila en la tabla principal.
- **Ninguna estructura ya congelada de `STR_ItemInfo` cambia de forma.** Todo lo nuevo vive en estructuras propias del Sistema de Interacción Contextual (`ContextActionSystem`).
- **Separación física/lógica de las listas de filtrado del contenedor.** "Qué ítems admite este contenedor" (física: gobierna Drag&Drop y el pre-filtro de `Transfer`) y "qué acciones están apagadas aquí" (lógica: independiente del ítem) son responsabilidades distintas y viven en listas separadas — nunca se mezclan en una sola `GameplayTagContainer`, para no forzar a una función de filtro a interpretar ambigüedad sobre qué tipo de tag está revisando.

---

## Estructuras

> Estado: **implementadas en Unreal**, en `Game/ContextActionSystem/Data/Structs/`. Los nombres de esta sección son ya los nombres reales de los assets, no provisionales.

### STR_Requirements
> Pieza reutilizable que representa "qué tags se necesitan y cómo deben coincidir". Se usa tres veces dentro de `STR_ActionDefinition`.

| Campo | Tipo | Contenedor |
|----|---|---|
| TagRequirement | GameplayTag | GameplayTagContainer |
| MatchMode | ENUM_ContextMatching (reutilizado, ya existente) | — |

### STR_ActionDefinition
> Una fila = una acción completa: identidad, requisitos, etiqueta y ruteo de ejecución.

| Campo | Tipo | Contenedor |
|----|---|---|
| ActionTag | GameplayTag | — |
| RequiredItemProperty | STR_Requirements | — |
| RequiredTargetCapability | STR_Requirements | — |
| RequiredRequesterCapability | STR_Requirements | — |
| DefaultLabel | Text | — |
| ContextualLabels | STR_ContextualLabel (reutilizado, ya existente) | Array |
| ExecMode | Enum_ActionExecMode | — |
| ExecutorReference | GameplayTag | — |
| AcceptNumValues? | Bool | — |
| CompareExternal? | Bool | — |

**Notas de uso de campos:**
- `RequiredItemProperty` vacío = la acción no depende del ítem (ej. Drop, Transfer, Split, Inspect).
- `RequiredItemProperty` con tag(s) de la familia `Item.Property.Use.*` = además de gatillar validez, ese mismo campo es la fuente de verdad para la resolución de la acción implícita.
- `RequiredRequesterCapability` vacío = cualquier solicitante puede pedir la acción. Es el campo que valida capacidades del *actor* (independiente del `InventoryComponent`), pensado para desbloqueos progresivos (ej. "puede inspeccionar").
- `RequiredTargetCapability` valida capacidades de **un contenedor**, nunca del solicitante. Cuál contenedor depende de `CompareExternal?` (**cambio del 07/11/2026**):
  - `CompareExternal? = false` (Split, Drop, Use…): el contenedor **clicado** (origen, `SourceContainer`).
  - `CompareExternal? = true` (Transfer): el contenedor **de destino** (`ExternalTarget` en el menú, `TargetContainer` en el manager). Para Transfer, `Container.Capability.AllowMove` significa "este contenedor acepta que le metan ítems". Un contenedor sin `AllowMove` deja sacar, pero no deja meter (ej. un cofre de misión).
  - Requisito vacío = pasa (chequeo explícito en `ValidateCapabalities`).
  - Antes, esta capacidad se revisaba siempre en el contenedor clicado; con Transfer la regla quedaba invertida (un cofre sin `AllowMove` dejaba meter, pero no sacar).
- `ExecutorReference` (**significado vigente desde el rediseño de la ejecución, 05/11/2026**): tag `System.Handler.*` que identifica el **sistema dueño** que ejecuta la acción (ej. `System.Handler.Inventory`, `System.Handler.Equipment`). `BP_ExecActionManager.FindHandler` lo compara (exacto) contra el `GetHandlerTag` de cada componente candidato. Vacío = la acción no la ejecuta ningún sistema (filas de sistema de la UI como `Action.MultiUse.OpenMenu`/`ExecuteRandom`), y el manager responde `System.Action.Result.Fail.NoHandler` si se intenta ejecutar.
  - *Significado anterior (descartado para este campo)*: con `ExecMode = Effect` iba a ser la categoría de efecto (`Effect.Consume`, `Effect.Throw`). Esa categoría queda pendiente de reubicar (otro campo de la fila o lógica interna del sistema dueño de Use).
- `ExecMode` ya **no decide el enrutado** de la ejecución (lo decide `ExecutorReference`). Se conserva el campo; su uso futuro, si lo tiene, es interno de cada sistema.
- `AcceptNumValues?` marca si esta acción opera sobre una cantidad parcial del stack (Split, Transfer, Drop) y por lo tanto necesita un widget con selector de cantidad al construir el botón, en vez de uno estándar. Es un campo 100% data-driven a propósito (a diferencia de excepciones puntuales como el ocultamiento de Split a `Quantity == 1`, que sí quedan hardcodeadas por ser reglas fijas de una sola acción — ver `4-2`, sección de construcción de botones, para la lógica completa que combina este campo con `STR_Slot.Quantity`).
- `CompareExternal?` marca si la acción depende del contenedor externo emparejado (`ExternalTarget`) en vez del contenedor clicado. Con `true`: (1) la fila se omite del menú si no hay `ExternalTarget` válido; (2) `RequiredTargetCapability` se valida contra el destino, no contra el contenedor clicado; (3) su label contextual se resuelve contra `ExternalTarget.GetContainerType()` en vez de `ClickedHandler.GetContainerType()`; y (4) en el manager, solo las filas con `true` conservan `TargetContainer` (con `false`, `RequestAction` lo deja en None). Hoy solo `ActionTransfer` lo tiene activo (el nombre útil depende de hacia dónde va el ítem: ej. "Guardar" cuando el destino es `Container.Type.Chest`). Lógica completa en `4-2`, `BuildActionsMenu`.

**Nota sobre `ContextualLabels` y su comparación:** la coincidencia entre `SourceContext` (de cada `STR_ContextualLabel`) y el tag de contexto contra el que se compara es de **igualdad exacta** (`EqualEqual_GameplayTag`), no jerárquica — es la única comparación de tags de todo el sistema que no usa `HasTag`/`HasAnyTags` con `bExactMatch=false`, confirmado revisando la implementación del sistema viejo. Contra qué tag se compara (`ClickedHandler` o `ExternalTarget`) depende de la acción y es lógica de construcción del menú, no un campo de esta estructura — ver `4-2`.

**Mecanismo de enlace Acción ↔ Efecto (para `ExecMode = Effect`):**
El propio ítem, en `STR_ItemInfo.StatData`, declara qué efecto concreto le corresponde a cada categoría: cada fila de `EffectRules` (vía numérica, `STR_StatEffect`) etiqueta su campo `EffectTag` con el mismo valor que el `ExecutorReference` de la acción, y cada fila de `ConsumeEffects` (vía actor, `STR_ConsumeEffect`) etiqueta su campo `DataTag` de la misma forma. Como cada ítem normalmente solo tiene una fila relevante por categoría de efecto, no hace falta granularidad adicional (tags hijos) para evitar colisiones: la coincidencia general del tag es suficiente. Esto no modifica la forma de `STR_StatEffect` ni `STR_ConsumeEffect` (siguen congeladas); es una convención de contenido, no una estructura nueva. La clase de actor a spawnear para la vía A vive en `STR_ConsumeEffect.EffectActor` (del propio ítem), nunca en la acción.

### STR_ActionRequest
> El paquete de datos uniforme con el que cualquier solicitante construye el menú de acciones válidas. No es el payload de ejecución (ese es `STR_ActionExec`, más abajo).

| Campo | Tipo | Contenedor |
|----|---|---|
| Requester | Actor | — |
| Target | Interfaz `BPI_SlotHandler` (existente, reutilizada tal cual) | — |
| ItemSlotData | STR_Slot (existente) | — |

> `ContextTag` y `ActionTag` quedaron descartados de esta estructura: pertenecen al momento de ejecución de una acción ya resuelta, no a la construcción del menú de acciones válidas (que es lo único que resuelve `STR_ActionRequest`). `ContextTag` se obtiene directamente de `Target` (`GetContainerType`) cuando se necesita.
>
> **Equivalencia de nombres con el payload:** `STR_ActionRequest.Target` es el contenedor **clicado**, que en `STR_ActionExec` corresponde a **`SourceContainer`** (no a `TargetContainer`). Se decidió no modificar esta estructura (ya hay lógica construida sobre ella); la equivalencia queda documentada aquí.

### STR_ActionExec
> El payload genérico de ejecución. Reemplaza a `STR_UseItemPayload` (descartado). Lo construye cualquier solicitante (menú, hotbar, tecla rápida, IA de PNJ) y lo recibe `BP_ExecActionManager.RequestAction`; el manager lo reenvía tal cual (con la acción ya resuelta) al `ExecuteAction` del sistema dueño. Es 100% genérico: los datos específicos de cada sistema (receta, posición de construcción, stats) los resuelve el propio componente con lo que ya conoce.

| Campo | Tipo | Contenedor | Equivale a (payload viejo) |
|----|---|---|---|
| Requester | Actor | — | ActionInitiator |
| ActionTag | GameplayTag | — | ActionTag |
| ItemSlot | STR_Slot | — | SlotData |
| Quantity | Integer (por defecto 1) | — | Cuantity |
| SourceSlotIndex | Integer (por defecto -1) | — | SlotSourceIndex |
| SourceContainer | Interfaz `BPI_SlotHandler` | — | SourceHandler |
| TargetContainer | Interfaz `BPI_SlotHandler` | — | TargetHandler |

**Notas de uso:**
- `Requester`: quien pide la acción (jugador o PNJ). Pendiente decidir si el menú debe pasar el pawn del jugador en lugar del dueño del contenedor clicado (hoy `WBP_Slot` pasa `GetOwnerActor`).
- `ActionTag` vacío = el manager resuelve la acción (por defecto → MultiUse por contexto). El manager escribe la acción resuelta en el payload antes de entregarlo al sistema dueño y antes de avisar por `OnActionExecuted`.
- `ItemSlot`: identidad del ítem (`ItemID`, `InstanceID`) y copia del slot al momento de pedir. Toda petición trae un ítem: `RequestAction` rechaza `ItemID = None` (`Fail.InvalidItem`). No se valida `Quantity > 0` porque el inventario ya limpia siempre cualquier slot con cantidad ≤ 0. El manager la compara con el slot real (`SourceContainer.GetSlotData(SourceSlotIndex)`) como medida de seguridad (`Fail.StaleSlot`).
- `Quantity`: cuánto operar (valor del botón numérico o 1). Independiente de `ItemSlot.Quantity`, que es lo que había en el slot.
- `SourceSlotIndex = -1`: la acción no sale de un slot (ej. Craft de una receta); el manager omite las validaciones de slot y cantidad.
- `SourceContainer`: de dónde sale el ítem; también es el contenedor contra el que se validan capacidades y `ExcludeActions`.
- `TargetContainer`: quién ejecuta / hacia dónde va. Puede quedar vacío (el manager busca el ejecutor en el Requester). El menú siempre envía el contenedor externo; `RequestAction` lo deja en None cuando la fila resuelta tiene `CompareExternal? = false`.
- Campos descartados del payload viejo: `ContextTag` (se obtiene de `SourceContainer.GetContainerType()`) y `ActionTrigger` (nadie lo usaba; se reconsidera solo si una regla cambia según el origen).

### STR_ActionResult
> La respuesta de toda ejecución. La devuelve el sistema dueño desde `ExecuteAction` (éxito o fallo propio) o el propio manager (fallos previos a la ejecución, vía `MakeFailResult`).

| Campo | Tipo | Contenedor | Valor por defecto |
|----|---|---|---|
| Success? | Boolean | — | false |
| ResultTag | GameplayTag | — | vacío |
| QuantityProcessed | Integer | — | 0 |
| ReturnedItems | STR_Slot | **Array** | vacío |

**Notas de uso:**
- `Success? = false` por defecto a propósito: si un ejecutor olvida llenar el resultado, o el objeto no implementa la interfaz (Unreal devuelve la struct por defecto), la acción cuenta como fallida, no como exitosa.
- `ResultTag`: código del resultado, bajo `System.Action.Result.*` (análogo a códigos HTTP). Ver *Tags del sistema*.
- `QuantityProcessed`: lo realmente operado. En Transfer, el remanente se queda en el origen (solo se descuenta lo procesado), así que no se "devuelve".
- `ReturnedItems` es **Array** a propósito (se evaluó dejarlo como valor único y se descartó): casos con varios ítems sin lugar — desequipar una mochila que concede slots (`GrantedInventorySlots`), arma a dos manos que libera ambas manos, Craft con subproductos (`STR_Recipes.OutputResult` es Array), desarmar un objeto. Es el respaldo de "nada se pierde".
  - **Pendiente de diseño (05/11/2026):** el destino de los `ReturnedItems` no es solo "devolver al origen o soltar en el mundo". Ejemplo: el contenido de una mochila que se desequipa debe quedar en un **almacenamiento oculto anclado al `InstanceID`** del ítem contenedor, para volver a enlazarlo fácilmente cuando se equipe de nuevo, sin spawnearlo en el mundo ni pasarlo al inventario principal. Se diseña más adelante.

### STR_Context_Action
> Estructura puente, propia del pipeline de resolución de MultiUse. Empaqueta "una acción concreta + en qué contextos es válida" (una acción, múltiples contextos posibles), para que la función de resolución (`GetMultiUseActions`) no tenga que devolver dos arrays paralelos, y la función de filtro (`ValidateMultiUseActions`, en construcción) no tenga que volver a consultar `ItemInfo.MultiUseBehavior.ContextRelation` para saber en qué contexto es válida cada acción ya resuelta.

| Campo | Tipo | Contenedor |
|----|---|---|
| ActionRule | STR_ActionDefinition | — |
| Context | GameplayTag | GameplayTagContainer |

**Notas de uso:**
- `ActionRule` es la fila real de `DT_ActionDefinition` ya resuelta a partir de un tag de `GrantedActions` (vía `ActionRulesListMap` en `GI_Sunyi`).
- `Context` es el `RequiredContext` (`GameplayTagContainer`) de la `STR_ContextRelation` que concedió esa acción — se mantiene como container (no se colapsa a un solo tag) porque una misma acción puede ser válida en más de un contexto a la vez (ej. válida en inventario principal, bolsillo y cofre, pero no en hotbar). La validación contra el `ContextTag` del target es un `HasTag` simple; `MatchingContext` (`Any`/`All`) de `STR_ContextRelation` no se traslada a esta estructura ni se usa en este flujo.
- No es una estructura de contenido (no se llena a mano en ninguna DataTable): se construye dinámicamente en tiempo de ejecución, una vez por cada acción concedida por MultiUse que un ítem tenga en su contexto actual.

---

## Listas de filtrado del contenedor (blanca/negra)

> No son estructuras nuevas del Sistema de Interacción Contextual: son variables del propio contenedor (`InventoryComponent` o equivalente), expuestas a través de `BPI_SlotHandler`. Se documentan aquí porque el sistema de acciones las consume directamente para construir/filtrar el menú.

| Nombre | Tipo | Contenedor | Ámbito | Descripción |
|----|---|---|---|---|
| AllowedItemTags (existente) | GameplayTag | GameplayTagContainer | Física | Lista blanca de propiedades de ítem que este contenedor admite (ej. un estante de armas). Acotada al ámbito físico: se usa para validar Drag&Drop y como filtro de `Transfer` contra el `ExternalTarget`, nunca para permitir/excluir acciones directamente. Expuesta vía `GetAllowedItems()` (existente). |
| ExcludedItemTags (existente) | GameplayTag | GameplayTagContainer | Física | Lista negra de propiedades de ítem que este contenedor rechaza. Mismo ámbito físico que la anterior: gobierna qué puede *estar/entrar* en el contenedor (Drag&Drop y pre-filtro de `Transfer`), no qué acciones se pueden ejecutar sobre lo que ya está adentro. Expuesta vía `GetExcludedItems()` (existente). |
| ExcludeActions (implementada) | GameplayTag | GameplayTagContainer | Lógica | Acciones apagadas en este contenedor puntual, independientes del ítem (ej. "en este cofre no se puede hacer Drop", sin que el ítem tenga ninguna restricción propia). Nombre final adoptado (descartando el nombre de trabajo `RejectedActions`). Expuesta vía el nuevo getter `GetExcludeActions()` en `BPI_SlotHandler`, ya implementado y consumido por `ValidateAcceptedActions` (ver `4-2`). |

**Por qué se separó `ExcludeActions` de `ExcludedItemTags`:** antes de esta decisión, la lista negra se planteaba como una sola lista que podía contener tanto tags de propiedad de ítem como tags de acción, filtrando ambos en una sola pasada. Se descartó ese enfoque: mezclar ambos tipos de exclusión en una sola `GameplayTagContainer` conflacionaba dos funcionalidades objetivamente distintas — una "física" (qué entra al contenedor) y una "lógica" (qué acciones están disponibles) — obligando a la función de filtro a asumir de qué tipo era cada tag. Con la separación, el filtro de acciones (`ValidateAcceptedActions`, ver `4-2`) solo necesita comparar `ActionTag` de cada fila contra `ExcludeActions` (con coincidencia jerárquica de raíz, para que excluir `Action.Container.Equip` excluya también `Equip.Wield` y cualquier hijo), sin tener que revisar `RequiredItemProperty` — esa responsabilidad queda enteramente del lado de `ExcludedItemTags`/`AllowedItemTags`.

---

## Enums

### Enum_ActionExecMode
> Determina cómo se ejecuta una acción ya resuelta. Implementado en `Game/ContextActionSystem/Data/Enums/`.

- `Container` (Nativa_Contenedor)
- `Effect` (Nativa_Efecto)
- `Delegate` (Delegada)

> `ENUM_ContextMatching` (`MatchAny` / `MatchAll`) se reutiliza tal cual, sin cambios; no se redefine aquí.
>
> Desde el rediseño de la ejecución, `Enum_ActionExecMode` ya no decide cómo se enruta una acción (lo decide `ExecutorReference`). Se conserva; las metodologías Container/Effect/Delegate pasan a ser internas de cada sistema (ver `4-0`).

---

## Tags del sistema

> Los tags de sistema van bajo `System.*` (archivo `Config/Tags/Sunyi_SystemTags.ini`), separados de los tags de acción (`Action.*`, `Sunyi_ActionTags.ini`). Así los resultados no cuelgan del árbol `Action`, que se compara jerárquicamente en `ExcludeActions` y en el menú.

| Familia | Uso | Tags |
|---|---|---|
| `System.Handler.*` | Sistema dueño de una acción. Va en `ExecutorReference` de la fila y en la variable `SystemHandler` de cada componente ejecutor. | `System.Handler.Inventory`, `System.Handler.Equipment` (futuros: `Craft`, `Build`…) |
| `System.Action.Result.*` | `STR_ActionResult.ResultTag`. Fallos del manager bajo `Fail`; los propios de cada sistema bajo `Fail.<Sistema>.*` (ej. `Fail.Craft.MissingMaterials`, `Fail.Inventory.NoSpace`). | **Creados:** `Fail.InvalidItem`, `Fail.NoRule`, `Fail.NoHandler`, `Fail.StaleSlot`, `Fail.InvalidQuantity`, `Fail.NotAllowed`. Sugerido: `System.Action.Result.Success` para que `ResultTag` nunca quede vacío |
| `System.Warning.Data.*` | Avisos de datos mal configurados, emitidos por `OnDataWarning` (pendiente de crear y asignar). | `System.Warning.Data.MultiUse.TagWithoutRelations`, `System.Warning.Data.MultiUse.RelationsWithoutTag` |

---

## Estado de implementación

- Carpeta técnica `Game/ContextActionSystem/` creada con subcarpetas `Data/Structs`, `Data/Enums`, `Data/DataTables`, espejando la convención del proyecto.
- `STR_Requirements`, `STR_ActionDefinition`, `STR_ActionRequest` y `Enum_ActionExecMode` implementados y verificados contra este documento.
- `CompareExternal?` agregado a `STR_ActionDefinition` (activo solo en `ActionTransfer`).
- `DT_ActionDefinition` (DataTable sobre `STR_ActionDefinition`) creada. Filas de contenido: `ActionUse`, `ActionSplit`, `ActionTransfer`, `ActionDrop`, `ActionInspect`, `ActionEquip`, `ActionEquipWield` (`Action.Container.Equip.Wield`, label "Agarrar", concedida por MultiUse vía `ContextRelation.GrantedActions`; usada en las pruebas con rama y yelmo) (ver `4-1-1` para el mapeo de tags heredados del sistema viejo usados como base). Filas de sistema para la UI de MultiUse agregadas: `ActionMultiUseOpenMenu` (`Action.MultiUse.OpenMenu`) y `ActionMultiUseExecuteRandom` (`Action.MultiUse.ExecuteRandom`), ambas `ExecMode = Delegate`, sin requisitos propios (se resuelven por presencia/ausencia en el mapa de reglas, no por matching de tags) — detalle de su consumo en `GetMultiUseGroups` (`4-2`).
- `STR_ActionExec` y `STR_ActionResult` implementadas (05/11/2026) en `Data/Structs`. `ReturnedItems` corregido a Array. `STR_ActionRequest` no se modificó (equivalencia `Target` → `SourceContainer` documentada).
- `STR_Context_Action` implementada. Usada por `GetMultiUseActions` (hoy en `BP_ExecActionManager`, migrada desde `BPFL_ContextActions` — ver `4-2`), la función que resuelve `ItemInfo.MultiUseBehavior.ContextRelation` contra `ActionRulesListMap` y devuelve el array de acciones MultiUse candidatas ya emparejadas con su contexto válido — detalle completo de esta función y su lógica en `4-2 - Implementación de Sistema de Acciones.md`.

---

## Dependencias con otros sistemas

- **`STR_ItemInfo.PropertyTags`** (familia `Item.Property.Use.*`): fuente de verdad para `RequiredItemProperty` y para la resolución de la acción implícita. Forma congelada, no se modifica.
- **`STR_ItemInfo.ItemTypeTags`**: no interviene en este sistema; se mantiene reservado a filtros/categorización.
- **`STR_MultiUse.ContextRelation`** (`RequiredContext` + `MatchingContext` + `GrantedActions`): fuente de acciones candidatas adicionales por contexto. Una acción concedida por esta vía omite `RequiredItemProperty`, pero no `RequiredTargetCapability` ni `RequiredRequesterCapability`.
- **`STR_EquipmentBehavior.ActionBindings`** (`STR_IntentionActions`): usado únicamente por la ejecución delegada de Equip, fuera de esta tabla.
- **`STR_StatData`** (`EffectRules` / `ConsumeEffects`): leído internamente solo por el sistema dueño de las acciones de uso/consumo. El enlace acción → efecto (antes vía `ExecutorReference` = `Effect.*`) queda pendiente de reubicar, porque `ExecutorReference` ahora identifica al sistema ejecutor (`System.Handler.*`).
- **`STR_ContextualLabel`**: reutilizada tal cual para `ContextualLabels`.
- **`ENUM_ContextMatching`**: reutilizado tal cual para `MatchMode`.

---

## Puntos pendientes de definir

- ~~Llenar `ExecutorReference` en `DT_ActionDefinition`~~ → hecho: Use/Split/Transfer/Drop/Inspect → `System.Handler.Inventory`; Equip/EquipWield → `System.Handler.Equipment`; filas de sistema MultiUse → vacío.
- Fila `ActionUnequip`: tiene el `ActionTag` duplicado (`Action.Container.Equip`). Se resuelve cuando se trabaje el EquipmentSystem (fuera de esta fase).
- `ActionInspect`: diferida hasta el InstanceSystem / LearnSystem (ver `4-0`).
- Dónde vive la categoría de efecto (antes `Effect.*` en `ExecutorReference`): otro campo de la fila o lógica interna del sistema dueño de Use. Se decide al diseñar ese sistema.
- Rama de tags de capacidad del *actor* (`RequiredRequesterCapability`), hoy inexistente en el proyecto — a definir por acción a medida que se necesite (primer caso de prueba: Inspeccionar).
- Componente de Capacidades del Actor (mencionado en `4-0`): sigue siendo un elemento de arquitectura pendiente, necesario para que `RequiredRequesterCapability` tenga de dónde leer en tiempo de ejecución. No es un entregable de esta fase, pero bloquea la ejecución real de acciones que dependan de él.
- ~~Forma exacta del payload de ejecución~~ → resuelto: `STR_ActionExec` + `STR_ActionResult`. El menú ya ejecuta por `RequestAction`; el puente temporal al sistema viejo se quitó (sin respaldo). Ver `4-2`.

### Observaciones de revisión sobre `DT_ActionDefinition` (pendientes de decidir/corregir)

> Detectadas al revisar el export de `DT_ActionDefinition.json` (04/11/2026). No se ha cambiado nada; quedan anotadas para decidir.

- **`ExecMode` de Equip**: `ActionEquip` y `ActionEquipWield` tienen `ExecMode = Container`. Con el rediseño, `ExecMode` ya no enruta la ejecución, así que deja de ser bloqueante; lo que sí hace falta es su `ExecutorReference = System.Handler.Equipment`.
- **`ActionEquipWield.RequiredItemProperty = Item.Property.Use.Equippable`**: rompe dos reglas de este documento: (1) cada tag `Item.Property.Use.*` debe ser el `RequiredItemProperty` de una sola fila (hoy `Use.Equippable` está en `ActionEquip` y en `ActionEquipWield`; la implícita resuelve bien solo porque `ActionEquip` va antes en la tabla y `ValidateActiveAction` corta en el primer match), y (2) una acción concedida por MultiUse omite `RequiredItemProperty`. Propuesta: dejarlo vacío.
- **Requisito vacío ≠ "cualquiera" en el menú** (parcialmente resuelto): el manager ya lo aplica en `ValidateCapabalities` (requisito vacío = pasa), y el menú lo usa para Transfer (`BuildActionsMenu` llama a `ValidateCapabalities`). Pero `ValidateBasicActions` y `ValidateMultiUseActions` todavía usan `HasAnyTags`, que contra un `TagRequirement` vacío devuelve `false`. Por eso `ActionInspect` (con `RequiredTargetCapability` vacío) nunca aparece en el menú. Lo mismo ocurrirá con `RequiredRequesterCapability`, cuya nota dice "vacío = cualquier solicitante". Para que se cumpla, la validación necesita un chequeo explícito "si `TagRequirement` está vacío → pasa" antes del `SwitchEnum` de `MatchMode` (o una convención de datos distinta).
- **`ActionUse.ExecutorReference = None`**: bloqueado por la taxonomía `Effect.*` (ver punto pendiente arriba).
- **`ActionEquipWield` usa `Container.Capability.AllowUse`** como capacidad requerida: cambiará a `Container.Capability.AllowMultiUse` cuando se cree ese tag (ver `4-2`, Pendiente).
- Contrato de función que `BPI_SlotHandler` debe exponer para que la construcción del menú consulte capacidades de forma genérica (revisar qué funciones de la interfaz actual se mantienen, cuáles se extienden y cuáles se reemplazan; detalle en `4-1-1`).
