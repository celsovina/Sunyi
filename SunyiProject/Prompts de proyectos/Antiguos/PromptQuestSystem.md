# **Sistema de misiones dinámicas en Unreal Engine 5.3** para un juego RPG-survival 
Desarrollo con BLUEPRINTS
## **Enums Definidos**
- ENUM_QuestType: MainQuest, SideQuest, RecurrentQuest, WorldEvent
- ENUM_QuestStatus: Available, Active, Completed, Failed, Blocked
- ENUM_ObjectiveType: Collect, Kill, Escort, TalkTo, GoTo, Craft, etc
- ENUM_PrerequisiteType: QuestCompleted, ItemOwned, RecipeLearned, etc
- ENUM_FailureType: TimeExpired, NPCDied, ItemDestroyed, etc
- ENUM_Stats: Strength, Agility, Intelligence, etc. (10-12 stats)
- ENUM_Faction: ThievesGuild, MagesCollege, Companions, etc
- ENUM_ClueTriggerType: OnItemPickedUp, OnItemUsed, OnNPCInteraction, OnZoneEntered, OnQuestProgress
- ENUM_FlagType: WorldState, StoryProgress, TemporaryEvent, MissionDifficulty, UnlockInteraction, RelationshipChange
- ENUM_RelationshipType: NPC, Group, Faction
## **Estructuras Definidas**
1. **STR_QuestInfo**
   QuestID (Name), QuestType (ENUM_QuestType), Faction (ENUM_Faction), Status (ENUM_QuestStatus), Objectives (array<STR_Objective>), Clues (array<STR_Clue>), Branches (array<STR_QuestBranch>), QuestEndings (array<STR_QuestEnding>), FailureConditions (array<STR_FailureCondition>), Rewards (STR_Reward), Prerequisites (array<STR_Prerequisite>), RequireAllRequisites? (Bool), ChainProgress(array<STR_ChainProgress>), RecurrenceSettings (STR_Recurrence), TimeLimit (Float), NPCMissionConfig (STR_NPCMissionConfig), UIData (STR_UIData), MissionFlags (array<STR_MissionFlags>)
2. **STR_Objective**
   ObjectiveID (Name), Type (ENUM_ObjectiveType), TargetTag (Name), Quantity (Int32), isHidden (Bool), Branches (Array<STR_QuestBranch>), CurrentProgress (Int32)
3. **STR_QuestBranch**
   BranchID (Name), Steps (array<Name>), ActivationCondition (array<STR_Prerequisite>), OutcomeFlags (array<STR_MissionFlags>)
4. **STR_Clue**
   ClueID (Name), TriggerType (ENUM_ClueTriggerType),  TriggerTarget (Name), LinkedBranch (Name), HiddenDescription (Text), ClueText (Text), AutoAddToJournal? (Bool)
5. **STR_QuestEnding**
   EndingID (Name), TriggerCondition (array<STR_Prerequisite>), OverrideRewards? (Bool), Rewards (STR_Reward), OutcomeFlags (array<STR_MissionFlags>), QuestStatus (ENUM_QuestStatus)
6. **STR_FailureCondition**
   Type (ENUM_Failure), TargetTag (Name)
7. **STR_Reward**:  
   Items (Map<Name, Int32>), Money (Int32), XP (Float), Unlocks (Array<Name>)
8. **STR_Prerequisite**:  
   Type (ENUM_PrerequisiteType), TargetID (Name), Quantity (Int32).
9. **STR_ChainProgress**
   MissionChainID (Name), ChainIndex (Int32), RequiredItems (array<STR_RequiredItem>), IsSkippable? (Bool), UnlockEvent (Name)
10. **STR_Recurrence**
   IsRecurrent (Bool), RespawnTime (Float), MaxRecurrences (Int32), CurrentRecurrences (Int32)
11. **STR_NPCMissionConfig**
   MissionGiver (array<STR_MissionGiver>), MissionTaker (STR_MissionTaker)
12. **STR_MissionGiver**
    NPCTag (Name), DialogueID (Name), AdditionalRewards (Map<Name, Int32>)
13. **STR_MissionTaker**
    AllowedNPC (array<Name>), RequiredStats (Map<ENUM_Stats, Int32>), RequiredEquipment (array<Name>), MaxParticipants (Int32), AllowMultipleNPC? (Bool)
14. **STR_UIData**
    QuestName (Text), QuestDescription (Text), QuestIcon (Texture2D), ObjectiveDescription (array<Text>), CustomMessage (Text)
15. **STR_MissionFlags**
    Category (ENUM_FlagType), Flags (array<Name>)
16. **STR_RequiredItems**
    ItemID (Name), Quantity (Int32), IsConsumable? (Bool), IsOptional? (Bool)
17. **STR_TargetObjectiveLink**
   TargetID (Name), ObjectiveID (Name), QuestID (Name), Type (ENUM_ObjectiveType)
>¿Necesitas más detalles sobre alguna estructura o enum? Responde con "sí" o "no". Si es "sí", especifica cuál.  
>Se creo la datatable `DT_QuestInfo`
## **Etapa Actual: QuestTakerComponent**
1. Crear blueprint object `QuestTakerComponent`
2. Funcion completar misiones
3. Comprobar funciones  
4. Recibir Recompensas
_Integración con inventario_
### **Desarrollo actual**
- QuestTakerComponent:
ActorComponent para gestionar misiones activas, completadas y su progreso.
1. **Variables**:
QuestLog: Array<STR_QuestInfo>: Copias modificables de misiones aceptadas.
ActiveQuestIDs: Array<Name>: IDs de misiones en curso.
CompletedQuestIDs: Array<Name>: IDs de misiones finalizadas.
TargetObjectiveLinks: Array<STR_TargetObjectiveLink>: Vincula objetivos con eventos del juego (ej: recolectar un ítem).
2. **Funciones**
- **AcceptQuest**
Propósito: Aceptar una misión validando prerrequisitos e inicializando objetivos.
Inputs:
QuestID (Name): ID único de la misión.
Outputs:
Success (Bool): Indica si la misión se aceptó correctamente.
Flujo:
Buscar QuestID en DT_QuestInfo.
Validar prerrequisitos con CheckPrerequisites.
Crear copia modificable de STR_QuestInfo para el QuestLog.
Vincular objetivos con eventos del juego (TargetObjectiveLinks).
Añadir misión a ActiveQuestIDs y QuestLog.
- **CheckPrerequisites**
Propósito: Verificar si el jugador cumple los requisitos para aceptar una misión.
Inputs:
CheckQuestInfo (STR_QuestInfo): Datos de la misión a validar.
Outputs:
Success (Bool): True si se cumplen todos los prerrequisitos.
Flujo:
Validar misiones completadas (QuestCompleted).
Verificar items en inventario (ItemOwned).
Comprobar recetas aprendidas (RecipeLearned).
Aplicar lógica AND/OR según RequireAllRequisites.
- **UpdateObjectiveProgress**
Propósito: Actualizar el progreso de un objetivo vinculado a un evento del juego.
Inputs:
TargetID (Name): ID del objetivo (ej: "RedApple").
Delta (Int32): Cantidad a sumar/restar (+1 al recolectar, -1 al usar).
Type (ENUM_ObjectiveType): Tipo de objetivo (Collect, Kill, etc.).
Outputs: Ninguno explícito (actualiza internamente QuestLog).
Flujo:
Buscar en TargetObjectiveLinks los objetivos vinculados a TargetID y Type.
Actualizar CurrentProgress en la misión correspondiente.
Disparar eventos OnObjectiveUpdated o OnObjectiveCompleted.
- **FindQuestInLog**
Propósito: Buscar una misión en el QuestLog.
Inputs:
QuestID (Name).
Outputs:
Found (Bool).
Index (Int32): Posición en el array QuestLog.
QuestInfo (STR_QuestInfo): Copia de los datos de la misión.
Flujo: Iteración secuencial con Break al encontrar coincidencia.
- **CheckAllObjectivesCompleted**
Propósito: Determinar si todos los objetivos de una misión están completos.
Inputs:
QuestInfo (STR_QuestInfo).
Outputs:
Bool: True si CurrentProgress >= Quantity para todos los objetivos.
Flujo: Iterar cada objetivo y comparar progreso vs. cantidad requerida.
- **CompleteQuest**
Propósito: Finalizar misión, otorgar recompensas y actualizar estados.
Inputs:
QuestID (Name).
Outputs:
Success (Bool).
Flujo:
Validar existencia y objetivos completos.
Eliminar items de objetivos Collect:
Si es stackable → RemoveStackableItems.
Si no es stackable → RemoveNonStackableItems.
Otorgar recompensas:
Items: Añadir al inventario (manejar overflow).
Dinero: PlayerState.Money += Reward.Money.
XP: PlayerState.AddXP(Reward.XP).
Actualizar estado de la misión a Completed.
- **FindItemInInventory**
Propósito: Calcular la cantidad total de un ítem en el inventario.
Inputs:
ItemID (Name).
Outputs:
Total (Int32): Cantidad total del ítem.
Index (Int32): Primer slot donde se encontró el ítem (-1 si no existe).
Flujo: Iterar todos los slots del inventario y sumar cantidades.
- **RemoveStackableItems**
Propósito: Eliminar una cantidad específica de ítems apilables.
Inputs:
ItemID (Name).
Quantity (Int32).
Outputs:
Success (Bool).
QuantityRemaining (Int32): Cantidad no eliminada.
Flujo:
Filtrar slots con el ítem y ordenar descendente.
Eliminar de los slots con mayor cantidad primero.
Actualizar inventario real usando índices originales.
- **RemoveNonStackableItems**
Propósito: Eliminar una cantidad de ítems no apilables (por slots completos).
Inputs:
ItemID (Name).
Quantity (Int32).
Outputs:
Success (Bool).
Flujo:
Iterar slots en orden inverso.
Eliminar slots completos hasta alcanzar Quantity.
3. **EventDispatchers lanzados**
- **OnQuestPrerequisites**: Call si no cumple prerrequisitos
- **OnAcceptedQuest**: Call para notificar que aceptó una misión
- **OnObjectiveUpdated**: Call para notificar actualización de objetivo
- **OnQuestCompleted**: Call para notificar una misión completadas
- **OnItemPicked**:  Call para notificar un item de misión recogido.  Assign conectado a `Update Objective Progress`
4. **Optimizaciones Clave**
**Early Exit**: Rompe loops al encontrar elementos para mejorar rendimiento.
5. **Pendientes**
- **Agregar función:** 
   - **TransferItemsToNPC**
Propósito: Transferir ítems del jugador al inventario de un NPC.
Inputs (Planeados):
ItemID (Name).
Quantity (Int32).
NPCTarget (STR_MissionGiver).
Outputs (Planeados):
Success (Bool).
Lógica Tentativa:
Verificar espacio en inventario del NPC.
Remover items del jugador (RemoveStackableItems/RemoveNonStackableItems).
Añadir items al NPC (si no hay espacio, spawnear cerca). 
> Agregar en desarrollo de QuestGiverComponent
5. **Pasos a seguir**
- construcción de NPC
   - Actor básico para NPC humanos `BP_NPCBase`
   - Estructura de datos del NPC (Nombre, ID, etc) 
   **Temas clave**: 
   - La apareciencia base del NPC (cabeza, tono de piel) se declara en la estructura?
   - **Todos los NPC no enemigos, pueden ser `QuestGiver` y `QuestTaker` al mismo tiempo _pero todavía no se vana declarar las condiciones_**
   - **Todos los NPC cuentan con `InventoryComponent` y `EquipmentComponent`**
   - **Para NPC enteramente enemigos (Bandios, invasores, etc), es necesario una estructura/datatable independientes? _Tambien incluye otros enemigos  como animales y monstruos?_**
- Quest giver _Configuración de los NPCs que dan las misiones_
   - Variables
   - Funciones principales
   - subfunciones
   - integración con sistema actual de interacción
- GUI Quest y QuestLog de jugador _Gui de Test para el QuestGiver_
- Sistema de interacción avanzado _Recolección de items, apertura de inventarios (cofres/NPC amigos, loot, Tiendas,), hablar/interactuar con NPC_
   **Temas clave**
   - Como se va a llevar a cabo el registro e interacción de los `NPC QuestTaker`?
## **Etapas futuras (Orden de prioridad)**:  
- construcción de lista de MissionFlags/OutcomeFlags, IDs UnlockEvent/UnlockInteraction, y estructuras de interacción para Branches y finales de las misiones 
- Configuración de tipos de misiones (Recolección, asesinato, escolta, etc) Diseño modular para posible expansión futura
- Establecer fracaso, bloqueo y desbloqueo de misiones
- Multiples QuestGiver dan la misma misión
- NPC que realizan las misiones  _que NPCs pueden hacer las misiones_
- Fracaso o exito de las misiones hechas por los NPCs
- Misiones procedurales/generativas
## Misión de Test: 
### Recolección de Manzanas  _Ya está crada_
- **Tipo**: Recolección (ENUM_ObjectiveType: Collect).  
- **Objetivo**: Recoger 1 manzana (ItemID: "RedApple", Quantity: 1).  
- **Recompensa**: 2 manzanas (Items: {"RedApple": 2}).  
- **Configuración**:  
  - QuestID: "TQ001_CollectApple".  
  - QuestType: MainQuest (ENUM_QuestType: MainQuest).  
  - Status: Available (ENUM_QuestStatus: Available).  
### Consigue una espada  _Ya está crada_
- **Tipo**: Recolección (ENUM_ObjectiveType: Collect).  
- **Objetivo**: Recoger 1 manzana (ItemID: "RedApple", Quantity: 1).  
                Recoger 1 espada (ItemID: "Sword", Quantity: 1)
- **Recompensa**: 2 manzanas (Items: {"RedApple": 2}).  
- **Configuración**:  
  - QuestID: "TQ002_GetTheSword".  
  - QuestType: MainQuest (ENUM_QuestType: MainQuest).  
  - Status: Available (ENUM_QuestStatus: Available).  
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
   - **Sin Maps**: Usa arrays y structs para evitar complejidad en Blueprints.
