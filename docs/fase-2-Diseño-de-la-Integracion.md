## Fase 2: Diseño de la Integración

En esta fase se define cómo los servicios de la arquitectura SOA se comunican entre sí, estableciendo los mecanismos de integración, los patrones utilizados, las estrategias de manejo de errores y reintentos, y los principales flujos de comunicación.

La solución utiliza un modelo de integración híbrido que combina **comunicación síncrona mediante APIs REST** y **comunicación asíncrona mediante eventos y mensajería**.

---

### 1. Modelo de comunicación

La comunicación entre los servicios se divide en dos mecanismos principales:

| Mecanismo     | Uso                                               | Tecnología propuesta | Características                                                                 |
| ------------- | ------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------- |
| **Síncrono**  | Operaciones que requieren una respuesta inmediata | REST/HTTPS           | Comunicación directa, respuesta inmediata y contratos HTTP definidos            |
| **Asíncrono** | Procesos desacoplados y notificaciones de eventos | Kafka / RabbitMQ     | Comunicación mediante eventos, menor acoplamiento y procesamiento independiente |

### 1.1 Comunicación síncrona

Se utilizará **REST sobre HTTPS** cuando un servicio necesite una respuesta inmediata de otro servicio para continuar con el procesamiento.

Por ejemplo, durante la creación de un pago, el servicio de Gestión de Pagos puede consultar al servicio de Gestión de Cuentas y Fondos para validar la disponibilidad del dinero.

```text
Cliente
   |
   | HTTPS
   v
API Gateway
   |
   | REST
   v
Gestión de Pagos
   |
   | POST /accounts/funds/validate
   v
Gestión de Cuentas y Fondos
   |
   | Response
   v
Gestión de Pagos
```

Este mecanismo es apropiado para operaciones como:

* Validación de tokens.
* Consulta de información.
* Validación de fondos.
* Evaluación de riesgo.
* Consulta del estado de un pago.

### 1.2 Comunicación asíncrona

Se utilizará un **Message Broker** para publicar y consumir eventos relacionados con las transacciones.

Este mecanismo permite que un servicio publique un evento sin necesidad de conocer directamente qué servicios lo consumirán.

Por ejemplo:

```text
Gestión de Pagos
       |
       | PaymentApproved
       v
+-------------------+
|   Message Broker  |
+-------------------+
       |
       +-------------------> Liquidación
       |
       +-------------------> Notificaciones
       |
       +-------------------> Auditoría
```

Entre los eventos principales se pueden definir:

* `PaymentCreated`
* `PaymentApproved`
* `PaymentRejected`
* `PaymentSettled`
* `PaymentCancelled`

La comunicación asíncrona permite desacoplar procesos como la liquidación, las notificaciones y la auditoría del procesamiento principal del pago.

---

# 2. Patrones de integración

La arquitectura utiliza varios patrones de integración para organizar la comunicación entre los servicios.

## 2.1 API Gateway

El **API Gateway** funciona como punto de entrada para los consumidores externos del sistema.

```text
                  +----------------+
                  | Web / Mobile   |
                  | Applications   |
                  +-------+--------+
                          |
                       HTTPS
                          |
                          v
                  +---------------+
                  |  API Gateway  |
                  +-------+-------+
                          |
             +------------+------------+
             |            |            |
             v            v            v
         Pagos       Autenticación   Consulta
```

### Responsabilidades del API Gateway

* Recibir solicitudes externas.
* Enrutar solicitudes hacia los servicios correspondientes.
* Validar autenticación y autorización.
* Aplicar políticas de seguridad.
* Controlar el acceso a las APIs.
* Implementar rate limiting cuando sea necesario.
* Centralizar aspectos comunes de la comunicación.
* Generar o propagar identificadores de correlación.

El API Gateway evita que los consumidores necesiten conocer directamente la ubicación o implementación interna de cada servicio.

---

## 2.2 Event-Driven Architecture

La arquitectura utiliza un modelo **orientado a eventos** para comunicar cambios importantes en el estado de las transacciones.

Por ejemplo:

```text
Gestión de Pagos
       |
       | PaymentApproved
       v
 Message Broker
       |
       +------> Liquidación
       |
       +------> Notificaciones
       |
       +------> Auditoría
```

El productor publica el evento y los consumidores lo procesan de forma independiente.

### Beneficios

* Reduce el acoplamiento entre servicios.
* Permite agregar nuevos consumidores sin modificar el productor.
* Facilita el procesamiento asíncrono.
* Permite escalar consumidores de manera independiente.
* Mejora la resiliencia ante fallos temporales.

---

## 2.3 Patrón Saga

Para gestionar el flujo distribuido de un pago se propone utilizar el patrón **Saga**.

Una transacción de pago puede involucrar diferentes servicios:

```text
Gestión de Pagos
       |
       v
Validación de Riesgo
       |
       v
Validación de Fondos
       |
       v
Reserva de Fondos
       |
       v
Liquidación
```

Debido a que estos pasos pueden involucrar diferentes servicios y datos, no se recomienda utilizar una única transacción distribuida entre todas las fuentes.

En su lugar, cada servicio ejecuta su propia operación y, si posteriormente ocurre un error, se ejecutan acciones compensatorias.

### Ejemplo

```text
1. Crear pago
       |
       v
2. Validar riesgo
       |
       v
3. Reservar fondos
       |
       v
4. Ejecutar liquidación
       |
       X
   Error en liquidación
       |
       v
5. Liberar fondos
       |
       v
6. Marcar pago como rechazado
```

La compensación permite mantener la consistencia del proceso sin necesidad de una transacción distribuida.

### Operaciones compensatorias

| Operación             | Acción compensatoria |
| --------------------- | -------------------- |
| Reserva de fondos     | Liberar fondos       |
| Creación de pago      | Cancelar pago        |
| Liquidación parcial   | Ejecutar reversión   |
| Publicación de evento | Reprocesar evento    |
| Notificación fallida  | Reintentar envío     |

---

# 3. Flujo principal de creación de un pago

El flujo principal combina comunicación síncrona y asíncrona.

```text
Cliente
   |
   | 1. POST /payments
   v
API Gateway
   |
   | 2. Validar autenticación
   v
Autenticación
   |
   | 3. Token válido
   v
Gestión de Pagos
   |
   | 4. Evaluar riesgo
   v
Riesgo y Fraude
   |
   | 5. APPROVED
   v
Gestión de Pagos
   |
   | 6. Validar fondos
   v
Gestión de Cuentas y Fondos
   |
   | 7. Fondos disponibles
   v
Gestión de Pagos
   |
   | 8. PaymentApproved
   v
Message Broker
   |
   +----------------------+
   |                      |
   v                      v
Liquidación          Auditoría
   |
   | PaymentSettled
   v
Message Broker
   |
   v
Notificaciones
   |
   v
Cliente
```

---

# 4. Flujo de comunicación síncrona

Para una operación que requiere respuesta inmediata:

```text
┌──────────┐       REST        ┌──────────────────┐
│  Pagos   │ ────────────────> │ Riesgo y Fraude  │
└──────────┘                   └────────┬─────────┘
      ^                                 │
      |          Response               │
      └────────────────────────────────┘
```

Por ejemplo:

```http
POST /api/v1/risk/evaluate
```

El servicio de Riesgo responde:

```json
{
  "paymentId": "PAY-001",
  "decision": "APPROVED"
}
```

El servicio de Gestión de Pagos puede continuar únicamente después de recibir una respuesta válida.

---

# 5. Flujo de comunicación asíncrona

Para operaciones que no requieren respuesta inmediata:

```text
┌──────────────────┐
│ Gestión de Pagos │
└────────┬─────────┘
         |
         | PaymentApproved
         v
┌─────────────────────┐
│    Message Broker   │
└───────┬───────┬─────┘
        |       |
        v       v
┌────────────┐ ┌───────────────┐
│Liquidación │ │  Notificación │
└──────┬─────┘ └───────────────┘
       |
       | PaymentSettled
       v
┌─────────────────────┐
│    Message Broker   │
└──────────┬──────────┘
           |
           v
     ┌───────────┐
     │ Auditoría │
     └───────────┘
```

---

# 6. Manejo de errores

Cada servicio debe gestionar los errores de forma controlada y devolver información consistente a sus consumidores.

## 6.1 Errores HTTP

Para las APIs REST se utilizarán códigos HTTP estándar:

| Código                      | Situación                                         |
| --------------------------- | ------------------------------------------------- |
| `200 OK`                    | Operación procesada correctamente                 |
| `201 CREATED`               | Recurso creado correctamente                      |
| `400 BAD REQUEST`           | Datos de entrada inválidos                        |
| `401 UNAUTHORIZED`          | Usuario no autenticado                            |
| `403 FORBIDDEN`             | Usuario sin permisos                              |
| `404 NOT FOUND`             | Recurso no encontrado                             |
| `409 CONFLICT`              | Conflicto con el estado actual                    |
| `422 UNPROCESSABLE ENTITY`  | Datos válidos sintácticamente pero no procesables |
| `429 TOO MANY REQUESTS`     | Límite de solicitudes excedido                    |
| `500 INTERNAL SERVER ERROR` | Error interno                                     |
| `503 SERVICE UNAVAILABLE`   | Servicio temporalmente no disponible              |

Los errores deben utilizar una estructura común para facilitar su interpretación:

```json
{
  "code": "PAYMENT_001",
  "message": "Insufficient funds",
  "timestamp": "2026-09-27T15:00:00Z",
  "traceId": "abc-123-def"
}
```

---

# 7. Reintentos

Los errores temporales de comunicación no deben provocar inmediatamente el fallo definitivo de una operación.

Para errores transitorios se utilizarán reintentos controlados con **exponential backoff**.

Ejemplo:

```text
Primer intento
      |
      X Error temporal
      |
    1 segundo
      |
Segundo intento
      |
      X Error temporal
      |
    2 segundos
      |
Tercer intento
      |
      X Error temporal
      |
    4 segundos
      |
Cuarto intento
      |
      v
Procesamiento exitoso
```

Los reintentos deben aplicarse únicamente a errores que puedan ser temporales, como:

* `503 Service Unavailable`.
* Timeout.
* Fallos temporales de red.
* Indisponibilidad temporal del Message Broker.

No se deben reintentar automáticamente errores funcionales como:

* Fondos insuficientes.
* Usuario no autorizado.
* Datos inválidos.
* Pago inexistente.

---

# 8. Dead Letter Queue

Cuando un mensaje no puede procesarse después de los reintentos configurados, se enviará a una **Dead Letter Queue (DLQ)**.

```text
                  Message Broker
                       |
                       v
                 PaymentApproved
                       |
                       v
                 Liquidación
                       |
                    ERROR
                       |
                 Retry 1
                       |
                    ERROR
                       |
                 Retry 2
                       |
                    ERROR
                       |
                 Retry 3
                       |
                    ERROR
                       |
                       v
                Dead Letter Queue
                       |
                       v
             Revisión / Reprocesamiento
```

La DLQ permite conservar los mensajes fallidos para su posterior análisis y reprocesamiento, evitando su pérdida.

---

# 9. Idempotencia

Las operaciones críticas deben ser **idempotentes** para evitar que un reintento provoque una operación duplicada.

Por ejemplo, si el cliente envía nuevamente una solicitud para crear un pago debido a un timeout, el sistema debe poder identificar que la operación ya fue procesada.

Se puede utilizar un identificador de idempotencia:

```http
POST /api/v1/payments
Idempotency-Key: 8f3a1b20-1234-4567
```

El servicio de Gestión de Pagos utilizará esta clave para determinar si una solicitud corresponde a una operación nueva o a un reintento de una operación existente.

Esto es especialmente importante para operaciones financieras como:

* Débitos.
* Reservas de fondos.
* Liquidaciones.
* Reversiones.

---

# 10. Timeouts y Circuit Breaker

Las llamadas síncronas entre servicios deben tener un **timeout** definido para evitar que una dependencia lenta bloquee indefinidamente el procesamiento.

Ejemplo:

```text
Gestión de Pagos
       |
       | Request
       v
Riesgo y Fraude
       |
       | Sin respuesta
       |
      TIMEOUT
       |
       v
Circuit Breaker
       |
       v
Respuesta controlada
```

El patrón **Circuit Breaker** permite evitar llamadas repetitivas hacia un servicio que se encuentra temporalmente indisponible.

Sus estados principales son:

```text
       ┌─────────┐
       │ CLOSED  │
       └────┬────┘
            |
       Fallos consecutivos
            |
            v
       ┌─────────┐
       │  OPEN   │
       └────┬────┘
            |
       Tiempo de espera
            |
            v
       ┌────────────┐
       │ HALF-OPEN  │
       └─────┬──────┘
             |
       +-----+-----+
       |           |
    Éxito        Error
       |           |
       v           v
    CLOSED        OPEN
```

Esto evita que la indisponibilidad de un servicio genere un efecto dominó sobre toda la plataforma.

---

# 11. Trazabilidad y correlación

Todas las solicitudes y eventos deben utilizar un identificador de correlación que permita seguir una transacción a través de los diferentes servicios.

Ejemplo:

```text
traceId = 7f8a91c2-45de-4a91
```

El identificador se propaga durante todo el flujo:

```text
Cliente
   |
   | traceId
   v
API Gateway
   |
   | traceId
   v
Gestión de Pagos
   |
   | traceId
   +------> Riesgo
   |
   +------> Fondos
   |
   +------> Message Broker
                 |
                 +----> Liquidación
                 |
                 +----> Auditoría
                 |
                 +----> Notificaciones
```

Esto facilita:

* Diagnóstico de errores.
* Seguimiento de transacciones.
* Análisis de tiempos de respuesta.
* Auditoría.
* Monitoreo distribuido.

---

# 12. Diagrama general de integración

```text
                                  ┌──────────────────┐
                                  │ Web / Mobile App │
                                  └────────┬─────────┘
                                           │
                                         HTTPS
                                           │
                                           v
                                  ┌──────────────────┐
                                  │   API Gateway    │
                                  └────────┬─────────┘
                                           │
                                           v
                              ┌─────────────────────────┐
                              │ Autenticación y          │
                              │ Autorización             │
                              └────────────┬────────────┘
                                           │
                                           v
                              ┌─────────────────────────┐
                              │   Gestión de Pagos       │
                              └──────┬────────┬─────────┘
                                     │        │
                              REST   │        │ REST
                                     │        │
                                     v        v
                              ┌──────────┐ ┌──────────────┐
                              │  Riesgo  │ │   Cuentas    │
                              │ /Fraude  │ │   y Fondos   │
                              └──────────┘ └──────────────┘
                                     │        │
                                     └────┬───┘
                                          │
                                          v
                                  ┌───────────────┐
                                  │ Message Broker│
                                  └───────┬───────┘
                                          │
                       ┌──────────────────┼──────────────────┐
                       │                  │                  │
                       v                  v                  v
                ┌─────────────┐   ┌──────────────┐   ┌─────────────┐
                │ Liquidación │   │Notificaciones│   │  Auditoría  │
                └──────┬──────┘   └──────────────┘   └─────────────┘
                       │
                       │ PaymentSettled
                       v
                ┌───────────────┐
                │ Message Broker │
                └───────────────┘
```

---

# 13. Resumen de la estrategia de integración

La arquitectura propuesta utiliza diferentes mecanismos según la naturaleza de cada interacción:

| Necesidad                         | Mecanismo                | Patrón                       |
| --------------------------------- | ------------------------ | ---------------------------- |
| Acceso de clientes                | REST/HTTPS               | API Gateway                  |
| Autenticación                     | REST/HTTPS               | Request/Response             |
| Evaluación de riesgo              | REST/HTTPS               | Synchronous Request/Response |
| Validación de fondos              | REST/HTTPS               | Synchronous Request/Response |
| Liquidación                       | Eventos                  | Event-Driven                 |
| Notificaciones                    | Eventos                  | Event-Driven                 |
| Auditoría                         | Eventos                  | Event-Driven                 |
| Transacciones distribuidas        | Eventos + compensaciones | Saga                         |
| Errores temporales                | Retry + Backoff          | Resiliencia                  |
| Mensajes que no pueden procesarse | Dead Letter Queue        | Messaging                    |
| Solicitudes repetidas             | Idempotency-Key          | Idempotencia                 |
| Dependencias indisponibles        | Circuit Breaker          | Resiliencia                  |
| Seguimiento de operaciones        | Correlation ID           | Trazabilidad                 |

## Principios de integración

La estrategia de integración se basa en los siguientes principios:

* **Bajo acoplamiento:** los servicios se comunican mediante contratos y eventos, evitando dependencias sobre implementaciones internas.
* **Comunicación síncrona controlada:** REST se utiliza únicamente cuando la respuesta es necesaria para continuar el flujo.
* **Comunicación asíncrona:** los eventos se utilizan para desacoplar procesos como liquidación, notificaciones y auditoría.
* **Resiliencia:** se incorporan timeouts, reintentos, exponential backoff, circuit breaker y Dead Letter Queues.
* **Idempotencia:** las operaciones financieras críticas deben poder procesarse de forma segura ante reintentos.
* **Trazabilidad:** las transacciones utilizan identificadores de correlación para seguir el flujo completo.
* **Consistencia eventual:** los procesos asíncronos pueden actualizar sus estados de manera independiente después de completarse la operación principal.
* **Compensación:** el patrón Saga permite revertir operaciones previamente realizadas cuando una etapa posterior del proceso falla.
* **Seguridad:** toda comunicación externa utiliza HTTPS y los servicios validan la identidad y autorización correspondiente.
