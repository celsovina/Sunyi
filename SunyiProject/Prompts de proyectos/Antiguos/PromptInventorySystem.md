# **Desarrollo de sistema de inventario basado en slots en Unreal Engine 5.3** para un juego RPG-survival 
Desarrollo con BLUEPRINTS
## **Enums definidos**
- **ENUM_ItemType** (Categoría general del ítem)  
None, Equipment, Resource, Consumable, Recipe, Book, Ammo, Quest, Backpack  
- **ENUM_EquipmentType** (Tipo de equipo)  
None, Head, FaceUp, FaceDown, Earring, Jacket, ChestArmor, Torso, Pants, Boots, Hands, ShoulderL, ShoulderR, NeckLace, RingL, RingR, Weapon, Shield, Ammo, Bow  
- **ENUM_TextContentType** (Tipo de libro/nota)  
SkillTome, InformationBook, Note  
- **ENUM_Rarity** (Rareza del ítem)  
Common, Uncommon, Rare, Epic, Legendary  
- **ENUM_Stat** (Estadísticas modificables)  
Health, Stamina, Mana, Thirst, Sleep, Strength, Agility  
- **ENUM_CraftingBench** (Estación de crafteo requerida)  
None, Forge, AlchemyTable, CookingPot, Workbench  
- **ENUM_InteractionType** (Tipo de interacciones)
None, Item, Storage, NPCType(Friendly(para gestionar inventario de aliado), Shop(Para tiendas), Giver(Para misiones), etc), Loot
- **ENUM_CurrencyType** (Typos de moneda)
Gold, Silver, Bronze, Gems
---
## **Estructuras Definidas**
1. **STR_ItemInfo**
  - ItemID (Name): Identificador único (ej: "Potion_Health").  
  - ItemType (ENUM_ItemType): Categoría principal.   
  - BasicInfo:  (Sub estructura **STR_BasicInfo**) 
    - Mesh (StaticMesh): Mesh estático.  
    - SkeletalMesh (SkeletalMesh): Mesh animado.  
    - Scale (Vector): Escala del mesh (ej: (0.5, 0.5, 0.5)).  
    - IsStackable (bool): ¿Puede apilarse?  
    - MaxStack (int32): Máximo por slot (solo si IsStackable=True).  
    - BasePrice (STR_CurrencyValue): Precio base en tiendas.
    - Tags (GameplayTagContainer): Etiquetas dinámicas (ej: "Quest.Important").  
    - ItemClass (Class Reference): Clase padre para spawn en mundo.  
  - UIData
    - DisplayName (Text): Nombre localizable (ej: "Poción de Vida").  
    - Description (Text): Descripción detallada (admite parámetros dinámicos).  
    - Icon (UTexture2D*): Icono del ítem. 
    - RarityType (ENUM_Rarity) 
    - RarityColor (LinearColor): Color de rareza opcional (para bordes/efectos).
  - CraftingInfo:  (Sub estructura **STR_CraftingInfo**)
    - IsCraftable (bool): ¿Se puede fabricar?  
    - RequiredComponents (Map<Name, int32>): ItemID → Cantidad requerida.  
    - ResultQuantity (int32): Cantidad creada al craftear (default=1).  
    - CraftingXP (float): Experiencia otorgada al fabricar.  
    - LevelRequired (int32): Nivel mínimo para craftear.  
    - CraftingBench (ENUM_CraftingBench): Estación requerida (ej: Forge).  
  - EquipmentInfo:  (Sub estructura **STR_EquipmentInfo**)
    - EquipmentType (ENUM_EquipmentType): Solo si ItemType=Equipment.  
    - Pockets (int32): Espacio adicional de inventario (ej: 8).  
    - LinkedInventoryID (Name): ID único del container asociado (ej: "Mochila_123").  
  - ConsumableInfo:  (Sub estructura **STR_ConsumableInfo**)
    - EffectBlueprint (Blueprint Class Reference): Efecto al consumir.  
    - StatsModifiers (Map<ENUM_StatType, float>): Modificadores instantáneos.  
  - TextContentInfo:  (Sub estructura **STR_TextContentInfo**)
    - TextContentType (ENUM_TextContentType): Tipo de texto.  
    - SkillTag (GameplayTag): Habilidad asociada (ej: "Skill.Fireball").  
    - NoteText (Text): Texto si es Note/InformationBook.  
2. **STR_Slot**
  ItemID (Name): Identificador del item
  Quantity (Int32): Cantidad del item (Por defecto 1)
3. **STR_Currency**
  CurrencyType (ENUM_CurrencyType)
  Quantity (Int32)
---
## **Etapa actual** (Foco global)
1. Configuración de enums y estructuras
Requerimiento:
- Paso a paso detallado de creación de los enums y las estructuras
2. DataTable `DT_ItemInfo`
  Gestiona Los datos de cada item
3. `InventoryComponent`
- Variables:
InventoryContent (Array<STR_Slot>): Slots disponibles (base + expansiones).
BaseSize (Int32): tamaño default del inventario (10 para jugador, variable para NPC)
MaxSlots (int32): Calculado desde EquipmentInfo.Pockets.
- Eventos disparadores:
OnInventoryUpdated (Event Dispatcher): Notifica cambios en UI.
- Funciones clave:
1. AddItem(ItemID, Quantity):
  - Buscar slots existentes con mismo ItemID y espacio en MaxStack.
  - Si no hay espacio, añadir nuevo slot.
  - Si no hay espacio en inventario, no recoge el item.
2. RemoveItem(ItemID, Quantity):
  - Reducir Quantity. Si llega a 0, vacía el slot.
3. SwapSlots(IndexA, IndexB): 
  - Intercambiar ItemID y Quantity.
4. LinkToHotbar(InventoryIndex, HotbarIndex): 
 - Vincular slot de inventario a hotbar.
4. `BP_Item` (Actor para ítems en el mundo)
Configuración:
- Construction Script:
  - Cargar STR_ItemInfo desde DataTable usando ItemID.
  - Asignar Mesh/SkeletalMesh y Scale desde BasicInfo.
  - Configurar física (Simulate Physics si no está en inventario).
5. `InteractionComponent`
1. Componentes:
  - Interfaz BP_Interactable:
    - Función GetInteractionType() → ENUM_InteractionType (Item, Storage, NPC_Shop).
  - BP_InteractionComponent (Player):
    - Raycast para detectar actores con BP_Interactable.
    - Ejecutar acciones según tipo:
      ej: Item: Pickup (AddItem). Storage: Abrir inventario del contenedor.
6. `GUI`
Widgets Clave:
  - WBP_Inventory:
    - Secciones:
      Inventario (GridPanel con ScrollBox).
      Equipo (UniformGrid para slots de EquipmentType).
      Hotbar (HorizontalBox con 10 slots).
    - Drag & Drop:
      OnDragDetected: Almacenar índice de origen.
      OnDrop: Validar destino y llamar a SwapSlots/MergeSlots.
    - Visuales:
      Mostrar RarityColor como borde del slot.
      Mostrar Quantity como texto (si es 0, el slot queda vacío).
7. Save/Load 
  **Critico**: Guardar/cargar sistema de inventario y estado de items en el mundo al cerrar/abrir el juego (GameInstance)
## **Desarrollo Actual** (Implementación)
1. `BP_Item`
  - se creó flujo de organización `GameDataTypes` para almacenar estructuras, enums y blueprints que comparten multiples sistemas
  - Se agregó blueprint object `BP_ItemManager` para realizar de puedente entre sistemas e items
  - Configuración de GameInstance `GI_Sunyi` para hacer a `BP_ItemManager` en **Singleton**
    - Variables: `ItemManagerClass` de tipo `BP_ItemManager class reference`, `IntemManagerRef` de tipo `BP_ItemManager Object reference`
    - Configuración: `EventInit`->`Construct BP_ItemManager`
                      Inputs de `Construct BP_ItemManager`: `Get BP_ItemManager`->`_Class_`(La variable se ha seteado con la clase BP_ItemManager), `SelfReference`->`Outer`
                      Outputs de `Construct BP_ItemManager`: `ReturnValue` -> Set `IntemManagerRef` 
  - **SIGIENTE**
    - Configurar variables y funciones de `BP_ItemManager`
2. `InventoryComponent`
3. `InteractionComponent`
**Nota**: **Tener presente el proceso de creación delas carpetas en el editor**
### **Optimizaciones Clave**
**Early Exit**: Rompe loops al encontrar elementos para mejorar rendimiento.
**Manejo de FunctionLibraries**: Funciones que se repitan en varios componentes, crearlas en un `functionlibrary`
## **Pasos a seguir**
1. `GUI`
2. `HotbarComponent`
3. Save/Load
---
## **Etapas futuras**
1. Sistema de Equipamiento
Requerimientos:
- Slots UI específicos por ENUM_EquipmentType.
- Actualizar stats del jugador al equipar (ej: defensa +10 por armadura).
- Evento OnEquip para efectos visuales (cambiar mesh del personaje).
2. Sistema de Pockets
Requerimientos:
- BP_ContainerManager (GameInstance) para guardar inventarios vinculados (LinkedInventoryID).
- UI dinámica con pestañas (Mochila, Ropa, Accesorios).
- Spawn de BP_Container al desequipar ítems no vacíos.
3. Sistema de Tiendas
Requerimientos:
- Precios dinámicos (BasePrice * multiplicador por rareza).
- Interfaz de compra/venta con arrastre de ítems.
- Diálogos de NPC con ENUM_InteractionType = NPC_Shop.
4. Sistema de Currency (CurrencyComponent)
5. Crafting
Requerimientos:
- Widget de crafting con filtrado por ENUM_CraftingBench.
- Validación de nivel y componentes.
- Experiencia acumulable (CraftingXP) para desbloquear aprendizaje de recetas.
---
## **Instrucciones Adicionales**
⚠️ **INSTRUCCIONES CRÍTICAS (LEER ANTES DE RESPONDER)** ⚠️
1. **Confirmación Visual**:  
   - **Todas las respuestas deben comenzar con ✅** si se han aplicado las instrucciones. si no usar el emoji ❌
   - Solo el emoji, sin texto adicional (ej: "✅ Respuesta...")
2. **Token Management**:  
  - Cada  ≈3000 tokens consumidos, lanza una alerta de estilo 
     **Hemos consumido _≈3000 tokens_, restantes aprox:** `Tokens totales - total de tokens consumidos` sin esperar confirmación.
  - Prioriza detalles técnicos críticos (estructuras, enums, flujos de trabajo) y omite repeticiones o información menos relevante.
  - Usa un formato claro y conciso para el resumen, manteniendo la coherencia del contexto.
  - Cuando se supere el umbral de ≈4000 tokens, al resumen ponle esta etiqueta "Resumen automático aplicado debido al límite de tokens. Se han priorizado los detalles técnicos críticos"
  - AÑADE EL TOTAL DE TOKENS USABLES DE LA CONVERSACIÓN A LA ALERTA DE TOKENS USADOS
  **Nota**: No es necesario pedir confirmación para resumir. Simplemente aplica el resumen cuando se alcance el límite de tokens. 
3. **Formato de Respuestas**:  
  - **Respuestas cortas y concretas**: Máximo 2-3 párrafos por mensaje.  
  - Usa **bullet points**, **negritas** y `code snippets` para claridad sin verbosidad.  
  - Excepciones: Si necesitas explicar un sistema complejo, extiéndete solo lo necesario.  
  - No adjuntes imágenes en las respuestas bajo ninguna circunstancia.
  - Si necesitas ilustrar un concepto, usa descripciones textuales o ejemplos de código.
4. **Foco Estricto**:  
  - Responde solo a lo preguntado en el prompt inicial, a menos que se solicite expansión. 
  - Las imágenes no son útiles en este contexto, ya que no se visualizan correctamente y pueden aumentar el consumo de tokens innecesariamente. Enfócate en respuestas textuales y evita cualquier mención o enlace a imágenes.
  **CRITICO: EVITA A TODA COSTA IMAGENES EN BASE64**
5. **Respuestas por etapas**
  - Responde **solo al alcance de la etapa actual**
  - No menciones futuras etapas **hasta que se soliciten explícitamente**.
6. **Especificaciones tecnicas**
  - Si necesitas ilustrar un concepto, **usa descripciones textuales o ejemplos de flujo de blueprints _Sin codigo tradicional_**
  - **Maps permitidos solo si**:  
    - Son *read-only* (ej: DataTables para consulta de recetas).  
    - No requieren manipulación en tiempo real (evitar `Add`/`Remove` en Blueprints). 
