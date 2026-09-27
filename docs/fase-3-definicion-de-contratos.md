# Fase 3: Definición de Contratos

En esta fase se definen los contratos de comunicación entre los servicios que conforman la arquitectura SOA. Los contratos establecen de forma explícita las operaciones disponibles, estructuras de solicitud y respuesta, validaciones, reglas de negocio, códigos de error y modelos de datos.

El objetivo es garantizar una integración **consistente, predecible, segura y desacoplada** entre los diferentes consumidores y proveedores de servicios.

---

## 1. Estándares de los contratos

Los servicios REST utilizarán los siguientes estándares:

| Característica                  | Estándar           |
| ------------------------------- | ------------------ |
| Protocolo                       | HTTPS              |
| Arquitectura                    | REST               |
| Formato de datos                | JSON               |
| Especificación                  | OpenAPI 3.x        |
| Autenticación                   | Bearer Token / JWT |
| Identificación de transacciones | `paymentId`        |
| Trazabilidad                    | `traceId`          |
| Idempotencia                    | `Idempotency-Key`  |
| Versionamiento                  | `/api/v1/`         |
| Comunicación asíncrona          | Eventos JSON       |
| Message Broker                  | Kafka / RabbitMQ   |

Los contratos deben ser independientes de la implementación interna de cada servicio. Un consumidor debe conocer **qué operación puede utilizar y qué información debe enviar o esperar**, pero no cómo está implementado el servicio.

---

# 2. Especificación OpenAPI

Los servicios que exponen APIs REST tendrán una especificación OpenAPI que permita documentar y validar sus contratos.

Una especificación simplificada para el servicio de **Gestión de Pagos** sería:

```yaml
openapi: 3.0.3

info:
  title: Payment Management API
  description: API para gestionar el ciclo de vida de los pagos
  version: 1.0.0

servers:
  - url: https://api.example.com/api/v1

paths:

  /payments:
    post:
      summary: Crear un pago
      operationId: createPayment

      security:
        - bearerAuth: []

      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema:
            type: string

        - name: X-Trace-Id
          in: header
          required: true
          schema:
            type: string

      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePaymentRequest'

      responses:
        '201':
          description: Pago creado correctamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'

        '400':
          description: Solicitud inválida

        '401':
          description: No autenticado

        '403':
          description: Sin permisos

        '409':
          description: Operación duplicada o conflicto

        '422':
          description: Regla de negocio no cumplida

        '500':
          description: Error interno

  /payments/{paymentId}:
    get:
      summary: Consultar un pago
      operationId: getPayment

      security:
        - bearerAuth: []

      parameters:
        - name: paymentId
          in: path
          required: true
          schema:
            type: string

      responses:
        '200':
          description: Información del pago
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaymentResponse'

        '401':
          description: No autenticado

        '403':
          description: Sin permisos

        '404':
          description: Pago no encontrado

        '500':
          description: Error interno

components:

  securitySchemes:

    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:

    CreatePaymentRequest:
      $ref: '#/components/schemas/CreatePaymentRequest'

    PaymentResponse:
      $ref: '#/components/schemas/PaymentResponse'
```

La especificación completa puede mantenerse en un archivo:

```text
docs/
└── openapi/
    ├── authentication-api.yaml
    ├── payment-api.yaml
    ├── risk-api.yaml
    └── accounts-api.yaml
```

---

# 3. Contratos de los servicios REST

## 3.1 Servicio de Autenticación y Autorización

### Endpoint

```http
POST /api/v1/auth/token
```

### Request

```json
{
  "username": "usuario123",
  "password": "********"
}
```

### Response

```json
{
  "accessToken": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

### Validaciones

* El usuario debe existir.
* Las credenciales deben ser válidas.
* La cuenta debe encontrarse habilitada.
* El usuario debe tener permisos para la operación solicitada.
* El token debe encontrarse vigente al acceder a los servicios protegidos.

### Errores

| Código HTTP | Código     | Descripción                      |
| ----------- | ---------- | -------------------------------- |
| `400`       | `AUTH_001` | Datos de autenticación inválidos |
| `401`       | `AUTH_002` | Credenciales incorrectas         |
| `403`       | `AUTH_003` | Usuario sin permisos             |
| `500`       | `AUTH_500` | Error interno de autenticación   |

---

# 4. Servicio de Gestión de Pagos

## 4.1 Crear un pago

### Endpoint

```http
POST /api/v1/payments
```

### Headers

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: 8f3a1b20-1234-4567
X-Trace-Id: abc-123-def
```

### Request

```json
{
  "sourceAccount": "123456789",
  "destinationAccount": "987654321",
  "amount": 150000,
  "currency": "COP",
  "reference": "FACTURA-2026-001"
}
```

### Response

```json
{
  "paymentId": "PAY-20260927-00001",
  "status": "PENDING",
  "amount": 150000,
  "currency": "COP",
  "reference": "FACTURA-2026-001",
  "createdAt": "2026-09-27T15:00:00Z"
}
```

### Validaciones

* `sourceAccount` es obligatorio.
* `destinationAccount` es obligatorio.
* Las cuentas deben tener un formato válido.
* La cuenta de origen debe existir.
* La cuenta de destino debe existir.
* `amount` debe ser mayor que cero.
* `currency` debe corresponder a una moneda soportada.
* `reference` no debe superar la longitud máxima definida.
* El usuario debe estar autenticado y autorizado.
* La operación debe utilizar una clave de idempotencia.

### Reglas de negocio

1. Una cuenta no puede realizar un pago hacia sí misma.
2. El monto debe encontrarse dentro de los límites permitidos.
3. La cuenta de origen debe disponer de fondos suficientes.
4. La transacción debe superar la validación de riesgo.
5. Un mismo `Idempotency-Key` no debe generar múltiples pagos.
6. Un pago aprobado debe continuar hacia el proceso de liquidación.

---

# 5. Consultar un pago

### Endpoint

```http
GET /api/v1/payments/{paymentId}
```

### Request

```http
GET /api/v1/payments/PAY-20260927-00001
Authorization: Bearer <token>
X-Trace-Id: abc-123-def
```

### Response

```json
{
  "paymentId": "PAY-20260927-00001",
  "status": "SETTLED",
  "amount": 150000,
  "currency": "COP",
  "sourceAccount": "123456789",
  "destinationAccount": "987654321",
  "reference": "FACTURA-2026-001",
  "createdAt": "2026-09-27T15:00:00Z",
  "updatedAt": "2026-09-27T15:02:00Z"
}
```

### Estados permitidos

```text
PENDING
APPROVED
REJECTED
SETTLED
CANCELLED
```

### Validaciones

* `paymentId` es obligatorio.
* El pago debe existir.
* El consumidor debe estar autenticado.
* El consumidor debe tener autorización para consultar la operación.

---

# 6. Servicio de Validación de Riesgo y Fraude

## 6.1 Evaluar una transacción

### Endpoint

```http
POST /api/v1/risk/evaluate
```

### Request

```json
{
  "paymentId": "PAY-20260927-00001",
  "customerId": "CUS-001",
  "amount": 150000,
  "currency": "COP",
  "sourceAccount": "123456789"
}
```

### Response

```json
{
  "paymentId": "PAY-20260927-00001",
  "decision": "APPROVED",
  "riskScore": 15
}
```

### Posibles decisiones

```text
APPROVED
REJECTED
REVIEW
```

### Validaciones

* El `paymentId` debe existir.
* El monto debe ser mayor que cero.
* La moneda debe estar soportada.
* El cliente debe encontrarse identificado.
* La información mínima necesaria para evaluar el riesgo debe estar disponible.

### Reglas de negocio

* Las transacciones que incumplan las reglas de riesgo pueden ser rechazadas.
* Las transacciones que requieran análisis adicional pueden quedar en estado `REVIEW`.
* Una transacción rechazada por riesgo no debe continuar hacia la liquidación.

---

# 7. Servicio de Gestión de Cuentas y Fondos

## 7.1 Validar disponibilidad de fondos

### Endpoint

```http
POST /api/v1/accounts/funds/validate
```

### Request

```json
{
  "accountId": "123456789",
  "amount": 150000,
  "currency": "COP"
}
```

### Response

```json
{
  "accountId": "123456789",
  "available": true,
  "availableBalance": 850000,
  "currency": "COP"
}
```

### Validaciones

* La cuenta debe existir.
* La cuenta debe estar activa.
* El monto debe ser mayor que cero.
* La moneda debe coincidir con la moneda de la cuenta.
* El saldo disponible debe ser suficiente.

### Regla principal

```text
availableBalance >= requestedAmount
```

Si esta condición no se cumple, el servicio debe responder indicando que no existen fondos suficientes.

---

# 8. Servicio de Liquidación de Pagos

El servicio de Liquidación utiliza principalmente comunicación asíncrona.

### Evento de entrada

```text
PaymentApproved
```

### Payload

```json
{
  "eventId": "evt-123456",
  "eventType": "PaymentApproved",
  "occurredAt": "2026-09-27T15:01:00Z",
  "paymentId": "PAY-20260927-00001",
  "sourceAccount": "123456789",
  "destinationAccount": "987654321",
  "amount": 150000,
  "currency": "COP"
}
```

### Evento de salida

```text
PaymentSettled
```

### Payload

```json
{
  "eventId": "evt-789012",
  "eventType": "PaymentSettled",
  "occurredAt": "2026-09-27T15:02:00Z",
  "paymentId": "PAY-20260927-00001",
  "status": "SETTLED"
}
```

### Reglas de negocio

* Solo pueden liquidarse pagos aprobados.
* Un pago ya liquidado no debe volver a liquidarse.
* El evento debe contener un identificador único.
* El procesamiento debe ser idempotente.
* Ante errores temporales, el mensaje debe ser reintentado.
* Si se excede el número máximo de reintentos, el mensaje debe enviarse a una DLQ.

---

# 9. Servicio de Notificaciones

El servicio recibe eventos generados durante el procesamiento de los pagos.

### Evento de entrada

```text
PaymentSettled
```

### Payload

```json
{
  "eventId": "evt-789012",
  "paymentId": "PAY-20260927-00001",
  "status": "SETTLED",
  "notificationType": "PAYMENT_SUCCESS"
}
```

### Resultado esperado

El servicio determina el canal de comunicación configurado y genera la notificación correspondiente.

Los canales soportados pueden ser:

```text
EMAIL
SMS
PUSH
```

### Reglas de negocio

* El destinatario debe encontrarse identificado.
* Debe existir una plantilla para el tipo de evento.
* Los mensajes fallidos deben poder reintentarse.
* Un error en la notificación no debe revertir automáticamente una liquidación exitosa.

---

# 10. Servicio de Auditoría

El servicio de Auditoría recibe eventos generados por los demás servicios.

### Evento

```text
AuditEvent
```

### Payload

```json
{
  "eventId": "audit-001",
  "traceId": "abc-123-def",
  "paymentId": "PAY-20260927-00001",
  "service": "payment-service",
  "operation": "CREATE_PAYMENT",
  "userId": "USR-001",
  "status": "SUCCESS",
  "timestamp": "2026-09-27T15:00:00Z"
}
```

### Reglas

* Los eventos de auditoría deben conservar el identificador de trazabilidad.
* Los registros no deben modificarse después de ser almacenados.
* Cada evento debe tener un identificador único.
* La auditoría no debe modificar el resultado de una transacción.

---

# 11. Contrato estándar de errores

Todos los servicios REST utilizarán una estructura de error común.

```json
{
  "code": "PAYMENT_001",
  "message": "Insufficient funds",
  "timestamp": "2026-09-27T15:00:00Z",
  "traceId": "abc-123-def",
  "details": []
}
```

### Campos

| Campo       | Tipo     | Descripción                     |
| ----------- | -------- | ------------------------------- |
| `code`      | String   | Código único del error          |
| `message`   | String   | Descripción del error           |
| `timestamp` | DateTime | Fecha y hora del error          |
| `traceId`   | String   | Identificador para trazabilidad |
| `details`   | Array    | Información adicional del error |

---

# 12. Códigos de error

Los servicios utilizarán códigos HTTP estándar acompañados de códigos funcionales.

| HTTP  | Código        | Descripción                          |
| ----- | ------------- | ------------------------------------ |
| `400` | `COMMON_001`  | Solicitud inválida                   |
| `401` | `AUTH_001`    | Token inválido o ausente             |
| `403` | `AUTH_002`    | Usuario sin permisos                 |
| `404` | `PAYMENT_001` | Pago no encontrado                   |
| `409` | `PAYMENT_002` | Operación duplicada                  |
| `409` | `PAYMENT_003` | Conflicto de estado                  |
| `422` | `PAYMENT_004` | Regla de negocio incumplida          |
| `422` | `FUNDS_001`   | Fondos insuficientes                 |
| `422` | `RISK_001`    | Transacción rechazada por riesgo     |
| `429` | `COMMON_002`  | Límite de solicitudes excedido       |
| `500` | `COMMON_500`  | Error interno                        |
| `503` | `COMMON_503`  | Servicio temporalmente no disponible |

---

# 13. Manejo de excepciones

Los errores se clasifican en dos categorías principales.

## Errores funcionales

Representan situaciones donde la solicitud no puede procesarse debido a una condición de negocio.

Ejemplos:

```text
Fondos insuficientes
Cuenta inexistente
Pago inexistente
Usuario sin permisos
Transacción rechazada por riesgo
Pago ya liquidado
```

Estos errores **no deben ser reintentados automáticamente**.

## Errores transitorios

Son errores que pueden desaparecer después de un período corto.

Ejemplos:

```text
Timeout
Servicio temporalmente indisponible
Error temporal de red
Message Broker temporalmente indisponible
```

Estos errores pueden procesarse utilizando:

* Reintentos.
* Exponential backoff.
* Circuit breaker.
* Dead Letter Queue.

---

# 14. Esquemas JSON

Los modelos de datos deben definirse mediante esquemas que permitan validar la estructura de los mensajes.

## 14.1 CreatePaymentRequest

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "CreatePaymentRequest",
  "type": "object",
  "required": [
    "sourceAccount",
    "destinationAccount",
    "amount",
    "currency",
    "reference"
  ],
  "properties": {
    "sourceAccount": {
      "type": "string",
      "minLength": 5,
      "maxLength": 30
    },
    "destinationAccount": {
      "type": "string",
      "minLength": 5,
      "maxLength": 30
    },
    "amount": {
      "type": "number",
      "exclusiveMinimum": 0
    },
    "currency": {
      "type": "string",
      "enum": [
        "COP",
        "USD",
        "EUR"
      ]
    },
    "reference": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    }
  }
}
```

---

## 14.2 PaymentResponse

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "PaymentResponse",
  "type": "object",
  "required": [
    "paymentId",
    "status",
    "amount",
    "currency",
    "reference",
    "createdAt"
  ],
  "properties": {
    "paymentId": {
      "type": "string"
    },
    "status": {
      "type": "string",
      "enum": [
        "PENDING",
        "APPROVED",
        "REJECTED",
        "SETTLED",
        "CANCELLED"
      ]
    },
    "amount": {
      "type": "number",
      "exclusiveMinimum": 0
    },
    "currency": {
      "type": "string"
    },
    "sourceAccount": {
      "type": "string"
    },
    "destinationAccount": {
      "type": "string"
    },
    "reference": {
      "type": "string"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    },
    "updatedAt": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

---

## 14.3 PaymentApprovedEvent

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "PaymentApprovedEvent",
  "type": "object",
  "required": [
    "eventId",
    "eventType",
    "occurredAt",
    "paymentId",
    "amount",
    "currency"
  ],
  "properties": {
    "eventId": {
      "type": "string"
    },
    "eventType": {
      "type": "string",
      "const": "PaymentApproved"
    },
    "occurredAt": {
      "type": "string",
      "format": "date-time"
    },
    "paymentId": {
      "type": "string"
    },
    "sourceAccount": {
      "type": "string"
    },
    "destinationAccount": {
      "type": "string"
    },
    "amount": {
      "type": "number",
      "exclusiveMinimum": 0
    },
    "currency": {
      "type": "string"
    }
  }
}
```

---

# 15. Reglas de compatibilidad y versionamiento

Los contratos deben poder evolucionar sin afectar inmediatamente a los consumidores existentes.

Para esto se utilizará versionamiento de las APIs:

```text
/api/v1/payments
/api/v2/payments
```

Los cambios compatibles pueden incluir:

* Agregar campos opcionales.
* Agregar nuevos eventos.
* Agregar nuevos valores cuando los consumidores puedan ignorarlos correctamente.

Los cambios incompatibles deben generar una nueva versión del contrato.

Ejemplo:

```text
v1 → v2
```

La versión anterior debe mantenerse durante un período de transición para permitir la migración de los consumidores.

---

# 16. Contratos de eventos

Los eventos deben contener metadatos comunes para facilitar su trazabilidad y procesamiento.

Estructura recomendada:

```json
{
  "eventId": "evt-123456",
  "eventType": "PaymentApproved",
  "eventVersion": "1.0",
  "occurredAt": "2026-09-27T15:01:00Z",
  "traceId": "abc-123-def",
  "payload": {
    "paymentId": "PAY-20260927-00001",
    "amount": 150000,
    "currency": "COP"
  }
}
```

### Campos comunes

| Campo          | Propósito                                     |
| -------------- | --------------------------------------------- |
| `eventId`      | Identificar de forma única el evento          |
| `eventType`    | Identificar el tipo de evento                 |
| `eventVersion` | Controlar la evolución del contrato           |
| `occurredAt`   | Indicar cuándo ocurrió el evento              |
| `traceId`      | Mantener la trazabilidad                      |
| `payload`      | Contener la información específica del evento |

---

# 17. Flujo contractual de un pago

Los contratos definidos permiten establecer el siguiente flujo:

```text
┌──────────────┐
│    Cliente   │
└──────┬───────┘
       │
       │ POST /payments
       ▼
┌──────────────────┐
│ Gestión de Pagos │
└────────┬─────────┘
         │
         │ POST /risk/evaluate
         ▼
┌──────────────────┐
│ Riesgo y Fraude  │
└────────┬─────────┘
         │
         │ APPROVED
         ▼
┌──────────────────┐
│ Gestión de Pagos │
└────────┬─────────┘
         │
         │ POST /funds/validate
         ▼
┌──────────────────┐
│ Cuentas y Fondos │
└────────┬─────────┘
         │
         │ AVAILABLE
         ▼
┌──────────────────┐
│ Gestión de Pagos │
└────────┬─────────┘
         │
         │ PaymentApproved
         ▼
┌──────────────────┐
│  Message Broker  │
└───────┬─────┬────┘
        │     │
        ▼     ▼
┌────────────┐ ┌───────────────┐
│Liquidación │ │   Auditoría   │
└──────┬─────┘ └───────────────┘
       │
       │ PaymentSettled
       ▼
┌──────────────────┐
│  Message Broker  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Notificaciones  │
└──────────────────┘
```

---

# 18. Organización de los contratos

Los contratos pueden mantenerse en el repositorio con una estructura similar a la siguiente:

```text
docs/
└── contracts/
    ├── openapi/
    │   ├── authentication-api.yaml
    │   ├── payment-api.yaml
    │   ├── risk-api.yaml
    │   └── accounts-api.yaml
    │
    ├── schemas/
    │   ├── create-payment-request.json
    │   ├── payment-response.json
    │   ├── payment-approved-event.json
    │   └── payment-settled-event.json
    │
    └── events/
        ├── payment-created.json
        ├── payment-approved.json
        ├── payment-rejected.json
        └── payment-settled.json
```

---

# 19. Resumen de contratos

| Servicio                         | Tipo de contrato | Operación/Eventos                      |
| -------------------------------- | ---------------- | -------------------------------------- |
| **Autenticación y Autorización** | REST             | `POST /auth/token`                     |
| **Gestión de Pagos**             | REST             | `POST /payments`, `GET /payments/{id}` |
| **Riesgo y Fraude**              | REST             | `POST /risk/evaluate`                  |
| **Cuentas y Fondos**             | REST             | `POST /accounts/funds/validate`        |
| **Liquidación**                  | Eventos          | `PaymentApproved`, `PaymentSettled`    |
| **Notificaciones**               | Eventos          | `PaymentSettled`                       |
| **Auditoría**                    | Eventos          | `AuditEvent`                           |

---

# 20. Principios aplicados en la definición de contratos

Los contratos definidos cumplen los siguientes principios:

* **Contratos explícitos:** cada servicio documenta claramente sus operaciones y modelos de datos.
* **Interoperabilidad:** se utilizan estándares como REST, HTTPS, JSON y OpenAPI.
* **Versionamiento:** los contratos cuentan con mecanismos para evolucionar sin afectar inmediatamente a los consumidores.
* **Validación:** los datos de entrada y salida poseen reglas y esquemas definidos.
* **Manejo consistente de errores:** todos los servicios utilizan una estructura común para comunicar errores.
* **Idempotencia:** las operaciones financieras críticas pueden procesarse de manera segura ante reintentos.
* **Trazabilidad:** los contratos incorporan identificadores de correlación.
* **Desacoplamiento:** los consumidores dependen del contrato y no de la implementación interna.
* **Seguridad:** las APIs protegidas requieren autenticación y autorización.
* **Consistencia:** los eventos utilizan una estructura común y versionada.
* **Evolución controlada:** los cambios incompatibles requieren una nueva versión del contrato.

## Resultado de la Fase 3

Con la definición de estos contratos se establece una frontera clara entre los servicios de la arquitectura SOA. Cada consumidor conoce **cómo invocar un servicio, qué información debe proporcionar, qué respuesta puede recibir, cuáles son las reglas de negocio y cómo interpretar los errores**.

De esta manera, las fases anteriores quedan conectadas:

```text
FASE 1
Definición de Servicios
        │
        ▼
¿Qué servicios existen?
        │
        ▼
FASE 2
Diseño de Integración
        │
        ▼
¿Cómo se comunican?
        │
        ▼
FASE 3
Definición de Contratos
        │
        ▼
¿Qué deben enviar y recibir?
¿Qué reglas deben cumplir?
¿Cómo se manejan los errores?
```
