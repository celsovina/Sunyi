# Progreso de Implementación: `EquipmentComponent` (Data-Only + Preparación de Orquestación)

Este documento resume **lo que ya está implementado**, **qué decisiones de diseño tomamos**, y **qué falta** para iniciar el orquestador (Inventario ↔ Equipo ↔ MultiUse) sin perder contexto.

---

## 1. Objetivo actual (etapa “Pre”)

- Construir un `EquipmentComponent` **modular**, **desacoplado** y **atómico** para equipar/desequipar ítems **solo con datos** (sin UI y sin visuales todavía).
- Mantener el principio de **no pérdida de ítems**: el reemplazo devuelve `ReplacedItem` para que el orquestador lo regrese al inventario.
- Preparar el sistema para ítems con **múltiples usos** (MultiUse) donde el slot puede depender de la **intención** (acción), sin crear 20 acciones por slot.

---

## 2. Convenciones y Tags (estado actual)

### 2.1 Slots de equipo
- **Prefijo**: `Equipment.*` (slots “puros”, sin tipo de criatura).
- Archivo: `Config/Tags/Sunyi_EquipmentTags.ini`.
- Incluye (entre otros): `Equipment.Head`, `Equipment.FreeHandLeft`, `Equipment.FreeHandRight`, `Equipment.Ammo`, `Equipment.Bow`, `Equipment.Backpack`, `Equipment.Mount`.
- Decisión clave: se cambió de “Weapon/Shield” a **FreeHandLeft/FreeHandRight** para no asumir diestro/zurdo.

### 2.2 Tipos de criatura
- **Prefijo**: `Character.*`.
- Archivo: `Config/Tags/Sunyi_CharacterTags.ini`.
- Ejemplos actuales: `Character.Human.Player`, `Character.Human.NPC`.

### 2.3 Acciones (intención)
- Archivo: `Config/Tags/Sunyi_ActionTags.ini`.
- Root existente: `Action.Container.Equip`.
- Nuevas acciones de intención (ya registradas):  
  - `Action.Container.Equip.Wield`  
  - `Action.Container.Equip.Wear`  
  - `Action.Container.Equip.Ammo`  
  - `Action.Container.Equip.Throw`

Acciones de soporte MultiUse (ya registradas):
- `Action.MultiUse.OpenMenu`
- `Action.MultiUse.ExecuteRandom`

---

## 3. Cambios de datos en ItemInfo (centralizado en `DT_ItemInfo`)

### 3.1 Estructura nueva
- **`STR_IntentionActions`**
  - `ActionTag` (GameplayTag)
  - `SlotCandidates` (Array\<GameplayTag\>) **ordenado** (prioridad, ej. Right→Left)

### 3.2 `STR_EquipmentBehavior` (se amplió)
Campos existentes:
- `GrantedInventorySlots`
- `StorageRestrictionTags`

Campos añadidos:
- `EquippableSlots` (GameplayTagContainer): **superset** de slots donde este ítem puede equiparse.
- `AllowedCreature` (GameplayTagContainer): roots `Character.*` permitidos.
- `ActionBindings` (Array\<STR_IntentionActions\>): **intención → slots candidatos** (resuelve “modo” sin hardcodear en lógica).

**Documento actualizado:** `\.cursor/Contextos/2 - Estructuras de ItemInfo.md`.

### 3.3 Estado actual de datos de prueba
- Ítems configurados para test: **rama** y **yelmo**.
- Ambos tienen binding en `ActionBindings` para `Action.Container.Equip.Wield` con `SlotCandidates`:
  - `Equipment.FreeHandRight`
  - `Equipment.FreeHandLeft`

---

## 4. Perfil de criatura (DataTable `DT_CreatureEquipment`)

### 4.1 Propósito
Resolver “qué criatura es” + “qué configuración de equipo aplica” **desde datos oficiales**.

### 4.2 Matching por root (multi-fila a futuro)
- Se valida por texto: `ActorTagName == RootName` **o** `ActorTagName startsWith RootName + "."`.
- Cuando hay múltiples matches, se elige el **más específico** (mayor profundidad por puntos `.` / “last index”).

### 4.3 Slots y nombres de UI por criatura
Se añadió un array de struct para evitar desajustes:
- **`STR_EquipmentSlotData`**
  - `SlotTag` (GameplayTag)
  - `DisplayName` (Text)

En el componente se usa como:
- **`EquipmentSlotValidations`** (Array\<STR_EquipmentSlotData\>) = lista ordenada de slots de la criatura + etiquetas para UI.

---

## 5. EquipmentComponent — variables y rol (data-only)

### 5.1 Variables relevantes
- `EquipmentList` (Array\<STR_EquipmentData\> o equivalente): storage principal de “equipo puesto”.
  - Cada elemento contiene al menos: `EquipmentType/SlotTag` + `ItemData (STR_Slot)`.
  - Se hace `Resize` por `EquipmentSize` (derivado del perfil de criatura).
- `SpawnedActors`, `SpawnedSKMeshes`: existen pero **no se usan aún** (quedan para etapa de visuales).
- `CreatureTag` (GameplayTag): se setea desde `DT_CreatureEquipment`.
- `EquipmentSlotValidations` (Array\<STR_EquipmentSlotData\>): slots permitidos y ordenados para la criatura.

### 5.2 Convención de “slot vacío”
Se considera vacío si:
- SlotTag inválido **o**
- `ItemID == None` **o** `Quantity == 0`

Esta convención se usa de forma consistente en:
- `SlotOccupied`
- `FindEmptySlot`
- `GetAllEquippedItems`

---

## 6. Funciones implementadas (resumen funcional)

### 6.1 Consultas / helpers
- **`GetSlotData(SlotTag)`**: busca en `EquipmentList` y devuelve `STR_Slot` + Success.
- **`SlotOccupied(SlotTag)`**: usa `GetSlotData` y devuelve bool (ocupado si ItemID válido y Quantity > 0).
- **`GetAllEquippedItems()`**: devuelve una lista “densa” sin huecos (snapshot de lectura).
- **`FindEmptySlot()`**: devuelve el primer índice vacío (Found + Index / -1).

### 6.2 Escritura segura (sin pérdida)
- **`AddToEquipment(SlotTag, ItemData)`**
  - Si el SlotTag ya existe: guarda el ítem previo como `ReplacedItem`, limpia y escribe el nuevo.
  - Si no existe: usa `FindEmptySlot` y escribe en ese índice.
  - Devuelve: Success + `ReplacedItem` (para devolver a inventario).
- **`RemoveFromEquipment(SlotTag)`**
  - Busca, guarda el ítem removido y limpia el índice.
  - Devuelve: Success + `RemovedItem`.

### 6.3 Validación (usando `DT_ItemInfo` → EquipmentBehavior)
- **`FindValidSlot(ItemToEquip: STR_Slot)`**
  - Obtiene ItemInfo con ItemManager.
  - Verifica `EquippableSlots` no vacío.
  - Verifica criatura: `CreatureTag` ∈ `AllowedCreature` (si `AllowedCreature` no está vacío).
  - Recorre `EquipmentSlotValidations` (orden del perfil) y filtra solo los SlotTag presentes en `EquippableSlots`.
  - Devuelve array ordenado de slots válidos (en tu implementación puede devolver tags o structs; lo importante es conservar orden).
- **`CanEquipItem(ItemToEquip, SlotTag)`**
  - Verifica: ItemInfo válido, `EquippableSlots` contiene SlotTag, criatura permitida y SlotTag existe en `EquipmentSlotValidations`.

### 6.4 API pública data-only
- **`EquipItem(ItemToEquip)`**
  - Llama `FindValidSlot`.
  - Lógica de lateralidad: prueba slots en orden y usa el **primer slot libre**; si todos ocupados, usa el primero (reemplazo).
  - Llama `AddToEquipment` y retorna `Success`, `EquippedSlot`, `ReplacedItem`.
- **`UnequipItem(SlotTag)`**
  - Verifica con `GetSlotData`.
  - Llama `RemoveFromEquipment` y retorna `Success` + `UnequipItem`.

---

## 7. Decisiones clave (para no perder coherencia)

- **Acciones vienen de MultiUse (no de ActionBindings)**: el set de acciones (qué botones aparecen) se obtiene desde `STR_MultiUse.ContextRelation → GrantedActions` (filtradas por `SourceContext`).  
  `ActionBindings` **no genera acciones**; solo ayuda a ejecutar equipamiento (resolver candidatos de slot) cuando la acción es `Action.Container.Equip.*`.
- **No hardcodear slots en el orquestador**: el orquestador pasa la intención (`ActionTag`) y el ítem resuelve candidatos de slot vía `ActionBindings` (solo cuando aplica a equipamiento).
- **No 20 acciones nuevas por slot**: se usan **acciones de intención** `Action.Container.Equip.*` + `ActionBindings` del ítem.
- **Reemplazo seguro**: nunca “pierdes” el ítem equipado anterior; `AddToEquipment` lo devuelve como `ReplacedItem`.
- **Desequipar y espacio en inventario**: política actual acordada:
  - El `EquipmentComponent` solo quita y devuelve.
  - El orquestador valida inventario antes de llamar a desequipar (si no hay espacio, no desequipa).

---

## 7b. Menú contextual: avance necesario para Equipment

Para habilitar acciones `Action.Container.Equip.*` sin hardcode, se hizo/ajustó lo siguiente:

- **ActionList ahora es data-driven**:
  - Se creó un cache en `GI_Sunyi` que carga `DT_ActionList` al iniciar (singleton).
  - `GetValidActions` usa el **array cacheado** (en lugar del array hardcode).
  - (Opcional en progreso) cache adicional como **Map** para consultas directas de reglas por ActionTag.

- **Fix crítico (HasMultiUse)**:
  - Las acciones concedidas por MultiUse se agregan al inicio desde `GrantedActions`.
  - En la fase de “acciones generales” (loop del ActionList global), se **omite** cualquier regla cuya `ItemProperty == Item.Property.HasMultiUse` para evitar que se cuelen `Wear/Ammo/Throw` cuando el ítem solo concede `Wield`.
  - Resultado: el menú solo muestra las acciones realmente concedidas por MultiUse + acciones generales (Drop/Split/Inspect/Transfer/Use) según reglas.

---

## 8. Qué falta (próxima etapa)

### 8.1 Orquestador (Inventario ↔ Equipo)
Implementar el flujo transaccional:
- Equipar: remover del inventario → `EquipItem` → si `ReplacedItem` válido, devolver al inventario.
- Desequipar: validar “inventario acepta” → `UnequipItem` → añadir al inventario.

### 8.2 Equip explícito (Drag & Drop a slot gráfico)
Añadir una API explícita (si aún no existe):
- `EquipItemToSlot(ItemToEquip, TargetSlotTag)` (usa `CanEquipItem` + `AddToEquipment`)

### 8.3 UI + Visuales (posterior)
- UI: usar `EquipmentSlotValidations.DisplayName` + estado desde `EquipmentList`.
- Visuales: poblar `SpawnedActors` / `SpawnedSKMeshes` en Apply/ClearVisuals.

