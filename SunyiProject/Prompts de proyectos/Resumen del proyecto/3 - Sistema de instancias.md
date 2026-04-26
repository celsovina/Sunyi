# **Resumen de STR_InstanceData**
---
## 1. Vistazo Conceptual

`STR_InstanceData` es la estructura que contiene **toda la información que hace que un ítem sea una instancia única**. No se crea para todos los ítems, solo para aquellos que se desvían de su plantilla `STR_ItemInfo`. Se gestiona de forma descentralizada: cada actor con un inventario (jugador, PNJ, cofre) tiene un `InstanceManagementComponent` que contiene un `Array` de las instancias de los ítems que posee. El `InstanceID` (GUID) en el `STR_Slot` sirve como clave para encontrar la entrada correspondiente en este array local.

---
## 2. ENUMS requeridos

* **`ENUM_ModificationType`**:
    * **Propósito:** Define cómo se calcula un modificador de stat en `STR_Modifications`.
    * **Valores:** `MOD_Plain` (Aditivo), `MOD_BasePercent` (Porcentaje de la Base), `MOD_TotalPercent` (Porcentaje del Total).

* **`ENUM_ModificationOperation`**:
    * **Propósito:** Define si un modificador es un bono o una penalización.
    * **Valores:** `MOP_Increase` (Suma), `MOP_Decrease` (Resta).

---
## 3. Estructura Final y de Soporte

### **Estructuras de Soporte Requeridas**

* **`STR_Modifications`**:
    * `ModifierSourceTag` (`FGameplayTag`): Origen del modificador (ej. `Buff.Whetstone`).
    * `StatToModify` (`FGameplayTag`): Estadística afectada.
    * `ModificationType` (`ENUM_ModificationType`): `MOD_Plain`, `MOD_BasePercent`, `MOD_TotalPercent`.
    * `Operation` (`ENUM_ModificationOperation`): `MOP_Increase` o `MOP_Decrease`.
    * `Value` (`float`): Valor de la modificación (siempre positivo).

* **`STR_Instance_TrackedStat`**:
    * `StatTag` (`FGameplayTag`): El tag del contador a rastrear.
    * `CurrentValue` (`int32`): El valor actual del contador.

* **`STR_Instance_TimedEffect`**:
    * `EffectTag` (`FGameplayTag`): El tag que identifica el efecto temporal.
    * `TimeRemaining` (`float`): Segundos restantes.

* **`STR_ContextualDataLink`**:
    * `ContextTag` (`FGameplayTag`): El "verbo" de la relación (ej. `Quest.Objective.DeliverTo`).
    * `TargetIDs` (`TArray<FName>`): Lista de `IDs` de objetivos específicos.
    * `TargetProperties` (`FGameplayTagContainer`): Contenedor de `Tags` de objetivos genéricos.

* **`STR_ActiveDestiny`**:
    * `DestinyID` (`FName`): El ID del destino activo.
    * `DestinyDiscovered?` (`bool`): Si el jugador ya es consciente de este destino.

### **Estructura Principal: `STR_InstanceData`**

* **`InstanceID` (`GUID`):** El identificador único y global.

#### **Datos de Descripción**
* **`CustomName` (`FText`):** Nombre personalizado (solo en creación).
* **`CustomDescription` (`FText`):** Descripción personalizada.
* **`RarityOverride` (`ENUM_Rarity`):** Rareza específica de la instancia.

#### **Datos de Stats y Estado**
* **`InstanceTags` (`FGameplayTagContainer`):** Tags de estado permanentes (Robado, etc.).
* **`ItemID_StateOverride` (`FName`):** ID a usar para apariencia/stats si el ítem está degradado.
* **`SocketedGems` (`TArray<FName>`):** Lista de gemas/encantamientos engarzados.
* **`AppliedModifiers` (`TArray<STR_Modifications>`):** El historial de todas las modificaciones de estadísticas aplicadas.
* **`StatsOverrides` (`TArray<STR_Instance_StatModifier>`):** El "caché persistente" con los valores finales y pre-calculados de los stats.
* **`ActiveTimedEffects` (`TArray<STR_Instance_TimedEffect>`):** Lista de buffs/debuffs temporales sobre el ítem.

#### **Ciclo de Vida**
* **`CurrentUses` (`int32`):** Usos restantes.
* **`CurrentDecayTime` (`float`):** Tiempo de descomposición restante.

#### **Progresión e Historia**
* **`ActiveDestinies` (`TArray<STR_ActiveDestiny>`):** Lista de los destinos que esta instancia está siguiendo.
* **`TrackedStats` (`TArray<STR_Instance_TrackedStat>`):** Lista de los contadores numéricos para los hitos.
* **`ContextualData` (`TArray<STR_ContextualDataLink>`):** Lista de las relaciones complejas del ítem.

#### **Creación y Propiedad**
* **`OwnerID` (`FName`):** El dueño original.
* **`CrafterID` (`FName`):** El creador de la instancia.