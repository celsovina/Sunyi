# **Resumen de Arquitectura Final y Completa: `STR_ItemInfo`**
---
## 1. Enums de Soporte del Sistema

* #### **`ENUM_Rarity`**:
    * **Descripción:** Define la rareza de un ítem, que afecta a su valor y, potencialmente, a sus estadísticas.
    * **Valores:** `Common`, `Uncommon`, `Rare`, `Epic`, `Legendary`.

* #### **`ENUM_ContextMatching`**:
    * **Descripción:** Define la lógica de comparación para un `GameplayTagContainer`.
    * **Valores:** `MatchAny` (OR), `MatchAll` (AND).

* #### **`ENUM_ComparisonMethod`**:
    * **Descripción:** Define los operadores para comparaciones numéricas.
    * **Valores:** `EqualTo`, `NotEqualTo`, `GreaterThan`, `GreaterThanOrEqualTo`, `LessThan`, `LessThanOrEqualTo`.

* #### **`ENUM_EffectApplicationMethod`**:
    * **Descripción:** Define cómo y cuándo se aplica un efecto de stat de un ítem.
    * **Valores:** `OnEquip_Passive`, `OnConsume_Instant`, `OnConsume_Timed`, `OnConsume_Permanent`.

* #### **`ENUM_EffectConditionType`**:
    * **Descripción:** Define si una condición comprueba un `Tag` o un `Stat` numérico.
    * **Valores:** `HasTag`, `StatCheck`.

* #### **`ENUM_IngredientRequirementType`**:
    * **Descripción:** Define si un ingrediente de receta se busca por su `ItemID` específico o por una `PropertyTag` genérica.
    * **Valores:** `MatchPropertyTag`, `MatchSpecificItemID`.

* #### **`ENUM_BuildingSnapType`**:
    * **Descripción:** Define el comportamiento de una pieza en la cuadrícula de construcción.
    * **Valores:** `Foundation`, `Wall`, `Floor_Ceiling`, `Roof`, `Stairs`, `Inset` (para puertas/ventanas), `Freeform`.

* #### **`ENUM_TrapTrigger`**:
    * **Descripción:** Define cómo se activa una trampa.
    * **Valores:** `OnProximity`, `OnDirectContact`, `OnTimer`, `OnSignal`, `OnDamageReceived`, `OnTripwire`.

---
## 2. Estructuras de Soporte Reutilizables

* #### **`STR_ContextRelation`**:
    * **Descripción:** Define una regla para el sistema `MultiUse`, determinando qué acciones están disponibles en qué contextos.
    * **`RequiredContexts`** (`FGameplayTagContainer`): Tags necesarios para que la regla se active.
    * **`MatchingContext`** (`ENUM_ContextMatching`): Lógica (Y/O) para evaluar los `RequiredContexts`.
    * **`GrantedActions`** (`FGameplayTagContainer`): Tags de las acciones que se habilitan.

* #### **`STR_FXCondition`**:
    * **Descripción:** Define una única condición reutilizable para un efecto o un hito de destino.
    * **`ConditionType`** (`ENUM_EffectConditionType`): El tipo de condición a verificar.
    * **`RequiredTag`** (`FGameplayTag`): El tag que el personaje/mundo debe tener.
    * **`StatToCompare`** (`FGameplayTag`): El tag del stat a comparar.
    * **`ComparisonMethod`** (`ENUM_ComparisonMethod`): El operador de comparación.
    * **`ComparisonValue`** (`float`): El valor numérico contra el cual se compara.

* #### **`STR_StatEffect`**:
    * **Descripción:** Define un efecto simple de modificación de stat o aplicación de tag.
    * **`EffectTag`** (`FGameplayTag`): El stat o estado a aplicar.
    * **`ApplicationMethod`** (`ENUM_EffectApplicationMethod`): Cómo y cuándo se aplica el efecto.
    * **`Value`** (`float`): Valor numérico de la modificación.
    * **`Duration`** (`float`): Duración en segundos del efecto.
    * **`ActivationConditions`** (`TArray<STR_FXCondition>`): Condiciones que deben cumplirse para que este efecto se active.
    * **`ConditionLogic`** (`ENUM_ContextMatching`): Lógica (Y/O) para evaluar las condiciones.

* #### **`STR_OnConsumeEffect`**:
    * **Descripción:** Define un efecto complejo que necesita lógica de Blueprint, actuando como un "payload" de datos.
    * **`EffectActor`** (`TSubclassOf<BP_Effect>`): El actor que se spawnea para ejecutar la lógica.
    * **`DataPayload_ID`** (`FName`): Campo genérico para pasar IDs (RecipeID, QuestID, etc.).
    * **`DataPayload_Text`** (`FText`): Campo para pasar bloques de texto a un visor de documentos.
    * **`DataPayload_Tag`** (`FGameplayTagContainer`): Campo para pasar `GameplayTags`.
    * **`DataPayload_Integer`** (`int32`): Campo para pasar valores enteros.
    * **`DataPayload_Float`** (`float`): Campo para pasar valores flotantes.

* #### **`STR_CraftingIngredientSlot`**:
    * **Descripción:** Define un requisito de ingrediente flexible para cualquier sistema de fabricación.
    * **`RequirementType`** (`ENUM_IngredientRequirementType`): Selector para `MatchPropertyTag` o `MatchSpecificItemID`.
    * **`AcceptedIngredientProperty`** (`FGameplayTag`): El tag de propiedad del ingrediente genérico.
    * **`SpecificItemID`** (`FName`): El `ItemID` específico del ingrediente.
    * **`RequiredQuantity`** (`int32`): La cantidad necesaria.

---
## 3. Módulos de Datos de `STR_ItemInfo`

* #### **`STR_UiItemData`**:
    * **Descripción del Subsistema:** Almacena toda la información necesaria para representar el ítem en la Interfaz de Usuario.
    * **`DisplayName`** (`FText`): Nombre del ítem visible al jugador.
    * **`DisplayDescription`** (`FText`): Descripción para tooltips e inspección.
    * **`RarityType`** (`ENUM_Rarity`): Rareza base del ítem, para colorear el fondo, el borde, etc.
    * **`Icon`** (`TSoftObjectPtr<UTexture2D>`): Icono 2D para el slot de inventario.
    * **`UIMesh`** (`TSoftObjectPtr<UStaticMesh>`): Malla 3D para la ventana de previsualización del ítem.
    * **`UIMeshSize`** (`float`): Factor de escala para la `UIMesh`.

* #### **`STR_WorldRepresentation`**:
    * **Descripción del Subsistema:** Define la apariencia visual del ítem en el mundo del juego.
    * **`WorldMesh`** (`TSoftObjectPtr<UStaticMesh>`): La malla que se muestra cuando el ítem está en el suelo.
    * **`WorldMeshSize`** (`FVector`): Escala de la `WorldMesh`.
    * **`SK_Equipment`** (`TSoftObjectPtr<USkeletalMesh>`): La malla esquelética que se adjunta al personaje si el ítem se equipa y tiene una apariencia única.

* #### **`STR_InventoryBehavior`**:
    * **Descripción del Subsistema:** Define las reglas básicas de cómo se comporta el ítem dentro de un inventario.
    * **`IsStackable?`** (`bool`): Si `true`, múltiples ítems de este tipo pueden ocupar un solo slot.
    * **`MaxStackSize`** (`int32`): Si es apilable, este es el número máximo de ítems por stack.

* #### **`STR_MultiUseData`**:
    * **Descripción del Subsistema:** Gestiona el sistema de acciones contextuales, permitiendo que un ítem tenga diferentes "usos" según la situación.
    * **`AcceptMultiUse?`** (`bool`): Activa o desactiva este sistema para el ítem.
    * **`ContextRules`** (`TArray<STR_ContextRelation>`): Lista de reglas que definen qué acciones están disponibles en qué contextos.
    * **`bAcceptRandom`** (`bool`): Si hay múltiples acciones disponibles, permite que el sistema elija una al azar.

* #### **`STR_EquipmentBehavior`**:
    * **Descripción del Subsistema:** Define el comportamiento de un ítem al ser equipado en un personaje.
    * **`GrantedInventorySlots`** (`int32`): Número de slots de inventario que este equipo otorga (ej. una mochila).
    * **`StorageRestrictionTags`** (`FGameplayTagContainer`): Tags de ítems que NO se pueden guardar en el inventario concedido.

* #### **`STR_EconomicData`**:
    * **Descripción del Subsistema:** Define una regla de precio para un ítem, permitiendo valores dinámicos por región o contexto. `STR_ItemInfo` puede tener un array de estas reglas.
    * **`TransactionContexts`** (`FGameplayTagContainer`): Contextos donde aplica la regla (Tienda Normal, Mercado Negro).
    * **`RegionContexts`** (`FGameplayTagContainer`): Regiones geográficas donde aplica la regla.
    * **`AcceptedPrices`** (`TMap<ENUM_CurrencyType, int32>`): El precio del ítem en diferentes tipos de moneda.

* #### **`STR_ItemLifeData`**:
    * **Descripción del Subsistema:** Gestiona el ciclo de vida, la durabilidad y el deterioro del ítem.
    * **`HasLife?`** (`bool`): Activa o desactiva este sistema.
    * **`MaxUses`** (`int32`): Durabilidad máxima por número de usos.
    * **`UseTransforms`** (`TMap<float, FName>`): Reglas de transformación por % de uso restante.
    * **`UseBreakLoot`** (`TMap<FName, int32>`): Loot generado al romperse por uso.
    * **`MaxDecayTime`** (`float`): Durabilidad máxima por tiempo (en segundos).
    * **`TimeStateChanges`** (`TMap<float, FGameplayTag>`): Reglas de cambio de estado por % de tiempo restante.
    * **`TimeDecayLoot`** (`TMap<FName, int32>`): Loot generado al descomponerse por tiempo.

* #### **`STR_StatData` (Módulo de Efectos Unificado)**:
    * **Descripción del Subsistema:** El motor central para todos los efectos del ítem, tanto simples como complejos. Absorbe la funcionalidad de Aprendizaje, Desbloqueo y Objetos Arrojadizos.
    * **`EffectRules`** (`TArray<STR_StatEffect>`): Lista de efectos simples (modificar stats, aplicar tags).
    * **`OnConsumeEffects`** (`TArray<STR_OnConsumeEffect>`): Lista de efectos complejos (iniciar misiones, aprender recetas, spawnear proyectiles).

* #### **`STR_CraftingOriginData`**:
    * **Descripción del Subsistema:** Enlaza un ítem con las recetas que pueden crearlo. Es una optimización para búsquedas rápidas.
    * **`CraftingRecipeIDs`** (`TArray<FName>`): Contiene los IDs de las recetas en `DT_Recipes` que producen este ítem.

* #### **`STR_ItemModificationData`**:
    * **Descripción del Subsistema:** Define cómo un ítem ya existente puede ser modificado.
    * **`RepairDetails`** (`STR_RepairItem`): Detalles para reparar este ítem.
    * **`UpgradeDetails`** (`STR_UpgradeItem`): Detalles para mejorar este ítem.
    * **`DisassembleDetails`** (`STR_DisassembleItem`): Detalles para desmantelar este ítem.

* #### **`STR_EnchantmentData`**:
    * **Descripción del Subsistema:** Define la capacidad y las reglas de un ítem para ser encantado con gemas.
    * **`IsEnchantable?`** (`bool`): Si el ítem puede ser encantado.
    * **`MaxEnchantmentSlots`** (`int32`): Número de ranuras para gemas.
    * **`AcceptedEnchantmentTags`** (`FGameplayTagContainer`): Tipos de encantamientos que acepta.
    * **`RequiredEnchantingLevel`** (`int32`): Habilidad necesaria para encantar este ítem.

* #### **`STR_TrapData`**:
    * **Descripción del Subsistema:** Define las propiedades mecánicas de un ítem de tipo trampa.
    * **`PlacedTrapActorClass`** (`TSubclassOf<AActor>`): El actor a spawnear al colocar la trampa.
    * **`ValidTargetTags`** (`FGameplayTagContainer`): Tags que un objetivo debe tener para activar la trampa.
    * **`IgnoreTargetTags`** (`FGameplayTagContainer`): Tags que impedirán la activación.
    * **`AcceptedBaitTags`** (`FGameplayTagContainer`): Cebos compatibles.
    * **`TriggerType`** (`ENUM_TrapTrigger`): Cómo se activa la trampa.
    * **`TriggerDelay`** (`float`): Retraso en segundos tras la activación.

* #### **`STR_LightingData`**:
    * **Descripción del Subsistema:** Define las propiedades de los ítems de iluminación equipables.
    * **`EquippedActorClass`** (`TSubclassOf<AActor>`): El actor a spawnear en la mano del personaje.
    * **`LightColor`** (`FLinearColor`): El color de la luz.
    * **`bCastsShadows?`** (`bool`): Si la luz proyecta sombras.
    * **`ExtinguishingConditions`** (`FGameplayTagContainer`): Tags de condiciones que apagan la luz.

* #### **`STR_ConstructionData`**:
    * **Descripción del Subsistema:** Define las propiedades mecánicas de las piezas de construcción.
    * **`PlacedActorClass`** (`TSubclassOf<AActor>`): El actor a spawnear en el mundo.
    * **`SnapType`** (`ENUM_BuildingSnapType`): Cómo se conecta a la cuadrícula.
    * **`bRequiresFoundation?`** (`bool`): Si necesita ser colocado sobre un cimiento.

---
## 4. Estructura Raíz Final: `STR_ItemInfo`

*Esta es la estructura final que se usará en la `DataTable` `DT_ItemInfo`.*

* **`ItemID`** (`FName`): El ID único del ítem.
* **`ItemTypeTags`** (`FGameplayTagContainer`): Las categorías generales del ítem.
* **`PropertyTags`** (`FGameplayTagContainer`): Las propiedades inherentes del ítem.
* **`EmpiricalDiscoveryPaths`** (`TArray<FGameplayTagContainer>`): **(Ajuste Final)** Lista de conjuntos de tags que actúan como "condiciones pasivas" que sistemas externos (actores del mundo, gestores de eventos) pueden leer para disparar un descubrimiento o evento.
* **`UiData`** (`STR_UiItemData`)
* **`WorldRepresentation`** (`STR_WorldRepresentation`)
* **`InventoryBehavior`** (`STR_InventoryBehavior`)
* **`MultiUseData`** (`STR_MultiUseData`)
* **`EquipmentBehavior`** (`STR_EquipmentBehavior`)
* **`EconomicRules`** (`TArray<STR_EconomicData>`)
* **`ItemLifeData`** (`STR_ItemLifeData`)
* **`StatData`** (`STR_StatData`)
* **`CraftingOriginData`** (`STR_CraftingOriginData`)
* **`ItemModificationData`** (`STR_ItemModificationData`)
* **`EnchantmentData`** (`STR_EnchantmentData`)
* **`TrapData`** (`STR_TrapData`)
* **`LightingData`** (`STR_LightingData`)
* **`ConstructionData`** (`STR_ConstructionData`)