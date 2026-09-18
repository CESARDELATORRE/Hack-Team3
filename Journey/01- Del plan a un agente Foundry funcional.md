# Del plan a un agente Foundry funcional: diario de la implementación

_18 de septiembre de 2026_
_Como una SPA de Vue pasó de ser un plan de arquitectura a conversar con un agente real de Azure AI Foundry._

---

## El punto de partida

El proyecto nace como una prueba de concepto para Iberostar: un asistente gastronómico que permita a un huésped preguntar por platos, bebidas y maridajes, recibir recomendaciones en español o inglés y avanzar hacia un pedido de prueba. El plan proponía una experiencia móvil, una conversación sencilla, validaciones de catálogo y un pequeño backoffice para revisar pedidos.

La primera decisión importante fue mantener el MVP pequeño. El [plan de implementación](../Specs/Vision-Caso-Uso/implementation-plan.md) contemplaba una Vue SPA y una integración directa con Azure AI Foundry desde JavaScript, dejando un backend .NET como evolución para cuando hicieran falta secretos mejor protegidos, persistencia empresarial o integraciones reales.

> **Usuario:** "añade un diagrama mermaid con la arquitectura de la solucion"

El diagrama terminó describiendo el recorrido esencial: el huésped entra por la Vue SPA, la capa de servicio prepara la conversación, el agente de Azure AI Foundry responde y la parte operativa conserva catálogo, pedidos de prueba y observabilidad. Esa imagen ayudó a separar la experiencia del usuario de la futura capa empresarial.

**Artefactos:** [Plan de implementación](../Specs/Vision-Caso-Uso/implementation-plan.md) y [diagrama de arquitectura dentro del plan](../Specs/Vision-Caso-Uso/implementation-plan.md#32-diagrama-de-alto-nivel).

## La primera conversación con el agente

La aplicación ya tenía una pantalla de chat en [App.vue](../src/src/App.vue). El componente mantenía una sesión local, mostraba los mensajes y exponía una función `callFoundry` para enviar la consulta. La configuración se leía desde variables de entorno, de modo que el endpoint y la clave no quedaran escritos en el código fuente.

Al principio, la llamada se parecía demasiado a una llamada genérica a un modelo: construía un `payload` con `model`, mensajes y parámetros de generación. El usuario aclaró el detalle que cambió el rumbo de la implementación:

> **Usuario:** "el endpoint no es un modelo de openai sino un agente en foundry"

La URL ya identificaba al agente en su ruta de Foundry. Por eso se eliminó `model` del cuerpo de la petición. El cliente debía tratar la operación como una invocación del endpoint de Responses de un agente, no como una selección de modelo desde el navegador.

La siguiente señal vino directamente del servidor:

> **Usuario:** "Azure Foundry error (400): {\"error\":{\"code\":\"BadRequest\",\"message\":\"Missing required query parameter: api-version\"}}"

La solución fue hacer explícita la versión en `.env` mediante `VITE_FOUNDRY_API_VERSION` y añadirla a la URL con `URL.searchParams`. Esta construcción también evita concatenar manualmente la query string y conserva cualquier parámetro que ya tuviera el endpoint.

El resultado quedó conceptualmente así:

```js
const apiVersion =
  import.meta.env.VITE_FOUNDRY_API_VERSION || "2025-05-15-preview";

const requestUrl = new URL(endpoint);
requestUrl.searchParams.set("api-version", apiVersion);

const response = await fetch(requestUrl, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "api-key": apiKey,
    Accept: "application/json",
  },
  body: JSON.stringify(payload),
});
```

La aplicación también conserva un prompt de sistema orientado al dominio: usar solo el catálogo, no inventar ingredientes, alérgenos, precios ni disponibilidad, pedir aclaraciones cuando falte contexto y derivar al personal cuando el dato sea crítico.

## El error que reveló el contrato real

Con `api-version` ya presente apareció un error más específico:

> **Usuario:** "Azure Foundry error (400): Invalid value: ''. Supported values are: ... 'message' ... param: input[1]"

El índice `input[1]` apuntaba al segundo elemento del array enviado al endpoint. El cuerpo contenía `role` y `content`, pero no describía el tipo de elemento. El servicio de Responses no podía interpretar ese objeto como un mensaje válido.

La corrección fue pequeña, pero importante: cada elemento del array pasó a declarar `type: 'message'` junto con su rol y contenido.

```js
input: [
  {
    type: "message",
    role: "system",
    content: formatSystemPrompt(),
  },
  {
    type: "message",
    role: "user",
    content: messageText,
  },
];
```

Este fue el momento de entender que la API no estaba fallando por el texto gastronómico ni por el agente, sino porque el cliente estaba enviando una estructura incompleta. El mensaje de error, aunque seco, funcionó como una descripción del contrato: entre los tipos admitidos estaba `message`, y ese era precisamente el tipo que faltaba.

## Del build a la prueba real

Cada cambio se verificó primero con el build de Vite. El proyecto compiló correctamente después de quitar `model`, añadir `api-version` y tipar los mensajes. Aun así, compilar no demostraba que las credenciales, la URL y el contrato remoto fueran correctos.

La prueba definitiva fue levantar la SPA y enviar desde el chat una consulta real:

> **Usuario:** "Recomiéndame un plato vegetariano del catálogo."

El agente respondió con una recomendación concreta, precio y alérgenos. La aplicación mostró la respuesta en la burbuja del asistente, sin el error 400. La experiencia completa quedó validada: carga de la pantalla, activación del formulario, llamada HTTP, respuesta del agente y renderizado del contenido.

Ese último paso cerró el circuito. El código no solo compilaba: el navegador podía hablar con el agente de Foundry y recibir una respuesta útil para el dominio gastronómico.

## Lo que aprendimos

### Sobre la tecnología

- Un endpoint de agente de Foundry puede exponer una interfaz compatible con Responses sin convertirse por eso en un endpoint de modelo genérico.
- La identidad del agente está en la ruta configurada; no debe duplicarse enviando `model` en el cuerpo.
- `api-version` forma parte obligatoria de la URL y conviene configurarlo mediante entorno.
- Los objetos de `input` necesitan declarar explícitamente `type: 'message'` cuando se envían como mensajes.
- Un build correcto y una prueba HTTP real validan cosas distintas; hacen falta ambos.

### Sobre el proceso

- Los errores del servicio fueron más útiles cuando se leyó su parámetro exacto: primero faltaba `api-version`, después faltaba el tipo del elemento `input[1]`.
- La implementación avanzó mejor con cambios pequeños y verificables: modificar la URL, compilar, corregir el payload y probar desde el navegador.
- El plan de arquitectura fue una guía de alcance, no una excusa para incorporar infraestructura que el POC aún no necesitaba.

## Lo que queda pendiente

La integración funciona para la demo, pero todavía hay decisiones de producto y seguridad por resolver. La API key se consume desde variables `VITE_*`, por lo que queda disponible en el bundle del navegador: esto puede ser aceptable únicamente para una prueba controlada y requiere rotar la clave y mover la llamada a un backend antes de cualquier entorno compartido o productivo.

También quedan por implementar el catálogo estructurado, las validaciones de precio y disponibilidad, el resumen editable del pedido, la persistencia de pedidos y el backoffice. En una siguiente fase, la capa .NET podrá centralizar secretos, validaciones, idempotencia, logging y autenticación.

La primera piedra, sin embargo, ya está colocada: la interfaz de Iberostar conversa con el agente correcto, usando el contrato correcto y devolviendo una respuesta comprobable.

---

_Written: 18 de septiembre de 2026_
