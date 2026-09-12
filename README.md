Proyecto: Evaluador y Coordinador de Reclamos B2B
Autor: Alberto Tomás Di Chiara
Orquestador: n8n Enlace github
Repositorio GitHub: https://github.com/Cococabj/Entrega-final-coder-automation
Base de Datos / Dashboard: Airtable Shared View - Dashboard de Control y KPIs 
1. Mapa de Arquitectura e Inserción del Flujo
1.1 Diagrama del Sistema (PDF - Diagrama Integrado)
El flujo opera como un pipeline asíncrono y resiliente que canaliza los reclamos entrantes a través de Telegram, los procesa mediante un Agente de IA con memoria contextual, evalúa la validez del problema, aplica un punto de validación humana (HITL) por Gmail y distribuye la resolución final hacia el cliente y los sectores internos.



1.2 Detalle de Nodos del Orquestador (n8n)
Telegram Trigger (telegramTrigger): Escucha mensajes entrantes de clientes en tiempo real.
Crea registro de Airtable (airtable): Inserta la entrada inicial en estado "Pendiente" con el canal asignado como "Telegram".
AI Agent (gpt-5-mini): Ejecuta la evaluación del reclamo usando gpt-5-mini.
Simple Memory (memoryBufferWindow): Mantiene un contexto de conversación por usuario (contextWindowLength: 2) asociándolo al ID de Telegram (from.id).
Structured Output Parser (outputParserStructured): Garantiza la devolución estricta en formato JSON con campos tipados.
Switch (switch): Evalúa es_reclamo_valido (true/false) para derivar el caso.
Gmail HITL (gmail - sendAndWait): Detiene la ejecución y envía una solicitud de aprobación de doble opción al supervisor.
If (if): Evalúa la respuesta del supervisor ($json.data.approved == true).
Solicita información al cliente (telegram - customForm): Envía un formulario dinámico a Telegram para adjuntar factura/foto/contacto.
Crea registro de error (airtable): Captura las fallas en la invocación de la IA (ruta de resiliencia onError: continueErrorOutput).
2. Manual Operativo de Datos
2.1 Esquema de la Base de Datos (Airtable: Evaluador de Reclamos)
Tabla Principal: Reclamos
ID_Reclamo (Auto Number / String): Identificador único del reclamo.
Cliente (Single Line Text): Nombre completo obtenido de Telegram (message.from.first_name + last_name).
Canal_Ingreso (Single Select): Telegram, Email, Correo electrónico.
Detalle_Reclamo (Long Text): Cuerpo del mensaje original emitido por el cliente.
Decision_IA (Single Select): Cambio de Producto, Reemplazo, Reembolso parcial / total, Disculpas y cupón, Rechazado.
Resolucion_Propuesta_IA (Long Text): Respuesta formal redactada dinámicamente por la IA.
Sector_Derivado (Link to Sectores): Relación de tabla hacia Ventas, Atención al Cliente, Mantenimiento o Sistemas.
Estado (Single Select): Pendiente, Procesado por IA, Aprobado por Humano, Requiere Revisión Manual.
Aprobado (Checkbox): true / false.
Fila_Errores (Link to Registro_Errores): Relación 1:N hacia los fallos técnicos registrados en la transacción.
Tabla Secundaria: Registro_Errores (tblUpZwvLeXG88WCC)
ID_Error (Auto Number / String): Identificador del error registrado.
ID_Reclamo_Afectado (Link to Reclamos): Vínculo relacional al ID del reclamo donde falló el proceso.
Fecha_Hora (Created Time): Timestamp de la falla.
Detalle_Tecnico (Long Text): Traceback o mensaje de error raw enviado por el nodo AI Agent.
Resuelto (Checkbox): Estado de atención del error técnico.

3. Matriz de Optimización de Costos y Modelos de IA
Tarea en el Flujo
Modelo Elegido
Razón Técnica de Selección
Costo por 1M Tokens (Entrada / Salida)
Ahorro Estimado vs Modelos Flagship
Clasificación y Extracción de Reclamos
gpt-5-mini
Alta capacidad de razonamiento estructurado con ventana de contexto liviana. Latencia ultrabaja para respuestas inmediatas en bot.
~$0.15 / $0.60 USD
~85% de ahorro comparado con la implementación directa en GPT-4o o Claude 3.5 Sonnet.
Generación de Respuestas HITL Estructuradas
Prompt acotado + Max Tokens (< 200 palabras)
Restricción explícita en el System Prompt para evitar el desborde de tokens y alucinaciones de texto extenso.
N/A (Optimización por Prompting)
~40% de consumo menor de tokens de salida en cada iteración del agente.
Procesamiento de Adjuntos/Documentos (Futuro)
Batch API (OpenAI/Anthropic)
Recomendado para procesamiento diferido nocturno de facturas o reclamos no urgentes acumulados.
Descuento del 50% en tarifas base
50% de ahorro directo en tareas que toleran ejecución asíncrona de hasta 24h.


4. Documentación de Seguridad, Resiliencia y HITL
4.1 Minimización de Datos y Seguridad de Privacidad
Sin Hardcoding: Todas las credenciales de API (Telegram API, OpenAI API, Gmail OAuth2 y Airtable OAuth2) están desacopladas en el gestor de credenciales seguro de n8n (credentials).
Control de Alcance: El bot de Telegram no guarda información financiera sensible directa; los archivos y fotos de facturación son gestionados por un formulario interactivo enviado post-aprobación del supervisor.
Protección contra Bucles Infinitos:
La integración de contextWindowLength: 2 limita la profundidad del buffer de memoria para evitar bucles de consumo por peticiones repetitivas.
La lógica de actualización en Airtable utiliza coincidencia por ID_Reclamo único para prevenir registros duplicados.
4.2 Resiliencia y Rutas de Error (Error Handlers)
Error Trigger / Catch en IA: El nodo AI Agent posee la directiva onError: "continueErrorOutput".
Fallback de Base de Datos: Cuando ocurre una falla de token, caída del proveedor de LLM o formato inválido, el flujo deriva la ejecución al nodo Crea registro de error, el cual registra el traceback técnico en la tabla Registro_Errores vinculada al reclamo afectado.
4.3 Mecanismo Human-in-the-Loop (HITL)
Punto de Pausa Activa (Gmail HITL): Utiliza el nodo con operación sendAndWait. El sistema no emite órdenes operativas ni reembolsos automáticamente.
Doble Validación: El correo enviado al supervisor expone los detalles del reclamo y la propuesta de la IA.
Aprobado (true): Cambia el estado en Airtable a "Aprobado por Humano", dispara la recolección de datos adicionales al cliente y notifica al área operativa por correo.
Rechazado (false): Cambia el estado a "Requiere Revisión Manual" y remite las observaciones del supervisor al equipo de soporte sin enviar promesas erróneas al cliente
5. Dashboard de Control y KPIs
El monitoreo del ecosistema se realiza mediante la vista compartida de Airtable (Shared View), que actúa como panel de control en tiempo real.
Link Público al Dashboard: Airtable Shared View - Dashboard de Control y KPIs (Configurado en modo lectura)


Métricas Principales Monitoreadas:
Tasa de Aprobación HITL: % de casos validados por humanos vs. rechazados o derivados a revisión manual
Ratio de Validez de Reclamos: Distribución de reclamos válidos vs. mensajes ambiguos/spam
Tasa de Error del Sistema: Número de registros en Registro_Errores sobre el total de ejecuciones procesadas.
Distribución por Sectores: Clasificación de carga de trabajo derivada a Ventas, Atención al Cliente, Mantenimiento y Sistemas.
