# Resumen de Arquitectura: BPC_EquipmentComponent

## 1. Objetivo Principal
Construir un componente de actor modular y desacoplado (`BPC_EquipmentComponent`) para gestionar el sistema de equipamiento de cualquier personaje (jugador, PNJ, etc.). El sistema debe ser capaz de manejar ítems que cambian una malla esquelética y ítems que spawnean un actor en el mundo.

---

## 2. Arquitectura de Datos Interna
Decidimos no usar una única estructura compleja, sino gestionar los datos y los visuales en colecciones separadas pero vinculadas por un `FGameplayTag` que identifica cada slot de equipo.

### Variables del Componente (`BPC_EquipmentComponent`)
* **`EquipmentList`**: Un `TMap<FGameplayTag, STR_Slot>`
    * **Propósito**: Es la lista maestra de **datos**. Almacena la información (`STR_Slot`) de cada ítem equipado. La clave (`FGameplayTag`) identifica el slot (ej. `Slot.Equip.Head`).
* **`SpawnedActors`**: Un `TMap<FGameplayTag, BP_ItemBase*>`
    * **Propósito**: Gestiona los **visuales de tipo Actor**. Almacena la referencia al actor `BP_ItemBase` que se spawnea para ítems como cascos, armas o escudos. La clave es el `SlotTag` correspondiente.
* **`SpawnedSKMeshes`**: Un `TMap<FGameplayTag, SkeletalMesh*>`
    * **Propósito**: Gestiona los **visuales de tipo Skeletal Mesh**. Almacena la referencia al asset `SkeletalMesh` que se aplica a una parte del cuerpo del personaje para ítems como camisetas o pantalones. La clave es el `SlotTag`.

---

## 3. Lógicas Clave de Diseño
* **Ítems Genéricos para Slots Pares**: Los ítems como anillos, hombreras o armas a una mano son genéricos. No crearemos versiones "Derecha" e "Izquierda" del mismo ítem. La lógica para decidir en qué lado equipar (si el jugador no lo especifica) será: `Intentar Derecha -> Si ocupado, Intentar Izquierda -> Si ambos ocupados, Sobrescribir Derecha`.
* **Gestión de Instancias Descentralizada**: El `BPC_EquipmentComponent` es responsable de gestionar la `STR_InstanceData` de los ítems que posee. La `InstanceData` se mueve junto con el `STR_Slot` desde otros componentes (como el inventario) a este, manteniendo la carga del sistema distribuida.

---

## 4. Funciones del Componente

### 4.1. Funciones Públicas (La API del Componente)

#### **Comandos (Modifican el estado)**
* **`EquipItem`**
    * **Propósito**: Inicia el proceso completo para equipar un ítem.
    * **Entradas**: `ItemToEquip (STR_Slot)`.
    * **Salidas**: `Success? (bool)`, `EquippedSlot (FGameplayTag)`.
* **`UnequipItemFromSlot`**
    * **Propósito**: Inicia el proceso para desequipar un ítem de un slot específico.
    * **Entradas**: `SlotToUnequip (FGameplayTag)`.
    * **Salidas**: `Success? (bool)`, `UnequippedItem (STR_Slot)`.

#### **Consultas (Obtienen datos, son "read-only")**
* **`GetEquippedItemInSlot`**
    * **Propósito**: Obtiene los datos del ítem equipado en un slot.
    * **Entradas**: `TargetSlot (FGameplayTag)`.
    * **Salidas**: `ItemData (STR_Slot)`, `Found? (bool)`.
* **`IsSlotOccupied`**
    * **Propósito**: Verifica si un slot está ocupado. **Debe ser una función Pura.**
    * **Entradas**: `TargetSlot (FGameplayTag)`.
    * **Salidas**: `IsOccupied? (bool)`.
* **`GetAllEquippedItems`**
    * **Propósito**: Devuelve todos los ítems equipados.
    * **Entradas**: Ninguna.
    * **Salidas**: `EquippedItems (Array<STR_Slot>)`.

#### **Validación (Comprueban si una acción es posible)**
* **`CanEquipItemInSlot`**
    * **Propósito**: Verifica si un ítem es compatible con un slot.
    * **Entradas**: `ItemToCheck (STR_Slot)`, `TargetSlot (FGameplayTag)`.
    * **Salidas**: `CanEquip? (bool)`.
* **`FindValidSlotForItem`**
    * **Propósito**: El "repartidor". Encuentra en qué slot(s) se puede equipar un ítem.
    * **Entradas**: `ItemToCheck (STR_Slot)`.
    * **Salidas**: `ValidSlots (Array<FGameplayTag>)`.

### 4.2. Subfunciones Internas (Helpers)

#### **Helpers de Lógica**
* **`ApplyVisualsForItem`**: Orquesta la aplicación de visuales, decidiendo si spawnear un actor o cambiar una malla.
* **`ClearVisualsForSlot`**: Orquesta la eliminación de visuales (destruye actor o restaura malla).
* **`AddDataToMaps`**: Centraliza la lógica de añadir la info a los 3 `TMaps`.
* **`RemoveDataFromMaps`**: Centraliza la lógica de eliminar la info de los 3 `TMaps`.

#### **Helpers Puros**
* **`GetItemVisualType`**: Lee el `ItemInfo` y devuelve si el ítem es de tipo `Actor`, `Skeletal` o `Ninguno`.
* **`GetSlotData`**: Busca de forma segura los datos de un slot en el `TMap` `EquipmentList`.
* **`GetDefaultSkeletalMeshForSlot`**: Devuelve la malla por defecto (ej. torso desnudo) para restaurarla.

---

## 5. Assets Creados / A Crear
* **Componente**: `BPC_EquipmentComponent` en la ruta `Content/EquipmentSystem/Blueprints/Components/`.
* **Actor de Ítem**: `BP_ItemBase`, que es el actor que se spawnea en el mundo para representar ítems estáticos equipados.