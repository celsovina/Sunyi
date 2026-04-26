# **Contexto de Desarrollo: Sistema de Inventario RPG-Survival en Unreal Engine 5.3 (100% Blueprints)**

---

## **Reglas de la Sesión**

⚠️**CRÍTICO: NO DES NINGUNA RESPUESTA, NI SIQUIERA UN ACUSE DE RECIBO. ESPERA LA INSTRUCCIÓN ESPECÍFICA _"Iniciemos"_ ANTES DE RESPONDER. CUALQUIER RESPUESTA ANTES DE ESA INSTRUCCIÓN SERÁ CONSIDERADA UN ERROR.**⚠️
⚠️ **INSTRUCCIONES CRÍTICAS (LEER ANTES DE RESPONDER)** ⚠️

1.  **Confirmación Inicial**: No aplica para exportación.
2.  **Confirmación Visual**: Todas las respuestas deben comenzar con ✅ si se han aplicado las instrucciones, si no usar el emoji ❌.
3.  **Token Management**: Al final de cada mensaje, lanzar alerta de consumo de tokens.
          - El formato debe ser exactamente:  
         **Hemos consumido _≈[tokens consumidos en el mensaje] tokens_, restantes aprox: [tokens restantes de la sesión]. Tokens totales aprox: [tokens maximos para la sesión]**
4.  **Formato de Respuestas**: Cortas y concretas. Uso de **bullet points**, **negritas**. Diagramas de secuencia si es necesario para ilustraciones. Sin imágenes adjuntas.
5.  **Respuestas con opiniones**: Incluir: **Qué haría**, **Cómo lo haría**, y **Justificación**.
6.  **Foco Estricto**:
        _ Responde solo a lo preguntado, a menos que se solicite expansión.
        _ ⚠️ **CRITICO: LAS IMAGENES NO SON UTILES. ENFOCATE EN RESPUESTAS TEXTUALES.**⚠️
7.  **Respuestas por etapas**: Responder solo al alcance de la etapa actual y estructurar la sección actual de desarrollo en puntos y sub puntos más pequeños, de acuerdo a la información especificada.
8.  **Especificaciones técnicas**:
    - **Prohibición de C++**: Solo Blueprints.
    - **NO USAR CAST**: Estrictamente prohibido. Usar interfaces.
    - **Maps permitidos solo si**: Son _read-only_ o no requieren manipulación en tiempo real.

---

## 1. **Blueprints e Interfaces Clave (Relevantes a la Discusión Reciente)**

- **`BP_ItemManager`**: Gestiona acceso a `DT_ItemInfo`.\* **`InventoryComponent`**: Gestiona un inventario (`Array<STR_Slot>`), implementa `BPI_SlotHandler`, tiene dispatcher `OnInventoryUpdated`.
- **`GI_Sunyi` (GameInstance)**:
  - Spawnea y gestiona la instancia única de `BP_TooltipStudio`.
  - Implementa `BPI_SystemAccess` para proveer acceso a managers y al `BP_TooltipStudio`.
- **`BPI_SystemAccess` (Interfaz)**:
  - Funciones: `GetItemManager`, `GetSpawnManager`, `GetTooltipStudio` (retorna `BP_TooltipStudio`).
- **`WBP_Inventory`**: UI principal de inventario.
  - Contiene la "Drop Zone" y su lógica.
  - Gestiona el array `PendingUse` (de tipo `STR_PendinToUse`).
  - Implementa la lógica de `Event SpecialZoneReport` (para añadir a `PendingUse` con anti-duplicados) y `Event CancelPendingTransaction` (para quitar de `PendingUse`).
  - En `Event Destruct`, revierte las transacciones en `PendingUse`.
  - Lógica del botón "Tirar": Usa `PendingUse` para spawnear `BP_Bag`, llama a `ClearAllInventory` en el `TargetHandler` de la Drop Zone, y limpia `PendingUse`.\* **`WBP_Slot`**: Widget para un slot individual.
  - En su función de `Tooltip Widget Binding`:
    - Obtiene `STR_ItemInfo` del `ItemManager`.
    - Prepara `STR_TooltipDisplayData` (transformando datos, ej. `Map` de stats a `Array` de `STR_ToolTipStats`, y el `ENUM_Rarity` a `Text` y `LinearColor` para el tooltip).
    - Crea `WBP_ItemTooltip`, pasándole `STR_TooltipDisplayData` vía "Expose on Spawn".
  - Lógica `OnDrop`: Llama a `SpecialZoneReport` si el destino es una zona especial; llama a `CancelPendingTransaction` si el origen es una zona especial.
  - Lógica de limpieza para el tooltip 3D: gestiona la llamada a `ClearDisplayedItem` del estudio cuando el tooltip se oculta.
- **`WBP_SlotGrid`**: Contenedor de `WBP_Slot`. Se le pasa un `InventoryComponent`.
  - Se suscribe al `OnInventoryUpdated` del `InventoryComponent` para refrescarse (`GenerateSlots`).\* **`WBP_ItemTooltip`**: Widget del tooltip.
  - Recibe `STR_TooltipDisplayData` (expuesta en spawn).
  - En `Event Construct`, llama a una función `RefreshTooltipVisuals`.
  - `RefreshTooltipVisuals` usa los datos recibidos para poblar todos los `TextBlock`, `Image`, y generar dinámicamente listas para stats y precios usando sub-widgets (`WBP_Tooltip_StatLine`, `WBP_Tooltip_PriceLine`) en `WrapBox` o `VerticalBox`.
  - Para 3D: Se comunica con `BP_TooltipStudio` para mostrar el `UIDisplayMesh`. Llama a `ClearDisplayedItem` del estudio en su `Event Destruct`.
- **`BP_TooltipStudio` (Actor)**:
  - Spawneado "on demand" y gestionado como singleton por `GI_Sunyi`.
  - Contiene `SceneCaptureComponent2D` (`ItemCaptureCamera`) que renderiza a un `RenderTarget` (`RT_TooltipView`).
  - Contiene un `StaticMeshComponent` permanente (`DisplayMeshHolder`) anclado frente a la cámara.
  - Funciones:
    - `ShowItemMesh(InMeshToDisplay, InItemTags, InDesiredScale)`: Configura `DisplayMeshHolder`, calcula `bounds` para paneo, e inicia Timelines.
    - `ClearDisplayedItem()`: Limpia el mesh y detiene Timelines.
  - Dos `Timelines`: Uno para rotación constante (`TL_ItemRotation`), otro para paneo vertical condicional (`TL_ItemPan`) de ítems largos (trigger por tag, usa `Lerp` y `GetLocalBounds`).
  - Paneo se activa si tag presente y `ItemHeight > TooltipViewHeightUU`.
- **`RT_TooltipView` (RenderTarget)**: Asset para la captura de escena.\* **`M_ToolTipView` (Material)**: Material de UI, translúcido, que usa `RT_TooltipView`. Para transparencia, usa el canal Alpha del `RenderTarget` o Luma Keying. Configuración de `SceneCaptureComponent2D` (ShowFlags, CaptureSource HDR con Alpha) es crucial para el fondo transparente.

---

## \*\*Se proporciona archivo de Tags para mayor comprensión

---

## 2. **Consideraciones de desarrollo**

⚠️ **Restricciones de diseño que deben respetarse.** ⚠️

1.  **Sin sistema de peso (`weight`)**: Limitado por slots/stack.
2.  **No es multiplayer (`NO REPLICACIÓN`)**: Single-player.
3.  **NO USAR CAST**: Prohibido. Usar interfaces.
4.  **Uso compartido (Jugador y NPCs)**: Sistemas compatibles para ambos.
5.  **Nomenclatura**: Variables internas/Struct/Enum en **Inglés** (ej. `ActionID`, `ECA_Use`). Textos visibles al usuario (ej. `ButtonLabel` en `STR_ContextActionData`) en **Español**. Booleanos como `VariableName?`.

---

## **📂 Comando de Resumen Portátil**

⚠️ **INSTRUCCIÓN ESPECIAL**:

- Si el usuario escribe **`/EXPORTAR_CONTEXTO`**, genera un resumen **DEL PROCESO FINAL TRABAJADO HASTA EL MOMENTO**:  
    1. Objetivo: Permitir copia/pega en nuevos chats o modelos sin pérdida de coherencia.
    2. El resumen debe venir en formato _markdown_ embebido en un bloque de código.
