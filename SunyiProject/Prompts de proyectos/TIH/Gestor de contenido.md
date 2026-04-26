# Resumen del Proyecto: Gestor de Contenido xRLive

## 1. Descripción General del Proyecto (Funcionalidad)

El objetivo es desarrollar un sistema autocontenido dentro de un plugin de Unreal Engine llamado **`xRLive_ContentManager`**. Este sistema permitirá a una aplicación principal (el "Launcher") cargar paquetes de contenido externos (`.pak` o `.xrlive`) de forma dinámica y asíncrona.

Los requisitos clave de la funcionalidad son:

* **Cero Configuración Manual:** El usuario final del plugin (un desarrollador en Blueprints) no deberá realizar ninguna configuración manual en el editor, como añadir actores o componentes al nivel. El sistema debe funcionar de forma automática.
* **Interfaz Simple en Blueprint:** El uso del sistema desde Blueprints debe ser a través de nodos "globales", simples de encontrar y usar, sin la necesidad de obtener referencias a actores o componentes externos.
* **Carga Asíncrona:** El proceso de carga de un paquete de contenido no debe bloquear el juego. El sistema debe notificar al Blueprint a través de eventos cuando la carga ha comenzado y cuando está lista.
* **Gestión de Contenido:**
    * El sistema debe ser capaz de montar un archivo `.pak` o `.xrlive`.
    * Debe poder buscar y cargar un mapa principal contenido en el pak. (El mapa debe llamarse igual que el proyecto)
    * Una vez el mapa esté cargado, debe permitir buscar clases de `UUserWidget` por su nombre desde dentro del pak.
    * Debe proporcionar una forma de desmontar el contenido actual para permitir la carga de un nuevo paquete de contenido.

## 2. Descripción del Desarrollo (Arquitectura)

Para cumplir con todos los requisitos, especialmente el de "cero configuración" y la "simplicidad en Blueprint", implementaremos una arquitectura híbrida de dos partes: un "Cerebro" con estado y una "Interfaz Pública" sin estado.

* **El Cerebro (Backend):** Será un objeto de C++ único, persistente y automático. Su función es contener toda la lógica pesada y, lo más importante, guardar el **estado** del sistema (ej: qué pak está cargado, si el sistema está listo, etc.).
* **La Interfaz Pública (Frontend):** Será una colección de clases de C++ cuyo único propósito es generar nodos de Blueprint globales y fáciles de usar. Estos nodos no contendrán lógica compleja; actuarán como "atajos" o "fachadas".

El flujo de trabajo es simple: el desarrollador en Blueprint llama a un nodo de la Interfaz Pública. Internamente, ese nodo buscará al Cerebro (que ya existe automáticamente) y le delegará la orden. El Cerebro ejecuta la tarea y notifica el resultado, que la Interfaz devuelve al Blueprint.

## 3. Clases a Desarrollar

Para implementar esta arquitectura, crearemos las siguientes tres clases de C++:

### a) El Cerebro

* **Nombre de la Clase:** `UContentManagement`
* **Clase Padre en C++:** `UGameInstanceSubsystem`
* **Rol y Responsabilidades:**
    * Actúa como el motor central y la única fuente de verdad del sistema.
    * Es creado y gestionado **automáticamente** por el motor, garantizando que siempre exista uno (y solo uno) durante toda la sesión de juego.
    * Mantiene el estado del sistema, como la ruta del pak actualmente montado y la bandera de seguridad `bIsExperienceReady`.
    * Contiene la implementación real para montar/desmontar paks, buscar assets en el `AssetRegistry` y gestionar el proceso de carga asíncrono de dos etapas.
    * Declara y dispara los delegados (`OnMapLoadStarted`, `OnExperienceReady`, etc.) para notificar los cambios de estado.

### b) La Interfaz para la Carga Asíncrona

* **Nombre de la Clase:** `UAsyncLoadExperienceAction`
* **Clase Padre en C++:** `UBlueprintAsyncActionBase`
* **Rol y Responsabilidades:**
    * Su único propósito es generar el nodo global `AsyncLoadExperience` en Blueprint.
    * Este nodo tendrá múltiples pines de ejecución de salida (`OnMapLoadStarted`, `OnExperienceReady`, `OnFail`) para manejar la naturaleza asíncrona de la carga.
    * Internamente, su lógica consistirá en encontrar el subsistema `UContentManagement` y llamar a su función de carga, suscribiéndose a sus delegados para saber qué pin de salida disparar.

### c) La Interfaz para Funciones Simples

* **Nombre de la Clase:** `UxRLive_ContentLibrary`
* **Clase Padre en C++:** `UBlueprintFunctionLibrary`
* **Rol y Responsabilidades:**
    * Actúa como una "caja de herramientas" para proporcionar nodos globales para las operaciones más simples.
    * Contendrá las funciones `static` que generarán los nodos `UnmountExperience` y `FindWidgetClassByName` en Blueprint.
    * La lógica interna de cada función será extremadamente simple: encontrar el subsistema `UContentManagement` y llamar a la función interna correspondiente.