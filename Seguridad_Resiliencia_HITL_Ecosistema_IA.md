# Seguridad, Resiliencia y Human-in-the-Loop

## Ecosistema IA de Gestión de Reseñas – Days Inn Parque Termal Dolores

## 1. Objetivo

Este documento describe las medidas de seguridad, resiliencia y control humano incorporadas al workflow de n8n para reducir riesgos operativos, evitar envíos incorrectos y mantener trazabilidad sobre cada ejecución.

La arquitectura combina validación de datos, rutas de error, reintentos automáticos, recuperación controlada de información mediante RAG y un punto obligatorio de aprobación humana antes de contactar al huésped.

---

## 2. Minimización y protección de datos

El sistema utiliza únicamente la información necesaria para procesar una reseña:

- Nombre del huésped.
- Email.
- Fecha de estadía.
- Calificación.
- Comentarios positivos.
- Aspectos a mejorar.
- Datos técnicos vinculados a la ejecución.

No se almacenan contraseñas, claves API ni credenciales dentro de las bases de Notion.

Las credenciales de Notion, Gmail y Gemini se gestionan mediante el sistema de Credentials de n8n y no aparecen hardcodeadas dentro de los nodos del workflow.

El email se utiliza como identificador lógico para evitar duplicar huéspedes. La información se separa entre las bases Huéspedes, Reseñas y Ejecuciones y Errores para evitar duplicación innecesaria de datos.

---

## 3. Validación de datos de entrada

El nodo:

**2. Validar y Normalizar**

comprueba la existencia de los campos obligatorios antes de continuar:

- Nombre.
- Email.
- Fecha de estadía.
- Calificación.
- Aspectos positivos.

El resultado de la validación se almacena en la variable booleana:

`valido`

El nodo:

**3. ¿Datos Válidos?**

decide si el workflow puede continuar.

### Camino válido

`valido = true`

El flujo continúa hacia la búsqueda o creación del huésped.

### Camino inválido

`valido = false`

La ejecución se desvía hacia:

**E. Etapa Validación → E. Registrar Error**

El AI Agent no se ejecuta y no se contacta al huésped.

Esta segunda capa de validación protege al sistema incluso si el origen de datos cambia o un payload defectuoso alcanza directamente el webhook.

---

## 4. Resiliencia ante fallos de APIs

Los nodos críticos de integración están configurados con:

**Retry On Fail = true**

y con rutas específicas de error.

Las etapas protegidas incluyen:

- Notion / Base de datos.
- Recuperación del contexto RAG.
- AI Agent / Gemini.
- Gmail.
- HITL.

Cuando una operación crítica falla, el flujo utiliza la salida de error y deriva hacia un nodo de clasificación de etapa.

Ejemplos:

- **E. Etapa Base de Datos**
- **E. Etapa RAG**
- **E. Etapa IA**
- **E. Etapa Gmail**
- **E. Etapa HITL**

Todas estas rutas convergen en:

**E. Registrar Error**

---

## 5. Registro centralizado de errores

La base de Notion **Ejecuciones y Errores** funciona como registro técnico del sistema.

Ante un fallo se almacena:

- ID Ejecución.
- Estado de ejecución = Error.
- Etapa del workflow.
- Nodo relacionado.
- Mensaje de error.
- Número de reintentos.

Esto permite identificar rápidamente en qué componente ocurrió un problema y alimentar el Dashboard de Control.

La arquitectura permite distinguir errores de:

- Validación.
- Base de datos.
- RAG.
- Inteligencia Artificial.
- Gmail.
- HITL.

---

## 6. Human-in-the-Loop (HITL)

El sistema incorpora un punto obligatorio de supervisión humana antes de ejecutar la acción crítica de contactar al huésped.

Después de que el AI Agent genera el borrador, el nodo:

**11. Guardar Análisis y Borrador**

actualiza la reseña con:

- Borrador IA.
- Sentimiento IA.
- Urgencia IA.
- Requiere Revision = true.
- Aprobado = false.
- Estado = Pendiente de aprobación.

Posteriormente:

**12. Notificar Revisor Interno**

envía un email al responsable con:

- Nombre del huésped.
- Calificación.
- Reseña original.
- Sentimiento.
- Urgencia.
- Borrador generado.
- Enlace directo al registro en Notion.

En este punto el AI Agent ya terminó su trabajo, pero el sistema todavía no puede contactar al huésped.

---

## 7. Espera y validación humana

El nodo:

**13. Esperar Aprobación**

detiene temporalmente la ejecución.

Actualmente el intervalo está configurado en:

**1 minuto**

Luego:

**14. Consultar Estado Reseña**

vuelve a leer el registro en Notion.

El nodo:

**15. ¿Aprobado?**

evalúa el checkbox `Aprobado`.

### Si Aprobado = false

El flujo no envía ningún email al huésped y pasa al control anti-bucle.

### Si Aprobado = true

El flujo continúa hacia:

**16. Enviar Respuesta al Huésped**

De esta manera, la decisión final queda exclusivamente en manos del operador humano.

---

## 8. Protección contra bucles infinitos

El workflow evita que el polling de aprobación continúe indefinidamente.

El nodo:

**17. ¿Límite de Chequeos?**

evalúa el número de iteraciones mediante:

`$runIndex >= 12`

Si todavía no se alcanzó el límite, el workflow vuelve al nodo:

**13. Esperar Aprobación**

Si se alcanza el límite, la ejecución se desvía hacia:

**E. Etapa HITL → E. Registrar Error**

y el proceso finaliza sin contactar al huésped.

Con el intervalo actual de un minuto, el diseño contempla aproximadamente hasta 12 comprobaciones antes del corte de seguridad.

---

## 9. Control anti-alucinación mediante RAG

El AI Agent recibe un contexto privado construido desde la Base de Conocimiento de Notion.

La secuencia es:

**8. Consultar Base de Conocimiento**
→
**9. Consolidar Contexto RAG**
→
**10. AI Agent - Analizar y Redactar**

El prompt del agente establece que debe utilizar exclusivamente:

- La reseña del huésped.
- La Base de Conocimiento proporcionada.

El modelo tiene prohibido inventar:

- Servicios.
- Horarios.
- Promociones.
- Políticas.
- Beneficios.
- Información del hotel no presente en las fuentes autorizadas.

Durante las pruebas se validaron consultas sobre mascotas, traslados y beneficios inexistentes en el RAG. El agente evitó confirmar información no respaldada.

---

## 10. Separación entre generación y acción

Una decisión de seguridad central es separar:

**Generar contenido**

de:

**Ejecutar una acción externa**

El AI Agent nunca dispone de autonomía para enviar directamente la respuesta.

La secuencia obligatoria es:

**AI Agent**
→
**Guardar Borrador**
→
**Notificar Humano**
→
**Esperar**
→
**Aprobar**
→
**Gmail**

Esta separación reduce el riesgo de que una respuesta incorrecta, una alucinación o un error de contexto llegue directamente al cliente.

---

## 11. Seguridad del canal de salida

La comunicación final se ejecuta mediante Gmail únicamente después de que:

- La reseña fue validada.
- El huésped fue identificado.
- El RAG fue recuperado.
- El AI Agent finalizó correctamente.
- El borrador fue guardado.
- El revisor humano aprobó el contenido.

Después del envío, el workflow almacena:

- Respuesta final.
- Gmail Thread ID.
- Fecha de envío.
- Estado = Enviado.

Esto mantiene trazabilidad entre la reseña, la aprobación humana y la comunicación efectivamente realizada.

---

## 12. Camino infeliz

El sistema contempla explícitamente situaciones donde el flujo no debe avanzar.

Entre ellas:

### Datos incompletos
Se registra error y no se procesa con IA.

### Error de Notion
Se aplican reintentos y, si persiste, se registra el incidente.

### Error de RAG
No se permite continuar hacia generación o envío si el contexto no puede recuperarse correctamente.

### Error de IA
No existe borrador válido y no se contacta al huésped.

### Falta de aprobación
El flujo continúa esperando y no envía nada.

### Límite HITL alcanzado
La ejecución se detiene y registra el incidente.

### Error de Gmail
Se registra la falla y no se considera la ejecución como exitosa.

---

## 13. Evidencias obtenidas durante las pruebas

El workflow fue probado con múltiples escenarios:

- Reseña de 5 estrellas.
- Reseña positiva con consulta respondible desde el RAG.
- Reseña negativa.
- Reseña crítica clasificada con urgencia Alta.
- Consulta con información inexistente en el RAG.
- Experiencia mixta.
- Reutilización de huésped existente mediante email.
- Creación de huésped nuevo.
- Ciclos HITL sin aprobación.
- Aprobación posterior y envío exitoso.

Estas pruebas permitieron comprobar tanto el camino feliz como distintas rutas de control y seguridad.

---

## 14. Riesgos residuales y mejoras futuras

Aunque el prototipo incorpora múltiples controles, existen mejoras posibles:

- Registrar dinámicamente el número real de reintentos.
- Registrar consumo real de tokens.
- Calcular costo de IA por ejecución.
- Incorporar alertas automáticas ante tasas de error elevadas.
- Añadir roles o permisos más restrictivos para la aprobación humana.
- En producción, ampliar los controles de privacidad y retención de datos personales.
- Sustituir o complementar el polling HITL por un mecanismo event-driven si el entorno lo permite.

---

## 15. Conclusión

La arquitectura implementada evita que el ecosistema dependa únicamente de la salida de un modelo de Inteligencia Artificial.

La validación inicial, las rutas de error, los reintentos, el RAG, la protección anti-bucle y el Human-in-the-Loop crean una serie de barreras de control antes de que cualquier comunicación llegue al huésped.

De esta manera, el sistema combina automatización y autonomía operativa con supervisión humana, trazabilidad y mecanismos de recuperación ante fallos.
