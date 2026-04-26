# Resumen de Módulos de Datos para STR_ItemInfo

Se acordó que, para la actualización del sistema de items, para que este fuera más robusto y funcionara para multiples escenarios, además sin destruir los sistemas desarrollados actualmente, la información se iba a dividir en 2 estructuras. `STR_ItemInfo` va a ser la definición de los items en el mundo. `STR_InstanceData` va a ser la información "sobreescrita" de la instancia del item en los inventarios. Además `STR_Slot`, va a ser el puente para la información de las 2 estructuras.

Esta arquitectura funcionará de la siguiente forma: 
* `STR_ItemInfo`, contiene la información base de cada uno de los items del juego. Estos datos son **inamomibles** y se encargan de "dar forma" al objeto. Contendrá la información más básica de **todos** los posibles sistemas que podrían utilizar o manipular el ítem.
* `STR_InstanceDate`, almacenará datos específicos del item despues de su creación en el mundo y que sean esencialmente diferentes de los de definición. Estos datos podrán ser **sobreescritos o actualizados** tanto por el jugador como por los NPC y solo afectarán al *elemento específico durante su existencia en el mundo* (su instancia).
* `STR_Slot`, será el puente entre la definición y la instancia, esto quiere decir que gracias a `STR_Slot` podremos acceder a los datos necesarios. Contendrá tres datos esenciales `ItemID`, que será el nombre del elemento en la tabla de *ItemInfo*, la cantidad, directamente administrada por el slot y un identificador único para cada instancia de elemento, que será con el que se busque y gestione la información dentro de `STR_InstanceData`.

Este es el fragmento de la propuesta de desarrollo para la actualización del sistema **sin romper ningún sistema existente**
_"Podemos tener en `STR_ItemInfo` **todos los datos generales del item y que no cambian con el tiempo** (estos incluyen datos de todos los sistemas implementados y que se van a implementar), por otro lado tener una nueva estructura, que se encarga de manejar todos los datos de instancia (tambien de todos los sistemas a implementar o implementados) y tenemos a `STR_Slot`, como el puente entre la tabla de datos estandarizados y la estructura de datos instanciados (`STR_Slot`, solo contaría con 2 o 3 datos para buscar y encontrar lo que sea que necesitemos). Con esto podríamos mantener la estructura sistemática actual, sin demasiados cambios arquitectónicos y sería más manejable."_

A continuación se presenta la lista definitiva y el orden de los 16 módulos de datos acordados que compondrán la estructura `STR_ItemInfo`, junto con una descripción de su propósito.

### 1. Documento de Identidad (Datos Básicos y de UI)
*Define la identidad fundamental y visual del ítem: su ID, tags de tipo y propiedad, nombre, descripción, icono y todos los modelos 3D asociados.*

### 2. Comportamiento en Inventario
*Define las reglas básicas de almacenamiento: si el ítem es apilable y la cantidad máxima por stack.*

### 3. Sistema Multi-Uso y de Acciones
*Define la lista de todas las acciones intrínsecas que un ítem puede realizar (`Usar`, `Lanzar`, `Barrer`) y el contexto requerido para cada una (`EnInventario`, `Equipado`, etc.).*

### 4. Comportamiento de Equipamiento
*Define en qué slot(s) de equipo se puede colocar el ítem y las propiedades de los contenedores que provee (ej. mochila).*

### 5. Datos Económicos
*Define la lista de precios de un ítem para diferentes tipos de transacción (venta normal, compra, valor en mercado negro, etc.).*

### 6. Ciclo de Vida del Ítem (ItemLife) y Deterioro
*Define la durabilidad máxima de un ítem y las reglas de cómo se transforma en otros ítems al alcanzar ciertos umbrales de uso o tiempo.*

### 7. Datos de Estadísticas
*Define los modificadores a estadísticas base (`Ataque`, `Defensa`, etc.) que un ítem posee. Es un módulo independiente consumido por otros sistemas.*

### 8. Sistemas de Fabricación
*Define todas las recetas y materiales necesarios para craftear, desensamblar, reparar o mejorar el ítem.*

### 9. Encantamiento y Modificación Mágica
*Define el número de encantamientos que un ítem puede soportar y la lista de los encantamientos que son compatibles con él.*

### 10. Datos de Obtención y Entorno (Investigación)
*Define las condiciones "empíricas" u ocultas bajo las cuales este ítem puede ser encontrado o creado en el mundo (requerimientos de herramienta, clima, lugar, etc.).*

### 11. Datos de Aprendizaje y Desbloqueo (Textos)
*Define el conocimiento "explícito" que un ítem otorga al ser consumido (desbloqueo de recetas, habilidades, pistas de texto, etc.).*

### 12. Propiedades de Objetos Arrojadizos
*Define la clase de proyectil que el ítem genera al ser lanzado, así como sus efectos en el área de impacto.*

### 13. Datos de Trampas y Cebos
*Define el funcionamiento de ítems que actúan como trampas, incluyendo el tipo de criaturas que pueden capturar y los cebos que aceptan.*

### 14. Sistema de Música e Interpretación
*Define, para los instrumentos musicales, los archivos de sonido (`SoundCues`) asociados a las melodías que pueden interpretar para generar efectos.*

### 15. Propiedades de Iluminación
*Define las características de un ítem como fuente de luz (radio, intensidad, color) y su capacidad máxima de combustible.*

### 16. Datos de Construcción
*Define las propiedades de los ítems que pueden ser colocados en el mundo como estructuras (paredes, suelos, cimientos, etc.).*