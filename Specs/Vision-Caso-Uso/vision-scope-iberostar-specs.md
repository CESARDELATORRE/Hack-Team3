# Vision-Scope — Asistente de restaurante y sumiller

**Cliente:** Iberostar Selection · **Equipo:** Turing  
**Estado:** propuesta de alcance para la fase 1 (POC), pendiente de validación con Iberostar.  
**Referencia:** presentación `Iberostar_Asistente_Buffet_y_Sumiller.pptx`, versión facilitada en esta conversación.

## 1. Visión y valor

Crear un asistente gastronómico que permita al huésped conversar sobre la oferta del restaurante, elegir platos y bebidas según sus gustos y presupuesto, y confirmar un pedido a cocina. Explicará ingredientes, preparación y maridajes, así como las características de los vinos disponibles, utilizando información verificable.

Para Iberostar, la propuesta busca facilitar la elección, reforzar la experiencia gastronómica de Honest Food y generar oportunidades de venta adicional. Para Turing, permite demostrar innovación multimodal aplicada a una necesidad real, con un alcance sencillo y resultados evaluables.

## 2. Objetivo de la fase 1

Construir una **prueba de concepto funcional de extremo a extremo** que demuestre el recorrido desde la conversación hasta un pedido registrado en un backoffice de prueba.

La POC validará viabilidad técnica, utilidad de las recomendaciones y calidad de la interacción. La mejora de ingresos y la aceptación de huéspedes reales se medirán en un piloto posterior.

**Usuarios:** huésped que consulta y confirma su selección; personal de restaurante que mantiene la disponibilidad y recibe pedidos de prueba; equipo evaluador que verifica el recorrido.

## 3. Alcance de la POC

| Área | Incluido en fase 1 |
| --- | --- |
| Contexto | Un hotel y un restaurante/buffet, con 20–30 recetas y una carta reducida de vinos y bebidas. |
| Acceso | Aplicación web adaptable a móvil (PWA), accesible mediante enlace o QR. |
| Conversación | Voz y texto en español e inglés. Preferencias, presupuesto y preguntas de seguimiento dentro de la sesión. |
| Conocimiento gastronómico | Ingredientes, elaboración y maridajes. Bodega, uva y añada cuando existan datos en las fichas. |
| Imagen | Lectura de carteles para identificar un plato y consultar su ficha. Confirmación ante ambigüedad. |
| Recomendaciones | Opciones del menú vigente con precio, disponibilidad y régimen de demostración. Alternativas ante productos agotados. |
| Pedido | Resumen editable con productos, cantidades e importe. Confirmación explícita y registro en el backoffice de prueba con identificador y estado. |
| Operación | Carga inicial de catálogo y control básico para modificar disponibilidad y consultar pedidos. |

**Decisión de alcance propuesta:** la presentación contempla pedidos reales en la visión general y, a la vez, una POC sin dependencia de producción. En fase 1, el pedido se persistirá en un entorno de prueba; no implicará preparación real, cobro ni integración con cocina en producción.

## 4. Fuera de alcance

- Pagos, cargos a habitación e integración productiva con PMS, caja, cocina o app corporativa.
- Despliegue multihotel, nuevos idiomas y operación continua con huéspedes reales.
- Perfiles persistentes, fidelización y personalización entre estancias.
- Diagnóstico nutricional, prescripción de dietas y cálculo de calorías por fotografía.
- Conocimiento gastronómico no respaldado por las fuentes disponibles.

## 5. Datos y condiciones de funcionamiento

El catálogo incluirá identificadores, recetas, ingredientes, alérgenos validados, menú por servicio, fichas de vinos, precios, moneda, disponibilidad y reglas del régimen. Se importará desde archivos; los datos ficticios estarán identificados.

El responsable de Alimentos y Bebidas validará las fichas. El asistente no inferirá alérgenos a partir de imágenes ni garantizará ausencia de contaminación cruzada. Ante datos críticos ausentes o caducados, informará de la limitación y derivará al personal.

La POC utilizará sesiones anónimas, sin conservar audio ni imágenes por defecto. Las claves permanecerán en el servidor y las herramientas estarán limitadas al catálogo y al pedido de prueba. Antes de registrar un pedido se revalidarán precio y disponibilidad; un reintento no deberá duplicarlo.

## 6. Demo y criterios de aceptación

**Recorrido:** el huésped pide una opción vegetariana y un vino de menos de 30 euros; consulta su elaboración, recibe un maridaje y revisa la selección. El operador agota el vino, el asistente ofrece una alternativa y el huésped confirma. El pedido aparece en el backoffice de prueba. Una consulta adicional con información de alérgenos incompleta provoca derivación al personal.

| Comprobación | Objetivo propuesto |
| --- | --- |
| Flujo completo | Demostrable en ES/EN, con confirmación y pedido visible sin duplicados. |
| Recomendaciones | ≥95 % válidas en un conjunto revisado de al menos 50 escenarios. |
| Casos críticos | 100 % de los casos críticos del conjunto informa o deriva correctamente. |
| Experiencia | ≥90 % de tareas completas en pruebas guiadas. |
| Agilidad | p95 de respuesta útil ≤5 s y disponibilidad actualizada ≤60 s en el entorno de prueba. |

Estos umbrales son metas pendientes de acuerdo, no resultados obtenidos. La salida exige además ausencia de fallos críticos abiertos y registro de limitaciones conocidas.

## 7. Plan y entregables

**Preparación:** validar catálogo, responsables y escenarios. Estimación de una semana, condicionada a la disponibilidad de datos.

**Construcción durante el hackathon:** estimación de 2–3 días con 3–4 personas. Primero, catálogo y conversación; después, voz, imagen y pedido; finalmente, pruebas, correcciones y ensayo. Duración y equipo por confirmar.

**Entregables:** PWA ejecutable, catálogo de demostración, backoffice de prueba, instrucciones de ejecución, guion de demo e informe breve de pruebas y pendientes.

## 8. Continuidad

Tras aceptar la POC, la fase 2 será un piloto en un hotel con integraciones autorizadas y medición de tiempo de elección, satisfacción, margen de bebidas y coste por sesión. Las fases siguientes ampliarán a una cohorte regional y, si se acredita valor y capacidad operativa, a otras marcas y regiones.

**Pendientes para iniciar:** responsable de Iberostar, hotel de referencia, muestra de datos, reglas comerciales, entorno de prueba y fechas del hackathon.
