# ejercicio-2
# Diseño de Alto Nivel: Endpoint de Consulta de Saldo de Comercio

**1. Componentes de Microservicios Involucrados y Sus Responsabilidades:**

Propongo una arquitectura con los siguientes microservicios:

* **API Gateway:**
    * **Responsabilidades:** Actúa como el único punto de entrada para todas las solicitudes de los clientes. Se encarga del enrutamiento de la solicitud al `ServicioConsultaSaldo`, la autenticación inicial, la autorización básica, la limitación de tasa para prevenir abusos.

* **ServicioConsultaSaldo (Microservicio Principal):**
    * **Responsabilidades:**
        * Expone el endpoint REST (ej. `GET /api/v1/comercios/{idComercio}/saldo`).
        * Valida la solicitud.
        * Manejs la lógica de negocio: inicia el proceso de consulta de saldo, que implica una comunicación asíncrona con el sistema legado.
        * Publica un mensaje de "solicitud de consulta de saldo" en una cola/tópico del Message Broker.
        * Espera la respuesta del sistema legado (que llegará a través de otra cola/tópico en el Message Broker) de forma síncrona para el cliente pero con un timeout interno.
        * Maneja errores, timeouts de la espera, y respuestas de patrones de resiliencia.
        * Una vez recibida la respuesta del legado, se encarga de proteger la información sensible antes de construir la respuesta final al API Gateway.
        * Registra detalles de la solicitud y su resultado para trazabilidad.

* **ProxySistemaLegado (Microservicio Adaptador):**
    * **Responsabilidades:**
        * Actúa como un intermediario y adaptador entre los microservicios y el sistema legado.
        * Se suscribe a la cola/tópico de "solicitud de consulta de saldo" del Message Broker.
        * Transforma el mensaje de solicitud al formato o protocolo que el sistema legado entiende.
        * Se comunica con el sistema legado para obtener el saldo.
        * Maneja la lógica de reintentos y timeouts específica para la interacción con el sistema legado.
        * Una vez obtenida la respuesta del legado, la transforma a un formato canónico y la publica en una cola/tópico de "respuesta de saldo" en el Message Broker, para que el `ServicioConsultaSaldo` la consuma.

* **Message Broker (ej. RabbitMQ):**
    * **Responsabilidades:**
        * Facilita la comunicación asíncrona, desacoplada y resiliente entre el `ServicioConsultaSaldo` y el `ProxySistemaLegado`.
        * Contendrá al menos dos colas/tópicos principales:
            1.  `cola_solicitudes_saldo`: Donde el `ServicioConsultaSaldo` publica las peticiones.
            2.  `cola_respuestas_saldo`: Donde el `ProxySistemaLegado` publica los resultados obtenidos del sistema legado.

**2. Mecanismos de Trazabilidad/Observabilidad:**
* **Logs:**
    * **Formato Estructurado (JSON):** Todos los microservicios generarán logs en formato JSON para facilitar su parseo y análisis.
    * **Información Clave en Logs:** Cada entrada de log incluirá como mínimo: `timestamp`, `nivelDeLog` (INFO, ERROR, WARN, DEBUG), `nombreDelServicio`, `traceId` (para rastreo distribuido), `correlationId` (si aplica, para seguir un flujo de negocio específico a través de múltiples interacciones), `idComercio` (cuando sea relevante), mensaje descriptivo, y `stackTrace` en caso de errores.
      
* **Distributed Tracing:**
    * **Implementación:** Se utilizará la especificación y SDKs de OpenTelemetry en cada microservicio.
    * **Generación y Propagación de IDs:** Un `traceId` único se generará al inicio de la solicitud. Este `traceId`, junto con los `spanId` para cada operación individual, se propagará a través de las cabeceras HTTP entre servicios y a través de las propiedades de los mensajes en el Message Broker.
    * **Recolección y Visualización:** Las trazas se enviarán a un colector OpenTelemetry y luego a un backend de rastreo como Zipkin, donde se podrán visualizar las interacciones completas, identificar cuellos de botella.
    * **Correlación:** El `traceId` se incluirá en todos los logs para permitir una fácil correlación entre una traza específica y los logs detallados generados durante su procesamiento.

---

**3. Estrategia de Concurrencia y Patrón de Resiliencia:**

Para asegurar alta disponibilidad y robustez:

* **Estrategia de Concurrencia:**
    * En los microservicios, especialmente `ServicioConsultaSaldo` y `ProxySistemaLegado`, se utilizarán **Hilos Virtuales**. Esto permitirá manejar un gran número de solicitudes concurrentes y operaciones de I/O con un uso muy eficiente de los hilos de plataforma del sistema operativo, mejorando la escalabilidad y el throughput.
* **Patrones de Resiliencia:**
    * **Circuit Breaker:**
        * Se aplicará en el `ProxySistemaLegado` para las llamadas directas al sistema legado. Si el sistema legado comienza a fallar repetidamente o a responder muy lento, el Circuit Breaker se "abrirá", evitando más llamadas al legado por un tiempo y devolviendo un error rápido o un fallback al `ProxySistemaLegado`.
        * También podría usarse en el `ServicioConsultaSaldo` si la publicación en el `Message Broker` falla consistentemente.
    * **Retry (Reintentos con Backoff Exponencial):**
        * Para operaciones transitorias fallidas, como la publicación de mensajes en el `Message Broker` o llamadas al sistema legado que podrían fallar por problemas de red temporales.
    * **Timeouts:**
        * Configurar timeouts en todas las interacciones de red:
            * Llamadas HTTP entre servicios.
            * Llamada del `ProxySistemaLegado` al sistema legado.
            * La espera del `ServicioConsultaSaldo` por una respuesta de la cola de mensajes.
        * Esto evita que los hilos queden bloqueados indefinidamente.
---

**4. Modelo de Cifrado para los Datos Sensibles en Tránsito y en Reposo:**

* **Identificación de Datos Sensibles:** El "saldo disponible" es el dato más obviamente sensible. También se considerarán sensibles los identificadores y cualquier token de autenticación.
* **En Tránsito:**
    * **TLS/HTTPS:** Toda la comunicación HTTP, tanto la externa (cliente -> API Gateway) como la interna entre microservicios, se realizará obligatoriamente sobre HTTPS utilizando TLS.
    * **Cifrado en el Message Broker:** La comunicación entre los microservicios y el `Message Broker` debe estar cifrada usando TLS.
    * **Gestión de Secretos:** Todas las credenciales (claves de API, contraseñas para el Message Broker, claves de cifrado de aplicación) deben ser almacenadas y gestionadas de forma segura.
* **Protección en la Respuesta:**
    * El `ServicioConsultaSaldo` solo debe devolver la información estrictamente necesaria en la respuesta (ej. el saldo).
    * Evitar exponer identificadores internos o cualquier otro dato que no sea esencial para el cliente.
    * La autorización es manejada por el API Gateway y el `ServicioConsultaSaldo`.

---

**6. Justificación de Decisiones Clave:**

* **Microservicios:** Se eligió esta arquitectura para promover el desacoplamiento, la escalabilidad independiente de los componentes, el aislamiento de fallos, y permitir desarrollos y despliegues más ágiles y focalizados. La separación del `ProxySistemaLegado` es importante para poder aislar la complejidad del sistema legado del resto de la plataforma moderna.
* **Mensajería Asíncrona con el Legado:** Desacopla el flujo principal del `ServicioConsultaSaldo` de la latencia y disponibilidad directa del sistema legado, mejorando la resiliencia general del sistema.
* **Patrones de Resiliencia (Circuit Breaker, Retry, Timeout):** Son fundamentales en arquitecturas distribuidas para construir sistemas tolerantes a fallos que puedan manejar problemas transitorios y prevenir fallos en cascada.
* **Observabilidad Completa (Logs, Tracing):** Esencial para operar, monitorear la salud y el rendimiento, y depurar problemas eficientemente en un sistema compuesto por múltiples servicios interconectados. Faltan las métricas pero no sé bien como implementar esto.
* **Seguridad en Capas (TLS, Cifrado de Datos, Gestión de Secretos):** Se aplica un enfoque de defensa en profundidad en todos los niveles de la arquitectura.
* **Hilos Virtuales:** Para la estrategia de concurrencia, opto por hilos virtuales para manejar eficientemente un alto volumen de operaciones de I/O concurrentes que son inherentes a este tipo de servicio, simplificando el código concurrente y mejorando la escalabilidad.
* Me faltaron los diagramas UML pero no tengo conocimiento de como hacerlos, es un tema que voy a investigar.
