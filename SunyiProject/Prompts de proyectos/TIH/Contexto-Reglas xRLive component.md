## ⚠️**CRÍTICO: CADA VEZ QUE USE LA EXPRESIÓN **`/REVISAR_CODIGO`** REVISA EL CODIGO QUE TE PROPORCIONÉ Y CUANDO ME DES RESPUESTAS QUE IMPLIQUEN LA CREACIÓN, AJUSTE O ELIMINACIÓN DE FUNCIONES **PROPORCIANE EL CÓDIGO COMPLETO DE LA FUNCIÓN SIN OMITIR LAS PARTES QUE NO CAMBIAN**. MARCA CON EL EMOJI DE UN PC (💻) _CADA VEZ QUE LEAS EL CÓDIGO PROPORCIONADO_. SIEMPRE PROPORCIONA EL CÓDIGO COMPLETO DE LA FUNCIÓN MODIFICADA, SIN OMISIONES DE NINGÚN TIPO**⚠️

# **Contexto General**

Se desarrolló un sistema en UE5.1 que consiste en un plugin que permite generar paks de contenido a partir de proyectos completos (se empaqueta el contenido de la carpeta content del proyecto principal) llamado `xRLive_Package`, para ser utilizados dentro de un proyecto más grande llamado `xRLive_Launcher` y un plugin utilizado dentro `xRLive_Launcher` para poder montar los paks en formato `xrlive` llamado `xRLive_ContentManager` y utilizarlos dentro del launcher.

## **INDICACIONES CRITICAS**

⚠️**CRITICO: TE VOY A PROPORCIONAR MI CÓDIGO EN DISTINTOS MENSAJES. SOLO RESPONDE _DE ACUERDO_ HASTA QUE TE DE LA INSTRUCCIÓN LITERAL _He terminado de darte mi código_ SIN CURSIVA**⚠️
**Token Management**: Al final de cada mensaje, lanzar alerta de consumo de tokens.
      - El formato debe ser exactamente:  
     **Hemos consumido _≈[tokens consumidos en el mensaje] tokens_, restantes aprox: [tokens restantes de la sesión]. Tokens totales aprox: [tokens maximos para la sesión]**

## **📂 Comando de Resumen Portátil**
⚠️ **INSTRUCCIÓN ESPECIAL**:
- Si el usuario escribe **`/EXPORTAR_CONTEXTO`**, genera un resumen **DEL PROCESO FINAL TRABAJADO HASTA EL MOMENTO**:  
  1. Objetivo: Permitir copia/pega en nuevos chats o modelos sin pérdida de coherencia.
  2. El resumen debe venir en formato _markdown_ embebido en un bloque de código.

---

Código en `FPakkerCore.cpp`
```
```
Código en `FPakkerCore.h`
```
```
Código en `xRLive_Package.cpp`
```
```
Código en `xRLive_Package.h`
```
```








---

Código en `xrlive_UIpanelCompnent.h`

```

```

Código en `xrlive_UIpanelCompnent.cpp`

```

```

Código en `WBP_WidgetController.h`

```

```

Código en `WBP_WidgetController.cpp`

```

```



Se desarrolló un sistema en UE5.1 que sincroniza controles de UI (Sliders, Checkboxes, Dropdowns, Texts) entre un Launcher y PAKs mediante un archivo JSON plano (Value0, Value1, ...).
Objetivo: Garantizar persistencia bidireccional (UI ↔ JSON) y evitar corrupción de datos.