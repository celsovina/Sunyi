# Sistema de Interacción Contextual — Implementación de Sistema de Acciones

> Nombre técnico en Unreal (carpeta y assets): **`ContextActionSystem`**. Documento hermano de `4-1 - Estructuras de Datos.md`: mientras `4-1` documenta **qué datos existen**, este documento documenta **qué funciones/Blueprints existen y cómo trabajan**.

---

## Descripción del sistema

Este es el documento permanente de comportamiento del Sistema de Interacción Contextual: cada función/Blueprint nuevo, sus variables, su lógica y las decisiones de diseño que la sostienen. A diferencia de los documentos `4-1-N` (mapa de reemplazo del sistema viejo, temporales durante la migración), este documento **no se archiva ni se reemplaza al terminar la migración** — es la referencia viva de cómo funciona el sistema, y se edita directamente cada vez que se agrega o cambia una función.

No repite el detalle de estructuras/enums/DataTables, que vive en `4-1`. Solo referencia esos nombres cuando hace falta para entender una función.

---

## Principios de diseño (a nivel de implementación)

- **Bucket-based collection**: la construcción de la lista final de acciones acumula resultados en variables/arrays separados por rol (implícita, básicas, multiuso, requester) dentro de la misma función orquestadora, y se concatenan en un orden fijo al final. Esto desacopla el orden de validación (pensado por eficiencia) del orden de presentación en el menú (pensado por UX).
- **Despacho de `MatchMode` vía `SwitchEnum`**, no `Select`: al ser un nodo de ejecución, `SwitchEnum` solo evalúa la rama elegida (`HasAnyTags` o `HasAllTags`), evitando llamar a la comparación no usada en cada fila — un `Select` (nodo puro) evaluaría ambas siempre.
- **Corte temprano con `ForEachLoopWithBreak`** en búsquedas de "el primer que matchea" (ej. acción implícita), en vez de recorrer el array completo.
- **Separación de responsabilidades entre funciones aunque compartan el mismo array de entrada**: cuando dos validaciones son conceptualmente independientes (ej. "¿tiene acción implícita?" vs "¿qué acciones básicas admite el contenedor?"), se mantienen como funciones separadas aunque ambas iteren el mismo `ActionList`. El costo de una segunda iteración sobre una tabla pequeña es despreciable frente al beneficio de mantener responsabilidades claras y funciones reutilizables por separado.
- **El gate de capacidad de contenedor (`AllowUse`, futuro `AllowMultiUse`) se resuelve en la función orquestadora**, nunca dentro de las funciones que resuelven "qué quiere hacer el ítem" (implícita) o "qué reglas tiene la tabla" (básicas/multiuso). Mantiene la pregunta "¿qué puede hacer el ítem?" separada de "¿qué permite el contenedor aquí?".
- **`RequiredItemProperty` nunca debe incluir `Item.Property.HasMultiUse`**, ni siquiera en `MatchAll` junto a otra propiedad. `RequiredItemProperty` describe la acción NATURAL de un ítem (una propiedad → una acción concreta: `Use.Consumable`→Consumir, `Use.Equippable`→Equipar, etc.). `HasMultiUse` no describe ninguna acción específica, es la bandera de entrada al pipeline de MultiUse (`GetMultiUseActions`/`ValidateMultiUseActions`), que ya la evalúa nativamente como guarda. Si una acción debe depender de que el ítem sea MultiUse, el mecanismo correcto es agregarla al `ContextRelation.GrantedActions` del ítem, no condicionar una fila de `DT_ActionDefinition` con esa propiedad.
- **Migración de namespace (dos etapas).** Primero `BPFL_Utilities` → `BPFL_ContextActions`; después (05/11/2026) **`BPFL_ContextActions` → `BP_ExecActionManager`**. Las siete funciones (`ValidateActionMenu`, `ValidateBasicActions`, `ValidateActiveAction`, `GetMultiUseActions`, `ValidateMultiUseActions`, `GetMultiUseGroups`, `ValidateAcceptedActions`) viven ahora en el manager, con sus mismos nombres, entradas y salidas. Motivo: las validaciones de capacidad del contenedor son reglas comunes a cualquier acción y tienen que estar donde valida el manager, para que las peticiones que no usan UI (hotbar, tecla rápida, PNJ) no queden sin validar; y una sola copia de la lógica evita que el menú y el manager diverjan. `BPFL_ContextActions` se borra completa tras pasar la prueba en PIE. `BPFL_Utilities.SetHandlers` sigue siendo usada por `BuildActionsMenu` (resolver origen/destino es tarea de la UI). De aquí en adelante, toda referencia a estas funciones asume `BP_ExecActionManager`.
- **Un Object no puede llamar a un Blueprint Function Library ni a `PrintString`.** Ambos necesitan World Context, que un Object no tiene. Por eso: las funciones se migraron (no se pueden llamar desde la BPFL), los `PrintString` de `ValidateMultiUseActions` se reemplazaron por el dispatcher `OnDataWarning`, y el manager no usa `GetGameInstance`: `GI_Sunyi` le entrega lo que necesita al construirlo.
- **Parámetros compartidos se pasan hacia abajo, no se vuelven a resolver.** `ActionRules` (el mapa de reglas, hoy la variable `ActionRulesMap` del manager) se lee una sola vez en `ValidateActionMenu` y se pasa como parámetro explícito a cada función que lo necesita (`GetMultiUseActions`, `ValidateMultiUseActions`, `GetMultiUseGroups`), en vez de que cada una vuelva a pedirlo. Esto es distinto del caso de `ExcludeActions` en `ValidateAcceptedActions`: como esa lista se obtiene con una única llamada de interfaz directa sobre el mismo `ClickedHandler` que la función ya recibe (no hay cache compartido que valga la pena evitar re-pedir), la función la resuelve internamente en vez de recibirla como parámetro adicional.

---

## Variables

### `GI_Sunyi`
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ActionRulesList | STR_ActionDefinition | Array | Cache plano de todas las filas de `DT_ActionDefinition`. Reemplaza al viejo `ActionRules` (deprecado, no borrado). Se entrega al manager al construirlo. |
| ActionRulesListMap | STR_ActionDefinition | Map\<Name, STR_ActionDefinition\> | Mismo cache, indexado por el nombre del `ActionTag`. Reemplaza al viejo `ActionRulesMap` (deprecado, no borrado). Se entrega al manager como `ActionRulesMap`. |
| ExecActionClass | Class de BP_ExecActionManager | — | Clase a construir (mismo patrón que `ItemManagerClass` / `UseItemClass`). |
| ExecActionRef | BP_ExecActionManager | — | Instancia única del manager. La devuelve `GetExecActionManager` (`BPI_GIAccess`). |

### `BP_ExecActionManager` (variables de miembro y dispatchers)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ItemManagerRef | BP_ItemManager | — | Instance Editable + Expose on Spawn. Lo entrega `GI_Sunyi`. Usado por `ResolveAction` (`GetItemInfo`). |
| ActionRulesList | STR_ActionDefinition | Array | Instance Editable + Expose on Spawn. Copia de solo lectura de las reglas, entregada por `GI_Sunyi`. |
| ActionRulesMap | STR_ActionDefinition | Map\<Name, STR_ActionDefinition\> | Instance Editable + Expose on Spawn. Ídem, indexado por nombre del `ActionTag`. Cumple la regla del proyecto: Map solo de lectura. |
| OnActionExecuted | Event Dispatcher (`ActionExec`: STR_ActionExec, `ActionResult`: STR_ActionResult) | — | Se dispara al final de **toda** petición, con éxito o fallo, para trazabilidad completa. Lo escuchan los sistemas que reaccionan a acciones de otros (Quest, Destinos, UI, logs), no los que ejecutan. |
| OnDataWarning | Event Dispatcher (`ItemID`: Name, `Alert`: GameplayTag) | — | Aviso de datos mal configurados (hoy: inconsistencias de MultiUse). Reemplaza los `PrintString` que no se pueden usar en un Object. Lo escucha `GI_Sunyi`. |

### `BP_ExecActionManager.ValidateActionMenu` (locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| (reglas) | — | — | Ya no hay copia local: usa directamente las variables `ActionRulesList` y `ActionRulesMap` del manager (antes `LocalActionList`/`LocalActionRules` obtenidas de `GI_Sunyi.ExposeActionRules`). |
| LocalCapabalities | GameplayTag | GameplayTagContainer | Capacidades del contenedor "clicado" (`ClickedHandler.GetCapabalities`). |
| LocalContextTag | GameplayTag | — | `ClickedHandler.GetContainerType()`. Obtenido y guardado, todavía sin uso dentro de las validaciones actuales. |
| LocalRejectedItems | GameplayTag | GameplayTagContainer | `ClickedHandler.GetExcludedItems()`. Obtenido, pendiente de uso en el filtro de listas negras. |
| BasicActions | STR_ActionDefinition | Array | Salida de `ValidateBasicActions`. |
| ActiveAction / ValidActiveAction? | STR_ActionDefinition / bool | — | Salida de `ValidateActiveAction`: la acción implícita y si existe alguna. |
| MultiUseActions | STR_ActionDefinition | Array | Salida de `ValidateMultiUseActions`, solo asignada si `Success = true`. |
| MultiUseActionsRaw | STR_ActionDefinition | Array | Copia de `MultiUseActions` guardada antes de intentar el colapso en `GetMultiUseGroups`. Es el bucket "sin agrupar", usado como fallback directo cuando `Grouped? = false`, y pensado a futuro como fuente para construir el submenú de "Abrir menú". |
| GroupedMultiUse | STR_ActionDefinition | Array | Salida `GroupedMultiUse` de `GetMultiUseGroups`, solo se usa cuando `Grouped? = true`. |
| NeedsAllowUse? | bool | — | Resultado del gate `AllowUse` aplicado sobre `ActiveAction.RequiredTargetCapability` (vía `SwitchEnum` de `MatchMode` → `HasAnyTags`/`HasAllTags` contra `LocalCapabalities`), combinado luego con `ValidActiveAction?` (`BooleanAND`) para decidir si la implícita entra al ensamblaje final. |
| ActionListRaw / LocalActionListRaw | STR_ActionDefinition | Array | Acumulador final de la función. Se limpia y se llena en el orden implícita → multiuso (agrupado o crudo) → básicas, y es el valor que retorna la función (vía `Return Node`). |

### `BP_ExecActionManager.GetMultiUseActions` (locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| FoundActions | STR_Context_Action | Array | Acumulador de salida. Se llena dentro del loop y se retorna como `MultiUseActions`. |
| Context | GameplayTagContainer | — | Variable puente: guarda el `RequiredContext` de la `STR_ContextRelation` actual antes de entrar al loop anidado (sobre los tags de `GrantedActions`), para poder usarlo al construir el `STR_Context_Action` sin tener que romper la struct de nuevo. |

### `BP_ExecActionManager.ValidateMultiUseActions` (locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| HastMultiUseTag | bool | — | `ItemInfo.PropertyTags.HasTag(Item.Property.HasMultiUse)`. (Nombre con typo heredado del desarrollo, "Hast" en vez de "Has"; pendiente de limpieza cosmética). |
| HasRelations | bool | — | `Array_IsNotEmpty(ItemInfo.MultiUseBehavior.ContextRelation)`. Indica si el ítem tiene datos de contexto configurados. |
| ActionToCheck | STR_ActionDefinition | — | Variable puente por iteración del loop principal: guarda `ActionRule` del `STR_Context_Action` actual para no repetir el `Break` en los 4 puntos donde se lee (Context, RequiredTargetCapability, MatchMode, valor final a agregar). |
| LocalMultiUseActions | STR_ActionDefinition | Array | Acumulador de salida. Se retorna como `MultiUseActions`. |

### `BP_ExecActionManager.GetMultiUseGroups` (locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ActionRules | STR_ActionDefinition | Map\<Name, STR_ActionDefinition\> | Parámetro de entrada (no variable puramente local, pero listada aquí porque también se consulta vía `Get` desde varios puntos del grafo). Alimentada por `LocalActionRules` de `ValidateActionMenu`, evita que la función tenga que resolver el cache por su cuenta. |

### `BP_ExecActionManager.ValidateAcceptedActions` (locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ExcludeList | GameplayTag | GameplayTagContainer | `ClickedHandler.GetExcludeActions()`. A diferencia de `ActionRules`, se resuelve dentro de la propia función (ver nota de diseño sobre parámetros compartidos) porque es una sola llamada de interfaz directa sobre el mismo handler ya recibido. |
| ActionTag | GameplayTag | — | Tag de la fila actual del loop externo, extraído una vez por iteración para no repetir el `Break` de `ActionRule`. |
| ActionRule | STR_ActionDefinition | — | Fila actual del loop externo (sobre `ActionListRaw`), guardada para poder agregarla a `CleanList` al final si sobrevive el filtro. |
| IsExcluded? | bool | — | Flag por iteración del loop externo: se resetea a `false` al empezar cada elemento, y se marca `true` si el loop interno (sobre los tags de `ExcludeList`) encuentra una coincidencia jerárquica. |
| CleanList | STR_ActionDefinition | Array | Acumulador de salida. Se retorna como `ActionList`. |

### `WBP_ContextMenu` (variables de miembro)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ClickedHandler | BPI_SlotHandler | — | Contenedor donde se hizo clic. Lo asigna `BuildActionsMenu` desde la salida `SourceHandler` de `BPFL_Utilities.SetHandlers`. |
| ExternalTarget | BPI_SlotHandler | — | Contenedor externo emparejado (ej. cofre abierto junto al inventario). Lo asigna `BuildActionsMenu` desde la salida `TargetHandler` de `SetHandlers`. Puede no ser válido. |
| FinalActionList | STR_ActionDefinition | Array | Resultado de `ValidateAcceptedActions`; es la lista que se recorre para construir los botones. |
| ContextTag | GameplayTag | — | Tag de contexto contra el que se comparan los `ContextualLabels` de la fila actual (`ClickedHandler` o `ExternalTarget` según `CompareExternal?`). |
| ContextLabel | Text | — | Label resuelto de la fila actual (`DefaultLabel` o el contextual que coincida). |
| CurrentRequest | STR_ActionRequest | — | Copia completa de la entrada `ActionSource` de `BuildActionsMenu` (`Requester`, `Target` = contenedor clicado, `ItemSlotData`). La lee `SelectClickedAction` para armar el `STR_ActionExec`. `CurrentRequest.Target` y `ClickedHandler` son el mismo contenedor. |
| SlotIndex | int | — | Expose on Spawn. `WBP_Slot` le pasa su `ContentIndex`. `SelectClickedAction` lo usa como `SourceSlotIndex` (no viene en `STR_ActionRequest`). |
| SlotTags | GameplayTagContainer | — | Expose on Spawn, categoría **DEPRECATED**. `WBP_Slot` la sigue pasando, pero ya ningún grafo del menú la lee. Se borra en la limpieza. |
| GI Sunyi Ref | GI_Sunyi | — | Expose on Spawn, categoría **DEPRECATED**. Ya no se usa (el menú pide el manager con `GetGameInstance` → `GetExecActionManager`). Se borra en la limpieza. |
| Item Manager | BP_ItemManager | — | Expose on Spawn. Usado por `SetHandlers` y `GetItemInfo` en `BuildActionsMenu`. |
| SlotData | STR_Slot | — | Variable vieja, nunca asignada. Ya no se usa (el ítem llega en `CurrentRequest.ItemSlotData`). Se borra en la limpieza. |

### `WBP_ContextMenu.BuildActionsMenu` (entrada y locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ActionSource | STR_ActionRequest | — | Entrada de la función. |
| LocalReqItem | STR_ItemInfo | — | `ItemInfo` del ítem clicado, obtenido con `Item Manager.GetItemInfo(ActionSource.ItemSlotData.ItemID)`. |
| ActionTag | GameplayTag | — | `ActionTag` de la fila actual del loop sobre `FinalActionList`. |
| AcceptNumValues | bool | — | `AcceptNumValues?` de la fila actual. |

### `WBP_ContextMenu.Build Action Btn` (entradas y locales)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| OptionText | Text | — | Label final resuelto previamente por `BuildActionsMenu`. |
| ItemData | STR_Slot | — | Datos del slot usados para consultar `Quantity`; provienen de `ActionSource.ItemSlotData`. |
| TargetAction | GameplayTag | — | Acción objetivo representada por el botón. Se usa este nombre para no confundirla con otros valores llamados `ActionTag` en grafos cercanos. |
| AcceptNumValues | bool | — | Indica si la definición de la acción permite operar sobre una cantidad parcial del stack; proviene de `STR_ActionDefinition.AcceptNumValues?`. |
| NumericBtn | WBP_NumericButton | — | Local. Referencia al botón numérico recién creado, para agregarlo a `Container` y enlazar su delegate. |
| ClickedBtn | WBP_ActionButton | — | Local. Referencia al botón estándar recién creado, mismo propósito. |

### `WBP_ContextMenu.SelectClickedAction` (entradas)
| Nombre | Tipo | Contenedor | Descripción |
|-----|---|---|----|
| ActionTag | GameplayTag | — | Acción del botón pulsado. |
| Quantity | int | — | Cantidad elegida. `1` fijo si viene de un botón estándar; el valor del selector si viene de un botón numérico. |

---

## Funciones

### `GI_Sunyi.CachedActionRules`
> Reemplazo funcional de `BuildActionRulesCache` (deprecada). Llamada desde `ReceiveInit`, igual que la vieja.

Recorre todos los `RowNames` de `DT_ActionDefinition`, extrae el `ActionTag` de cada fila, obtiene su `Name`, y con eso llena `ActionRulesList` y `ActionRulesListMap` (clave = nombre del `ActionTag`).

### `GI_Sunyi.ExposeActionRules`
> Getter directo, evita repetir `GetDataTableRowNames` en cada apertura de menú. Devuelve `ActionRulesList` y `ActionRulesListMap`.

### `WBP_ContextMenu.BuildActionsMenu`
> Reemplaza `InitializeMenu` como punto de entrada del menú contextual.

**Entradas:** `ActionSource` (`STR_ActionRequest`).

Guarda `ActionSource` completo en la variable de miembro `CurrentRequest` (para que `SelectClickedAction` arme la petición de ejecución), desglosa `ActionSource`, llama `BPFL_Utilities.SetHandlers` (reutilizado conceptualmente para resolver Source/Target; es la única llamada de esta función que sigue en la biblioteca vieja, ver nota de migración), valida que `ItemSlotData` no esté vacío (`BooleanOR` de `ItemID != None` y `Quantity != 0`, cubriendo tanto slots vacíos por ausencia de ítem como por cantidad cero), limpia los hijos del contenedor de UI, obtiene `ItemInfo` del ítem clicado y lo guarda en `LocalReqItem`, y llama a `ValidateActionMenu(ClickedHandler, ItemData=LocalReqItem)` → `ActionListRaw`.

**Filtro de lista negra ya conectado:** el `ActionListRaw` devuelto por `ValidateActionMenu` se pasa directo a `ValidateAcceptedActions(ClickedHandler, ActionListRaw)` → `ActionList`, aplicando el filtro de acciones excluidas antes de continuar. El filtro lo sigue llamando el widget (no se integró dentro de `ValidateActionMenu`, por decisión).

**Migración completada (05/11/2026):** las dos llamadas ya usan el manager: `GetGameInstance` → `GetExecActionManager` (`BPI_GIAccess`) → `ValidateActionMenu` / `ValidateAcceptedActions` del manager. Mismas entradas y salidas. Probado en PIE sin problemas; `BPFL_ContextActions` se borró.

**Conectada desde:** `WBP_Slot.OnPreviewMouseButtonDown` (rama click derecho, solo si el slot tiene `ItemID != None`), que ya no llama a `InitializeMenu`. Si ya existe un `ContextMenuRef` lo quita (`RemoveFromParent`), crea un `WBP_ContextMenu` nuevo pasándole por Expose on Spawn `SlotIndex = ContentIndex`, `SlotTags`, `GI Sunyi Ref` e `Item Manager` (no pasa `SlotData`), obtiene el actor propietario con `SlotHandler.GetOwnerActor()` y llama `BuildActionsMenu` con `ActionSource = (Requester = Owner, Target = SlotHandler, ItemSlotData = SlotData)`. Después hace `AddToViewport` y posiciona `OptionsContainer` en la posición del mouse. Nota: `Requester` es el dueño del contenedor clicado, no necesariamente el jugador (ej. al clicar un slot de cofre, el Requester sería el cofre); relevante para el futuro bucket de Requester.

**Validación de entrada:** un `Branch` sobre `ItemID != None OR Quantity != 0` deja pasar el flujo; si el slot está vacío la función termina sin construir nada. Recién tras esa validación hace `ClearChildren` sobre `Container`.

**Resolución de labels y destino externo:** guarda el resultado de `ValidateAcceptedActions` en `FinalActionList` y la recorre (`ForEachLoop`). Por cada fila guarda `ActionTag`, inicializa `ContextLabel` con `DefaultLabel` y guarda `AcceptNumValues`. Un `Branch` sobre `CompareExternal? AND NOT IsValid(ExternalTarget)` omite la fila (rama `true` sin conectar, el loop sigue). Si continúa, `CompareExternal?` selecciona el origen de `ContextTag`: `ExternalTarget.GetContainerType()` cuando es verdadero o `ClickedHandler.GetContainerType()` cuando es falso. Después recorre `ContextualLabels` con `ForEachLoopWithBreak`; una igualdad exacta `SourceContext == ContextTag` reemplaza `ContextLabel` y corta el loop. Si no hay coincidencia permanece `DefaultLabel`.

**Construcción del botón:** en el `Completed` del loop de `ContextualLabels` (se dispara haya habido break o no) llama `Build Action Btn(OptionText = ContextLabel, ItemData = ActionSource.ItemSlotData, TargetAction = ActionTag, AcceptNumValues)`. `Test_Print_Actions` ya fue retirado.

**Pendiente dentro de esta función:** el bucket de Requester.

### `WBP_ContextMenu.Build Action Btn`
> Construye y agrega al menú el widget visual correspondiente a una acción ya validada. No ejecuta la acción.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| OptionText | Text |
| ItemData | STR_Slot |
| TargetAction | GameplayTag |
| AcceptNumValues | bool |

Primero evalúa `TargetAction == Action.Container.Split AND ItemData.Quantity <= 1`; si es verdadero termina sin crear ningún botón, porque ese stack no se puede dividir.

Si la fila continúa, evalúa `ItemData.Quantity > 1 AND AcceptNumValues`. Si es falso crea `WBP_ActionButton`. Si es verdadero vuelve a comprobar `TargetAction == Action.Container.Split`: para Split crea `WBP_NumericButton` con `StackQuantity = Quantity - 1`; para cualquier otra acción numérica crea `WBP_NumericButton` con `StackQuantity = Quantity`.

En las tres ramas de creación pasa `OptionText` como `ButtonLabel` y `TargetAction` como `ActionTag` del widget, guarda la referencia (`NumericBtn` o `ClickedBtn`) y la agrega al `VerticalBox` `Container`.

**Enlace de clics:** justo después de agregar el widget, enlaza su delegate a un evento del menú: `WBP_ActionButton.OnActionButtonClicked(ActionTag)` → `HandleOptionClicked`, y `WBP_NumericButton.OnActionNumericButtonClicked(Quantity, ActionTag)` → `HandleOptionNumeric`. La función no ejecuta la acción; solo construye y enlaza.

### `WBP_ContextMenu.HandleOptionClicked` / `HandleOptionNumeric` (Custom Events)
> Receptores de los clics de los botones. Unifican ambos tipos de botón en una sola llamada.

- `HandleOptionClicked(ActionTag)` → `SelectClickedAction(ActionTag, Quantity = 1)` (cantidad fija para botones estándar).
- `HandleOptionNumeric(Quantity, ActionTag)` → `SelectClickedAction(ActionTag, Quantity)` (cantidad del selector).

### `WBP_ContextMenu.SelectClickedAction`
> Ejecuta la acción elegida a través de `BP_ExecActionManager.RequestAction` (07/11/2026). Reemplazó al puente temporal que usaba `STR_UseItemPayload` + `GI_Sunyi.OnUseRequested`. **Sin respaldo al sistema viejo**: las acciones que ningún sistema implementa todavía (Use, Inspect, Equip) responden `Fail.NotImplemented` o `Fail.NoHandler`, y es lo esperado.

**Entradas:** `ActionTag` (GameplayTag), `Quantity` (int).

`Get CurrentRequest` → *Break STR_ActionRequest* → *Make STR_ActionExec*:
| Campo de `STR_ActionExec` | Origen |
|---|---|
| Requester | `CurrentRequest.Requester` |
| ActionTag | entrada `ActionTag` |
| ItemSlot | `CurrentRequest.ItemSlotData` |
| Quantity | entrada `Quantity` |
| SourceSlotIndex | `SlotIndex` |
| SourceContainer | `CurrentRequest.Target` (contenedor clicado) |
| TargetContainer | `ExternalTarget` **siempre**, sea válido o no. El filtro según `CompareExternal?` lo hace `RequestAction` (ver más abajo). |

`GetGameInstance` → `GetExecActionManager` (*Message* de `BPI_GIAccess`) → `RequestAction(ActionExec)` → cierra el menú (`RemoveFromParent`) → `SetUserFocus(GetPlayerController(0))`. El `ActionResult` aún no se usa.

**Detalle menor pendiente:** `SetUserFocus` se llama sobre el propio menú después de `RemoveFromParent`; si se busca devolver el foco al juego o al inventario, el destino debería ser ese widget, o usar `Set Input Mode`.

### `BP_ExecActionManager.ValidateActionMenu`
> Función orquestadora. Prefetchea todo el contexto común y llama a las validaciones específicas.

**Entradas:** `ClickedHandler` (`BPI_SlotHandler`), `ItemData` (`STR_ItemInfo`).

**Salidas:** `ActionListRaw` (lista final ordenada) y `MultiUseActions` (las acciones MultiUse sin agrupar, `MultiUseActionsRaw`, para el futuro submenú "Multi uso" y para resolver `ExecuteRandom`).

Usa directamente las reglas del manager (`ActionRulesList` / `ActionRulesMap`); ya no consulta `GI_Sunyi`. En una `Sequence`, resuelve `LocalCapabalities` y `LocalContextTag` desde `ClickedHandler`, garantizando que estén listos antes de llamar a las validaciones. Llama `ValidateBasicActions(ActionRulesList, LocalCapabalities)` → `BasicActions`, y `ValidateActiveAction(ActionRulesList, ItemData)` → `ActiveAction` + `ValidActiveAction?`. `ClickedHandler` conserva su nombre; en términos de `STR_ActionExec` corresponde a `SourceContainer`.

**Gate `AllowUse` sobre la implícita:** rompe `ActiveAction.RequiredTargetCapability` y despacha por `SwitchEnum` de `MatchMode` a `HasAnyTags`/`HasAllTags` contra `LocalCapabalities`, guardando el resultado en `NeedsAllowUse?`. Un `BooleanAND(ValidActiveAction?, NeedsAllowUse?)` alimenta la condición de un `Branch`: solo si ambos son verdaderos se limpia `ActionListRaw` y se agrega `ActiveAction` antes de continuar; si no, solo se limpia (ninguna de las dos ramas pierde la limpieza inicial).

**Conexión de MultiUse y colapso de grupo (`GetMultiUseGroups`):** ambas ramas del branch anterior convergen en la llamada a `ValidateMultiUseActions(LocalActionRules, ItemData, LocalContextTag, LocalCapabalities)`. Si `Success = true`, el resultado se guarda en `MultiUseActions` y se copia también a `MultiUseActionsRaw` (el bucket "sin agrupar", conservado para el fallback y para un futuro submenú), y se llama a `GetMultiUseGroups(MultiUseActionsRaw, ItemData, LocalActionRules)`. Su salida `Grouped?` decide la rama: si es `true`, se guarda `GroupedMultiUse` y se hace `Array_Append(ActionListRaw, GroupedMultiUse)`; si es `false` (0 o 1 acción MultiUse, o porque `Action.MultiUse.OpenMenu` no está cargado en `ActionRules` — ver `GetMultiUseGroups` más abajo), se ignora la salida de `GetMultiUseGroups` y se hace `Array_Append(ActionListRaw, MultiUseActionsRaw)` directamente, mostrando las acciones individuales sin colapsar. Si `Success = false` (ítem sin MultiUse válido), se salta directo a la continuación sin ningún `Array_Append` de MultiUse. Todos los caminos convergen en un único `Array_Append(ActionListRaw, BasicActions)` final, garantizando el orden fijo acordado: implícita → multiuso (agrupado o crudo) → básicas.

**Retorno:** el `then` del último `Array_Append` alimenta un `Return Node` con salida `ActionListRaw` (alimentada por un `Get` de la variable local, renombrada internamente a `LocalActionListRaw` por consistencia de nombres, mismo dato).

**Pendiente dentro de esta función:** el filtro de listas blanca/negra de ítem (`AllowedItemTags`/`ExcludedItemTags`) y el bucket de Requester se resuelven fuera de esta función (ver `Decisiones de diseño`); el filtro de lista negra de *acciones* ya no es pendiente de esta función — vive en `ValidateAcceptedActions`, llamada por `BuildActionsMenu` sobre el resultado combinado.

### `BP_ExecActionManager.ValidateBasicActions`
> Filtra las acciones básicas de contenedor (Split, Transfer, Drop…), excluyendo la implícita.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| ActionList | Array de STR_ActionDefinition |
| Capabilities | GameplayTagContainer |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| LocalValidActions | Array de STR_ActionDefinition |

Recorre `ActionList`; descarta las filas donde `RequiredItemProperty.TagRequirement` **no** está vacío (esas son candidatas a implícita, no básicas). Para el resto, usa `SwitchEnum` sobre `RequiredTargetCapability.MatchMode` para despachar a `HasAnyTags`/`HasAllTags` contra `Capabilities`. Las filas que matchean se agregan al array de salida.

**Probado y confirmado en PIE**: con el tag de capacidad corregido (ver `Decisiones de diseño`), `Use` queda correctamente excluida de este bucket (por tener `RequiredItemProperty` no vacío), y `Split`/`Transfer`/`Drop` aparecen correctamente cuando el contenedor tiene la capacidad correspondiente.

### `BP_ExecActionManager.ValidateActiveAction`
> Resuelve la acción implícita del ítem, independiente de las capacidades del contenedor.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| ActionList | Array de STR_ActionDefinition |
| ItemInfo | STR_ItemInfo |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| ActiveAction | STR_ActionDefinition |
| Success | bool |

Usa `ForEachLoopWithBreak` (corta en el primer match). Para cada fila con `RequiredItemProperty.TagRequirement` no vacío, compara contra `ItemInfo.PropertyTags` usando el `MatchMode` propio de esa fila. En el primer match: guarda la fila en `ActiveAction`, `Found = true`, rompe el loop. Si ninguna fila matchea (ej. un palo o una piedra — recursos sin acción implícita propia), `Found` queda en `false` por defecto; comportamiento correcto y esperado, no un error.

La función **no conoce ni valida** `RequiredTargetCapability` ni ninguna capacidad de contenedor: ese gate (`AllowUse`) es responsabilidad de `ValidateActionMenu`.

### `BP_ExecActionManager.GetMultiUseActions`
> Traduce `STR_MultiUse.ContextRelation` (datos crudos del ítem) a una lista de `STR_Context_Action` (acción resuelta + contexto), usando `ActionRules` como diccionario. Es el paso de "resolución", no de "filtrado" — no valida capacidades ni contexto, solo convierte tags en structs completas.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| ActionRules | Map\<Name, STR_ActionDefinition\> |
| ItemInfo | STR_ItemInfo |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| MultiUseActions | Array de STR_Context_Action |

Recorre `ItemInfo.MultiUseBehavior.ContextRelation` (`ForEachLoop`). Por cada relación, guarda su `RequiredContext` en la variable puente `Context`, rompe `GrantedActions` (GameplayTagContainer) a un array de tags y lo recorre en un segundo `ForEachLoop` anidado (defensivo: la disciplina de datos dice que cada `ContextRelation` debería tener un solo tag en `GrantedActions`, pero la función soporta múltiples sin romperse). Por cada tag, obtiene su `Name` y hace `Map_Find` sobre `ActionRules`. Si se encuentra (`Branch` sobre el booleano `Found` de `Map_Find`, evitando el patrón inseguro `Array_Find` + `GetArrayItem`), construye un `STR_Context_Action` (`ActionRule` = valor encontrado, `Context` = la variable puente) y lo agrega a `FoundActions`. Si no se encuentra, esa entrada se descarta silenciosamente (tag mal escrito o fila borrada de `DT_ActionDefinition`).

### `BP_ExecActionManager.ValidateMultiUseActions`
> Filtra las acciones MultiUse del ítem contra el contexto actual (`ContextTag`) y las capacidades del contenedor (`Capabilities`). Es el paso de "filtrado" que sigue a `GetMultiUseActions`.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| ActionRules | Map\<Name, STR_ActionDefinition\> |
| ItemInfo | STR_ItemInfo |
| ContextTag | GameplayTag |
| Capabilities | GameplayTagContainer |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| MultiUseActions | Array de STR_ActionDefinition |
| Success | bool |

**Guardas de consistencia de datos** (antes de resolver nada): calcula `HastMultiUseTag` (¿el ítem tiene `Item.Property.HasMultiUse`?) y `HasRelations` (¿`ContextRelation` tiene contenido?). Avisa de las dos inconsistencias posibles con el dispatcher **`OnDataWarning(ItemID, Alert)`** (reemplaza los `PrintString`, que no compilan en un Object): tag presente sin datos ("tiene multiples usos activo, pero no hay usos detallados") o datos presentes sin tag ("tiene multiples usos, pero la función no está activa"). Por ahora `Alert` sale vacío; los mensajes originales quedaron como comentarios en el grafo, y los tags previstos son `System.Warning.Data.MultiUse.TagWithoutRelations` / `RelationsWithoutTag` (ver `4-1`). Quien muestra el aviso es `GI_Sunyi` (ver `HandleDataWarning`). Solo si ambos son `true` (caso consistente) se procede a llamar `GetMultiUseActions` y filtrar; en cualquier otro caso la función retorna `MultiUseActions` vacío con `Success = false`.

**Filtrado** (uno por cada `STR_Context_Action` devuelto por `GetMultiUseActions`, `ForEachLoop`): primero `Context.HasTag(ContextTag)` — descarta la acción si el contexto actual (ej. `Container.Type.Player.Inventory.Primary`) no está en la lista de contextos válidos de esa relación. Si pasa, usa el mismo patrón que `ValidateBasicActions`: `SwitchEnum` sobre `RequiredTargetCapability.MatchMode` para despachar a `HasAnyTags`/`HasAllTags` contra `Capabilities`. Si también pasa, agrega el `ActionRule` (no el `STR_Context_Action` completo — el `Context` ya cumplió su propósito de filtro) a `LocalMultiUseActions`, y en ese mismo paso marca una variable local `Found = true`.

**Contrato robusto de `Success`:** en vez de devolver un `Success` fijo en `true` al terminar el loop (ambiguo si el array quedó vacío), la salida `Success` está cableada directamente a `Found`. Así, el llamador (`ValidateActionMenu`) puede ramificar sobre un booleano explícito en vez de inferir "¿hubo resultados?" a partir del contenido del array, evitando depender de que un array vacío se interprete siempre correctamente en cada punto de consumo.

### `BP_ExecActionManager.GetMultiUseGroups`
> Colapsa el bucket de acciones MultiUse ya validadas en dos botones de sistema ("Abrir menú" / "Acción aleatoria") cuando hay más de una, replicando la lógica de `UI_OpenMenu`/`System_Random` del sistema viejo. No valida nada nuevo del ítem ni del contenedor — solo decide cómo se presenta un conjunto ya válido.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| MultiUseList | Array de STR_ActionDefinition |
| ItemInfo | STR_ItemInfo |
| ActionRules | Map\<Name, STR_ActionDefinition\> |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| GroupedMultiUse | Array de STR_ActionDefinition |
| Grouped? | bool |

Comprueba primero si `MultiUseList` tiene más de un elemento (`Array_Length > 1`). Si no (0 o 1), no hace nada más: `Grouped?` queda en `false` por defecto y `GroupedMultiUse` queda vacío — el llamador (`ValidateActionMenu`) ya sabe, por contrato, que debe usar el bucket sin agrupar (`MultiUseActionsRaw`) en ese caso, así que un array vacío aquí no es ambiguo.

Si hay más de una acción, busca la fila `Action.MultiUse.OpenMenu` en `ActionRules` (`Map_Find` sobre el nombre del tag, hardcodeado como literal — es una acción de sistema, no depende de datos del ítem, a diferencia de los tags que resuelve `GetMultiUseActions`). Si **no se encuentra** (`Found = false` del `Map_Find`), la función corta ahí mismo: no llega a evaluar `Action.MultiUse.ExecuteRandom` ni agrega nada a `GroupedMultiUse`, dejando `Grouped? = false`. Este es un cortocircuito intencional, no un caso sin cubrir: por disciplina de datos `Action.MultiUse.OpenMenu` debería existir siempre que haya más de una acción MultiUse, así que su ausencia es un error de contenido (fila borrada o `DT_ActionDefinition` desactualizada) — degradar a "mostrar las acciones individuales sin agrupar" es el comportamiento seguro, en vez de mostrar un botón "Acción aleatoria" huérfano sin su menú.

Si se encuentra, agrega esa fila a `GroupedMultiUse` y revisa `ItemInfo.MultiUseBehavior.AcceptRandom?` (flag propio del ítem). Si es `true`, busca `Action.MultiUse.ExecuteRandom` en `ActionRules` (mismo patrón de `Map_Find` con tag hardcodeado) y, si existe, también la agrega a `GroupedMultiUse`. Al final de este camino, `Grouped? = true`.

### `BP_ExecActionManager.ValidateAcceptedActions`
> Filtro de lista negra de *acciones* (lógico, no físico). Recibe el array ya ensamblado de `ValidateActionMenu` y descarta las filas cuyo `ActionTag` esté cubierto por `ExcludeActions` del contenedor, con coincidencia jerárquica de raíz.

**Entradas:**
| Parámetro | Tipo |
|-----|---|
| ClickedHandler | BPI_SlotHandler |
| ActionListRaw | Array de STR_ActionDefinition |

**Salidas:**
| Parámetro | Tipo |
|-----|---|
| ActionList | Array de STR_ActionDefinition |

Obtiene `ExcludeList` con `ClickedHandler.GetExcludeActions()` y limpia `CleanList`. Recorre `ActionListRaw` (`ForEachLoop`): por cada fila, guarda `ActionRule` y su `ActionTag`, resetea `IsExcluded? = false`, y entra a un loop interno (`ForEachLoopWithBreak`) sobre los tags individuales de `ExcludeList` (`BreakGameplayTagContainer`). Por cada tag de exclusión, envuelve el `ActionTag` de la fila actual en un contenedor de un solo tag (`MakeGameplayTagContainerFromTag`) y llama `HasTag(TagContainer=ese contenedor, Tag=tag de exclusión, bExactMatch=false)` — la dirección de la comparación es la clave del fix de esta sesión: con `bExactMatch=false`, `HasTag` también compara contra los tags *padre* del contenedor, así que preguntar "¿el tag de exclusión coincide con el `ActionTag` o alguno de sus padres?" es exactamente "¿este tag de exclusión, si es una raíz, cubre a este `ActionTag` hijo?". Si coincide, marca `IsExcluded? = true` y rompe el loop interno (no hace falta seguir revisando el resto de `ExcludeList`, ya está decidido). Al completarse el loop interno (con o sin break), si `IsExcluded? = false` se agrega `ActionRule` a `CleanList`; si quedó en `true`, la fila se descarta silenciosamente. Al completarse el loop externo, retorna `ActionList = CleanList`.

---

## Ejecución de acciones (rediseño 05/11/2026)

> Ejecución construida desde cero. Diseño general en `4-0`; estructuras (`STR_ActionExec`, `STR_ActionResult`) y tags en `4-1`.

### `BPI_ExecActionHandler` (interfaz)
> Ruta: `ContextActionSystem/Blueprints/Interfaces/`. Contrato único que implementa todo componente que ejecuta acciones (Inventory, Equipment y, a futuro, Craft, Build…). Reemplaza a `BPI_UseItemHandler`.

| Función | Entradas | Salidas | Descripción |
|---|---|---|---|
| `ExecuteAction` | `ActionExec` (STR_ActionExec, sin Pass-by-Reference) | `ActionResult` (STR_ActionResult) | El componente ejecuta la acción con su propia lógica y validaciones específicas, y responde. |
| `GetHandlerTag` | — | `HandlerTag` (GameplayTag) | Devuelve la variable `SystemHandler` del componente (`System.Handler.*`), para que el manager lo encuentre sin Cast. |

Como ambas funciones tienen salida, en el componente se implementan como **funciones** (no eventos), con respuesta síncrona. Al llamarlas como *Message* sobre un objeto que no implementa la interfaz, Unreal devuelve la struct por defecto (por eso `Success? = false` por defecto).

### `BP_ExecActionManager`
> Ruta: `ContextActionSystem/Blueprints/`. Tipo **Object** (mismo patrón que `BP_ItemManager` / `BP_UseItemManager`): único, sin presencia en el mundo, igual para jugador y PNJ. Lo construye `GI_Sunyi`. Reemplaza a `BP_UseItemManager`. Todas sus funciones quedan **Public** (decisión: no se sabe aún qué otros sistemas las usarán).

#### `RequestAction` — *implementada*
**Entrada:** `ActionExec` → **Salida:** `ActionResult`.
**Locales:** `LocalExec` (STR_ActionExec), `LocalRule` (STR_ActionDefinition), `LocalHandler` (Object), `LocalResult` (STR_ActionResult).
0. `ItemSlot.ItemID != None` → si no: `Fail.InvalidItem`. Solo se valida el `ItemID`: el inventario ya limpia siempre cualquier slot con `Quantity <= 0`.
1. `ResolveAction(ActionExec)` → si no `Found?`: `Fail.NoRule`. Si sí: `Set LocalRule`.
2. `Set LocalExec = ActionExec` → *Branch* sobre `LocalRule.CompareExternal?`:
   - **true** → *Set Members in STR_ActionExec* (`ActionTag` = `LocalRule.ActionTag`); `TargetContainer` se queda como llegó.
   - **false** → *Set Members in STR_ActionExec* (`ActionTag` = `LocalRule.ActionTag`, `TargetContainer` expuesto y sin conectar = **None**).
   - Las dos ramas guardan en `LocalExec` y siguen a `ValidateExec`. Desde aquí todo usa `LocalExec`, para que el ejecutor y el aviso conozcan la acción resuelta aunque la petición llegara vacía.
   - **Por qué se limpia `TargetContainer`:** `FindHandler` busca primero en el Target. Si el menú manda el cofre abierto como Target en un Split o un Drop, el cofre (también `System.Handler.Inventory`) coincide primero y ejecuta la acción sobre su propio slot `SourceSlotIndex`, no sobre el del contenedor clicado. Cambiar el orden de `FindHandler` no sirve, porque Transfer necesita que el destino sea quien ejecuta. El filtro va en el manager (y no en el menú) porque solo el manager conoce la acción resuelta (una petición con `ActionTag` vacío) y así aplica también a PNJ, hotbar y teclas rápidas.
3. `ValidateExec(LocalExec, LocalRule)` → si no `Valid?`: fallo con el `FailTag` devuelto.
4. `FindHandler(LocalExec, LocalRule.ExecutorReference)` → si no `Found?`: `Fail.NoHandler`. Si sí: `Set LocalHandler`.
5. `ExecuteAction` (*Message* de `BPI_ExecActionHandler`) sobre `LocalHandler` con `LocalExec` → `Set LocalResult`.
6. *Comment* PENDIENTE: destino de `LocalResult.ReturnedItems` (ver `4-1`: almacenamiento oculto anclado al `InstanceID`, por diseñar).
7. `FinishAction(LocalExec, LocalResult)` → Return (`ActionResult` = `Result`).

**Las cinco salidas pasan por `FinishAction`**: cada fallo es `FinishAction(ActionExec o LocalExec, MakeFailResult(tag))` → Return. Fallos 0 y 1 usan `ActionExec` (aún no existe `LocalExec`); fallos 3 y 4 usan `LocalExec` (con la acción resuelta, para que `OnActionExecuted` informe qué acción falló). Verificado en el Blueprint (07/11/2026).

**Probado en PIE (07/11/2026):** Split, Drop y Transfer desde el menú contextual, de punta a punta.

#### `ResolveAction` — *implementada (salvo MultiUse por contexto)*
**Entrada:** `ActionExec` → **Salidas:** `ActionRule` (STR_ActionDefinition), `Found?` (bool, **por defecto false** en los Return Node, para que ninguna rama devuelva `true` por accidente).
- `ActionTag` válido (explícita: menú, PNJ) → `GetTagName` → `Map_Find` en `ActionRulesMap` → Return (`Found?` = resultado del `Map_Find`).
- `ActionTag` vacío (hotbar, tecla rápida, doble clic) → `ItemManagerRef.GetItemInfo(ItemSlot.ItemID)` → `Set LocalItemFound` → si falla: `Found? = false` → si no: `ValidateActiveAction(ActionRulesList, LocalItemFound)` (la misma función del menú, así doble clic y menú coinciden) → con acción por defecto: Return (`ActionRule` = `ActiveAction`, `Found? = true`) → sin acción por defecto: `Found? = false` + *Comment* **PENDIENTE: MultiUse por contexto**.
- No aplica el gate `AllowUse` sobre la acción por defecto: lo cubre `ValidateExec` (capacidades del contenedor contra la fila resuelta).

#### `ValidateExec` — *implementada*
**Entradas:** `ActionExec`, `ActionRule` → **Salidas:** `Valid?` (por defecto false), `FailTag`. **Local:** `CurrentSlot` (STR_Slot). Reglas genéricas, en orden:
1. **A** `SourceContainer` (*Is Valid*) → no válido: Return `Valid? = true` sin tag (no hay contenedor que revisar).
2. **B** `SourceSlotIndex >= 0` → False (la acción no sale de un slot) salta a D.
3. **C** `GetSlotData(SourceSlotIndex)` → `CurrentSlot` → `ItemID` == **y** `InstanceID` == (*Equal Guid*) → si no: `Fail.StaleSlot` (**medida de seguridad**: el inventario refresca la UI en cada operación, pero el menú guarda una copia al abrirse y no escucha `OnInventoryUpdated`). Luego `Quantity >= 1` **y** `Quantity <= CurrentSlot.Quantity` → si no: `Fail.InvalidQuantity`. El tope de Split (`Quantity - 1`) **no** va aquí: es regla específica de Inventory.
4. **D** `ValidateCapabalities(ActionExec, ActionRule)` → si no `CapabilityFound?`: `Fail.NotAllowed`.
5. **E** `ValidateExcludedActions(ActionExec, ActionRule)` → si `FoundExcluded?`: `Fail.NotAllowed`.
6. **F** *Comment*: "Faltan las capacidades del requester" (`RequiredRequesterCapability`, pendiente del componente de capacidades del actor).
7. **G** Return `Valid? = true` sin tag.

D, E, F y G están **duplicados** a propósito, una cadena para la ruta "sin slot" (B-False) y otra para "con slot" (C-True), por organización visual. Cualquier validación genérica nueva debe agregarse en **las dos** cadenas.

#### `ValidateCapabalities` — *implementada*
**Entradas:** `ActionExec`, `ActionRule` → **Salida:** `CapabilityFound?` (bool). **Locales:** `CapabilityOk?` (bool), `LocalCapabilities` (GameplayTagContainer).
`RequiredTargetCapability.TagRequirement` vacío (`GetNumGameplayTagsInContainer == 0`) → `CapabilityOk? = true` (**requisito vacío = pasa**, explícito). Si no: `SourceContainer.Get Capabilities` → `LocalCapabilities` → *Switch on ENUM_ContextMatching* (`MatchMode`) → MatchAny: `HasAnyTags`, MatchAll: `HasAllTags` (`TagContainer` = `LocalCapabilities`, `Other` = requisito, exact = false) → `CapabilityOk?`. Las tres rutas convergen en el Return. Extraída como función por organización; candidata a reutilizarse en `ValidateBasicActions`/`ValidateMultiUseActions` cuando se alinee el menú con "requisito vacío = pasa" (hoy el menú todavía oculta, por ejemplo, Inspect).

#### `ValidateExcludedActions` — *implementada*
**Entradas:** `ActionExec`, `ActionDefinition` (por referencia, solo lectura) → **Salida:** `FoundExcluded?` (bool).
`SourceContainer.GetExcludeActions` → `HasAnyTags(TagContainer = MakeGameplayTagContainerFromTag(ActionDefinition.ActionTag), Other = ExcludeActions, exact = false)`. La acción va en *Tag Container* para que Unreal incluya sus tags padre: excluir `Action.Container.Equip` también excluye `Action.Container.Equip.Wield`. Usa el `ActionTag` de la fila (el resuelto), no el del payload. Un solo nodo hace lo que `ValidateAcceptedActions` hace con un loop.

#### `FindHandler` — *implementada*
**Entradas:** `ActionExec`, `HandlerTag` (= `ExecutorReference`) → **Salidas:** `Handler` (Object), `Found?` (por defecto false). **Locales:** `Cadidates` (Array de Object; nombre con errata), `Requester`, `SourceContainer`, `TargetContainer`, `FinalHandler`, `Found`.
- `HandlerTag` no válido → `Found? = false` (filas sin sistema, como "Multi uso" / "Random").
- *Clear* `Cadidates` → *Sequence* que guarda `Requester`, `SourceContainer` y `TargetContainer` (como Object) → segunda *Sequence* que arma los candidatos con *Validated Get* y **AddUnique**, en orden: Target → Source → componentes del dueño del Target (`GetOwnerActor` → `GetComponentsByInterface(BPI_ExecActionHandler)` → *For Each* → AddUnique) → componentes del Requester (ídem).
- **AddUnique** evita duplicados (el Target vuelve a aparecer entre los componentes de su dueño y el Source entre los del Requester) conservando la primera aparición, que es la que respeta el orden. *For Each + AddUnique* en vez de *Append* porque `GetComponentsByInterface` devuelve ActorComponent y `Cadidates` es Object.
- Búsqueda: *For Each Loop with Break* → `Set FinalHandler` = elemento → `GetHandlerTag` (*Message*) → `MatchesTag` exacto con `HandlerTag` → coincide: `Found = true` → Break. *Completed* → Return (`Handler` = `FinalHandler`, `Found?` = `Found`).
- Nota: `FinalHandler` se asigna antes de comparar, así que sin coincidencia `Handler` queda con el último candidato revisado (no vacío). No afecta: `RequestAction` solo usa `Found?`.

#### `MakeFailResult` — *implementada* (Private, Pure)
**Entrada:** `FailTag` → **Salida:** `ActionResult` (`Success? = false`, `ResultTag = FailTag`, `QuantityProcessed = 0`, `ReturnedItems` vacío). El manager solo arma resultados de fallo (los de éxito los arma el sistema dueño); con esta función cada rama de fallo es un solo nodo y todos los fallos se arman igual.

#### `FinishAction` — *implementada* (Private)
**Entradas:** `ActionExec`, `ActionResult` → **Salida:** `Result`. `Call OnActionExecuted(ActionExec, ActionResult)` → Return. Salida única de `RequestAction`: garantiza que el aviso se dispare en todas las ramas. (La salida se llama `Result` porque Unreal no permite una entrada y una salida con el mismo nombre.)

#### Funciones migradas desde `BPFL_ContextActions`
`ValidateActionMenu`, `ValidateBasicActions`, `ValidateActiveAction`, `GetMultiUseActions`, `ValidateMultiUseActions`, `GetMultiUseGroups`, `ValidateAcceptedActions`. Lógica idéntica a la original (se migró sin corregir nada, para que la prueba de regresión en PIE dé exactamente los mismos botones). Diferencias: reglas desde variables del manager, `OnDataWarning` en vez de `PrintString`, y `ValidateActionMenu` devuelve además `MultiUseActions` sin agrupar. Detalle de cada una en la sección *Funciones*.

### `InventoryComponent` — ejecutor de acciones de contenedor
> `InventoryComponent` es **el** sistema de administración de ítems. Los demás sistemas (Crafting, Equipment…) usan inventarios a través de `BPI_SlotHandler` (`RemoveAnItem`, `PickupInActions`), no tienen su propio Split/Transfer/Drop. Por eso esas acciones tienen `ExecutorReference = System.Handler.Inventory`.

Implementa `BPI_ExecActionHandler`. Variable `SystemHandler = System.Handler.Inventory`. Las funciones `ExecX` son propias del componente, **no** van en la interfaz.

| Función | Lógica |
|---|---|
| `GetHandlerTag` | Devuelve `SystemHandler`. |
| `ExecuteAction` | *Switch on Gameplay Tag* sobre `ActionExec.ActionTag`: Split / Transfer / Drop → `ExecX` → *Set* `Action_Result` (local). **Default** → *Make STR_ActionResult* con `Fail.NotImplemented`. Un solo Return. |
| `ExecSplit` | `GetSlotData(self, SourceSlotIndex)` → si `Quantity < slot.Quantity` → `ActionSplitItem` → `Success` / `Fail.Inventory.NoSpace` (con `Success? = false`). Si no → `Fail.Inventory.InvalidSplit`. |
| `ExecDrop` | `RemoveItem(self, SourceSlotIndex, Quantity, ShouldSpawn = true)` → `Success` / `Fail.InvalidItem`. |
| `ExecTransfer` | Lo ejecuta el inventario **de destino**. `TargetContainer` válido y `== Self`, si no `Fail.Inventory.NoTarget` → `CanAcceptItem(ItemID, IgnoreSpaceCheck = true)`, si no `Fail.Inventory.TargetRejected` → `PickUpItem(Make STR_Slot(ItemID, InstanceID, ActionExec.Quantity), NeedStack = true)` → `Processed = Quantity − QuantityRemaining` → `Processed > 0`, si no `Fail.Inventory.NoSpace` → `SourceContainer.RemoveAnItem(SourceSlotIndex, Processed, false)` → `Success` con `QuantityProcessed = Processed`. |

**Reglas de Transfer:** se agrega primero en el destino y se descuenta del origen solo lo procesado; el excedente nunca sale del origen. El `STR_Slot` que se entrega se arma con `ActionExec.Quantity` (pasar `ItemSlot` directo duplicaba ítems). `IsStackable?` se comprueba en `FindStack`; `MaxStack = 1` para los no apilables. El `InstanceID` se ignora por ahora al apilar (futuro: pociones crafteadas con el mismo `ItemID`).

### `GI_Sunyi` — construcción del manager
En `ReceiveInit`: `CachedActionRules` → **Construct Object from Class** (`Class = ExecActionClass`, `Outer = Self`, `ItemManagerRef`, `ActionRulesList`, `ActionRulesMap = ActionRulesListMap`) → **Set `ExecActionRef`** → **Bind Event to `OnDataWarning`** → `HandleDataWarning` (Custom Event: *Format Text* "Item: {Item} / Mensaje: {msj}" con `ItemID` y `GetTagName(Alert)` → *Print*, Development Only). El *Construct* va **después** de `CachedActionRules` para que las reglas no lleguen vacías. `ReceiveInit` del GameInstance corre antes que cualquier nivel o widget, así que el manager ya existe cuando alguien lo pide.

### `BPI_GIAccess.GetExecActionManager`
Salida `ExecActionManager` (BP_ExecActionManager). Implementada en `GI_Sunyi` devolviendo `ExecActionRef`. Uso desde cualquier Blueprint: `GetGameInstance` → `GetExecActionManager` (*Message*) → funciones del manager. Sin Cast.

### Suscripción a `OnActionExecuted`
No la hace el GI. Se suscribe cada sistema que necesita reaccionar a acciones de otros: actores/componentes en `BeginPlay`; widgets en `Construct`, con *Unbind* en `Destruct` (los widgets se crean y destruyen muchas veces y el manager vive con el GI). Los sistemas que **ejecutan** acciones no se suscriben: implementan `BPI_ExecActionHandler`.

---

## Decisiones de diseño

- **El gate `AllowUse` vive en `ValidateActionMenu`, no en `ValidateActiveAction`.** "¿El ítem tiene una acción natural?" (ítem) y "¿el contenedor permite usarla aquí?" (contenedor) son preguntas independientes; mezclarlas rompería la reutilización de `ValidateActiveAction` en contextos donde el contenedor no sea relevante.
- **`ValidateBasicActions` y `ValidateActiveAction` se mantienen separadas**, a pesar de recorrer el mismo `ActionList`. Se evaluó fusionarlas para ahorrar una iteración y se descartó a favor de la separación de responsabilidades.
- **Bug de tipeo corregido en `DT_ActionDefinition.json`**: cuatro filas usaban `Container.Capabality.AllowUse/AllowSplit/AllowMove/AllowDrop` (con error), en vez de `Container.Capability.*` (el tag real, correctamente registrado en `Config/Tags/Sunyi_ContainerTags.ini` — los archivos de `Config/Tags` no tienen ningún error). El error se había heredado por similitud visual con los nombres de función viejos (`GetCapabalities`, `HandlerCapabalities`, que sí tienen el typo pero son solo identificadores de Blueprint, no tags). Ya corregido; el futuro tag `Container.Capability.AllowMultiUse` debe crearse sin el typo.
- **Separación `AllowUse` / `AllowMultiUse`**: el sistema viejo usaba el mismo tag (`Container.Capability.AllowUse`) como gate tanto para la acción implícita como para MultiUse. El sistema nuevo introduce `Container.Capability.AllowMultiUse` (tag nuevo, aún no creado en `Config/Tags`) para poder permitir el uso natural de un ítem sin necesariamente permitir sus usos alternativos.
- **Atomicidad de `STR_ContextRelation`**: se confirmó que cada `GrantedActions` debe referenciar una sola acción independiente de `DT_ActionDefinition`, nunca dos o más tags representando alternativas de la misma relación (ej. "equipar como arma" vs "equipar como munición" son dos `STR_ContextRelation` separadas, cada una con su propio `RequiredContext` y un único tag en `GrantedActions`). Evita ambigüedad de "cuál de las dos se ejecuta" al momento de la acción explícita. `GetMultiUseActions` respeta esta disciplina pero no la impone en código: itera `GrantedActions` como container completo (soportaría más de un tag por relación sin romperse), la atomicidad es una regla de autoría de datos, no una validación estructural.
- **Bug encontrado y corregido en `ValidateMultiUseActions`**: la variable `HasRelations` se calculaba originalmente con `Array_IsEmpty(ContextRelation)` cableado directo (sin negar), invirtiendo su significado real ("vacío" en vez de "tiene contenido"). Esto rompía en cascada las tres compuertas que la usan: la alerta de "tag sin datos" y la de "datos sin tag" se disparaban en los casos exactamente opuestos a los pensados, y el gate real que decide si se llama a `GetMultiUseActions` solo dejaba pasar el caso inconsistente (tag activo + `ContextRelation` vacío), bloqueando cualquier ítem MultiUse correctamente configurado. Corregido reemplazando el nodo por `Array_IsNotEmpty`, que restaura el significado correcto de la variable y corrige las tres compuertas sin tocarlas.
- **Bug de contenido encontrado en pruebas PIE — tags de capacidad desactualizados en contenedores existentes**: tras corregir el typo `Container.Capabality.*` → `Container.Capability.*` en `DT_ActionDefinition.json`, las básicas (`Split`/`Transfer`/`Drop`) fallaban para todos los ítems sin importar el contenedor. Diagnóstico paso a paso confirmó que `GetCapabalities()` en el inventario del jugador seguía devolviendo la lista de capacidades con la ortografía vieja (`Container.Capabality.AllowUse/AllowSplit/AllowMove/AllowDrop`), nunca actualizada tras el fix del typo en la tabla. Como son cadenas de tag distintas, `HasAnyTags` nunca encontraba coincidencia — fallo uniforme, no ligado a un contenedor específico. Corregido actualizando los tags de capacidades por defecto del inventario del jugador a la ortografía correcta; queda pendiente auditar el resto de contenedores (cofres, bolsas de drop, etc.) por el mismo problema.
- **Mismo bug encontrado en PIE en `BP_Chest` — el menú contextual no construye ningún botón al abrir un cofre**: con un contenedor `Container.Type.Chest` abierto, `ValidateBasicActions` devolvía `false` en todos los `HasAnyTags`. Diagnóstico: `Capabilities` del cofre seguía con `Container.Capabality.AllowUse/AllowSplit/AllowMove/AllowDrop` (ortografía vieja), mientras las filas de `DT_ActionDefinition` piden `Container.Capability.*`. Como también fallan `AllowUse` (implícita y MultiUse), `FinalActionList` queda vacía y no se crea ningún botón; no está relacionado con `ContextualLabels`. Auditoría de `Content/` (búsqueda del texto `Container.Capabality` en todos los `.uasset`/`.umap`): solo aparece en `LevelPrototyping/TestInteractables/BP_Chest.uasset` (valor por defecto del Blueprint; ninguna instancia colocada en el mapa lo sobrescribe) y en `DT_ActionList.uasset` (sistema viejo, a deprecar, no afecta al menú nuevo). Los tags mal escritos ya no existen en `Config/Tags`; el asset conserva el nombre serializado, por eso el tooltip los sigue mostrando. **Corrección**: reasignar en `BP_Chest` → `InventoryComponent` → `Capabilities` los cuatro tags desde el árbol `Container.Capability.*`. **Prevención (opcional)**: agregar Gameplay Tag Redirects (`Container.Capabality.X` → `Container.Capability.X`) en Project Settings → GameplayTags, para que cualquier asset con el nombre viejo se corrija al cargarse.
- **Bug de datos encontrado y corregido en pruebas PIE — fuga de la acción implícita "Equipar" hacia ítems MultiUse no equipables**: al agregar `ActionEquip` (implícita) y `ActionEquipWield` (MultiUse) para probar el flujo, la rama (ítem MultiUse sin acción implícita propia) mostraba tanto "Equipar" como "Agarrar", cuando solo debía mostrar "Agarrar". Causa: `ActionEquip.RequiredItemProperty` usaba `MatchAny` con dos tags — `Item.Property.HasMultiUse` **y** `Item.Property.Use.Equippable` — por lo que cualquier ítem MultiUse (tenga o no la propiedad de ser equipable) calzaba con la implícita. No es un problema de jerarquía de tags ni de `HasAnyTags` necesitando `bExactMatch`; es un problema de autoría de datos, resuelto por la nueva regla de principios de diseño (`RequiredItemProperty` nunca debe incluir `HasMultiUse`). Corregido quitando `Item.Property.HasMultiUse` del `RequiredItemProperty` de `ActionEquip` en `DT_ActionDefinition.json`, dejando solo `Item.Property.Use.Equippable`.
- **Prueba de regresión en PIE (manzana / rama / casco) confirma el comportamiento correcto tras ambos fixes**: manzana → Soltar, Transferir, Dividir, Usar (básicas + implícita, sin MultiUse). Rama → Soltar, Transferir, Dividir, Agarrar (básicas + MultiUse, **sin** la fuga de "Equipar"). Casco → Soltar, Transferir, Dividir, Agarrar, Equipar (básicas + MultiUse + implícita, coexistiendo correctamente porque el casco sí es genuinamente equipable además de tener la relación de contexto que otorga "Agarrar").
- **Bug de dirección de comparación encontrado y corregido en `ValidateAcceptedActions`**: el primer intento comparaba `ExcludeList.HasTag(ActionTag, bExactMatch=false)`, que solo detecta coincidencia jerárquica cuando `ActionTag` es padre de algo en `ExcludeList` — el caso inverso al que se necesitaba (raíz en la lista negra, hijo en la fila de acción). Corregido invirtiendo la dirección: se envuelve el `ActionTag` de la fila en un contenedor de un solo elemento y se pregunta, por cada tag de `ExcludeList`, si ese contenedor `HasTag(tag, bExactMatch=false)` — así `bExactMatch=false` evalúa los tags padre del `ActionTag` contenido, que es la dirección jerárquica correcta para que excluir una raíz (ej. `Action.Container.Equip`) excluya también a sus hijos (`Action.Container.Equip.Wield`).
- **`GetMultiUseGroups` usa tags hardcodeados para `Action.MultiUse.OpenMenu` y `Action.MultiUse.ExecuteRandom`**, a diferencia de `GetMultiUseActions` que resuelve tags dinámicamente desde los datos del ítem. Es una decisión consciente: estas dos acciones son de sistema (fijas, no varían por ítem ni por contenido de `DT_ActionDefinition` más allá de existir o no), así que no hay nada que "resolver" dinámicamente — la única pregunta es si la fila está cargada en `ActionRules`.
- **Dependencia estructural intencional entre `OpenMenu` y `ExecuteRandom` en `GetMultiUseGroups`**: si `Action.MultiUse.OpenMenu` no se encuentra en `ActionRules`, la función corta antes de siquiera evaluar `Action.MultiUse.ExecuteRandom`, por construcción del grafo (el chequeo de `ExecuteRandom` vive anidado dentro de la rama de "`OpenMenu` encontrado"). Se evaluó como posible inconsistencia y se confirmó que es el comportamiento correcto: por disciplina de datos, `OpenMenu` debe existir siempre que haya más de una acción MultiUse, así que su ausencia ya es un error de contenido — degradar mostrando las acciones individuales (vía `Grouped? = false` en `ValidateActionMenu`) es preferible a mostrar un botón "Acción aleatoria" sin su menú contenedor.
- **`SlotData.Quantity > 1` reemplaza a `Item.Property.IsStackable` como criterio para decidir si mostrar el selector de cantidad.** Se evaluó comparar la propiedad del ítem (como hacía el sistema viejo, sobrecargando el campo `ItemProperty` de la fila de acción para ese propósito) y se descartó a favor de mirar directamente `Quantity` del slot clicado: es el dato real que importa (si hay algo que operar parcialmente *en esta instancia concreta*), no depende de que cada ítem apilable tenga el tag bien puesto, y evita heredar la sobrecarga de significado que tenía ese campo en el sistema viejo.
- **`AcceptNumValues?` se agregó como campo nuevo a `STR_ActionDefinition`** (ver `4-1`) en vez de resolver "¿esta acción opera por cantidad?" comparando contra un set de `ActionTag`s hardcodeados dentro de la función de construcción. A diferencia de `Action.MultiUse.OpenMenu`/`ExecuteRandom` (acciones de sistema sin ningún dato de ítem contra el cual comparar, por lo que ahí sí se justifica hardcodear), acá sí hay una variación real de contenido por fila (Split/Transfer/Drop la tienen, el resto no, y una acción futura podría necesitarla también), así que corresponde que sea 100% data-driven.
- **El ocultamiento total de Split cuando `Quantity <= 1` (y el tope de su spinner en `Quantity - 1`, nunca `Quantity` completa) se mantiene como regla fija de esa acción puntual, no como campo de datos.** `Build Action Btn` comprueba Split dos veces con propósitos distintos: primero para omitir un stack indivisible y, tras confirmar que la acción admite valores numéricos, para aplicar exclusivamente su límite `Quantity - 1`. Las demás acciones numéricas reciben `Quantity` completa.
- **`CompareExternal?` separa las acciones cuyo label depende del destino externo.** Una fila con este flag usa `ExternalTarget.GetContainerType()` para resolver su label; las demás usan `ClickedHandler.GetContainerType()`. Transferir lo activa porque el nombre útil de la acción depende de hacia dónde va el ítem (por ejemplo, Mover, Guardar o Entregar), no de su contenedor de origen.
- **La validación de `ExternalTarget` no vive en `ValidateActionMenu`/`ValidateBasicActions`.** Esas funciones no conocen el destino emparejado; `ExternalTarget` existe en el scope de `BuildActionsMenu`. Por eso el loop de construcción omite cualquier fila con `CompareExternal? = true` cuando el externo no es válido y usa el mismo flag para escoger el contexto del label.
- **`TargetAction` nombra la identidad de la acción dentro de `Build Action Btn`.** Los widgets siguen exponiendo internamente `ActionTag`, pero el parámetro de la función se llama `TargetAction` para distinguirlo de los múltiples valores `ActionTag` usados en otros puntos del sistema. El tag se conserva en el botón porque un label contextual no es una identidad estable.
- **Un solo punto de salida para los clics (`SelectClickedAction`).** Los dos tipos de botón llegan a la misma función: el estándar con `Quantity = 1` fijo y el numérico con la cantidad elegida. Gracias a eso, el cambio a `RequestAction` solo tocó el cuerpo de `SelectClickedAction`, sin tocar los botones ni `Build Action Btn`.
- **Sin respaldo al sistema viejo (07/11/2026).** El puente temporal (`STR_UseItemPayload` + `OnUseRequested`) se quitó y no se agregó un respaldo para `NoHandler`/`NotImplemented`: el sistema viejo se elimina completo. Use, Inspect y Equip quedan sin funcionar hasta implementarlas en el sistema nuevo.
- **El menú guarda `ActionSource` completo (`CurrentRequest`)** en vez de copiar sus campos a variables sueltas: un solo *Set*, y los tres datos (Requester, contenedor clicado, slot) viajan juntos. Corrigió también el bug de `SlotData` vacía del puente viejo.
- **El filtro de `TargetContainer` según `CompareExternal?` vive en `RequestAction`, no en el menú.** El menú siempre envía `ExternalTarget`; el manager lo limpia cuando la acción resuelta no lo necesita (ver `RequestAction`, paso 2).

### Decisiones del rediseño de la ejecución (05/11/2026)
- **Manager nuevo, no renombrado.** Se creó `BP_ExecActionManager` desde cero en vez de renombrar `BP_UseItemManager`: el grafo viejo estaba construido sobre lo descartado (`STR_UseItemPayload`, `DT_ActionPropertyRule`, `SourceHandler.UseItem`), y el puente temporal sigue dependiendo del viejo hasta que el nuevo esté probado.
- **Entrada por función con retorno, no por dispatcher.** `RequestAction` devuelve `STR_ActionResult` en el mismo nodo. Un dispatcher no devuelve nada (el solicitante, por ejemplo la IA de un PNJ, no sabría por qué falló) y, si se enlazan dos oyentes por error, la acción se ejecutaría dos veces.
- **Ejecutar por interfaz, avisar por dispatcher.** La ejecución va a **un** componente por llamada de interfaz (`ExecuteAction`); `OnActionExecuted` es solo un aviso posterior. Usar el dispatcher para ejecutar obligaría a cada sistema a filtrar "¿esto es mío?" (monolito repartido) y no devolvería resultado.
- **`OnActionExecuted` avisa siempre** (éxito y fallo), para trazabilidad completa.
- **El GI entrega los datos; el manager no conoce al GI.** Igual que los demás singletons: `GI_Sunyi` le pasa `ItemManagerRef` y las reglas por Expose on Spawn. Se descartó guardar una referencia a `GI_Sunyi` dentro del manager. Las reglas son de solo lectura (cargadas una vez desde `DT_ActionDefinition`), así que la copia del manager no se desactualiza.
- **`GetHandlerTag` + variable `SystemHandler` en vez de `ContainerType`.** `ContainerType` responde "qué contenedor / en qué contexto" (y ya lo usan `ContextualLabels` con igualdad exacta y `RequiredContext` de MultiUse); varios `ContainerType` (inventario del jugador, cofre, bolsa) son el mismo sistema. Enrutar con él obligaría a reorganizar sus tags y a que todo ejecutor implemente `BPI_SlotHandler`. El tag de sistema queda separado; la reorganización jerárquica de `ContainerType` puede evaluarse aparte.
- **Orden de búsqueda Target → Source → actor del Target → Requester.** Mirar primero los contenedores exactos de la petición resuelve los actores con dos componentes del mismo sistema (inventario + mochila) y el crafting simple sin banco (cae al Requester).
- **Resultados y avisos bajo `System.*`**, no bajo `Action.*` (separar tags de acción de tags de sistema; `Action.*` se compara jerárquicamente en `ExcludeActions`).
- **Remanentes en el origen.** Las operaciones cuentan slots, no stacks: el destino acepta lo que puede (slot libre o completar un stack) y al origen solo se le descuenta lo procesado; el remanente nunca sale del origen. Por eso el ejecutor debe agregar primero en el destino y descontar después (el sistema viejo hacía lo inverso y rescataba con `SpawnItemInWorld`). `ReturnedItems` queda como respaldo para ítems que sí salen sin destino asegurado.
- **Acciones fuera del menú**: acción por defecto del ítem; si no tiene, MultiUse por contexto. Los PNJ siempre envían la acción explícita.
- **Funciones migradas sin cambios de lógica**, para que la prueba de regresión del menú sea comparable. Las mejoras (por ejemplo "requisito vacío = pasa" en `ValidateBasicActions`) se hacen después.
- **Nombres de las funciones migradas sin cambiar** (`ValidateActionMenu`, `ClickedHandler`…), por decisión, aunque en el diseño se había planteado `GetValidActions` / `SourceContainer`.

---

## Estado de implementación

### Implementado
- `CachedActionRules` + `ExposeActionRules` en `GI_Sunyi`, llamada desde `ReceiveInit`. Funciones viejas (`BuildActionRulesCache`, `GetActionRules`) marcadas `Deprecated`, desconectadas pero no borradas.
- Migración de namespace: `ValidateActionMenu`, `ValidateBasicActions`, `ValidateActiveAction`, `GetMultiUseActions`, `ValidateMultiUseActions`, `GetMultiUseGroups` y `ValidateAcceptedActions` viven ahora en `BP_ExecActionManager` (antes en `BPFL_ContextActions`; ver `Principios de diseño`).
- Ejecución (05/11/2026): `STR_ActionExec`, `STR_ActionResult`, `BPI_ExecActionHandler`, `BP_ExecActionManager` (variables, dispatchers `OnActionExecuted`/`OnDataWarning`, `MakeFailResult`, `FinishAction`, `ResolveAction` salvo MultiUse por contexto, `ValidateExec` con `ValidateCapabalities` y `ValidateExcludedActions`, `FindHandler` y `RequestAction`), construcción en `GI_Sunyi` con suscripción a `OnDataWarning`, y `GetExecActionManager` en `BPI_GIAccess`.
- Menú migrado al manager y `BPFL_ContextActions` borrada (probado en PIE).
- `BPI_SlotHandler`: `GetCapabalities` renombrada a `Get Capabilities` (el pin de salida conserva el nombre `Capabalities`).
- Tags `System.Action.Result.Fail.*` creados (ver `4-1`).
- `BP_Chest`: capacidades corregidas a `Container.Capability.*`.
- `BuildActionsMenu` en `WBP_ContextMenu`, conectada desde `WBP_Slot.OnPreviewMouseButtonDown` (click derecho) vía `GetOwnerActor`. Confirmada la fase de datos completa: `ValidateActionMenu` → `ValidateAcceptedActions` → `FinalActionList`.
- Resolución de labels implementada en `BuildActionsMenu`: fallback a `DefaultLabel`, selección de `ClickedHandler`/`ExternalTarget` mediante `CompareExternal?`, comparación exacta contra `ContextualLabels` y omisión de filas que requieren un externo inexistente.
- `Build Action Btn` implementada y **conectada** al `Completed` del loop de `ContextualLabels` en `BuildActionsMenu` (`Test_Print_Actions` retirado): omite Split con `Quantity <= 1`, crea `WBP_ActionButton` o `WBP_NumericButton` según `AcceptNumValues AND Quantity > 1`, aplica `Quantity - 1` únicamente a Split, agrega el widget a `Container` y enlaza su delegate de clic.
- Clics enlazados: `OnActionButtonClicked` → `HandleOptionClicked`, `OnActionNumericButtonClicked` → `HandleOptionNumeric`, ambos → `SelectClickedAction` → `RequestAction` (07/11/2026; sin respaldo al sistema viejo).
- `InventoryComponent` implementa `BPI_ExecActionHandler` (`ExecSplit`, `ExecDrop`, `ExecTransfer`; el resto → `Fail.NotImplemented`). `ExecutorReference` llenado en `DT_ActionDefinition`. Tags `System.Handler.*`, `Fail.Inventory.*`, `Fail.Equipment.*`, `Fail.NotImplemented` y `System.Warning.Data.MultiUse.*` creados.
- `RequestAction` limpia `TargetContainer` cuando la acción no tiene `CompareExternal?`. Split, Drop y Transfer probados en PIE de punta a punta.
- `ValidateActionMenu` (orquestador) **completa**, incluyendo el colapso de MultiUse: `ValidateBasicActions`, `ValidateActiveAction`, `ValidateMultiUseActions` y `GetMultiUseGroups` conectados en el flujo, gate `AllowUse` aplicado sobre la implícita, ensamblaje en `ActionListRaw` (orden implícita → multiuso agrupado/crudo → básicas) y `Return Node` devolviendo el resultado. Probada y confirmada en PIE con manzana (básica/implícita) y con yelmo/rama (multiuso).
- `GetMultiUseActions` (resuelve `STR_MultiUse.ContextRelation` → `STR_Context_Action` vía `ActionRules`) y `ValidateMultiUseActions` (filtra por `ContextTag` + `Capabilities`, con guardas de consistencia de datos y `Success` cableado a un `Found` explícito). Bugs de `HasRelations` y de ambigüedad de `Success` encontrados y corregidos (ver `Decisiones de diseño`).
- `GetMultiUseGroups` **completa**: colapsa el bucket de MultiUse en "Abrir menú" / "Acción aleatoria" cuando corresponde, recibiendo `ActionRules` como parámetro y resolviendo `Action.MultiUse.OpenMenu`/`Action.MultiUse.ExecuteRandom` con `Map_Find` sobre tags hardcodeados. Dos filas de sistema (`ActionMultiUseOpenMenu`, `ActionMultiUseExecuteRandom`) agregadas a `DT_ActionDefinition.json` para soportarla.
- `ValidateAcceptedActions` **completa**: filtro de lista negra de acciones, con coincidencia jerárquica de raíz corregida (ver `Decisiones de diseño`). Ya conectada en `BuildActionsMenu` sobre el resultado de `ValidateActionMenu`. La variable/getter `ExcludeActions`/`GetExcludeActions()` en `BPI_SlotHandler` (antes pendiente como `RejectedActions`) ya existe e implementada.
- Bug de contenido: tags de capacidad desactualizados (`Container.Capabality.*`) en el inventario del jugador, corregidos a `Container.Capability.*`.
- Bug de datos: `ActionEquip.RequiredItemProperty` corregido en `DT_ActionDefinition.json` (quitado `Item.Property.HasMultiUse`, dejando solo `Item.Property.Use.Equippable`). Confirmado con prueba de regresión en PIE (manzana / rama / casco).

### Pendiente
- ~~Auditar el resto de contenedores por tags de capacidad desactualizados~~ → hecho: el único afectado era `BP_Chest`, ya corregido. Repetir la búsqueda de `Container.Capabality` si se crean contenedores nuevos duplicando assets viejos.
- Probar las rutas de fallo de `RequestAction` (`NoRule`, `InvalidItem`, `NotAllowed`, `InvalidQuantity`, `StaleSlot`, `NoHandler`, `NotImplemented`) con una suscripción temporal a `OnActionExecuted` que imprima `ActionTag` y `ResultTag`.
- Resolver la fila `ActionUnequip` de `DT_ActionDefinition` (`ActionTag` duplicado con `Action.Container.Equip`): quitarla o darle un tag propio con `RequiredItemProperty` vacío y una capacidad `AllowUnequip`.
- Crear el tag `Container.Capability.AllowMultiUse` en `Config/Tags/Sunyi_ContainerTags.ini`. `ValidateMultiUseActions` no tiene un gate propio para él: depende de que las filas MultiUse de `DT_ActionDefinition` lo incluyan en su `RequiredTargetCapability` para que el filtro de capacidades ya existente lo aplique.
- Filtro de listas blanca/negra de *ítem* (`AllowedItemTags`/`ExcludedItemTags`), pendiente de conectar en el pre-filtro de `Transfer` contra `ExternalTarget` (distinto del filtro de acciones, ya resuelto por `ValidateAcceptedActions`).
- Bucket de acciones de Requester (capacidades del actor, ej. Inspeccionar), función independiente aún no construida; requiere el componente de capacidades del actor, que tampoco existe todavía. Cuando exista, `BuildActionsMenu` deberá combinarlo con el resultado de `ValidateActionMenu` antes (o junto a) `ValidateAcceptedActions`.
- Probar en PIE la construcción visual con stacks de cantidad 1 y mayores, y el clic de cada tipo de botón.
- ~~Bug del puente temporal: `SlotData` vacía~~ → resuelto con `CurrentRequest`.
- Borrar del menú las variables que ya no se usan (`SlotTags`, `GI Sunyi Ref`, `SlotData`) y dejar de pasarlas desde `WBP_Slot` (paso de limpieza).
- Decidir quién es el `Requester` real: hoy `WBP_Slot` pasa el dueño del contenedor clicado (al clicar un slot del cofre, el Requester es el cofre). Afecta al bucket de Requester y a las capacidades del Requester.
- Observaciones de datos en `DT_ActionDefinition` (`ExecMode` de Equip, `RequiredItemProperty` de `ActionEquipWield`, requisitos vacíos que nunca pasan `HasAnyTags`, ej. `Inspect`): ver `4-1`, "Observaciones de revisión".

**Orden de trabajo vigente (reemplaza al anterior "menú principal → submenú MultiUse → Requester → payload"):**
1. **Payload y manager** (terminado):
   - ~~Conectar `BuildActionsMenu` al manager y borrar `BPFL_ContextActions`~~ → hecho.
   - ~~Terminar `ResolveAction`, `ValidateExec`, `FindHandler` y `RequestAction`~~ → hecho (MultiUse por contexto queda pendiente).
   - ~~Verificar que los fallos de `ValidateExec` y `NoHandler` avisen con `LocalExec`~~ → verificado.
   - ~~Llenar `ExecutorReference` y crear `System.Handler.*` / `System.Warning.Data.*`~~ → hecho (falta asignar `Alert` en `ValidateMultiUseActions`).
   - ~~Reemplazar `SelectClickedAction` por `RequestAction`~~ → hecho, con el filtro de `CompareExternal?` en el manager.
   - Destino de `ReturnedItems` (almacenamiento oculto anclado al `InstanceID`): por diseñar.
2. **Acciones de contenedor** (en curso): ~~`InventoryComponent` con Split/Drop/Transfer~~ → hecho y probado. Siguen, en orden: **Inspect** (no modifica el inventario), **Use** (definir antes dónde vive cada efecto), **Equip/Unequip** en `EquipmentComponent` (resolver antes la fila `ActionUnequip`).
3. **Limpieza de todo lo DEPRECATED** (lista completa en `4-0`), incluido quitar las funciones `ActionX` de `BPI_SlotHandler`.
- Submenú real de "Multi uso", ejecución de "Acción aleatoria", bucket de Requester y reglas de contexto de MultiUse: después de los tres pasos anteriores.
- Unificación futura (opcional): si otra UI necesita otro orden de presentación, separar el ensamblado de `ValidateActionMenu`.
