# Resumen de Cambio Propuesto: `BP_UseItemManager` (100% dinámico)

Este documento resume el **cambio de enfoque** para que `BP_UseItemManager` deje de depender de hardcode y pueda manejar acciones **explícitas** e **implícitas** de forma dinámica, usando los datos ya existentes en `DT_ItemInfo` (MultiUse + ActionTags), y manteniendo el sistema modular/desacoplado.

---

## 0. Estado de implementación (lo que ya se hizo)

Esta sección existe para que, si abrimos una nueva sesión de chat, tengamos un “snapshot” real de lo que quedó implementado y qué assets se tocaron.

### 0.1 Cambios de comportamiento confirmados

- **Acción explícita gana**: si `Payload.ActionTag` llega válido, **se respeta** y no se intenta inferir nada.
- **Acción implícita NO usa MultiUse**: si `Payload.ActionTag` llega vacío, se ejecuta **la acción default del ítem** (derivada de su propiedad `Item.Property.Use.*`) incluso si el ítem tiene acciones MultiUse.
- **Los sufijos de equip (`Action.Container.Equip.*`) son intenciones explícitas**: solo aparecen cuando la acción fue seleccionada deliberadamente (MultiUse). El equip default usa el root `Action.Container.Equip` sin sufijos.

### 0.2 Funciones ajustadas / nuevas (Blueprint)

- **`BP_UseItemManager`**
  - **`DetermineActionTag` (ajustada)**:
    - Primero retorna el `ActionTag` explícito si viene en el payload.
    - Si no viene acción explícita, resuelve la acción default buscando en el `ItemInfo.PropertyTags` un tag que pertenezca a la familia `Item.Property.Use.*`.
    - Esa propiedad `Item.Property.Use.*` se mapea a una acción root `Action.Container.*` mediante una tabla externa (ver sección Data).
    - Si no se encuentra ninguna `Item.Property.Use.*`, retorna un tag inválido (no hay acción implícita posible).
  - **`GetActionWithProperty` (nueva, pura)**:
    - Función interna de `BP_UseItemManager` que consulta una DataTable de reglas para mapear `Item.Property.Use.*` → `Action.Container.*`.

- **`InventoryComponent`**
  - **`ExecActions` (ajustada)**:
    - Antes de entrar al `Switch on GameplayTag`, normaliza el action usando `GetExecutableAction`.
    - Todas las acciones que pertenezcan a la familia `Action.Container.Equip.*` se enrutan a un único case `Action.Container.Equip`.
    - Dentro del case `Equip`, se conserva disponible el `FullActionTag` para diferenciar root vs intención específica cuando sea necesario (ej. `.Wield`, `.Wear`, etc.).
  - **`GetExecutableAction` (nueva, pura)**:
    - Devuelve dos valores: `RootAction` (para el switch) y `FullActionTag` (intención completa).
    - Regla: si el tag pertenece a `Action.Container.Equip` (familia), el `RootAction` se fuerza a `Action.Container.Equip` y el `FullActionTag` conserva el tag original.

### 0.3 Data/Assets nuevos (para mantenerlo data-driven sin tocar `DT_ItemInfo`)

Se creó un asset externo para evitar editar `DT_ItemInfo` (riesgo de corrupción por dependencia en múltiples sistemas).

- **DataTable**: `SunyiProject/Content/GameDataTypes/Data/DataTables/UseItem/DT_ActionPRopertyRule.uasset`
  - Responsabilidad: reglas 1:1 de mapeo **por propiedad de uso**.
  - Uso: buscar por `Item.Property.Use.*` para obtener `Action.Container.*` (ej. Consumable → Use, Equippable → Equip).
  - Estado actual: contiene al menos las reglas para **Use** y **Equip** (y se ampliará a futuro).

- **Struct**: `SunyiProject/Content/GameDataTypes/Data/Structs/UseItem/STR_PropertyRule.uasset`
  - Responsabilidad: representar la fila/regla usada por la DataTable anterior.

### 0.4 Assets tocados (inventario de cambios)

Lista de assets que quedaron modificados o creados durante este reajuste (útil para auditoría y para retomar en otra sesión):

- **Documentación interna (metadata / contexto)**
  - `.cursor/ContextFunctions/Funcion_DetermineActionTag.txt` (actualizado)
  - `.cursor/ContextFunctions/FuncionNew_DetermineActionTag.txt` (nuevo/actualizado)
  - `.cursor/ContextFunctions/FunctionExecActions.txt` (actualizado)
  - `.cursor/ContextFunctions/Function_GetExecutableAction.txt` (nuevo/actualizado)
  - `.cursor/Contextos/*` (varios archivos ajustados para reflejar el estado real del sistema)

- **Blueprints / Systems**
  - `SunyiProject/Content/GameDataTypes/System/BP_UseItemManager.uasset` (modificado)
  - `SunyiProject/Content/InventorySystem/Blueprints/Components/InventoryComponent.uasset` (modificado)
  - `SunyiProject/Content/GameDataTypes/Data/Structs/Items/UseStructs/STR_UseItemPayload.uasset` (modificado)
  - `SunyiProject/Content/GameDataTypes/Data/Structs/Items/STR_ItemInfo.uasset` (modificado)

- **Interfaces / utilidades relacionadas al flujo de uso**
  - `SunyiProject/Content/GameDataTypes/Blueprints/Interfaces/BPI_UseItemHandler.uasset` (modificado)
  - `SunyiProject/Content/GameDataTypes/Blueprints/Interfaces/BPI_SlotHandler.uasset` (modificado)
  - `SunyiProject/Content/GameDataTypes/Blueprints/Libraries/BPFL_Utilities.uasset` (modificado)

- **Data / Context Menu (relacionado a acciones visibles)**
  - `SunyiProject/Content/GameDataTypes/Data/DataTables/ContextMenu/DT_ActionList.uasset` (nuevo)
  - `SunyiProject/Content/GameDataTypes/Data/DataTables/ContextMenu/DT_ActionNames.uasset` (modificado)

- **Data para mapeo de propiedad → acción (UseItem)**
  - `SunyiProject/Content/GameDataTypes/Data/DataTables/UseItem/DT_ActionPRopertyRule.uasset` (nuevo)
  - `SunyiProject/Content/GameDataTypes/Data/Structs/UseItem/STR_PropertyRule.uasset` (nuevo)

- **Tags (config)**
  - `SunyiProject/Config/DefaultGameplayTags.ini` (modificado)
  - `SunyiProject/Config/Tags/Sunyi_ActionTags.ini` (modificado)
  - `SunyiProject/Config/Tags/Sunyi_ItemTags.ini` (modificado)
  - `SunyiProject/Config/Tags/Sunyi_ContainerTags.ini` (modificado)
  - `SunyiProject/Config/Tags/Sunyi_CharacterTags.ini` (modificado)
  - `SunyiProject/Config/Tags/Sunyi_EquipmentTags.ini` (nuevo/modificado, según rama)

- **EquipmentSystem (trabajo en paralelo que acompaña el flujo Equip)**
  - `SunyiProject/Content/EquipmentSystem/Blueprints/Components/EquipmentComponent.uasset` (nuevo)
  - `SunyiProject/Content/EquipmentSystem/Blueprints/Libraries/BPFL_GeneralEquipment.uasset` (nuevo)
  - `SunyiProject/Content/EquipmentSystem/Data/DataTables/DT_CreatureEquipment.uasset` (nuevo)
  - `SunyiProject/Content/EquipmentSystem/Data/Structs/STR_CreatureEquipment.uasset` (nuevo)
  - `SunyiProject/Content/EquipmentSystem/Data/Structs/STR_EquipmentData.uasset` (nuevo)
  - `SunyiProject/Content/EquipmentSystem/Data/Structs/STR_EquipmentSlotData.uasset` (nuevo)
  - `SunyiProject/Content/GameDataTypes/Data/Structs/Items/DefinitionStructs/SupportStructs/STR_IntentionActions.uasset` (nuevo/modificado)
  - `SunyiProject/Content/GameDataTypes/Data/Structs/Items/DefinitionStructs/STR_EquipmentBehavior.uasset` (modificado)

- **UI / mapas / actores (derivados de pruebas e integración)**
  - `SunyiProject/Content/UI/ContextMenu/WBP_ContextMenu.uasset` (modificado)
  - `SunyiProject/Content/UI/Inventory/WBP_Inventory.uasset` (modificado)
  - `SunyiProject/Content/UI/Equipment/WBP_HumanEquipment.uasset` (nuevo)
  - `SunyiProject/Content/ThirdPerson/Blueprints/BP_ThirdPersonCharacter.uasset` (modificado)
  - `SunyiProject/Content/ThirdPerson/Blueprints/BP_HumanNPC.uasset` (nuevo)
  - `SunyiProject/Content/ThirdPerson/Maps/ThirdPersonMap.umap` y `__ExternalActors__/*` (modificado)

### 0.5 Decisión pendiente ya tomada (para la siguiente fase)

- **`Unequip` será una acción explícita nueva** (no se implementará como `Transfer`), para mantener responsabilidades separadas y evitar el acoplamiento implícito de “transferir” con “reemplazar/ocupar slots”.

---

## 1. Estado actual (lo que existe hoy)

- `GI_Sunyi` crea el singleton `BP_UseItemManager` en `Init` y hace bind a `OnUseRequested`.
- `BP_UseItemManager.HandleUseRequest(Payload)`:
  - llama `DetermineActionTag`
  - actualiza `Payload.ActionTag`
  - llama `UseItem(Payload)` en el `SourceHandler` (por interfaz).
- Problema: `DetermineActionTag` y/o parte del flujo está **hardcodeado** (switch / mapeos fijos).

---

## 2. Objetivo del cambio

- Eliminar hardcode para determinar y ejecutar la acción.
- Soportar ítems con **múltiples usos** (banana, escoba, roca, olla, cuerda, espada…) sin explotar en cantidad de acciones por slot.
- Mantener el principio:
  - **Acción explícita gana** (si el usuario seleccionó un botón específico).
  - **Acción implícita** ocurre cuando el usuario dispara un “uso estándar” (doble clic, hotbar, drop sobre equipo, etc.).
  - Si el ítem no tiene acción estándar, **no hace nada** (solo ofrece MultiUse cuando corresponda).

---

## 3. Definiciones (importantes)

### 3.1 Acción explícita
Se da cuando el flujo ya trae un `ActionTag` concreto:
- Click en botón de menú contextual
- Click en menú radial
- (a futuro) Drag & drop “explícito” a un slot objetivo

**Regla:** si `Payload.ActionTag` es válido → **no se determina** nada, se ejecuta esa acción.

### 3.2 Acción implícita
Se da cuando el `Payload.ActionTag` llega vacío y el sistema debe inferir la intención:
- Doble clic sobre un slot
- Hotbar / trigger de input
- Drag & drop “implícito” (ej. soltar sobre equipo sin seleccionar “equipar como…”)

**Regla:** si no hay `ActionTag`, el manager debe resolver la acción “estándar” del ítem según datos.

---

## 4. Implementación actual: `DetermineActionTag` (explícito primero, implícito por propiedad)

> Nota: esta sección describe el comportamiento que quedó implementado. El enfoque de “implícito por MultiUse” fue descartado a propósito para respetar la regla: **una acción implícita siempre ejecuta el default del ítem**, independientemente de MultiUse.

### 4.1 Prioridad 1 — Acción explícita

- Si `Payload.ActionTag` es válido → devolverlo y terminar.

### 4.2 Prioridad 2 — Acción implícita por `Item.Property.Use.*`

Cuando `Payload.ActionTag` llega vacío, el manager:

- Obtiene el `ItemInfo` del ítem usando el `ItemID` del slot.
- Busca dentro de `ItemInfo.PropertyTags` un tag que pertenezca a la familia `Item.Property.Use.*`.
- Con ese tag de propiedad, consulta una regla externa (DataTable) para obtener el **root de acción** `Action.Container.*` correspondiente.
- Devuelve esa acción root como acción default (implícita).

**Restricción intencional (arquitectura)**:

- Un ítem puede tener muchas propiedades, pero para “uso implícito” **solo debe existir 1** propiedad `Item.Property.Use.*` (la default).  
  Si un ítem admite otras acciones (ej. baguette que también puede equiparse), esas rutas viven como acciones **explícitas** vía MultiUse.

**Motivo de la DataTable externa**:

- `DT_ItemInfo` se considera “congelada” (evitar corrupción/roturas por dependencias).  
  Por eso el vínculo propiedad→acción se saca a un asset dedicado: `DT_ActionPRopertyRule`.

---

## 5. Propuesta: ejecución dinámica por “root de acción”

En vez de manejar cada acción como caso aislado, la ejecución se organiza por **familias**:

- `Action.Container.Equip.*` → “flujo de equipamiento”
  - Ej.: Wield/Wear/Ammo/Throw son sub‑intenciones; el root es Equip.
  - El “complemento” (`.Wield`, `.Wear`, etc.) se usa para buscar bindings/slots candidatos cuando aplique.
- `Action.Container.Use` → consumir/usar (si el ítem lo soporta)
- `Action.Container.Split` → split stack (si aplica)
- `Action.Container.Drop` → drop
- `Action.Container.Transfer` → transfer
- `Action.Container.Inspect` → inspect

**Idea:** el manager puede mantener una ruta genérica:
1) determinar ActionTag final  
2) delegar a un “handler” por root de ActionTag (sin hardcode por cada variante)

---

## 6. Interacción con Equipment (intenciones + slots candidatos)

- Las acciones `Action.Container.Equip.*` no necesitan 20 acciones por slot.
- El slot final se resuelve por datos del ítem cuando la intención lo requiere:
  - `STR_EquipmentBehavior.ActionBindings` (Array<STR_IntentionActions>)
  - `STR_IntentionActions`: `ActionTag` + `SlotCandidates` ordenado (prioridad)

Esto permite:
- roca: Wield → FreeHands, Ammo/Throw → Ammo/Throw slots, etc.
- yelmo: Wear → Head, Wield → FreeHands (si aplica)

---

## 7. Qué cambia y qué NO cambia

### Cambia
- `DetermineActionTag` deja de ser switch hardcode:
  - prioriza acción explícita si existe en el payload
  - si es implícita, resuelve el default por `Item.Property.Use.*` + `DT_ActionPRopertyRule`
- `InventoryComponent.ExecActions` enruta por familias, normalizando `Action.Container.Equip.*` hacia `Action.Container.Equip` sin perder el tag completo (intención).
- Se agrega un asset externo (DataTable) para el mapeo propiedad→acción, evitando tocar `DT_ItemInfo`.

### No cambia (por ahora)
- El patrón de orquestación: `GI_Sunyi` → `OnUseRequested` → `HandleUseRequest` → `SourceHandler.UseItem(Payload)` se puede conservar si conviene.

---

## 8. Riesgos / puntos a decidir antes de implementar

- ¿Qué ítems deben tener “acción estándar”? (si un recurso no la tiene, no debe disparar nada en doble click/hotbar).
- ¿Qué contexts mínimos usamos para MultiUse? (ej. inventario primario, hotbar, equipado…) **solo para acciones explícitas**.
- Política de “multi action explícita”: OpenMenu vs ExecuteRandom (la ejecución random aún no está implementada).
- Dónde vive la transacción Inventario ↔ Equipo (idealmente el orquestador o el SourceHandler, sin pérdida de ítems).

