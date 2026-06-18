# 🗺️ ArcGIS AI Assistant — Arquitectura Híbrida

> **Documento técnico para reunión con cliente**

> Versión: 2.0 | Enfoque: Híbrido Enterprise + Online | Estado: Pre-reunión

---

## 📋 Tabla de Contenidos

1. [Contexto del Problema](#contexto-del-problema)
2. [¿Es Factible?](#es-factible)
3. [La Solución Híbrida](#la-solución-híbrida)
4. [Arquitectura del Enfoque Simplificado](#arquitectura-del-enfoque-simplificado)
5. [Flujo de Trabajo Detallado](#flujo-de-trabajo-detallado)
6. [El AI Assistant Component (Esri)](#el-ai-assistant-component-esri)
7. [Insumos Requeridos por el Cliente](#insumos-requeridos-por-el-cliente)
8. [Infraestructura y Capacidad](#infraestructura-y-capacidad)
9. [Implementación Paso a Paso](#implementación-paso-a-paso)
10. [Comparativa: Enfoque Complejo vs Enfoque Híbrido](#comparativa-enfoque-complejo-vs-enfoque-híbrido)
11. [Limitaciones y Consideraciones](#limitaciones-y-consideraciones)
12. [Estimación de Costos](#estimación-de-costos)
13. [Roadmap de Implementación](#roadmap-de-implementación)
14. [Preguntas para la Reunión](#preguntas-para-la-reunión)

---

## Contexto del Problema

El cliente tiene la siguiente situación:

| Componente | Estado | Acceso a Internet |
|------------|--------|:-----------------:|
| ArcGIS Enterprise | ✅ Instalado on-premise | ❌ Solo intranet |
| ArcGIS Online | ✅ Disponible | ✅ Sí |
| Computadoras de técnicos | ✅ Disponibles | ✅ Sí |
| Arquitectura MCP completa en Azure | ⚠️ Posible pero compleja | Requiere VPN |

**El problema central:** ArcGIS Enterprise no tiene salida a internet, lo que bloquea la integración directa con servicios de IA en la nube en la arquitectura original.

**La solución:** Un enfoque híbrido donde los datos críticos de Enterprise se replican o sincronizan hacia ArcGIS Online, y el AI Assistant de Esri (componente oficial) se usa como interfaz inteligente desde las computadoras de los técnicos que sí tienen internet.

---

## ¿Es Factible?

**Sí. Con el enfoque híbrido, absolutamente.**

| Tecnología | ¿Existe hoy? | Madurez |
|------------|:---:|---------|
| ArcGIS AI Assistant Component | ✅ | Beta oficial de Esri (SDK JS 5.0) |
| Sincronización Enterprise → Online | ✅ | Feature maduro de ArcGIS |
| ArcGIS Online con IA integrada | ✅ | LLM gestionado por Esri/Microsoft |
| Embeddings de WebMaps | ✅ | Disponible en ArcGIS Online |
| App web con `arcgis-assistant` | ✅ | Componente web estándar |

> El **AI Assistant Component** de Esri es un componente oficial del ArcGIS Maps SDK for JavaScript (versión 5.0, actualmente en beta). No hay que construir el LLM ni la integración — Esri ya lo hizo.

---

## La Solución Híbrida

```
┌─────────────────────────────────────────────────────────────────┐
│  ArcGIS ENTERPRISE (Intranet — sin internet)                    │
│                                                                 │
│  Feature Services  ──────────────────────────────────────────►  │
│  Map Services       [Sincronización / Replica hacia AGOL]       │
│  Geoprocessing      (Scheduled sync, Feature Service Sync,      │
│  Capas críticas      o exportación manual periódica)            │
└──────────────────────────────────────────────┬──────────────────┘
                                               │
                                    Datos replicados
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ArcGIS ONLINE (Con internet)                                   │
│                                                                 │
│  Hosted Feature Layers  ◄── datos replicados de Enterprise      │
│  WebMaps configurados   ◄── con embeddings generados            │
│  LLM integrado por Esri ◄── Azure OpenAI gestionado por Esri    │
└──────────────────────────────────────────────┬──────────────────┘
                                               │ HTTPS
                                               │
┌──────────────────────────────────────────────▼──────────────────┐
│  COMPUTADORA DEL TÉCNICO (Con internet)                         │
│                                                                 │
│  Navegador Web                                                  │
│  └── App Web (HTML/JS con ArcGIS Maps SDK 5.0)                  │
│       ├── arcgis-map (WebMap de ArcGIS Online)                  │
│       └── arcgis-assistant (chat IA integrado)                  │
│            ├── arcgis-assistant-navigation-agent                │
│            ├── arcgis-assistant-data-exploration-agent          │
│            └── arcgis-assistant-help-agent                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Flujo de Trabajo Detallado

### Paso 1 — Sincronización Enterprise → Online

```
ArcGIS Enterprise (Intranet)
        │
        │ Sync automático o programado
        │ (Feature Service Sync / Replica / Export)
        ▼
ArcGIS Online
  └── Hosted Feature Layer (copia de los datos)
  └── Se actualiza según frecuencia definida (diaria, horaria, etc.)
```

**Opciones de sincronización disponibles:**

| Método | Descripción | Mejor para |
|--------|-------------|-----------|
| **Distributed Collaboration** | Sincronización oficial entre Enterprise y Online | Recomendado, bidireccional |
| **Scheduled Export + Overwrite** | Script Python que exporta y sobreescribe en AGOL | Simple, unidireccional |
| **Feature Service Sync** | Réplica de geodatabase vía servicios REST | Datos críticos con historial |
| **ArcGIS Data Pipelines** | Automatización visual en AGOL | Sin programación |

### Paso 2 — Configuración del WebMap en ArcGIS Online

```
ArcGIS Online
  ├── 1. Crear WebMap con las capas de Enterprise (ya replicadas)
  ├── 2. Configurar metadatos de capas:
  │       - Nombres descriptivos en español
  │       - Aliases de campos claros
  │       - Descripciones de campos (para el LLM)
  │       - Popups bien configurados
  └── 3. Generar Embeddings del WebMap:
          - Portal → Item Settings → "Manage AI vector embeddings"
          - Clic en "Generate Embeddings"
          - Esto permite al LLM entender qué datos contiene el mapa
```

> ⚠️ **Los embeddings son obligatorios** para que el `arcgis-assistant` funcione correctamente. Son representaciones vectoriales de las capas y campos que el LLM usa para entender los datos sin verlos todos.

### Paso 3 — App Web con AI Assistant

El técnico accede desde su navegador a una aplicación web que combina el mapa con el asistente de IA:

```
Técnico escribe: "¿Cuántas parcelas sin propietario hay en la zona norte?"
        │
        ▼
arcgis-assistant (componente Esri)
        │
        ├── Orquestador determina la intención
        ├── Ruta al arcgis-assistant-data-exploration-agent
        ├── El agente consulta las capas del WebMap (AGOL)
        ├── LLM (gestionado por Esri/Azure) genera la respuesta
        └── Respuesta en lenguaje natural + resultados en el mapa
```

---

## El AI Assistant Component (Esri)

### ¿Qué es?

Es un **Web Component oficial de Esri**, parte del ArcGIS Maps SDK for JavaScript 5.0 (beta). Proporciona una interfaz de chat integrada directamente en aplicaciones web GIS.

**El LLM lo gestiona Esri** — no hay que contratar Azure OpenAI por separado ni configurar endpoints. El componente usa la infraestructura de IA de ArcGIS Online.

### Agentes disponibles out-of-the-box

| Agente | Qué hace |
|--------|----------|
| `arcgis-assistant-navigation-agent` | Navega el mapa: zoom, pan, ir a ubicaciones |
| `arcgis-assistant-data-exploration-agent` | Consulta y explora datos de Feature Layers |
| `arcgis-assistant-help-agent` | Responde preguntas sobre cómo usar la app |

### Implementación mínima (HTML)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Asistente GIS</title>
  <script type="module">
    import "@arcgis/map-components/components/arcgis-map";
    import "@arcgis/ai-components/components/arcgis-assistant";
    import "@arcgis/ai-components/components/arcgis-assistant-navigation-agent";
    import "@arcgis/ai-components/components/arcgis-assistant-data-exploration-agent";
    import "@arcgis/ai-components/components/arcgis-assistant-help-agent";
  </script>
  <style>
    html, body { margin: 0; padding: 0; height: 100%; }
    .app-container {
      display: flex;
      height: 100vh;
    }
    arcgis-map { flex: 1; }
    arcgis-assistant { width: 380px; }
  </style>
</head>
<body>
  <div class="app-container">

    <!-- Mapa con WebMap de ArcGIS Online (datos de Enterprise replicados) -->
    <arcgis-map id="main-map"
      item-id="AQUI_EL_ID_DEL_WEBMAP_EN_AGOL">
    </arcgis-map>

    <!-- Asistente IA conectado al mapa -->
    <arcgis-assistant
      reference-element="#main-map"
      heading="Asistente GIS"
      description="Consulta datos de catastro, redes y zonas de riesgo"
      entry-message="Hola, soy tu asistente geoespacial. Puedes preguntarme sobre las capas del mapa."
      copy-enabled
      feedback-enabled>

      <!-- Agentes registrados -->
      <arcgis-assistant-navigation-agent></arcgis-assistant-navigation-agent>
      <arcgis-assistant-data-exploration-agent></arcgis-assistant-data-exploration-agent>
      <arcgis-assistant-help-agent></arcgis-assistant-help-agent>

    </arcgis-assistant>

  </div>

  <script type="module">
    // Prompts sugeridos según los datos del cliente
    const assistant = document.querySelector("arcgis-assistant");
    assistant.suggestedPrompts = [
      "¿Cuántas parcelas hay en zona de riesgo de inundación?",
      "Muéstrame las vías en mal estado en el sector norte",
      "¿Cuál es el total de área de predios sin propietario?"
    ];
  </script>
</body>
</html>
```

### Configuración avanzada recomendada

```javascript
const assistant = document.querySelector("arcgis-assistant");

// Capturar feedback de usuarios para mejora continua
assistant.addEventListener("arcgisFeedback", (event) => {
  const feedback = event.detail;
  console.log("Feedback recibido:", feedback);
  // Enviar a sistema de análisis interno
  logFeedbackToServer(feedback);
});

// Capturar errores
assistant.addEventListener("arcgisError", (event) => {
  console.error("Error en asistente:", event.detail);
  showUserFriendlyError();
});

// Habilitar logs para desarrollo (desactivar en producción)
assistant.logEnabled = true; // Solo durante desarrollo
```

---

## Infraestructura y Capacidad

### ¿Qué servidores se necesitan?

Con el enfoque híbrido usando el AI Assistant Component de Esri, la infraestructura se simplifica drásticamente:

| Componente | Dónde corre | Gestionado por | Costo adicional |
|------------|-------------|----------------|:---:|
| LLM (IA) | ArcGIS Online / Esri Cloud | Esri | Créditos AGOL |
| Datos geoespaciales | ArcGIS Online (replicados) | Cliente + Esri | Incluido en AGOL |
| App Web (HTML/JS) | Cualquier servidor web | Proveedor | Mínimo |
| Sincronización datos | Enterprise → AGOL | Cliente | Incluido |
| ArcGIS Enterprise | On-premise del cliente | Cliente | Ya existe |

> **No se necesita Azure.** La IA está gestionada por Esri dentro de ArcGIS Online.

### Requerimientos del servidor para la app web

La aplicación web es estática (HTML + JavaScript). Los requerimientos son mínimos:

```
Servidor interno (recomendado para intranet):
  - IIS en Windows Server existente, o
  - Nginx en Linux
  - Disco: < 50 MB (app estática)
  - CPU/RAM: cualquier servidor web básico
  - Puerto 443 con certificado SSL
```

### Requerimientos de red

```
Técnico (navegador)
  └── Necesita internet para:
      ├── Acceder a ArcGIS Online (arcgis.com)
      ├── Cargar ArcGIS Maps SDK for JavaScript (CDN de Esri)
      └── Usar el AI Assistant (LLM en AGOL)

ArcGIS Enterprise (servidor on-prem)
  └── Necesita conectividad solo hacia:
      └── ArcGIS Online (para sincronización de datos)
          ├── Si no tiene internet: sincronización manual/programada
          └── Si tiene internet limitado: habilitar solo salida a arcgis.com
```

---

## Comparativa: Enfoque Complejo vs Enfoque Híbrido

| Aspecto | Arquitectura MCP Completa | Enfoque Híbrido (este) |
|---------|:---:|:---:|
| **Tiempo de implementación** | 8-12 semanas | 3-4 semanas |
| **Costo de desarrollo** | Alto | Bajo-Medio |
| **Costo operativo mensual** | $900-1,500 USD | Créditos AGOL |
| **Servidores Azure requeridos** | 4-5 servicios | 0 (opcional) |
| **Acceso internet Enterprise** | Requerido (VPN) | No requerido |
| **LLM gestionado por** | Proveedor | Esri |
| **Personalización del agente** | Total | Media |
| **Datos en tiempo real** | Sí | Según sync |
| **Riesgo técnico** | Medio-Alto | Bajo |
| **Escalabilidad futura** | Alta | Media |

> **Recomendación:** Empezar con el enfoque híbrido como MVP. Una vez demostrado el valor, escalar hacia la arquitectura MCP completa si los casos de uso lo justifican.

---

## Limitaciones y Consideraciones

### Limitaciones del AI Assistant Component (importantes para comunicar al cliente)

| Limitación | Descripción | Mitigación |
|------------|-------------|-----------|
| **Solo Feature Layers** | El asistente solo puede consultar Feature Layers. Map Image Layers, Raster, etc. no son consultables por IA | Crear Feature Layer views de los datos necesarios |
| **Componente en Beta** | El `arcgis-assistant` está marcado como beta. Puede cambiar en futuras versiones | Documentar versión usada, seguir release notes de Esri |
| **LLM no es configurable** | No se puede cambiar el modelo de LLM subyacente — lo gestiona Esri | Aceptarlo como parte del producto |
| **Requiere ArcGIS Online** | No funciona con ArcGIS Enterprise directamente | Por eso el enfoque híbrido |
| **Créditos de AGOL** | Las operaciones de IA consumen créditos — monitorear consumo | Establecer alertas de créditos en AGOL |
| **Datos no en tiempo real** | Los datos de Enterprise llegan con el delay de la sincronización | Definir frecuencia de sync según criticidad |
| **Idioma del LLM** | Funciona bien en español, pero la calidad mejora con buenos metadatos en español | Documentar campos y capas en español |

### Consideraciones de seguridad

```
Datos que SÍ salen de la intranet:
  ✓ Solo los datos replicados hacia ArcGIS Online
  ✓ El técnico define qué capas se sincronizan
  ✓ AGOL aplica la misma seguridad de roles que Enterprise

Datos que NUNCA salen:
  ✗ Datos que el cliente no replica a AGOL
  ✗ Datos de Enterprise que quedan solo en intranet

Recomendación: Solo replicar a AGOL datos que el técnico
ya puede ver en su trabajo habitual. No replicar datos
sensibles que no deban salir de la organización.
```

---

## Referencias Técnicas

| Recurso | URL |
|---------|-----|
| AI Assistant Component (Esri Docs) | https://developers.arcgis.com/javascript/latest/references/ai-components/components/arcgis-assistant/ |
| Web Map Setup para IA (Esri Docs) | https://developers.arcgis.com/javascript/latest/agentic-apps/ai-webmap-setup/ |
| Sample Code AI Assistant | https://developers.arcgis.com/javascript/latest/sample-code/ai-assistant/ |
| Introducción a Agentic Apps | https://developers.arcgis.com/javascript/latest/agentic-apps/ai-introduction/ |
| ArcGIS Distributed Collaboration | https://enterprise.arcgis.com/en/portal/latest/administer/windows/about-distributed-collaboration.htm |
| ArcGIS Data Pipelines | https://doc.arcgis.com/en/arcgis-online/analyze/get-started-with-data-pipelines.htm |
| ArcGIS SDK JS 5.0 - AI Components | https://developers.arcgis.com/javascript/latest/references/ai-components/ |

---

## Estructura del Repositorio

```
arcgis-ai-assistant-hibrido/
├── docs/
│   └── architecture/
│       └── architecture-overview.png    ← diagrama de arquitectura
├── src/
│   ├── index.html                       ← app principal
│   ├── app.js                           ← lógica del asistente
│   └── styles.css                       ← estilos personalizados
├── config/
│   └── assistant-config.js              ← prompts, WebMap ID, etc.
├── scripts/
│   └── sync-enterprise-agol.py          ← script de sincronización
└── README.md                            ← este archivo
```

---

*Documento técnico preparado para reunión con cliente.*
*Versión 2.0 — Enfoque Híbrido Enterprise + ArcGIS Online AI Assistant*