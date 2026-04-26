# Resumen de la Discusión Arquitectónica: STR_ItemInfo vs. STR_Slot

Esta discusión se centró en determinar el lugar correcto para almacenar diferentes tipos de datos de los ítems, con el objetivo de crear un sistema robusto, escalable y fácil de mantener.

## 1. El Dilema Inicial: Datos de Instancia

El problema surgió al querer implementar estados que varían para cada ítem individual, como el estado "Robado".

* **Propuesta Inicial:** Añadir un `GameplayTagContainer` de `InstanceTags` a `STR_Slot` para manejar estados como "Robado".
* **Objeción del Usuario:** Se estableció una regla de diseño fundamental: **NO MODIFICAR `STR_Slot`**, por temor a que, al ser una estructura base, cualquier cambio pudiera causar problemas en cascada en todo el sistema. La contrapropuesta fue usar el `GameplayTagContainer` ya existente dentro de `STR_ItemInfo`.

## 2. El Principio Clave: Definición vs. Instancia

Para resolver el dilema, se estableció una distinción arquitectónica crucial, usando la analogía de un plano de construcción vs. un coche real.

* ### `STR_ItemInfo` (La Definición)
    * Se definió como el **"plano de construcción"** o la **plantilla** de un ítem.
    * Contiene datos que son **idénticos para todas las instancias** de un `ItemID` específico (su nombre, descripción, peso, stats base, rareza base, `MaxStack`, etc.).
    * Esta estructura debe considerarse **inmutable en tiempo de ejecución**. Los datos se leen del `DataTable` y no se modifican.

* ### `STR_Slot` (La Instancia)
    * Se definió como el **"objeto real"** o la instancia única de un ítem en el mundo.
    * Contiene datos que son **únicos para ese slot específico**. Ya contenía `Quantity`.
    * Se concluyó que es el lugar **arquitectónicamente correcto** para todos los datos que pueden cambiar de una instancia a otra:
        * `DurabilidadActual`
        * `IsStolen` (o `InstanceTags` para manejar este y otros estados)
        * Encantamientos aplicados.
        * Stats modificados por mejoras.

## 3. Aplicación a Sistemas de Juego Específicos

Se analizó cómo este principio se aplicaría a diferentes sistemas futuros:

* **Sistema de Robo:** El estado "Robado" es de instancia. Ponerlo en `STR_ItemInfo` haría que todas las instancias de ese `ItemID` fueran robadas por definición. Por tanto, debe ir en `STR_Slot`.
* **Calidad/Rareza vs. Mejoras:**
    * Una **calidad inherente** que cambia los stats base (ej. "Manzana" vs. "Manzana Legendaria") se maneja con **dos `ItemID` diferentes** en el `DataTable`, cada uno con su propia `STR_ItemInfo`.
    * Una **mejora aplicada** a una instancia existente (ej. una "Espada de Hierro" mejorada a "+1") se maneja añadiendo **modificadores de instancia** en `STR_Slot`, que alteran los stats base leídos desde `STR_ItemInfo`.

## 4. El Problema Práctico de la Modificación de Estructuras

Se reconoció y validó la preocupación del usuario sobre la fragilidad del sistema al modificar estructuras base.

* **Confirmación:** Se confirmó que modificar un `struct` en Unreal Engine, especialmente uno anidado profundamente como `STR_Currency` dentro de `STR_ItemInfo`, puede causar una **cascada de errores de serialización** que obliga a refrescar y recompilar manualmente múltiples Blueprints dependientes.
* **Causa:** Esto sucede porque el motor necesita actualizar su "comprensión" de la memoria y las dependencias del `struct` modificado en todos los sitios donde se utiliza.

## 5. Conclusión y Próximos Pasos

Dada la criticidad de tener estructuras de datos estables y la frustración causada por la cascada de errores:

* Se acordó que la mejor decisión era **pausar el desarrollo de nuevas funcionalidades** que dependan de estas estructuras.
* El próximo paso acordado es tener una **sesión dedicada exclusivamente a analizar y reconstruir las estructuras de datos principales** (`STR_ItemInfo`, `STR_Slot`, etc.) para que sean lo más completas y "a prueba de futuro" posible, minimizando la necesidad de volver a editarlas.