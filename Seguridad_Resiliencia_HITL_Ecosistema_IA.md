# Seguridad, Privacidad, Resiliencia y Human-in-the-Loop

## Ecosistema IA de Gestión de Reseñas – Days Inn Parque Termal Dolores

## 1. Objetivo

Este documento describe los principales controles de seguridad, privacidad, resiliencia y supervisión humana implementados en el ecosistema de gestión de reseñas.

La arquitectura busca reducir cuatro riesgos principales:

- Procesar datos incompletos o inválidos.
- Generar respuestas con información no respaldada.
- Contactar al huésped sin revisión humana.
- Perder trazabilidad cuando una integración o etapa del workflow falla.

Para ello, el sistema combina validación de entrada, recuperación de conocimiento mediante RAG, Human-in-the-Loop (HITL), reintentos en integraciones críticas, rutas de contingencia y registro centralizado de ejecuciones y errores.

## 2. Minimización de datos

El workflow limita los datos tratados a aquellos necesarios para gestionar la reseña y mantener la trazabilidad operativa.

Entre los principales datos utilizados se encuentran nombre del huésped, email, fecha de estadía, calificación, aspectos positivos, aspectos a mejorar e información técnica necesaria para identificar la reseña y su ejecución.

El email funciona como identificador lógico para buscar y reutilizar huéspedes existentes, evitando crear registros duplicados innecesarios.

La información se distribuye entre bases diferenciadas de Notion: Huéspedes, Reseñas, Base de Conocimiento y Ejecuciones y Errores. Las credenciales de los servicios externos no se almacenan como información de negocio dentro de estas bases.

## 3. Validación de entrada

El nodo **2. Validar y Normalizar** extrae y normaliza los datos recibidos desde Tally y verifica los campos requeridos. El resultado se utiliza en **3. ¿Datos Válidos?**.

Si los datos son válidos, el workflow continúa hacia la identificación del huésped y creación de la reseña. Si son inválidos, se desvía a **E. Etapa Validación → E. Registrar Error** y no continúa hacia la generación de respuesta ni hacia el contacto con el huésped.

## 4. Persistencia y trazabilidad

Notion funciona como memoria persistente del ecosistema mediante cuatro bases:

- **Huéspedes:** identificación y reutilización de huéspedes mediante email.
- **Reseñas:** reseña original, análisis, borrador, aprobación y resultado final.
- **Base de Conocimiento:** información autorizada para el AI Agent.
- **Ejecuciones y Errores:** trazabilidad técnica de ejecuciones exitosas y fallidas.

Esta separación permite reconstruir qué ocurrió durante el procesamiento de una reseña.

## 5. RAG y control anti-alucinación

La secuencia **8. Consultar Base de Conocimiento → 9. Consolidar Contexto RAG → 10. AI Agent – Analizar y Redactar** recupera información autorizada desde Notion y la entrega al agente junto con la reseña.

Las instrucciones del agente establecen que la respuesta debe basarse exclusivamente en la información proporcionada por el huésped y el contenido disponible en la Base de Conocimiento. El modelo no debe inventar servicios, horarios, promociones, políticas, beneficios u otra información no respaldada.

Este mecanismo reduce el riesgo de alucinaciones, aunque no sustituye la supervisión humana posterior.

## 6. Generación estructurada

El AI Agent analiza la reseña y genera información estructurada, incluyendo sentimiento, urgencia y borrador de respuesta. Separar el análisis de la acción externa permite almacenar y revisar la salida antes de producir una comunicación real.

## 7. Human-in-the-Loop obligatorio

Después de generar el borrador, **11. Guardar Análisis y Borrador** actualiza la reseña y la deja pendiente de revisión. **12. Notificar Revisor Interno** informa al responsable humano.

Luego el workflow ejecuta:

**13. Esperar Aprobación → 14. Consultar Estado Reseña → 15. ¿Aprobado?**

El envío al huésped solamente puede producirse cuando existe aprobación.

### ¿Por qué el HITL está ubicado antes de Gmail?

Este punto representa la frontera entre un proceso interno y una acción externa. Hasta entonces la IA analiza y redacta, n8n procesa y Notion almacena. Gmail representa una comunicación con impacto directo sobre el huésped.

Por eso la arquitectura sigue el principio:

**IA genera → humano revisa → sistema ejecuta.**

## 8. Comportamiento sin aprobación

Si **Aprobado = false**, el workflow no envía la respuesta. Continúa con el mecanismo de espera y comprobación definido para el HITL. La ausencia de intervención humana nunca se interpreta automáticamente como aprobación.

## 9. Protección contra bucles de aprobación

Después de comprobar que la reseña todavía no fue aprobada, **17. ¿Límite de Chequeos?** evita una espera indefinida.

Mientras no se alcance el límite, el proceso puede volver a **13. Esperar Aprobación**. Si se alcanza, se deriva hacia **E. Etapa HITL → E. Registrar Error** y finaliza sin contactar al huésped.

## 10. Resiliencia ante fallos

El workflow incorpora mecanismos de resiliencia en integraciones críticas relacionadas con Notion/Base de Datos, recuperación RAG, AI Agent/Gemini y Gmail.

En los nodos donde está configurado, **Retry On Fail** permite reintentar operaciones ante determinados fallos transitorios.

No se asume que todos los nodos utilizan la misma cantidad de reintentos; la configuración concreta depende de cada nodo. Si el problema persiste, las rutas de contingencia identifican la etapa afectada y registran el incidente.

## 11. Rutas de contingencia

El workflow diferencia errores mediante rutas como:

- **E. Etapa Validación**
- **E. Etapa Base de Datos**
- **E. Etapa RAG**
- **E. Etapa IA**
- **E. Etapa Gmail**
- **E. Etapa HITL**

Todas convergen en **E. Registrar Error**, facilitando localizar el origen del fallo.

## 12. Registro centralizado de errores

Los incidentes se registran en **Notion – Ejecuciones y Errores**. Entre los campos contemplados se encuentran ID de ejecución, estado, etapa, nodo, mensaje de error e información de reintentos cuando corresponde.

El objetivo es evitar fallos silenciosos y mantener evidencia técnica de las interrupciones.

## 13. Seguridad del canal de salida

La secuencia conceptual es:

**RAG → AI Agent → Guardar Borrador → Revisión Humana → Aprobación → Gmail**

El AI Agent no dispone de una ruta directa hacia el huésped. Esta separación entre generación y ejecución reduce el riesgo de enviar automáticamente información inventada, respuestas inadecuadas o contenido generado con contexto insuficiente.

## 14. Registro del camino exitoso

Después del envío, el workflow continúa:

**16. Enviar Respuesta al Huésped → 18. Guardar Envío → 19. Registrar Ejecución Exitosa**

Así se conserva el resultado final y se diferencia una ejecución exitosa de aquellas almacenadas mediante rutas de error.

## 15. Camino feliz y camino infeliz

### Camino feliz

**Tally → Webhook → Validación → Identificación/creación del huésped → Creación de reseña → RAG → AI Agent → Guardado del borrador → Revisión humana → Aprobación → Gmail → Actualización de la reseña → Registro exitoso**

### Camino infeliz

**Datos inválidos / Base de Datos / RAG / IA / HITL / Gmail → identificación de etapa → E. Registrar Error → Notion – Ejecuciones y Errores**

Cuando un fallo impide completar de forma segura el proceso, el huésped no debe ser contactado.

## 16. Evidencias de prueba

Durante la validación se probaron escenarios de reseñas positivas, negativas y mixtas; distinta urgencia; consultas respondibles y no respondibles desde la Base de Conocimiento; reutilización y creación de huéspedes; espera HITL sin aprobación; aprobación humana y posterior envío; y recorridos completos de extremo a extremo.

## 17. Riesgos residuales y mejoras futuras

Para una implementación productiva podrían incorporarse:

- Registro más granular de reintentos reales.
- Métricas automáticas de latencia, tokens y costo por ejecución.
- Alertas ante tasas de error elevadas.
- Políticas formales de retención y eliminación de datos.
- Permisos más restrictivos según rol.
- Mayor monitoreo de servicios externos.
- Evolución del polling HITL hacia un mecanismo event-driven.
- Revisión periódica de la Base de Conocimiento.

## 18. Conclusión

La seguridad del ecosistema combina varias capas:

**Validación de entrada → Minimización de datos → Persistencia y trazabilidad → RAG anti-alucinación → Generación estructurada → Human-in-the-Loop → Control anti-bucle → Retry On Fail en integraciones críticas → Rutas de contingencia → Registro centralizado de ejecuciones y errores**

La decisión arquitectónica principal es mantener separadas la generación de contenido y la acción externa. El AI Agent puede analizar y redactar, pero la comunicación con el huésped queda condicionada a una aprobación humana previa.
