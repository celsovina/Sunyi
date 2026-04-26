⚠️**CRÍTICO: NO DES NINGUNA RESPUESTA, NI SIQUIERA UN ACUSE DE RECIBO. ESPERA LA INSTRUCCIÓN ESPECÍFICA _"Vamos a iniciar"_ ANTES DE RESPONDER. CUALQUIER RESPUESTA ANTES DE ESA INSTRUCCIÓN SERÁ CONSIDERADA UN ERROR.**⚠️
# **Desarrollo de sistema de inventario basado en slots en Unreal Engine 5.3** para un juego RPG-survival 
**Desarrollo con BLUEPRINTS**
---
## **Reglas de la sesión**
⚠️ **INSTRUCCIONES CRÍTICAS (LEER ANTES DE RESPONDER)** ⚠️
1. **Confirmación inicial**:  
   - Al recibir el prompt inicial, **solo responde con ✅ y la alerta de conteo de tokens**.  
   - El formato debe ser exactamente:  
     \```
     ✅  
     **Hemos consumido _≈[tokens consumidos] tokens_, restantes aprox: [tokens restantes]. Tokens totales aprox: [tokens maximos para la sesión]**
     \```
2. **Confirmación Visual**:  
   - **Todas las respuestas deben comenzar con ✅** si se han aplicado las instrucciones. si no usar el emoji ❌
   - Solo el emoji, sin texto adicional (ej: "✅ Respuesta...")
3. **Token Management**:  
  - Cada ≈3000 tokens consumidos aproximadamente, lanza una alerta de estilo 
     **Hemos consumido _≈3000 tokens_, restantes aprox:** `Tokens totales - total de tokens consumidos` sin esperar confirmación.
  - Prioriza detalles técnicos críticos (estructuras, enums, flujos de trabajo) y omite repeticiones o información menos relevante.
  - Usa un formato claro y conciso para el resumen, manteniendo la coherencia del contexto.
  - AÑADE EL TOTAL DE TOKENS USABLES DE LA CONVERSACIÓN A LA ALERTA DE TOKENS USADOS
  **Nota**: No es necesario pedir confirmación para resumir. Simplemente aplica el resumen cuando se alcance el límite de tokens. 
4. **Formato de Respuestas**:  
  - **Respuestas cortas y concretas**: Máximo 2-3 párrafos por mensaje.  
  - Usa **bullet points**, **negritas** para claridad sin verbosidad.  
  - Excepciones: Si necesitas explicar un sistema complejo, extiéndete.  
  - No adjuntes imágenes en las respuestas bajo ninguna circunstancia.
  - Si necesitas ilustrar un concepto, usa diagramas de secuencia.
5. **Respuestas con opiniones**:  
   - Cuando me hagas preguntas específicas (por ejemplo, "¿Cómo puedo optimizar X?" o "¿Es adecuado hacer Y?"), incluiré una opinión detallada en mi respuesta.  
   - La opinión consistirá en:  
     1. **Qué haría**: Una descripción clara del enfoque o solución que recomendaría.  
     2. **Cómo lo haría**: Los pasos, herramientas o métodos específicos que usaría (por ejemplo, nodos de Blueprints, estructuras de datos, etc.).  
     3. **Justificación**: Una explicación breve de por qué recomiendo ese enfoque.  
   - La opinión seguirá el formato estándar de respuestas (usando **bullet points**, `código`, y **negritas** para claridad).  
   - Se mantendrá dentro del límite de 2-3 párrafos, a menos que la complejidad del tema requiera una explicación más extensa.  
   - **Nota**: Solo incluiré opiniones detalladas en respuestas a preguntas específicas, no en respuestas generales o confirmaciones.
6. **Foco Estricto**:  
  - Responde solo a lo preguntado en el prompt inicial, a menos que se solicite expansión. 
  - ⚠️ **CRITICO: LAS IMAGENES NO SON UTILES PARA MI CONTEXTO. ENFOCATE EN RESPUESTAS TEXTUALES Y EVITA CUALQUIER MENCIÓN O ENLACE A IMAGENES. EVITA A TODA COSTA IMAGENES EN BASE64**⚠️
7. **Respuestas por etapas**
  - Responde **solo al alcance de la etapa actual**
  - No menciones futuras etapas **hasta que se soliciten explícitamente**.
8. **Especificaciones tecnicas**
  - **Prohibición de C++**:  
  - Bajo ninguna circunstancia incluyas código C++, funciones nativas de C++ o referencias a archivos `.cpp`/`.h`.  
  - **Alternativas permitidas**:  
    - Describir lógica con **flujos de Blueprints** (ej: "Usa un *Sequence Node* para...").  
    - Mención de **nodes específicos de Blueprints** (ej: *Make Array*, *For Each Loop*).  
    - Explicaciones técnicas con **texto estructurado** (no código).  
  - **Maps permitidos solo si**:  
    - Son *read-only* (ej: DataTables para consulta de recetas).  
    - No requieren manipulación en tiempo real (evitar `Add`/`Remove` en Blueprints). 
---
## 1. **Enums definidos**
- **ENUM_ItemType**: Categoría general del ítem (Equipment, Resource, Consumable, etc.).
- **ENUM_EquipmentType**: Tipo de equipo (Head, Weapon, Shield, etc.).
- **ENUM_TextContentType**: Tipo de texto (SkillTome, InformationBook, Note).
- **ENUM_Rarity**: Rareza del ítem (Common, Uncommon, Rare, etc.).
- **ENUM_Stat**: Estadísticas modificables (Health, Stamina, Strength, etc.).
- **ENUM_CraftingBench**: Estación de crafteo (Forge, AlchemyTable, etc.).
- **ENUM_InteractionType**: Tipo de interacción (Item, Storage, NPC, etc.).
- **ENUM_CurrencyType**: Tipos de moneda (Gold, Silver, Bronze, Gems).
- **ENUM_BuildType**: Tipo de construcción (Foundation, Wall, Furniture, Decoration, CraftingStation).
---
## 2. **Estructuras Definidas**
1. **STR_ItemInfo** (Contiene datos completos de un ítem (ID, tipo, mesh, stackable, etc.).)
  - **ItemID** (Name): Identificador único (ej: "Potion_Health").  
  - **ItemType** (ENUM_ItemType): Categoría principal. 

  - **BasicInfo**:  (Sub estructura **STR_BasicInfo**) 
    - Mesh (StaticMesh): Mesh estático  
    - SkeletalMesh (SkeletalMesh): Mesh animado
    - Scale (Vector): Escala del mesh
    - IsStackable (bool): ¿Puede apilarse?  
    - MaxStack (int32): Máximo por slot  
    - BasePrice (STR_Currency): Precio base en tiendas.
    - Tags (GameplayTagContainer): Etiquetas dinámicas (ej: "Quest.Important")
    - ItemClass (Class Reference): Clase padrede tipo BP_Item para spawn en mundo

  - **UIData**
    - DisplayName (Text): Nombre localizable (ej: "Poción de Vida")
    - Description (Text): Descripción detallada (admite parámetros dinámicos)
    - Icon (UTexture2D*): Icono del ítem
    - RarityType (ENUM_Rarity)
    - RarityColor (LinearColor): Color de rareza opcional (para bordes/efectos)

  - **CraftingInfo**:  (Sub estructura **STR_CraftingInfo**)
    - IsCraftable (bool): ¿Se puede fabricar?
    - RequiredComponents (Map<Name, int32>): ItemID → Cantidad requerida
    - ResultQuantity (int32): Cantidad creada al craftear (default=1)
    - CraftingXP (float): Experiencia otorgada al fabricar
    - LevelRequired (int32): Nivel mínimo para craftear
    - CraftingBench (ENUM_CraftingBench): Estación requerida

  - **EquipmentInfo**:  (Sub estructura **STR_EquipmentInfo**)
    - EquipmentType (ENUM_EquipmentType): Solo si ItemType=Equipment
    - Pockets (int32): Espacio adicional de inventario (ej: 8)
    - LinkedInventoryID (Name): ID único del container asociado (ej: "Mochila_123")

  - **ConsumableInfo**:  (Sub estructura **STR_ConsumableInfo**)
    - EffectBlueprint (Blueprint Class Reference): Efecto al consumir
    - StatsModifiers (Map<ENUM_Stat, float>): Modificadores instantáneos

  - **TextContentInfo**:  (Sub estructura **STR_TextContentInfo**)
    - TextContentType (ENUM_TextContentType): Tipo de texto
    - SkillTag (GameplayTag): Habilidad asociada (ej: "Skill.Fireball")
    - NoteText (Text): Texto si es Note/InformationBook

  - **STR_BuildInfo** (Sub estructura **STR_BuildInfo**):  
    - BuildID (Name): id de la estructura a construir
    - BuildType (ENUM_BuildType): Tipo de construcción (ej: Foundation, Wall).  

  - **ResearchInfo**: (Sub estructura **STR_ResearchInfo**):  
    - ResearchValue (Float): Valor de investigación (ej: 10 puntos).  
    - RequiredForResearch (Array<Name>): Lista de investigaciones que requieren este ítem.  
2. **STR_Slot**
  ItemID (Name): Identificador del item
  Quantity (Int32): Cantidad del item (Por defecto 1)
3. **STR_Currency**
  CurrencyType (ENUM_CurrencyType)
  Quantity (Int32)
---
## 3. **Blueprints creados**
1. **BP_ItemManager**: (Configurado e implementado)
 * Variables:
    - DT_Item (DT_Item): Referencia al DataTable de ítems
    - CachedItemData (Map<Name, STR_ItemInfo>): Cache de ítems
    - InitializationStatus (Bool): Verificación si el caché está cargado
 * Funciones:
    - GetItemInfo(ItemID): Obtiene datos de un ítem desde el DataTable o caché
    - PreloadAllItems(): Precarga todos los ítems en caché
    - SpawnItem(Slot, Quantity, Transform): Encuentra el item específico para generarlo en el mundo. 
    - GetAllItemsOfClass(Type(ENUM_ItemType)): Busca en `CachedItemData` la totalidad de items por tipo y devuelve la lista filtrada
2. **BP_ItemBase:** (Configurado)
 * Componentes:
    - SM_Item (StaticMeshComponent): Mesh del ítem
    - SK_Item (SkeletalMeshComponent): Mesh animado (opcional)
    - SphereCollision (SphereComponent): Para interacción
    - ItemComponent: Gestiona datos del ítem (ItemID, Quantity)
 * Funciones:
    - InitializeItem(Slot): Configura el ítem según STR_ItemInfo
    - ConfigureMeshAndCollision(): Asigna mesh y ajusta colisión
3. **GI_Sunyi (GameInstance)**: (Configurado e implementado)
 * Variables:
    - ItemManagerRef (BP_ItemManager): Singleton del gestor de ítems
 * Funciones:
    - EventInit: Inicializa BP_ItemManager
4. **InventoryComponent**:
 * Variables:
    - InventoryContent (Array<STR_Slot>): Lista del inventario
    - InventorySize (Int32): Tamaño ajustable del inventario
    - Currency (Array<STR_Currency>): Billetera del personaje
    - ItemManager (BP_ItemManager): Referencia de `BP_ItemManager`
 * Funciones: 
    - FindEmptySlot: busca un slot vacío.  
    - FindStack: busca un stack existente con espacio disponible.  
    - AddToStack: agrega ítems a un stack existente.  
    - AddToInventory: agrega ítems a un slot específico.  
    - PickUpItem: maneja la recolección de ítems, integrando las funciones anteriores.  
    - UseItem: Ejecuta el consumo/Uso de un item de acuerdo a su tipo
    - RemoveItem: elimina 1 item, varios items de un stack o un stack completo del inventario y valida si es drop, si Drop, spawnea el item o stack según sea el caso
5. **BP_Bag**: Funciona de contenedor para el drop de stacks 
6. **BP_SpawnManager** Gestiona el spawneo general de objetos 
  * Funciones:
    - ActionSpawnObject: Cumple la acción de usar SpawnActorFromClass
7. **WBP_Inventory**: Esquema visual del inventario (Contruye un WBP_InventoryGrid)
8. **WBP_InventoryGrid**: Contenedor generico de inventario (Organiza WBP_Slot)
9. **WBP_Slot**: Botón visual del item en el inventario (Contiene función de uso del item en evento clic)
10. **BPFL_Utilities** Librería de funciones de uso común
  *Funciones:
    - GetSpawnTransform: teniendo un actor como origen genera un trace de un largo específico
    - InteractLine: Linetrace de un largo especifico (Interact)
    - InteractSphere: Spheretrace de un diametro especifico (Interact)
    - SpawnItemInWorld: Usa GetSpawnTransform, SpawnItem y actionSpawnObject para generar items o bolsas de items en el mundo
---
## 4. **Consideraciones de desarrollo**
⚠️ **Estas son restricciones de diseño que deben respetarse en todo el desarrollo del sistema.** ⚠️

1. **Sin sistema de peso (`weight`)**:
   - El inventario **no tendrá un sistema de peso**. Los ítems solo estarán limitados por la cantidad de slots y el tamaño máximo de stack.

2. **No es multiplayer (`NO REPLICACIÓN`)**:
   - El juego **no es multiplayer**, por lo que **no se requiere replicación de datos**. Todos los sistemas deben funcionar en un entorno single-player.

3. **NO USAR CAST**: La utilización de nodos Cast, está estrictamente prohibida

4. **Uso compartido (Jugador y NPCs)**:
   - Todos los sistemas de inventario e interacción deben ser **compatibles tanto para el jugador como para los NPCs**. Esto incluye:
     - Gestión de inventario.
     - Interacción con objetos.
     - Uso de ítems (consumibles, equipamiento, etc.).

---
## **📂 Comando de Resumen Portátil**  
⚠️ **INSTRUCCIÓN ESPECIAL**:  
- Si el usuario escribe **`/EXPORTAR_CONTEXTO`**, genera un resumen **EN EL MISMO FORMATO QUE ESTE PROMPT**, con:  
  1. Todas las estructuras, enums y Blueprints definidos.  
  2. Exclusiones:  
     - Eliminar ejemplos redundantes.  
     - Simplificar descripciones a 1 línea por elemento.  
     - Mantener prohibiciones y formato de respuesta original.  
  3. Objetivo: Permitir copia/pega en nuevos chats o modelos sin pérdida de coherencia.  


## 4. **Proceso de desarrollo** (Necesidad de desarrollar en orden de prioridad)
  - creación de gameplayTags
  - Sistema DragAndDrop en el inventario (Swap items entre el mismo inventario o entre 2 inventarios distintos)
  - Creación y asignación de las funciones `GetSlotData`, `SetSlotData`,`CanAcceptItem`, `RemoveItem`, `SwapSlots` 
## 5. **Próximos pasos** (Despues de terminar sección _Preceso de desarrollo_)
  - Configuración `Drag&Drop` (Dejar canales de drop listos para hotbar y equipamento)
---


Vamos a iniciar

Estamos creando un menú contextual que funciona de la siguiente forma:

Cuando hay una GUI que requiera el manejo de items (Inventario, tienda, equipo, etc) y use WBP_Slot, al dar clic derecho sobre el slot en cuestión, en el punto donde se reconoce el clic, se desplegará el menú contextual (WBP_ContextMenu). Este menú utilizará para gestionar el slot (Usar, drop, split, etc); por lo tanto este menú estará directamente relacionado con el slot. El menú tendrá unas características especiales:

1. El contenido del menú cambiará de acuerdo al tipo de UI que se está visualizando

2. los controles de split, drop, sell (Si el item es stackable), Buy (Si el item es stackable), transfer (Si el item es stackable), no podrán ser un solo botón; deberán ser una combinación de spinbox y botón (ya que no sabemos cuantos elementos del stack se quieren transferir)

Se pensó de esta forma porque el SLot está construido de forma tal que sea generico y se pueda usar para cualquier sistema que necesite implementar slots (o uso de items)

Ya tenemos construido el contenedor `WBP_ContextMenu`, con su función basica de creación cuando se da clic derecho a un slot y destrucción cuando el cursor se sale del contenedor del menú (En la imagen 1 se muestra un ejemplo en acción. Los botones son netamente representativos). Tambien se creó un widget llamado `WBP_ContextButton`, el cual es un botón generico para la construcción del menú dinámicamente.

Se tienen 2 posibles propuestas para la creación dinámica del menú:
Validamos el tipo de contenedor (por medio de GameplayTag, para verificar si es inventario, estorage, tienda, equipo, loot, craft, etc)
1. Teniendo todos los botones creados, con su respectiva programación, solo mostramos/ocultamos los botones relacionados a la UI especifica.

2. Tener todas las funciones de la funcionalidad de cada botón y llamarlos por medio de un switch, de acuerdo al texto del botón.

Si tienes alguna otra propuesta, que no sea tan compleja de ejecutar y que respete las reglas de que el proyecto debe ser desacoplado, no contener ningún tipo de cast ni referencias directas entre elementos y componentes, es bienvenido

Si ves que mis propuestas son viables y validas, analizalas y dame una explicación del por que lo son