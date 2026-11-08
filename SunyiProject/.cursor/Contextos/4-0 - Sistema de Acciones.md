# Sistema de Interacción Contextual

> Nombre conceptual/de documentación: **Sistema de Interacción Contextual**. Nombre técnico en Unreal (carpeta y assets): **`ContextActionSystem`**. Cubre lo que antes eran tres sistemas separados: UseItem, MultiUse y Menú Contextual.

---

## Descripción del sistema

Este documento reemplaza, desde cero, el diseño anterior de UseItem + Menú Contextual + Equip (archivado en `.cursor/Contextos/Olds/`, documentos 4, 4-1, 4-2, 5 y 6). Esos documentos quedan solo como referencia histórica de lo que existía; no describen el sistema que se está construyendo.

El Sistema de Interacción Contextual es el punto único a través del cual **cualquier solicitante** (UI del jugador, hotbar, tecla rápida, IA de un PNJ, un trigger del mundo) pide **cualquier acción** sobre **cualquier objetivo** (un ítem en un slot, un contenedor, una estación de trabajo, un actor). El sistema decide qué acción corresponde, valida las reglas comunes y **entrega la ejecución al sistema dueño de esa acción**, sin que el solicitante ni el objetivo necesiten conocer los detalles del otro.

El alcance de esta fase es: **Menú Contextual dinámico + ejecución de acciones (payload, manager central y contrato de ejecución) + acciones de contenedor (Inventory: Split, Drop, Transfer, Use)**. Sistemas futuros (Crafting, Building, Quest, Destinos, Learning) quedan fuera de alcance; cuando existan se incorporarán como nuevos sistemas dueños de sus propias acciones, sin requerir cambios en el manager.

> **Fuera de esta fase (decisión 07/11/2026):**
> - **Equip / Unequip**: pertenecen al **EquipmentSystem**, que es un sistema separado del sistema de acciones. Se conectarán cuando se trabaje ese sistema.
> - **Inspect**: es parte del futuro **LearnSystem** (el sistema que da "conocimiento" a las entidades: recetas, lore, destinos, información de ítems). Existía como utilidad para ver los datos de un ítem, pero quedó como resto de esa idea. No se implementa hasta tener operativo el **InstanceSystem**.

> **Rediseño de la ejecución (05/11/2026).** La ejecución se rediseñó desde cero: se descartó el payload viejo (`STR_UseItemPayload`) y el modelo de "tres formas de ejecución" (Container / Effect / Delegate) como decisión del controlador. El modelo vigente es el **enrutado descentralizado** descrito abajo. Lo que antes eran las "tres formas" pasa a ser metodología interna de cada sistema (ver *Metodologías de efecto*).

---

## Principios de diseño

- **Pequeño, centralizado e indistinto del solicitante**: jugador y PNJ usan exactamente el mismo flujo. Todos los sistemas del proyecto siguen esta misma premisa.
- **Manager central que enruta, no ejecuta**: una sola pieza (`BP_ExecActionManager`) recibe la petición, resuelve qué acción es, aplica las validaciones comunes, encuentra al sistema dueño de esa acción, le entrega la petición y recibe el resultado. No sabe *cómo* se hace un Craft, un Split o un Equip.
- **Cada sistema es dueño de sus acciones**: `InventoryComponent` es **el** sistema de administración de ítems y sabe hacer Split/Drop/Transfer/Use (los demás sistemas usan inventarios a través de `BPI_SlotHandler`, no tienen su propio Split/Transfer/Drop); `EquipmentComponent` sabe hacer Equip/Unequip; a futuro `CraftComponent` hará Craft, `BuildComponent` hará Build, etc. Cada uno implementa su lógica, sus validaciones específicas y su propia transacción. Agregar un sistema nuevo nunca toca el manager: solo se agregan filas de datos y un componente que implementa el contrato.
- **Un solo contrato de ejecución**: todo sistema que ejecuta acciones implementa la interfaz `BPI_ExecActionHandler` (`ExecuteAction` + `GetHandlerTag`). El manager nunca distingue tipos de sistema: solo llama la interfaz y recibe un `STR_ActionResult`.
- **Enrutado por datos**: cada fila de `DT_ActionDefinition` declara en `ExecutorReference` qué sistema la ejecuta (`System.Handler.*`). Cada componente declara su sistema en su variable `SystemHandler`. El manager une ambos sin Cast (`GetComponentsByInterface` + `GetHandlerTag`).
- **Una acción = un sistema dueño**: si dos comportamientos distintos comparten verbo (ej. crafting simple en inventario vs crafting en banco), o comparten el mismo sistema con distinto componente (lo resuelve el orden de búsqueda), o son dos acciones distintas con dos tags y dos filas. Nunca una fila apunta a dos sistemas.
- **Acciones pedidas en un sistema y ejecutadas en otro**: ej. Equip se pide desde inventario/hotbar/tecla rápida y lo ejecuta `EquipmentComponent`; Unequip se pide y ejecuta en Equipment y afecta al inventario. El orden de búsqueda del manager lo resuelve sin que el solicitante sepa quién ejecuta.
- **Validaciones genéricas en el manager, específicas en cada sistema**:
  - *Genéricas* (definidas por datos, iguales para cualquier acción): `RequiredTargetCapability` contra las capacidades del contenedor, `RequiredRequesterCapability` (pendiente), `ExcludeActions` del contenedor, slot sin cambios, cantidad válida. Viven en el manager porque las peticiones que no pasan por el menú (hotbar, tecla rápida, PNJ) no tienen otra validación.
  - **En qué contenedor se revisan las capacidades** (07/11/2026): si la fila tiene `CompareExternal? = false` (Split, Drop, Use…), en el **origen** (el contenedor donde se hizo clic). Si tiene `CompareExternal? = true` (Transfer), en el **destino**: para Transfer, `AllowMove` significa "este contenedor acepta que le metan ítems". `ExcludeActions` siempre se revisa en el origen ("esta acción está apagada si haces clic aquí").
  - Ejemplo de diseño: un cofre de misión **sin** `AllowMove` deja sacar ítems pero no deja meterlos. Si el jugador saca un ítem que no es el de la misión, ya no puede devolverlo, y el cofre registra que fue tomado.
- **Drag & Drop (Swap) es interacción de UI, no una acción**: intercambiar o apilar slots arrastrando no pasa por `RequestAction` ni por `DT_ActionDefinition`. Pero cuando el arrastre va **entre contenedores distintos**, se valida con las **mismas reglas que Transfer** (reutilizando `ValidateExec` del manager), en los dos sentidos si hay intercambio: lo que entra al destino y lo que vuelve al origen. Dentro del mismo contenedor el Swap es libre.
- **Zona especial de soltar (DropZone)**: contenedor temporal para soltar varios ítems al mundo de una vez. Está **exenta de las reglas de Drop a propósito**: cualquier ítem, incluso de misión, puede soltarse por ahí. Solo declara `AllowMove` para que se pueda meter y sacar ítems arrastrando. Los demás contenedores siguen sus propias reglas.
  - *Específicas* (solo tienen sentido para esa acción): tope de Split (`Quantity - 1`), espacio en destino de Transfer, `CanEquipItem`, receta/materiales de Craft, "buildable" de Build. Viven dentro del `ExecuteAction` del sistema dueño y devuelven su propio `ResultTag`.
- **Ejecutar ≠ avisar**:
  - *Ejecutar* es una llamada directa de interfaz a **un** componente (`ExecuteAction`), que devuelve resultado. Los sistemas ejecutores no se suscriben a nada.
  - *Avisar* es un dispatcher (`OnActionExecuted`) que se dispara **siempre** (éxito o fallo) para trazabilidad completa. Se suscriben solo los sistemas que reaccionan a acciones de otros (Quest, Destinos, UI, logs).
- **Resolución de la acción** (cuando la petición no trae acción explícita):
  1. **Explícita** (menú, PNJ): se usa tal cual. Los PNJ siempre envían la acción explícita (la decide su IA).
  2. **Acción por defecto del ítem** (`Item.Property.Use.*` vía `RequiredItemProperty`): para peticiones fuera del menú (doble clic, hotbar, tecla rápida).
  3. **MultiUse por contexto**: solo si el ítem **no** tiene acción por defecto (ej. una roca: recurso sin acción default, con "Arrojar" y "Equipar" por MultiUse; en hotbar se arroja). Las reglas de contexto de MultiUse se definirán al configurar MultiUse.
- **Acción ≠ Efecto**: la acción gobierna el contexto de la interacción (¿consume el ítem?, ¿qué capacidad requiere?). El efecto es el resultado (curar, buff, daño) y lo ejecuta el sistema dueño de la acción con su propia maquinaria. Un mismo ítem puede tener efectos distintos ligados a acciones distintas (ej. poción: `Consumir` restaura maná, `Lanzar` hace daño).
- **Nada se pierde**: el remanente de una operación se queda en el origen (solo se descuenta lo procesado). Para ítems que salen de su lugar sin destino asegurado (ej. el `ReplacedItem` de Equip), el sistema los devuelve en `ReturnedItems` y el manager aplica una regla única de respaldo (devolver al origen o soltar en el mundo).
- **Data-driven real**: agregar una acción es agregar una fila de dato (y, si pertenece a un sistema nuevo, ese componente). Nunca se modifica la forma de `STR_ItemInfo` ni de sus sub-estructuras; su contenido (tags, valores) sí puede ampliarse.
- **Desacoplado**: el manager no conoce Equipment, Crafting, Quest, etc. Solo conoce tags de acción, tags de sistema y el contrato de interfaz.

---

## Metodologías de efecto (internas del sistema dueño de "Use")

> Antes eran las "tres formas de ejecución" decididas por el controlador. Ahora son la metodología interna con la que el sistema dueño de las acciones de uso/consumo ejecuta un efecto. El manager no participa. Se documentan aquí como referencia hasta que se diseñe ese sistema.

- **Contenedor**: acciones síncronas iguales para cualquier ítem (Split, Drop, Transfer). Las resuelve el propio contenedor. (Swap no es una acción: es interacción de UI, ver *Principios*.)
- **Efecto — Vía A** (comportamiento propio: animación, spawn en el mundo): actor pequeño y genérico, uno por tipo de efecto, que se crea, hace su trabajo y se autodestruye. Usa el patrón congelado de `STR_ConsumeEffect` (`EffectActor` + `DataID`/`DataTag`/`DataInt`/`DataFloat`).
- **Efecto — Vía B** (efecto numérico instantáneo, temporal o permanente): sin actor; entrada ligera en un ledger tickeado de forma centralizada. Usa `STR_StatEffect.ApplicationMethod` (`Appl_ConsumeInstant`, `Appl_ConsumeTimed`, `Appl_ConsumePermanent`, `Appl_EquipPassive`).
- **Delegada**: la entrada a otro sistema con lógica y estado propios. En el modelo actual, *toda* acción es delegada a su sistema dueño.

---

## Estructuras

> Las estructuras reales viven en `4-1 - Estructuras de Datos.md`. Resumen de las que sostienen el flujo:

- **`STR_ActionDefinition`** (`DT_ActionDefinition`): una fila = una acción (requisitos, labels, `ExecutorReference` = sistema que la ejecuta).
- **`STR_ActionRequest`**: datos con los que la UI construye el menú (Requester, contenedor clicado, slot).
- **`STR_ActionExec`**: el payload genérico de ejecución (Requester, ActionTag, ItemSlot, Quantity, SourceSlotIndex, SourceContainer, TargetContainer). 100% genérico: los datos específicos (receta, posición, stats) los resuelve cada sistema dentro de su componente.
- **`STR_ActionResult`**: la respuesta (Success?, ResultTag, QuantityProcessed, ReturnedItems).

---

## Flujo de una acción

```
Menú / Hotbar / Tecla rápida / PNJ
   └─ RequestAction(STR_ActionExec) ──► BP_ExecActionManager
                                          ├─ ResolveAction    → fila de DT_ActionDefinition   (falla: Fail.NoRule)
                                          ├─ ValidateExec     → reglas genéricas              (falla: StaleSlot / InvalidQuantity / NotAllowed)
                                          ├─ FindHandler      → componente dueño              (falla: Fail.NoHandler)
                                          ├─ ExecuteAction(ActionExec) ──► componente dueño (llamada directa de interfaz)
                                          │                            ◄── STR_ActionResult
                                          ├─ OnActionExecuted ──► Quest, Destinos, UI…   (aviso, éxito o fallo)
                                          └─ Return STR_ActionResult ──► solicitante
```

**Orden de búsqueda de `FindHandler`** (gana el primer componente cuyo `GetHandlerTag` coincide con el `ExecutorReference` de la fila):
1. `TargetContainer` · 2. `SourceContainer` · 3. demás componentes del actor dueño del Target · 4. componentes del Requester.

Ejemplos: Craft con forja abierta → `CraftComponent` de la forja; Craft sin forja → `CraftComponent` del propio Requester (crafting simple); Split en un cofre → el inventario del cofre; Equip desde inventario → `EquipmentComponent` del Requester. El menú siempre envía el contenedor externo como `TargetContainer`, pero **el manager lo limpia** (lo deja en None) cuando la fila resuelta tiene `CompareExternal? = false`. Así acciones como Split, Drop o Equip no se desvían hacia un contenedor externo abierto (un cofre, un maniquí): como `FindHandler` mira primero el Target, el cofre ejecutaría la acción sobre su propio slot. El filtro vive en el manager porque solo él conoce la acción resuelta (una petición puede llegar sin `ActionTag`) y porque así aplica también a PNJ, hotbar y teclas rápidas.

---

## Puntos pendientes de definir

- Reglas de contexto de MultiUse para la resolución fuera del menú (incluido qué gana si varias relaciones aplican al mismo contexto).
- Sistema dueño de las acciones de uso/consumo y dónde vive la categoría de efecto (antes pensada en `ExecutorReference`, hoy ocupado por el sistema ejecutor).
- Componente de Capacidades del Actor (para `RequiredRequesterCapability`).
- Acciones que toman tiempo (animaciones, crafting, construcción): si responden "aceptada" y avisan el final después vía `OnActionExecuted`.
- Avisos de acciones hechas por arrastre: el Drag & Drop valida con el manager pero ejecuta con `SwapSlots`/`DropOnStack`, así que **no dispara `OnActionExecuted`**. Un sistema que necesite saber qué salió de un contenedor (ej. el cofre de misión) debe escuchar el `OnInventoryUpdated` de ese inventario, que cubre todos los caminos (menú, arrastre, zona especial).
- Fuente genérica del "contenedor externo" para PNJ (posible: el objetivo actual del `InteractComponent`), en lugar de `SetHandlers`, que depende del HUD del jugador.

---

## Dependencias con otros sistemas

- **InventoryComponent**: implementa `BPI_ExecActionHandler` (`SystemHandler = System.Handler.Inventory`). Split, Drop y Transfer ya funcionan dentro de `ExecuteAction`; Use está pendiente. Sigue exponiendo `BPI_SlotHandler` para guardar/mover ítems (lo usan el Drag & Drop y los demás sistemas).
- **EquipmentComponent** (fuera de esta fase): cuando se trabaje el EquipmentSystem, implementará `BPI_ExecActionHandler` (`System.Handler.Equipment`) y usará sus funciones data-only (`EquipItem`, `UnequipItem`, `CanEquipItem`) dentro de su `ExecuteAction`.
- **GI_Sunyi**: crea `BP_ExecActionManager` y le entrega los datos que necesita (patrón de singletons del proyecto). Acceso desde cualquier Blueprint vía `BPI_GIAccess.GetExecActionManager`.
- **STR_ItemInfo y sub-estructuras** (`STR_MultiUse`, `STR_EquipmentBehavior`, `STR_StatData`, `STR_ConsumeEffect`, `STR_StatEffect`): fuente de verdad para resolución y efectos. Forma congelada.
- **Sistemas futuros** (Crafting, Building, Quest, Destinos, Learning): se incorporan como nuevos sistemas dueños de acciones o como suscriptores de `OnActionExecuted`, sin cambios en el manager.

---

## Aislamiento del sistema (carpeta técnica y assets deprecados)

- **Carpeta dedicada**: todos los assets nuevos viven en `Game/ContextActionSystem/`, carpeta hermana de `InventorySystem`, `EquipmentSystem`, etc. Se replica en `Content/` y en `.cursor/ContextFunctions/`.
- **Subestructura interna**:
  - `Blueprints/` — `BP_ExecActionManager` (Object). (`BPFL_ContextActions` ya se borró.)
  - `Blueprints/Interfaces` — `BPI_ExecActionHandler`.
  - `Data/Structs` — `STR_ActionDefinition`, `STR_ActionRequest`, `STR_Requirements`, `STR_Context_Action`, `STR_ActionExec`, `STR_ActionResult`.
  - `Data/DataTables` — `DT_ActionDefinition`.
  - `Data/Enums` — `Enum_ActionExecMode` (ya no decide el enrutado; ver `4-1`).
- **Importante**: `Game/InteractionSystem/` (`BPI_Interaction`, `InteractComponent`) es un sistema distinto (interacción de mundo "presiona E"). No mezclar con `ContextActionSystem`.

### Assets del sistema viejo implicados (a deprecar/borrar en la fase de limpieza)

Orden de trabajo acordado: **1. Payload y manager → 2. Acciones de contenedor (Split, Transfer, Drop) → 3. Limpieza de todo lo DEPRECATED → 4. Use.** Los pasos 1 y 2 están terminados. Hasta el paso 3 estos assets siguen existiendo:

> Lista revisada contra el export completo del 08/11/2026.

- `BP_UseItemManager` (reemplazado por `BP_ExecActionManager`).
- `GI_Sunyi`: el dispatcher `OnUseRequested` y su bind en el EventGraph, la construcción de `BP_UseItemManager`, `BuildActionRulesCache` (todavía aparece en el EventGraph), `GetActionRules`, `GetUseItem`, `ActionRules`/`ActionRulesMap` viejos.
- `BPI_GIAccess`: `GetUseItem`.
- `InventoryComponent`: los eventos `UseItem` (de `BPI_UseItemHandler`) y `ActionUseItem`, `ExecActions`, `GetExecutableAction`, `UseConsumable`, `ConsumeItem`, `ActionDisassemble`. Lo único implementado de Use en el sistema viejo es consumir 1 ítem de un stack consumible; se rehace al diseñar Use completo.
- `BPI_SlotHandler`: las funciones de acción concretas salen de la interfaz.
  - **Al quitar una función de una interfaz, Unreal no borra su grafo**: si solo tiene entradas, se vuelve un evento; si tiene salidas, se vuelve una función propia del Blueprint.
  - `ActionSplitItem` **se queda** como función interna de `InventoryComponent` (la usa `ExecSplit`).
  - `ActionDisassemble` es lógica de crafting y **se borra** (no se va a usar).
  - `ActionUseItem`: se borra (solo la usa el sistema viejo).
  - `ActionTransferItem`: hoy solo la usa `WBP_Inventory.RestoreInventories`. **Primero** se cambia esa llamada (ver *Zona especial* abajo) y después se borra.
- `BPFL_Utilities`: `GetValidActions`, `ValidateTargetActions`.
- `WBP_ContextMenu`: las variables sin uso `SlotTags`, `GI Sunyi Ref`, `SlotData` (y dejar de pasarlas desde `WBP_Slot`). `SelectClickedAction` ya usa `RequestAction`.
- `WBP_Slot` (EventGraph): el clic del `ActionButton` que llama a `GetGI_Sunyi` → **`OnUseRequested`**, y el nodo huérfano `BPFL_Utilities.GetValidActions`. El clic se reconecta a `RequestAction` con `ActionTag` vacío (acción por defecto del ítem). **El Drag & Drop no es deprecado**: usa `BPI_SlotHandler` (`CanAcceptItem`, `SwapSlots`, `DropOnStack`) y `ValidateDragTransfer`.
- `STR_UseItemPayload`, `BPI_UseItemHandler`, `STR_ActionRules`, `STR_ActionNames`, `DT_ActionList`, `DT_ActionNames`, `DT_ActionPropertyRule`.
- Antes de borrar cada asset, confirmar con **Reference Viewer** que nadie más lo usa. Orden sugerido: primero quien llama, después lo llamado: cambio de `RestoreInventories` → `WBP_Slot` → `GI_Sunyi` / `BPI_GIAccess` → `InventoryComponent` → `BPI_SlotHandler` → `BP_UseItemManager` / `BPI_UseItemHandler` → structs y DataTables → `BPFL_Utilities` → variables de `WBP_ContextMenu`.

### Lo que NO es del sistema viejo (no se borra)

- **Zona especial de soltar (`WBP_Inventory`)**: `PendingUse`, `Save Transaction`, `RestoreInventories`, `ClearCacheZone`, `SpawnLoot`, `SpecialZoneReport`, `SpecialZoneVisibility`, `CreateHost` y la struct `STR_PendingToUse`. Tienen "Use" en el nombre por historia, pero son **el registro de transacciones de la zona**, no del sistema viejo de uso. Ver *Zona especial* abajo.
- **Drag & Drop de `WBP_Slot`** (`OnDrop`, `ValidateDragTransfer`).
- **`ActionSplitItem`** (pasa a función interna).

### Zona especial: cómo funciona y cambio pendiente

| Momento | Qué pasa |
|---|---|
| Abrir el inventario (`Construct`) | `CreateHost` crea `BP_UI_Host` (`Container.Type.DropZone`) con un inventario temporal y le asigna `AllowMove`. |
| Meter un ítem en la zona | `WBP_Slot.OnDrop` → evento `SpecialZoneReport` → **`Save Transaction`** agrega un registro a **`PendingUse`** (`STR_PendingToUse`: `SourceHandle`, `SourceSlotIndex`, `ItemData`, `TargetSlotIndex`, `TargetHandler`). |
| Sacar un ítem de la zona | `ClearCacheZone` borra ese registro. |
| Botón de tirar | **`SpawnLoot`** suelta al mundo todo lo de la zona, la vacía y limpia `PendingUse`. |
| Cerrar el inventario sin tirar (`Destruct`) | **`RestoreInventories`** devuelve cada ítem a su inventario de origen y destruye el host. |

**Cambio pendiente en `RestoreInventories`** (primer paso de la limpieza). Hoy, por cada registro: zona → `RemoveAnItem` y después origen → `ActionTransferItem(ItemData)`. Dos problemas:
1. `ActionTransferItem` sale de `BPI_SlotHandler` en la limpieza, así que esta llamada se rompería.
2. `ActionTransferItem` solo llama a `PickUpItem` e **ignora lo que no cupo**. Como el ítem ya se quitó de la zona, si el origen no tiene espacio (ej. el jugador llenó el hueco con otro ítem antes de cerrar), ese ítem **se pierde**.

Corrección propuesta:
- Reemplazar `ActionTransferItem` por **`PickupInActions`(`ItemData`, `NeedStack? = true`)**, que ya está en `BPI_SlotHandler` y devuelve `QuantityRemaining`.
- Invertir el orden: **primero devolver al origen, después quitar de la zona solo lo que entró** (cantidad − `QuantityRemaining`), igual que `ExecTransfer`.
- Lo que no quepa en el origen se suelta al mundo, porque la zona se destruye al cerrar ("nada se pierde").
