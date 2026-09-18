# Especificación funcional — Asistente gastronómico para Iberostar

## 1. Feature Overview

### Feature Name
Asistente gastronómico y sumiller para huéspedes de restaurante y buffet

### Problem Statement
Los huéspedes del restaurante de Iberostar necesitan orientarse rápidamente entre la oferta gastronómica del buffet o restaurante, comparar opciones según sus gustos, presupuesto y necesidades dietéticas, y confirmar una elección sin perder tiempo ni generar fricción. Hoy la decisión puede depender de la disponibilidad del personal, la lectura de menús o la falta de información clara sobre ingredientes, maridajes y vinos.

### Proposed Solution
Se implementará un asistente conversacional en web móvil que permita al huésped consultar la carta, hacer preguntas en texto o voz, recibir recomendaciones personalizadas, validar disponibilidad y confirmar un pedido de prueba en un backoffice interno. El sistema combinará conocimiento del catálogo con información verificada del restaurante y ofrecerá derivación a personal cuando los datos no sean suficientes.

### Success Criteria
- El huésped puede completar un flujo de recomendación y pedido en español e inglés sin intervención del personal.
- El sistema ofrece recomendaciones válidas y actualizadas según disponibilidad, precio y preferencias.
- La selección final se confirma explícitamente antes de registrarse como pedido de prueba.
- El backoffice de prueba muestra el pedido con identificador, estado y datos básicos del huésped/operación.
- El asistente reconoce situaciones de riesgo (alergenos, información incompleta, producto agotado) y deriva al personal de forma clara.

---

## 2. User Impact

### Target Users
- Huésped del hotel que quiere consultar la oferta gastronómica antes de elegir.
- Personal de restaurante o F&B que mantiene catálogo, disponibilidad y revisión de pedidos.
- Equipo evaluador que valida la POC, la experiencia del usuario y la calidad de las recomendaciones.

### User Benefits
- Reducción del tiempo de decisión en la elección de platos y bebidas.
- Mejor comprensión de ingredientes, preparación y maridajes antes de pedir.
- Recomendaciones más alineadas con gustos, presupuesto y régimen de comida.
- Menor riesgo de errores en la selección de vinos y platos por falta de información.
- Tráfico de pedidos de prueba organizado y rastreable en un backoffice controlado.

### User Experience Changes
El huésped accede a una experiencia conversacional muy simple: pregunta, recibe recomendaciones, aclara dudas, revisa un resumen y confirma. El flujo se adapta a móvil, acepta voz o texto, y permite consultar información visual del cartel o ficha del producto cuando el nombre o la imagen no son suficientes.

---

## 3. Functional Requirements

### Core Functionality

1. The system shall allow a guest to open a session from a mobile web app using a QR code or direct link without a pre-created account.
2. The system shall support both text and voice interactions in Spanish and English within the same conversational session.
3. The system shall present the available restaurant offer, including dishes, beverages, and wines relevant to the active service and context.
4. The system shall allow the guest to ask about ingredients, preparation method, origin, paired beverages, and suitability for dietary preferences or restrictions.
5. The system shall analyze the guest’s preferences, budget, and prior selections to produce recommendations ranked by fit and commercial relevance.
6. The system shall filter unavailable or out-of-stock items and suggest alternatives when the original selection cannot be fulfilled.
7. The system shall support identifying a menu item from a photo or signage context, including product lookup by visible description when the name is ambiguous.
8. The system shall ask for confirmation before finalizing a selection or submitting an order.
9. The system shall allow the guest to modify the order summary, including quantities, items, and remove or replace products.
10. The system shall generate a final order summary with item names, quantities, unit price, and total estimated amount before confirmation.
11. The system shall register a confirmed order in a test backoffice with a unique identifier and a status value such as created, accepted, or rejected.
12. The system shall prevent duplicate registration when the user retries a confirmation or the backend responds repeatedly.
13. The system shall explicitly inform the guest when critical information is missing, outdated, or not validated, and redirect the user to staff assistance.
14. The system shall not infer allergen presence or cross-contamination safety from an image alone; it must rely on validated catalog data or personal assistance.
15. The system shall maintain a catalog with product IDs, names, descriptions, ingredients, allergens, prices, service availability, and wine metadata when available.
16. The system shall provide a backoffice interface for testing staff to review catalog entries, change availability, and inspect current orders.
17. The system shall log session activity in a way that preserves user privacy, without storing audio or images by default in the POC stage.
18. The system shall support an operational mode where the pricing and availability of items are revalidated immediately before final order submission.

### Input Specifications

The system shall accept the following inputs:
- Free-text conversational requests in Spanish and English.
- Voice input captured from the mobile web session.
- Optional image input when the guest points to a dish, menu card, or signage for identification.
- Product filters such as vegetarian, vegan, preference, budget ceiling, wine price max, and dish category.
- Catalog data loaded from structured files or import sources in the test environment.
- Order adjustments made by the user before final confirmation.

### Output Specifications

The system shall provide the following outputs:
- Conversational responses that answer product, ingredient, preparation, and pairing questions.
- Recommendation cards or suggestions with name, price, short description, and availability state.
- Alternative product suggestions when an item is unavailable.
- A summary of chosen items before order confirmation.
- Finalized order payload sent to the test backoffice, including identifier, timestamp, status, and finalized item list.
- Warnings or escalation messages for incomplete or critical data.

### Business Rules

1. The system shall not offer products that are marked unavailable in the active catalog or active service period.
2. Product recommendations shall be based on catalog entries validated by F&B or demo data owners.
3. If a guest requests a vegetarian, vegan, or seafood-sensitive option, the system shall filter by the approved catalog metadata and surface limitations when the data is incomplete.
4. The system shall price all returned recommendations from the current approved catalog before order submission.
5. If the price or availability changes after the user begins the flow, the system shall refresh and confirm the final pricing before registration.
6. The system shall not claim an allergen-safe statement unless the catalog explicitly contains validated data; otherwise it must inform the user and offer staff support.
7. The system shall not automatically charge the room, process a payment, or interact with production PMS or POS systems in the POC stage.
8. If no verified answer exists, the system shall respond with a limitation statement and suggest a staff recommendation rather than inventing a claim.
9. For ordering, the guest must confirm the final summary explicitly; a selection that is only partially acknowledged shall not be submitted.
10. Each confirmed order in the demo environment shall be traceable by a unique order ID and a clear status.

---

## 4. Dependencies and Constraints

### Dependencies
- Catalogo estructurado de platos, bebidas y vinos con precios, disponibilidad y definiciones.
- Módulo de backoffice de prueba para creación, consulta y revisión de pedidos.
- Capacidad para aceptar entrada de voz y texto en español e inglés.
- Motor de conversación con integración de reglas de negocio y validación del catálogo.
- Almacenamiento seguro de claves y acceso limitado a datos operativos.

### Constraints
- La POC está limitada a un único hotel y un restaurante o buffet con catálogo reducido.
- No se integrará con producción de caja, cocina, PMS ni pagos durante la fase 1.
- Se debe respetar el régimen comercial vigente para cada servicio y producto.
- Los datos de alérgenos, ingredientes y vinos deberán estar validados por responsables de F&B.
- Por seguridad y alcance del POC, las sesiones serán anónimas y no se conservarán imágenes o audio por defecto.

### Assumptions
- El catálogo base será suministrado en formato estructurado y con marca de datos ficticios si procede.
- Los responsables de la operación revisarán las fichas antes del piloto.
- El entorno de pruebas tendrá una disponibilidad funcional de productos cercana a la real, pero sin impacto comercial.
- Las pruebas de la POC serán guiadas y no con huéspedes reales en producción.

---

## 5. Risks and Mitigations

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Datos del catálogo incompletos o desactualizados | Recomiendo incorrectas y frustración del usuario | High | Validación de F&B antes del uso, revalidación al cierre del pedido y derivación al personal si falta información crítica |
| Confusión entre productos similares | Pedido incorrecto o recomendación poco útil | Medium | Confirmación de intención y aclaración por parte del asistente antes de cierre |
| Disponibilidad no sincronizada en tiempo real | Pedido basado en producto agotado o precio no vigente | High | Revalidación final al confirmar y actualización del estado antes del registro |
| Uso de voz con ruido o idioma mixto | Mala interpretación del usuario | Medium | Soporte de lenguaje claro, confirmación de entendimiento y fallback a texto |
| Incorrecta inferencia de alergias o ingredientes | Riesgo reputacional y sanitario | High | No inferir automáticamente; requerir datos validados o asistencia humana |
| Dependencia de una PWA móvil estable y usable | Mala experiencia en demos o sesiones reales | Medium | Probar en dispositivos objetivos, validación previa y fallback funcional simple |

---

## 6. Open Questions

1. ¿Qué hotel y restaurante serán la referencia operativa para la POC y qué catálogo exacto se usará?
2. ¿Qué nivel de validación del contenido de fichas (alérgenos, vinos, maridajes) exige Iberostar antes de mostrarlo al huésped?
3. ¿Se necesita una capa de identificación del huésped o bastará con una sesión anónima de demo?
4. ¿El backoffice de prueba debe incluir una vista de detalle de cada pedido, historial y estado de resolución?
5. ¿El flujo de pedido en la POC debe ser solo de registro o también debe producir señales para la cocina o al equipo de servicio?
6. ¿Se definirá una lista mínima de reglas dietéticas, alérgenos y restricciones de suma para la fase 1?
7. ¿Se requieren métricas de éxito oficiales para la demo (tiempo, tasa de completitud, tasa de recomendación válida) antes de la validación final?

---

## 7. References

### Related Documentation
- Vision scope original: [Specs/Vision-Caso-Uso/vision-scope-iberostar-specs.md](Vision-Caso-Uso/vision-scope-iberostar-specs.md)
- Presentación de referencia: Iberostar_Asistente_Buffet_y_Sumiller.pptx (documento anexado en la conversación)

### Research and Background
- Revisión del alcance de la POC para un asistente gastronómico multimodal.
- Casos de uso de recomendación, confirmación y derivación a personal.
- Requisitos relativos a resistencia operativa, claridad de datos y validación de alergias.

---

> **Engineering Sections (Optional)**
>
> Estas secciones sintetizan una primera propuesta técnica para la POC y deben revisarse con ingeniería antes del cierre final.
>
> En la implementación actual de la POC, el enfoque frontend-first ya quedó validado: la app permite conversación, recomendaciones en base a un catálogo demo, confirmación de pedido y un backoffice local. El sistema incluye un fallback seguro al catálogo local para mantener la funcionalidad disponible si el endpoint de Azure no responde o si la autenticación directa desde navegador exige ajustes. Además, incorpora selector de idioma y entrada por voz mediante Web Speech API cuando el navegador lo soporta, con texto como fallback. Esto refuerza la decisión del plan de mantener la capa .NET como evolución posterior, no como prerequisito del MVP.

---

## 8. Technical Approach

### Component Affected
- Frontend PWA para huésped
- Backend de conversación y orquestación de negocio
- Catalog service / data layer
- Backoffice de prueba para personal
- Módulo de validación de disponibilidad y precio

### Technology Stack
- Web app responsive / PWA para móvil
- Lógica de conversación con soporte de texto y voz
- Integración con catálogo de productos y vinos
- Servidor de backend para persistencia de ordenes y revisión de sesiones
- Almacenamiento seguro de secretos y configuración operativa
- [TBD] Stack exacto de LLM, framework de voz y base de datos a confirmar con ingeniería

### Integration Points
- Carga de catálogo desde archivos o fuentes administradas por Iberostar
- Validación de disponibilidad y precio antes de la confirmación final
- Registro de pedidos en entorno de pruebas
- Consulta y edición de pedidos desde el backoffice administrativo

### Data Considerations
- El sistema debe mantener un catálogo central con IDs, descripción, ingredientes, alérgenos, precios y disponibilidad.
- Debe registrar el estado del pedido y el historial mínimo necesario para auditar la demo.
- La POC debe operar con sesiones anónimas y sin persistencia por defecto de audio o imágenes.
- Los datos ficticios deben distinguirse claramente para evitar confusión en la demo.

---

## 9. Non-Functional Requirements

### Performance
- La respuesta útil del asistente debe estar dentro de un p95 inferior a 5 segundos para escenarios estándar de la POC.
- La disponibilidad del catálogo actualizada en el entorno de pruebas deberá reflejar los cambios dentro de 60 segundos.
- La aplicación web debe funcionar en dispositivos móviles con navegación estable y tiempos de carga aceptables.

### Security
- Las claves y secrets se almacenarán en servidor y no en el frontend.
- El acceso administrativo deberá estar protegido por autenticación y autorización mínimo.
- No se almacenarán imágenes ni audio en la POC sin consentimiento explícito y sin necesidad operativa.

### Reliability
- El sistema deberá manejar errores de validación del catálogo sin producir pedidos dobles ni estados inconsistentes.
- Los errores de backend deben devolver mensajes claros y no generar confirmación automática.
- Cada pedido final debe tener un chequeo de precio y disponibilidad antes de persistirse.

### Scalability
- La solución deberá soportar la carga de una demo y un entorno de prueba con un volumen limitado de sesiones concurrentes.
- La arquitectura deberá permitir ampliar catálogo y servicios sin reescrituras masivas en la fase posterior de piloto.

---

## 10. Testing Strategy

### Test Scenarios
1. Scenario 1: El huésped pide una recomendación vegetariana con presupuesto bajo y recibe platos y vino compatibles.
2. Scenario 2: El huésped pregunta por el maridaje de un plato concreto y el sistema responde con información verificada y alternativa si el vino no está disponible.
3. Scenario 3: El huésped intenta seleccionar un producto agotado; el sistema ofrece alternativas y mantiene la sesión sin romper el flujo.
4. Scenario 4: El huésped pregunta por alérgenos y el sistema detecta datos incompletos y deriva al personal.
5. Scenario 5: El usuario confirma el pedido y el backoffice registra el pedido sin duplicado ni pérdida de información.
6. Scenario 6: El sistema procesa un pedido en español y otro en inglés con consistencia comparable.

### Acceptance Criteria
- [ ] El flujo completo de recomendación + pedido de prueba se ejecuta sin intervención del personal en al menos el 90% de los escenarios guiados.
- [ ] Al menos el 95% de las recomendaciones en un conjunto validado de escenarios cumplen la intención del usuario y los datos del catálogo.
- [ ] Todos los casos críticos de alergias y datos incompletos derivan correctamente al personal o informan la limitación.
- [ ] La confirmación del pedido genera un identificador único y un estado visible en el backoffice.
- [ ] La aplicación detecta y evita duplicados en reintentos de confirmación.

---

## 11. Rollout Plan

### Deployment Approach
- Despliegue del POC en entorno de pruebas controlado.
- Acceso mediante enlace o QR para la demo y validación.
- Carga del catálogo demo y configuración de capacidad de operación.

### Rollback Strategy
- Deshabilitar el acceso a la PWA o al backoffice de prueba si aparece un problema crítico.
- Revertir catálogo a versión validada y reanudar la demo con datos de control.
- Mantener la versión anterior del catálogo para restaurar disponibilidad y precios.

### Monitoring and Metrics
- Tiempo de respuesta por sesión.
- Tasa de finalización de flujo.
- Tasa de recomendación válida.
- Citas de derivación a personal por datos incompletos o críticos.
- Número de pedidos confirmados, errores de duplicado y cambios de disponibilidad.

---

## Revision History

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2026-09-18 | 1.0 | Copilot | Initial functional specification derived from the vision-scope document |

---
