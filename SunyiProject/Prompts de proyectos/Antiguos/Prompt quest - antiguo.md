# **proyecto en unreal engine 5.3** juego RPG-survival **desarrollado con blueprints**
Tengo implementados 2 sistemas mediante componentes llamados `InventoryComponent` y `EquipmentComponent` _Para sistema de inventario y equipamento respectivamente_
- Mi sistema de inventario gestiona 4 cosas:
    1. Inventario del jugador / inventario de NPC (Puede usar al NPC como cofre cuando NPC es friendly)
        - Recolección de items _Sistema basico de interacción dentro del componente **Tecla E para recoger**_
        - Uso de items/equipo _Spawn de `BP_Effect` para items consumibles que tienen efectos ej: poción de salud_ 
        - `Drag&Drop` items _en el mismo inventario, entre inventarios, spawn 1 item al mundo o spawn una bolsa con un stack de Items_
        - Sistema de mochila _Aumento en la cantidad de slots del inventario de acuerdo al equipo `mochila`_
    2. Tiendas y comercio
    3. Cofres
    4. loot de stacks soltados 
    5. loot de NPC _Si variable IsDead? en NPC es true, el NPC funciona como cofre_
    6. Hotbar _mini inventario independiente que usa las teclas de 1 a 0 para usar items/Equipamento directamente sin entrar al inventario `I`_
- Aparte el sistema de quipo gestiona basicamente el equipamento (Son 20 piezas de equipo: cabeza, cara superior(Gafas), cara inferior(mascarilla), aretes, abrigo/capa, armadura pectoral, torso (Camisa), pantalón, zapatos, guantes, hombrera D, hombrera I, Collar, Anillo D, Anillo I, arma cuerpo a cuerpo (espada, hacha, etc), escudo, munición (flechas), arco, mochila)
    1. Asignación de items de tipo equipo a array `Equipment (Array <STR_Slot>)`
    2. Spawn de mesh respectiva en el character específico
        - si es static mesh, spawn `BPC_ItemName` y attach a socket respectivo en el esqueleto _ej: casco, espada, anillo_
        - Si es skeletal mesh (Ropa/Armadura), reeemplaza la sección del cuerpo específica _ej: Equipa item de tipo Torso (camisa), Sobre escribe torso desnudo por la camisa_
        - Mochila _Aumenta la cantidad de slots al inventario_
    3. Equipo entregado a NPC `Friendly` _Valida si NPC es `Friendly`, si true, al pasarle algún equipo, verifica si el nuevo equipo tiene mejores stats `Potency` que el item equipado actual, si es mejor, lo equipa automáticamente, sino lo deja en el propio inventario_
## **ENUMS definidos**
- ENUM_ItemType: None, ID_Card (Item único usado solamente para la primera misión), SpellBook (Libro de magia), Recipes, Consumable, Resources, Head, FaceUp, FaceDown, Earring, Jacket, ChestArmor, Torso, Pants, Boots, Hands, ShoulderL, ShoulderR, NeckLace, RingL, RingR, Weapon, Shield, Ammo, Bow, Backpack
- ENUM_ItemNames: Enum con nombre de todos los items registrado (Puede ser cambiado debido a problemas de escalabilidad)
- ENUM_CraftingBench: None, Smithy, Oven, Firepit, AlchemyTable, 
## **Estructuras definidas**
- **STR_ItemInfo**
  Scale (Vector) _Ajusta el tamaño del item en el mundo_, ItemNameList (ENUM_ItemNames) _ID del item_, ItemName(Name), Description (Text), ItemImage (Texture2D), ItemClass (ClassReference<BP_Item>) _Clase del actor que va a hacer spawn_, StaticMesh (StaticMesh) _Modelo que se va a mostrar en el mundo_, SkeletalMesh (SkeletalMesh)_Campo que va a actualizar las partes del cuerpo del actor o el arco (Si el item necesita animación)_, Stackable? (Bool), MaxStack (Int32), ItemType (ENUM_ItemType), Potency (Float) _Stat principal del item ej: Bread= +15 Hunger/ ChestPlate= +20 def_, AuxPotency (Float) _Stat secundario del item ej: BRead=+5 Health/ ChestPlate= +10 a la salud (110/100)_, PriceBase (Int32), Craftable? (Bool), Recipe (Map<`ClassReference<BP_ItemTipe>`, Int32>)_Items necesario para fabricar el objeto_, Bench (ENUM_CraftingBench) _El objeto se fabrica en un puesto especifico?_, LevelRequired (Int32) _Nivel de craft requerido para este item_, CraftingXP (Float) _XP de craft dada_, IsRecipe? (Bool) _Si el objeto listado es una receta_, ItemNames (DataTableRowHandle)_Item que va a crear la receta_, IsBook? (Bool)_Si el item creado es libro_, BookContent (Text)_Contenido escrito en el libro_, IsSpellBook? (Bool) _Si el item es libro de habilidad?_, SpellInfo (STR_SpellInfo) _Que habilidad va a aprender_, ItemEffects (`ClassReference<BP_ItemEffect>`)_Efecto otorgado por item consumible_, IsStolen? (Bool)_Flag si el tiem fue robado_, Pockets (Int32)_Aumenta la cantidad de slotsen el inventario_ (Solo funciona para la mochila _de momento_)
- **STR_Slot**
  ItemID (Name), Quantity(Int32)
## **Componentes declarados**
- InventoryComponent: Gestiona el inventario
- EquipmentComponent: Gestiona el equipo
- ItemComponent: Elemento que gestiona los items (Asigna tipo item y cantidad de acuerdo a `STR_Slot` para cargar la información específica en `BP_Item` y sus hijos)
## **Funciones declaradas**
### **InventoryComponent**
- **PickUpItem**
Verifica que el inventario tenga slots disponibles
Verifica que el item sea `stackable`. Si `stackable`= true busca stack disponible; si `stackable` = flase busca slot disponible
Si hay `stack` o `slot` disponibles, agrega item
Sino hay `stack` o `slot` disponibles, no recoge el item
- **InteractionTrace**
Genera un `LineTrace` de un largo específico para interactuar con el mundo
- **FindSlot**
Busca `Slot disponible en el inventario`
- **GetMaxStack**
Busca el item y devuelve `MaxStack`
- **AddToStack**
Agrega el item al primer `stack` compatible
- **FindEmptySlot**
Encuentra slot vacio en el inventario
- **AddItem**
Si `FindEmptySlot`=true agrega item al primer slot disponible
- **TransferSlots**
Hace swap sobre la posición de 2 items en el inventario
- **RemoveItem**
Elimina o 1 item o 1 stack del inventario
- **DropItem**
Hace spawn de un item o una bolsa con un stack en el mundo
- **GetItemData**
Consigue la info de un item desde el `DataTableRow` y devuelve el item
- **GetDropLocation**
Genera un `LineTrace` desde el actor para hacer spawn del item botado
- **UseItem**
Verifica el tipo de item y lo usa en consecuencia (Si es consumable, lo usa y lo elimina, si es de tipo equipo, lo usa y hace spawn al mesh o skeletal mesh en el actor)
- **ConsumeItem**
Hace spawn del `BP_ItemEffect` perteneciente al item consumido
- **GetQuantity**
Devuelve la cantidad total de items en el inventario
- **SaveInventory**
Guarda el inventario del jugador, NPC y cofres del mundo
- **LoadInventory**
Carga el inventario del jugador, NPC y cofres del mundo
- **UseItemHotbar**
Usa el asignado a un botón específico en el hotbar
- **ClearUsedHotbarSlot**
Si el item en el hotbar fue totalmente consumido/equipado, limpia el slot
- **SetHotbar**
Envía información del hotbar a interface `BPI_ShareSlot`
- **TransferItemsToInv**
Transfiere items desde inventarios externos (Hotbar y equipment) a el inventario _Mediante Drag&Drop_
- **RemoveItemHotbar**
Remueve 1 item del hotbar despues de usarlo
- **FindItemInInventory**
Busca un item específico y devuelve el total de este item en el inventario (No importa si es o no `Stackable`)
### **EquipmentComponent**
- **AssignEquip**
Valida si el componente es del jugador o un NPC y equipa o remueve el item al array equipment
- **SpawnMesh**
Valida si el componente es del jugador o un NPC y spawn la mesh en cuestión
- **RemoveMesh**
Remueve la mesh del equipo seleccionado. si es skeletal mesh torso, legs, hands o foot, pone las partes del cuerpo desnudas
- **GetItemData**
Consigue la info de un item desde el `DataTableRow` y devuelve el equipo
- **SpawnCloth**
Hace spawn del skeletal mesh en cuestión
- **ResetCloth**
pone las partes del cuerpo desnudas en el actor
- **SaveEquipment**
Guarda el estado de los equipos actuales
- **LoadEquipment**
Carga el estado de los equipos
- **RemoveEquipment**
Remueve el equipo. Si se están usando slots de la mochila, esta no se puede desequipar
- **GetDropLocation**
Genera un `LineTrace` desde el actor para hacer spawn del item botado
- **DropItem**
Spawn de item arrojado y guarda el estado de los items arrojados en el mundo
- **VerifyBackpackIsUsed**
Verifica si algún slot de la mochila se está usando

Para acceder a mis interfaces _GUI_ se está haciendo desde `WBP_HUD` mediante eventos conectados al sistema de interacción y comandos de teclado _Tecla I, para acceder al inventario_
---
 
Aquí tienes la lista completa de instrucciones críticas que solicitaste, organizadas y detalladas para su implementación en Unreal Engine 5.3 usando solo Blueprints:

1. Sistema de Inventario Modular
Objetivo:

Crear un sistema de inventario basado en slots que sea reutilizable para jugadores, NPCs, cofres, loot de enemigos y tiendas.

Requerimientos:

Los ítems pueden arrastrarse para cambiar de posición o combinarse.

El inventario debe ser expandible mediante el uso de mochilas o ropa con bolsillos (campo Pockets).

Usar DataTables para almacenar la información de los ítems (STR_ItemInfo).

2. Estructura de Datos
STR_ItemInfo:

Contiene toda la información del ítem:

BasicInfo: Nombre, descripción, icono, mesh, skeletal mesh, escala, stack máximo, tipo de ítem, rareza, etc.

CraftingInfo: Si es crafteable, componentes requeridos (usar Map para consultas), cantidad resultante, experiencia de crafting y nivel requerido.

EquipmentInfo: Tipo de equipo y espacio adicional de inventario (Pockets).

ConsumableInfo: Efecto al consumir (Blueprint) y modificadores de stats (usar Map para consultas).

LearningInfo: Tipo de libro (tomo de habilidad, nota, etc.), habilidad asociada y texto de la nota.

STR_Slot:

Solo contiene ItemID (Name) y Quantity (Int32).

Se usa para manipular la cantidad sin afectar la data base del ítem.

3. Enums
ENUM_ItemType:

plaintext
Copy
None, Equipment, Resource, Consumable, Recipe, Book, Ammo, Quest, Backpack
ENUM_EquipmentType:

plaintext
Copy
None, Head, FaceUp, FaceDown, Earring, Jacket, ChestArmor, Torso, 
Pants, Boots, Hands, ShoulderL, ShoulderR, NeckLace, RingL, 
RingR, Weapon, Shield, Ammo, Bow
ENUM_BookType:

plaintext
Copy
SkillTome, InformationBook, Note
ENUM_Rarity:

plaintext
Copy
Common, Uncommon, Rare, Epic, Legendary
ENUM_StatType:

plaintext
Copy
Health, Stamina, Mana, Thirst, Sleep, Strength, Agility
4. Sistema de Crafting
Requerimientos:

Los ítems pueden ser crafteables (IsCraftable=True).

Usar Map para RequiredComponents (solo consultas, no modificación).

Soporte para recetas que generen múltiples ítems (ResultQuantity).

Experiencia de crafting (CraftingXP) y nivel requerido (LevelRequired).

5. Sistema de Equipamiento
Requerimientos:

Los ítems de tipo Equipment pueden equiparse en slots específicos (ENUM_EquipmentType).

Equipar mochilas o ropa con bolsillos aumenta el espacio de inventario (Pockets).

6. Sistema de Consumibles
Requerimientos:

Los consumibles aplican efectos al usarse (EffectBlueprint).

Modifican stats del jugador usando un Map de StatsModifiers.

Ejemplo:

plaintext
Copy
Café:
  - StatsModifiers: { Thirst:5, Sleep:-25, Stamina:15 }
7. Sistema de Aprendizaje
Requerimientos:

Los libros (ENUM_BookType) pueden enseñar habilidades (SkillTag) o contener texto (NoteText).

Usar GameplayTags para habilidades (SkillTag) por su flexibilidad y jerarquías.

8. Sistema de Hotbar
Requerimientos:

Los ítems pueden asignarse a una hotbar (10 slots, teclas 1-0).

Al presionar la tecla, se usa el ítem correspondiente sin abrir el inventario.

9. Sistema de Rareza
Requerimientos:

Cada ítem tiene una rareza (ENUM_Rarity).

Asignar colores de rareza (RarityColor) para UI y efectos visuales.

Si RarityColor está vacío, usar colores por defecto basados en Rarity.

10. Sistema de Escalado de Ítems
Requerimientos:

Cada ítem tiene un campo Scale (Vector) para ajustar el tamaño de su mesh en el mundo.

Aplicar la escala en el Construction Script de BP_Item.

11. Sistema de GameplayTags
Requerimientos:

Usar GameplayTags para estados dinámicos (ej: ítem robado, quest importante).

Ejemplo:

plaintext
Copy
- Tags: "State.Stolen", "Quest.Important"
12. Sistema de DataTables
Requerimientos:

Usar DataTables para almacenar STR_ItemInfo.

Cada fila corresponde a un ItemID y contiene todos los sub-structs.

13. Directivas de Implementación
No usar replicación: El juego no requiere multiplayer.

Evitar Maps en Blueprints: Excepto para consultas de datos (RequiredComponents, StatsModifiers).

Cargar datos en BP_Item: Usar Construction Script para asignar mesh, escala, etc., desde STR_ItemInfo.

14. Ejemplos de Flujos Técnicos
Craftear un ítem:

Verificar RequiredComponents en el inventario.

Remover componentes y añadir el ítem craftedo (ResultQuantity).

Equipar una mochila:

Aumentar MaxSlots en el inventario usando Pockets.

Usar un consumible:

Ejecutar EffectBlueprint y aplicar StatsModifiers.

Aprender una habilidad:

Añadir SkillTag al jugador al usar un tomo de habilidad.

15. Notas Finales
Optimización:

Usar DataTables para evitar duplicación de datos.

Usar GameplayTags para estados dinámicos y filtros en UI.

Escalabilidad:

La estructura es modular y soporta futuras expansiones (ej: nuevos tipos de ítems, habilidades).
--------------------------------------
Genial, agrega a esto que me acabas de decir lo siguiente:
1. Cada item con pockets (Mochila, pantalón, chaqueta, etc) visualmente creará un container independiente del inventario basico del jugador que deberá mostrarse solo cuando el item en cuestión esté equipado si se desequipa el item, todo lo que tenía almacenado, se va a spawnear en el suelo 
2. Tenemos que implementar de forma independiente, pero simultanea un sistema de interacción que nos discrimine si con lo que estamos interactuando es un item, cofre, NPC (friendly, QuestGiver, Tienda, dialogo, etc), loot, entre otros
3. Se nos olvidó agregarlo ` STR_ItemInfo.CraftingInfo`, tambien debemos adjuntar un Enum llamado ENUM_CraftingBench, que nos validará si el item a fabricar necesita una estación de trabajo específica (ej: Herrería, forja, horno, olla, hoguera, etc)
4. La funcion del hotbar es un inventario totalmente separado del inventario normal _Pero puede interactuar con el inventario y el equipamento (Drag&Drop entre uno y otro)_ que almacenará items para su uso _Excepto municiones, esas deberan ser equipadas directamente del inventario_ 
> Considero que el hotbar debería ser un componente diferente, ¿Que opinas?

- Dame la versión actualizada de `STR_ITemInfo` junto con todas las sub estructuras, enums declarados y todos los requerimientos con la actualización solicitada