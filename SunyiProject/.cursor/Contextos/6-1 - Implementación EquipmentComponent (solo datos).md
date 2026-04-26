# Implementación EquipmentComponent — Solo datos

Lista de lo necesario para ejecutar equipar/desequipar **solo a nivel de datos** (sin widgets ni spawn de visuales). Orden recomendado para no bloquear pasos posteriores.

---

## 1. Lo que hace falta tener antes de codear

### 1.1 Tags en el proyecto (convención acordada)

- **Slots de equipo:** prefijo **`Equipment.<Posición>`** — sin distinguir tipo de criatura en el tag. Un solo set de tags reutilizable. Ejemplos: `Equipment.Head`, `Equipment.Torso`, `Equipment.ChestPlate`, `Equipment.Mount`, `Equipment.MainHand`, `Equipment.RingLeft`, `Equipment.Backpack`, etc. Archivo: `Config/Tags/Sunyi_EquipmentTags.ini` (o el .ini de slots que hayas creado).
- **Tipo de criatura:** se usa el **tag source “characters”** ya existente. Nomenclatura centralizada en `Sunyi_CharacterTags.ini`: `Character.Human.Player`, `Character.Human.NPC`. Más adelante se añaden otras criaturas con la misma jerarquía (ej. `Character.Horse.*`, `Character.Companion.*`). El componente guarda en `CreatureType` uno de estos tags; qué slots tiene cada criatura se define en `AllowedSlotTags` del componente (o en un perfil), no en el nombre del tag de slot.

Con esto los slots quedan puros y limpios; la compatibilidad criatura–slot es por datos (ítem dice EquippableSlots + AllowedCreatureTypes, componente dice CreatureType + AllowedSlotTags).

### 1.2 Datos del ítem: ampliar STR_EquipmentBehavior

Hoy `STR_EquipmentBehavior` solo tiene `GrantedInventorySlots` y `StorageRestrictionTags`. Para equipar por datos hace falta que el ítem declare:

- **En qué slot(s) puede ir:** por ejemplo `EquippableSlots` (`FGameplayTagContainer` o `TArray<FGameplayTag>`). Ej.: casco → Head, anillo → RingLeft + RingRight.
- **En qué criaturas se puede equipar:** por ejemplo `AllowedCreatureTypes` (`FGameplayTagContainer`). Ej.: casco → Human, armadura de lomo → Horse.

Eso se usa en **FindValidSlotForItem** y **CanEquipItemInSlot**: si el ítem no tiene criatura permitida o ningún slot, no se equipa (y se puede delegar a multi‑uso o nada).

Toca **añadir esos dos campos** a la estructura que alimenta la DataTable de ítems (y rellenarlos en los ítems equipables). Si la struct está en C++ habrá que recompilar; si es solo Blueprint/estructura de proyecto, se edita en el editor.

### 1.3 Configuración por criatura en el componente

Cada actor con `EquipmentComponent` debe saber:

- **Qué “tipo de criatura” es:** variable en el componente, ej. `CreatureType` (`FGameplayTag`), usando el tag source *characters* con nomenclatura centralizada (ej. `Character.Human.Player`, `Character.Human.NPC`). Asignada al crear el actor o desde un perfil (DataTable más adelante).
- **Qué slots tiene esa criatura:** variable ej. `AllowedSlotTags` (`TArray<FGameplayTag>` o `FGameplayTagContainer`). Ej. humano = [Head, MaskUpper, MaskLower, …], caballo = [Saddle, BackArmor, …]. Tamaño de ese array = número máximo de slots (20 humano, 6 caballo, etc.).
- **Array de equipo vacío:** `EquipmentList` = array de struct `(SlotTag: GameplayTag, ItemData: STR_Slot)`, con **tamaño = longitud de AllowedSlotTags** (o un `SlotCount` si lo prefieres). Al inicializar el componente, se crea el array de ese tamaño y todas las entradas se dejan “vacías” (tag inválido o vacío y ItemData vacío). No hace falta hardcodear qué tag va en cada índice; el tag se escribe cuando se equipa, según lo que diga el ítem.

La fuente de `AllowedSlotTags` (y opcionalmente `CreatureType`) puede ser después una DataTable o un asset de “perfil de equipo”; de momento puede ser un array configurado por defecto en el Blueprint del personaje (humano vs caballo, etc.).

### 1.4 Acceso a ItemInfo

Las funciones de validación (FindValidSlotForItem, CanEquipItemInSlot) necesitan leer `STR_ItemInfo` del ítem (al menos EquipmentBehavior con los nuevos campos). Eso implica tener **acceso al ItemManager** (o al sistema que devuelve ItemInfo por ItemID) desde el componente. Si ya usas `BPI_SystemAccess` / GameInstance para eso en inventario, el EquipmentComponent debe poder obtener ese acceso igual (mismo patrón desacoplado por interfaz).

---

## 2. Variables del EquipmentComponent (solo datos)

- **CreatureType** (`FGameplayTag`): tipo de criatura de este actor.
- **AllowedSlotTags** (`TArray<FGameplayTag>` o `FGameplayTagContainer`): lista de slots que esta criatura puede tener (define también el “tamaño” del equipo).
- **EquipmentList** (`Array` de struct): cada elemento = `SlotTag` (GameplayTag) + `ItemData` (STR_Slot). Longitud = AllowedSlotTags.Num() (o SlotCount). Inicializado vacío.
- **SpawnedActors / SpawnedSKMeshes:** se dejan como están; en esta fase no se usan para lógica, solo se tocarán cuando implementes visuales.

Representación de “slot vacío”: SlotTag sin valor o inválido y/o ItemData sin ItemID (o cantidad 0), según cómo definas “vacío” en tu struct.

---

## 3. Funciones a implementar (solo datos)

### 3.1 Inicialización

- **InitializeEquipment** (o en BeginPlay): con `AllowedSlotTags` ya asignado, crear o redimensionar `EquipmentList` al tamaño deseado y rellenar todas las posiciones como “vacías”. Si el tamaño lo tomas de un perfil/DataTable, aquí se aplicaría.

### 3.2 Consultas (puras, solo lectura)

- **GetSlotData(SlotTag):** recorre `EquipmentList`; si encuentra un elemento cuyo `SlotTag` coincide, devuelve ese `ItemData`; si no, devuelve “vacío” o fallo.
- **IsSlotOccupied(SlotTag):** usando GetSlotData (o el mismo bucle), devuelve true si hay ítem válido en ese slot.
- **GetEquippedItemInSlot(SlotTag):** mismo resultado que GetSlotData; puede ser alias o llamada interna.
- **GetAllEquippedItems:** recorre `EquipmentList` y devuelve un array con los `ItemData` de los elementos no vacíos.

### 3.3 Validación (puras, sin modificar estado)

- **FindValidSlotForItem(ItemToEquip):**
  - Obtener ItemInfo del ítem (ItemID).
  - Si no tiene EquipmentBehavior con EquippableSlots / AllowedCreatureTypes, devolver array vacío (no equipable; el llamador puede intentar multi‑uso).
  - Comprobar que `CreatureType` del componente esté en `AllowedCreatureTypes` del ítem. Si no, array vacío.
  - De los slots en `EquippableSlots` del ítem, quedarse solo con los que estén en `AllowedSlotTags` del componente. Esos son los candidatos válidos. Devolver ese array (orden: por ejemplo el mismo que en EquippableSlots, o derecho antes que izquierdo en pares).
- **CanEquipItemInSlot(ItemToEquip, SlotTag):** devuelve true si el ítem incluye ese slot en EquippableSlots, el componente tiene ese slot en AllowedSlotTags y el CreatureType del componente está en AllowedCreatureTypes del ítem.

### 3.4 Comandos (modifican EquipmentList; no tocan Spawned*)

- **AddDataToEquipmentList(SlotTag, ItemData):**
  - Recorrer `EquipmentList` buscando un elemento con ese `SlotTag`.
  - Si existe: sobrescribir solo `ItemData` en ese índice (mismo slot, ítem nuevo).
  - Si no existe: buscar el primer elemento “vacío” (SlotTag vacío/inválido) y escribir ahí `SlotTag` + `ItemData`.
  - Si no hay hueco (todo ocupado), no añadir (o definir política: ej. sobrescribir el primero del mismo “tipo” si aplica; por ahora lo simple es fallar si no hay hueco).
- **RemoveDataFromEquipmentList(SlotTag):**
  - Buscar en `EquipmentList` el elemento con ese `SlotTag`, “vaciar” ese elemento (quitar tag y ItemData o marcar como vacío). No tocar SpawnedActors/SpawnedSKMeshes en esta fase.

### 3.5 API pública (solo datos)

- **EquipItem(ItemToEquip — STR_Slot):**
  - Llamar FindValidSlotForItem(ItemToEquip). Si el array está vacío, devolver false (y opcionalmente disparar lógica de “no equipable, usar multi‑uso o nada”).
  - Elegir un slot de ese array: por ejemplo el primero. (Más adelante: si está ocupado, probar el siguiente; si todos ocupados, sobrescribir el primero, etc.)
  - Llamar AddDataToEquipmentList(slot elegido, ItemToEquip).
  - Devolver true y el slot usado. **No** llamar a ApplyVisuals.
- **UnequipItemFromSlot(SlotToUnequip — FGameplayTag):**
  - Con GetSlotData(SlotToUnequip) comprobar si hay ítem. Si no, devolver false.
  - Guardar ese ItemData para devolverlo.
  - Llamar RemoveDataFromEquipmentList(SlotToUnequip).
  - Devolver true y el ítem desequipado. **No** llamar a ClearVisuals.

---

## 4. Orden sugerido de implementación

1. Tags (slots + creature types) y ampliación de **STR_EquipmentBehavior** (EquippableSlots, AllowedCreatureTypes).  
2. Variables del componente (CreatureType, AllowedSlotTags, EquipmentList con struct SlotTag + ItemData) e **InitializeEquipment**.  
3. **GetSlotData**, **IsSlotOccupied**, **GetEquippedItemInSlot**, **GetAllEquippedItems**.  
4. **FindValidSlotForItem** y **CanEquipItemInSlot** (asegurando acceso a ItemInfo).  
5. **AddDataToEquipmentList** y **RemoveDataFromEquipmentList**.  
6. **EquipItem** y **UnequipItemFromSlot** usando solo lo anterior, sin visuales.

Con esto la ejecución del equipamiento a nivel de datos queda completa.

---

## 6. Construcción de funciones auxiliares (orden y lógica)

Construir primero estas funciones; el resto (FindValidSlotForItem, CanEquipItemInSlot, EquipItem, UnequipItemFromSlot) las usan.

### 6.1 GetSlotData
- **Entradas:** SlotTag (GameplayTag).
- **Salidas:** ItemData (STR_Slot), Found? (bool).
- **Lógica:** Recorrer el array EquipmentList. Por cada elemento, comparar su SlotTag (el del struct) con el SlotTag de entrada. Si coinciden, devolver ese ItemData y Found = true; si terminas el loop sin encontrar, devolver un STR_Slot vacío (o por defecto) y Found = false. No modificar nada.

### 6.2 IsSlotOccupied
- **Entradas:** SlotTag (GameplayTag).
- **Salidas:** Occupied? (bool).
- **Lógica:** Llamar GetSlotData(SlotTag). Si Found es true, comprobar si el ItemData devuelto es “válido” (por ejemplo ItemID no vacío o cantidad mayor que cero, según cómo definas “vacío”). Devolver true si hay ítem válido, false en caso contrario. Pura.

### 6.3 GetEquippedItemInSlot
- **Entradas:** SlotTag (GameplayTag).
- **Salidas:** ItemData (STR_Slot), Found? (bool).
- **Lógica:** Llamar GetSlotData(SlotTag) y devolver exactamente sus salidas. Es un alias para la API pública.

### 6.4 GetAllEquippedItems
- **Entradas:** Ninguna.
- **Salidas:** Items (Array de STR_Slot).
- **Lógica:** Crear un array vacío. Recorrer EquipmentList; por cada elemento, si el slot no está “vacío” (SlotTag válido y ItemData con ítem válido), añadir ese ItemData al array. Devolver el array.

### 6.5 FindEmptySlot (pura)
- **Entradas:** Ninguna (lee EquipmentList del componente).
- **Salidas:** Index (int) — índice del primer elemento “vacío”, o un valor inválido (ej. -1) si no hay hueco.
- **Lógica:** Recorrer EquipmentList desde el índice 0. Por cada elemento, comprobar si está “vacío” (SlotTag inválido o vacío, según la misma convención que en SlotOccupied/GetAllEquippedItems). Devolver el primer índice que cumpla; si ninguno, devolver -1 (o el valor que uses como “no encontrado”). No modificar nada. Idealmente marcada como **Pure** en Blueprint.

### 6.6 AddDataToEquipmentList (AddToEquipment)
- **Entradas:** SlotTag (GameplayTag), ItemData (STR_Slot).
- **Salidas:** Success? (bool), ReplacedItem (STR_Slot) — el ítem que estaba en el slot (vacío si no había nada); el llamador lo devuelve al inventario.
- **Lógica:** (1) Recorrer EquipmentList buscando un elemento cuyo SlotTag coincida con el de entrada. **(2a) Si existe:** guardar su ItemData en ReplacedItem; vaciar ese elemento (clear); en ese mismo índice escribir SlotTag e ItemData; Success = true; devolver. **(2b) Si no existe:** llamar FindEmptySlot; si devuelve un índice válido (≥ 0), escribir en ese índice SlotTag e ItemData; ReplacedItem = vacío; Success = true. Si FindEmptySlot devuelve -1, Success = false; ReplacedItem = vacío. No tocar SpawnedActors ni SpawnedSKMeshes.

### 6.7 RemoveDataFromEquipmentList
- **Entradas:** SlotTag (GameplayTag).
- **Salidas:** Success? (bool), RemovedItemData (STR_Slot) — opcional, para devolver lo que había.
- **Lógica:** Recorrer EquipmentList buscando el elemento cuyo SlotTag coincida. Si no encuentras, Success = false y opcionalmente RemovedItemData vacío. Si encuentras: guardar el ItemData de ese elemento en RemovedItemData (para devolverlo); vaciar ese elemento (quitar o resetear SlotTag e ItemData según tu convención de “vacío”); Success = true. No tocar visuales.

---

## 7. Estructura de FindValidSlot y CanEquipItem

### 7.1 FindValidSlot

**Entradas:** ItemToEquip (STR_Slot).  
**Salidas:** ValidSlots (Array de FGameplayTag). Vacío si el ítem no es equipable en esta criatura.

**Pasos:**

1. Obtener **ItemInfo** usando **ItemToEquip.ItemID** (ItemManager o equivalente). Si no hay ItemInfo válido, devolver array vacío.
2. Acceder al módulo **EquipmentBehavior** del ItemInfo (EquippableSlots, AllowedCreature). Si no existe o EquippableSlots está vacío, devolver array vacío.
3. Comprobar **CreatureType del componente** contra **AllowedCreature del ítem**: el CreatureType debe estar contenido en AllowedCreature (comparación de GameplayTag en container). Si no está, devolver array vacío.
4. Construir la lista de candidatos: de cada tag en **EquippableSlots** del ítem, comprobar si está en **AllowedSlotTags** del componente. Añadir a un array solo los que coincidan. Mantener el orden (ej. FreeHandRight antes que FreeHandLeft para lateralidad).
5. Devolver ese array (ValidSlots). Si no quedó ninguno, array vacío.

---

### 7.2 CanEquipItem

**Entradas:** ItemToEquip (STR_Slot), SlotTag (FGameplayTag).  
**Salidas:** CanEquip? (bool).

**Pasos:**

1. Obtener **ItemInfo** usando **ItemToEquip.ItemID**. Si no hay ItemInfo válido, devolver false.
2. Acceder a **EquipmentBehavior**. Si no existe o EquippableSlots vacío, devolver false.
3. Comprobar que **CreatureType** del componente esté en **AllowedCreature** del ítem. Si no, devolver false.
4. Comprobar que **SlotTag** (entrada) esté en **EquippableSlots** del ítem. Si no, devolver false.
5. Comprobar que **SlotTag** esté en **AllowedSlotTags** del componente. Si no, devolver false.
6. Si todas las comprobaciones pasan, devolver true (en tu caso: recorrer Equipment Slot Validations y comparar SlotTag; si coincide, true; si el loop termina sin coincidencia, false).

---

## 8. Contenido de EquipItem y UnequipItem

Ambas son funciones del **EquipmentComponent** (API pública). Solo datos; no llaman a ApplyVisuals ni ClearVisuals.

---

### 8.1 EquipItem

**Entradas:** ItemToEquip (STR_Slot).  
**Salidas:** Success? (bool), EquippedSlot (FGameplayTag), ReplacedItem (STR_Slot) — opcional; si hubo reemplazo, el gestor lo devuelve al inventario.

**Pasos:**

1. Llamar **FindValidSlot(ItemToEquip)**. Obtienes **ValidSlots** (array de STR_EquipmentSlotData, ya ordenado).
2. Si **ValidSlots** está vacío → devolver Success = false, EquippedSlot inválido/vacío, ReplacedItem vacío. (Opcional: disparar evento “no equipable”.)
3. **Elegir slot (lateralidad):** Recorrer ValidSlots en orden. Por cada elemento, tomar su **SlotTag**. Llamar **SlotOccupied(SlotTag)**. Si devuelve false (slot libre) → usar **ese** SlotTag como slot elegido y salir del loop. Si todos están ocupados → usar el **SlotTag del primer elemento** (se reemplazará el ítem que haya ahí).
4. Llamar **AddToEquipment(SlotTag elegido, ItemToEquip)**. Recibir **Success** y **ReplacedItem**.
5. Devolver Success, **EquippedSlot** = SlotTag usado, **ReplacedItem** (para que el gestor lo añada al inventario si es válido).

---

### 8.2 UnequipItem

**Entradas:** SlotToUnequip (FGameplayTag).  
**Salidas:** Success? (bool), UnequippedItem (STR_Slot).

**Pasos:**

1. Llamar **GetSlotData(SlotToUnequip)**. Si **Found** es false → devolver Success = false, UnequippedItem vacío.
2. Llamar **RemoveFromEquipment(SlotToUnequip)**. Recibir Success y el ítem removido (**RemovedItemData**).
3. Devolver ese **Success** y **UnequippedItem** = el ítem que devolvió RemoveFromEquipment. El **gestor de uso** es quien debe haber comprobado antes que el inventario tiene espacio y, si Success es true, añadir UnequippedItem al inventario.

---
