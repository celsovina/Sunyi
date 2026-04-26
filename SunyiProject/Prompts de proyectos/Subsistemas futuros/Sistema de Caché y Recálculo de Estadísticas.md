# **Resumen Funcional: Sistema de Caché y Recálculo de Estadísticas**
---
## 1. El Problema a Resolver

Necesitamos un sistema que permita que las estadísticas de un ítem sean modificadas por múltiples fuentes (calidad de fabricación, mejoras, buffs/debuffs) y que, al mismo tiempo, sea resiliente a parches de balanceo del juego (cuando un stat base en `DT_ItemInfo` cambia) y mantenga un alto rendimiento en tiempo de ejecución.

## 2. La Arquitectura de "Doble Capa"

La solución es un sistema de "doble capa" que reside en la `STR_InstanceData` del ítem y separa el "porqué" del "qué".

* **Capa 1: El Historial de Modificadores (`AppliedModifiers`)**
    * **Propósito:** Actúa como un "libro de contabilidad" o un historial inmutable de cada modificación que ha recibido la instancia.
    * **Datos Almacenados:** Guarda la **intención original** del modificador. Cada entrada en esta lista (`TArray<STR_Modifications>`) almacena de dónde vino el bono (`ModifierSourceTag`), qué stat afecta (`StatToModify`), de qué tipo es (`ModificationType`: Aditivo, Porcentaje de Base, etc.) y su valor (ej. `0.20` para un 20%).
    * **Función:** Esta lista es la **fuente de la verdad** para los cálculos. Permite recalcular los stats desde cero en cualquier momento.

* **Capa 2: Las Sobrescrituras de Stats (`StatsOverrides`)**
    * **Propósito:** Actúa como un **"caché persistente"**. Almacena el **resultado final y pre-calculado** de `StatBase + todos los modificadores aplicados`.
    * **Datos Almacenados:** Es una lista simple de `{StatTag, FinalValue}`.
    * **Función:** El 99% del tiempo, el juego lee directamente de esta lista para obtener el valor de un stat. Es una operación de búsqueda directa y extremadamente rápida, lo que garantiza el máximo rendimiento durante el gameplay.

## 3. El Flujo de Verificación y Recálculo

La "magia" del sistema reside en cuándo y cómo se actualiza la Capa 2 (`StatsOverrides`) a partir de la Capa 1 (`AppliedModifiers`).

* **Flujo 1: Al Aplicar una Nueva Modificación (en tiempo de juego)**
    1.  Se usa una piedra de afilar sobre una espada.
    2.  Se añade un nuevo `STR_Modifications` a la lista `AppliedModifiers` de la espada.
    3.  El sistema **inmediatamente** dispara un recálculo para el stat de daño.
    4.  Lee el `Daño Base` de `ItemInfo`, recorre la lista **completa** de `AppliedModifiers`, calcula el nuevo daño final, y actualiza el valor en `StatsOverrides`.

* **Flujo 2: Al Cargar la Partida (Verificación de Parche)**
    1.  El juego se inicia. Se necesita un mecanismo para detectar si los datos base han cambiado. Una forma simple es almacenar un "número de versión" o un "hash" de la `DT_ItemInfo` en el archivo de guardado.
    2.  Al cargar, el sistema compara el hash guardado con el hash de la `DT_ItemInfo` actual.
    3.  **Si son diferentes (hubo un parche):** El sistema activa un proceso de "re-base" o "migración". Para cada ítem instanciado en la partida, **invalida sus `StatsOverrides`** y los recalcula desde cero usando los **nuevos valores base** de la `DT_ItemInfo` y la lista `AppliedModifiers` guardada en la instancia.
    4.  **Si son iguales (no hubo parche):** No se hace nada. El juego confía en los valores de `StatsOverrides` guardados, asegurando tiempos de carga rápidos.

Este sistema garantiza tanto el máximo rendimiento en el juego como la capacidad de balancear y actualizar los stats base de los ítems sin "romper" o devaluar los ítems que los jugadores ya han mejorado.