# Resumen de Arquitectura Final: EquipmentComponent

## 1. Objetivo y Principios Fundamentales

* **Objetivo**: Construir un componente de actor, llamado **`EquipmentComponent`**, que sea modular y autocontenido para gestionar el sistema de equipamiento. Funcionará para cualquier actor (jugador, PNJ, etc.).
* **Principio de Instancias**: La gestión de `STR_InstanceData` es **descentralizada**. Cada componente (`InventoryComponent`, `EquipmentComponent`, etc.) es responsable de gestionar las instancias de los ítems que posee en un momento dado. La `InstanceData` se mueve entre componentes junto al `STR_Slot` del ítem.

---

## 2. Assets y Estructura de Carpetas

* **Componente Principal**: `EquipmentComponent` (Hereda de `ActorComponent`).
* **Actor de Ítem Spawneado**: `BP_ItemBase`. Es el actor que se crea en el mundo para representar visualmente ítems equipados como armas, cascos, etc.
* **Ruta de Archivos**: Todos los nuevos assets relacionados con este sistema se ubicarán en una carpeta raíz dedicada: `Content/EquipmentSystem/`.
    * **Componente**: `Content/EquipmentSystem/Blueprints/Components/`
    * **Interfaces**: `Content/EquipmentSystem/Interfaces/`
    * **Datos (si se necesitaran)**: `Content/EquipmentSystem/Data/`

---

## 3. Arquitectura de Datos Interna (Variables del Componente)

El `EquipmentComponent` no utilizará una estructura de datos compleja, sino tres `TMaps` independientes vinculados por un `FGameplayTag` que identifica cada slot.

* **`EquipmentList`**: `TMap<FGameplayTag, STR_Slot>`
    * **Rol**: La lista maestra de **datos**. Contiene la información (`STR_Slot`) de cada ítem equipado. La clave (`FGameplayTag`) identifica el slot de forma única (ej. `Slot.Equip.Head`).
* **`SpawnedActors`**: `TMap<FGameplayTag, BP_ItemBase*>`
    * **Rol**: Gestiona los **visuales de tipo Actor**. Almacena la referencia al `BP_ItemBase` que se ha spawneado y adjuntado al personaje. La clave es el `SlotTag` del ítem correspondiente.
* **`SpawnedSKMeshes`**: `TMap<FGameplayTag, SkeletalMesh*>`
    * **Rol**: Gestiona los **visuales de tipo Skeletal Mesh**. Almacena la referencia al asset `SkeletalMesh` que se ha aplicado a una parte del cuerpo del personaje (ej. una camiseta). La clave es el `SlotTag`.

---

## 4. Lógicas de Diseño Clave

* **Tipos de Ítems Visuales**: El sistema diferencia dos tipos de ítems equipables:
    1.  **`SkeletalMesh`**: Ítems que reemplazan la malla de un `SkeletalMeshComponent` ya existente en el personaje (ej. torso, piernas).
    2.  **`Actor Spawneado`**: Ítems que requieren spawnear un `BP_ItemBase` en el mundo y adjuntarlo a un socket del esqueleto del personaje (ej. armas, cascos, hombreras).
* **Slots Pares (Anillos, Hombreras, etc.)**:
    * Los ítems son **genéricos** (no existen versiones "Derecha" e "Izquierda" del mismo ítem).
    * La lógica de equipamiento implícito (doble clic, hotbar) será: `1. Intentar slot Derecho` -> `2. Si está ocupado, intentar slot Izquierdo` -> `3. Si ambos están ocupados, sobrescribir el slot Derecho`.
    * El equipamiento explícito (arrastrar y soltar) respetará la decisión del jugador y sobrescribirá el slot de destino directamente.

---

## 5. Funciones del Componente

### 5.1. Funciones Públicas (La API para interactuar con el componente)

#### **Comandos (Modifican el estado del equipo)**
* **`EquipItem`**: La función principal para iniciar el proceso de equipar un ítem.
    * **Entradas**: `ItemToEquip (STR_Slot)`.
    * **Salidas**: `Success? (bool)`, `EquippedSlot (FGameplayTag)`.
* **`UnequipItemFromSlot`**: La función principal para desequipar un ítem de una ranura específica.
    * **Entradas**: `SlotToUnequip (FGameplayTag)`.
    * **Salidas**: `Success? (bool)`, `UnequippedItem (STR_Slot)`.

#### **Consultas (Obtienen datos sin modificar nada)**
* **`GetEquippedItemInSlot`**: Obtiene los datos (`STR_Slot`) del ítem en un slot específico.
* **`IsSlotOccupied`**: Verifica si un slot tiene un ítem. **Debe ser una función Pura**.
* **`GetAllEquippedItems`**: Devuelve un array con todos los `STR_Slot` de los ítems equipados.

#### **Validación (Comprueban si una acción es posible)**
* **`CanEquipItemInSlot`**: Verifica si un ítem es compatible con un slot.
* **`FindValidSlotForItem`**: El "repartidor". Dado un ítem, devuelve un array de `FGameplayTag` con los slots donde podría equiparse.

### 5.2. Subfunciones Internas (Helpers para mantener el código limpio)

#### **Helpers de Lógica (Con ejecución)**
* **`ApplyVisualsForItem`**: Orquesta la aplicación de los visuales (llama a spawnear actor o a cambiar malla).
* **`ClearVisualsForSlot`**: Orquesta la eliminación de los visuales (destruye el actor o restaura la malla por defecto).
* **`AddDataToMaps`**: Centraliza la lógica para añadir la información de un ítem a los `TMaps` correspondientes.
* **`RemoveDataFromMaps`**: Centraliza la lógica para eliminar la información de un ítem de todos los `TMaps`.

#### **Helpers Puros (Sin ejecución)**
* **`GetItemVisualType`**: Lee el `ItemInfo` de un ítem y determina si su visual es de tipo `Actor`, `Skeletal` o `Ninguno`.
* **`GetSlotData`**: Busca de forma segura los datos de un slot en el `TMap` `EquipmentList`.
* **`GetDefaultSkeletalMeshForSlot`**: Devuelve la malla por defecto (ej. torso desnudo) para un slot esquelético, para poder restaurarla al desequipar.