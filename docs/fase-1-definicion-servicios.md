## Arquitectura SOA propuesta

### Fase 1: Definición de Servicios

En esta fase se identifican y definen los servicios necesarios para soportar las principales capacidades de un sistema de pagos. Cada servicio posee una responsabilidad específica, una interfaz definida y un nivel de independencia que permite reducir el acoplamiento entre los componentes de la solución.

| Servicio                             | Propósito                                                         | Responsabilidades principales                                             | Interfaz           |
| ------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------ |
| **1. Autenticación y Autorización**  | Gestionar la identidad y el acceso de los usuarios                | Login, emisión y validación de tokens, roles y permisos                   | REST/HTTPS         |
| **2. Gestión de Pagos**              | Gestionar el ciclo de vida de un pago                             | Crear, validar, consultar, aprobar y cancelar pagos                       | REST/HTTPS         |
| **3. Validación de Riesgo y Fraude** | Determinar si una transacción puede procesarse de forma segura    | Validar reglas de riesgo, listas de fraude y comportamiento transaccional | REST + eventos     |
| **4. Gestión de Cuentas y Fondos**   | Validar y gestionar la disponibilidad de fondos                   | Consultar cuenta, validar saldo, realizar débito y reserva de fondos      | REST/HTTPS         |
| **5. Liquidación de Pagos**          | Ejecutar la liquidación financiera de las transacciones aprobadas | Confirmar movimientos, gestionar estados de liquidación y conciliación    | Mensajería/eventos |
| **6. Notificaciones**                | Informar al usuario sobre el resultado de las operaciones         | Email, SMS, push y gestión de plantillas                                  | Mensajería         |
| **7. Auditoría**                     | Mantener la trazabilidad de las operaciones                       | Registrar eventos, usuario, fecha, operación y resultado                  | Eventos/logs       |

---

### 1. Servicio de Autenticación y Autorización

* **Propósito:** garantizar que solamente los usuarios y aplicaciones autorizadas puedan utilizar el sistema.

* **Responsabilidades:**

  * Autenticación de usuarios y aplicaciones.
  * Emisión y validación de tokens.
  * Gestión de roles y permisos.
  * Control de acceso a las operaciones.

* **Interfaz:** API REST sobre HTTPS.

* **Datos:**

  * Identificador de usuario.
  * Credenciales.
  * Roles.
  * Permisos.
  * Tokens.

* **Independencia:** puede funcionar de forma independiente y ser utilizado por los demás servicios que requieran validar la identidad y los permisos de un consumidor.

---

### 2. Servicio de Gestión de Pagos

* **Propósito:** administrar el ciclo de vida de una transacción de pago.

* **Responsabilidades:**

  * Crear pagos.
  * Validar la información de la transacción.
  * Consultar el estado de un pago.
  * Aprobar o rechazar pagos.
  * Cancelar pagos cuando corresponda.

* **Interfaz:** API REST sobre HTTPS.

* **Datos:**

  * Identificador del pago.
  * Monto.
  * Moneda.
  * Cuenta o medio de origen.
  * Cuenta o medio de destino.
  * Referencia.
  * Estado de la transacción.

* **Independencia:** concentra la lógica propia del ciclo de vida del pago, pero delega responsabilidades especializadas como autenticación, evaluación de riesgo y gestión de fondos a los servicios correspondientes.

---

### 3. Servicio de Validación de Riesgo y Fraude

* **Propósito:** evaluar una transacción antes de permitir su procesamiento, con el fin de identificar operaciones que puedan representar un riesgo.

* **Responsabilidades:**

  * Aplicar reglas de riesgo.
  * Consultar listas de fraude.
  * Analizar características de la transacción.
  * Evaluar el comportamiento transaccional.
  * Generar una decisión de aprobación o rechazo.

* **Interfaz:** API REST y comunicación mediante eventos.

* **Datos:**

  * Información de la transacción.
  * Identificador del usuario.
  * Monto.
  * Moneda.
  * Información contextual de la operación.
  * Resultado del análisis.
  * Nivel o puntuación de riesgo.

* **Independencia:** mantiene sus propias reglas y mecanismos de evaluación, permitiendo modificarlos o evolucionarlos sin alterar directamente la lógica del servicio de Gestión de Pagos.

---

### 4. Servicio de Gestión de Cuentas y Fondos

* **Propósito:** verificar y gestionar la disponibilidad de dinero necesaria para ejecutar un pago.

* **Responsabilidades:**

  * Consultar el saldo de una cuenta.
  * Validar la disponibilidad de fondos.
  * Reservar fondos.
  * Debitar fondos.
  * Gestionar los movimientos relacionados con la operación.

* **Interfaz:** API REST sobre HTTPS.

* **Datos:**

  * Identificador de cuenta.
  * Saldo.
  * Moneda.
  * Monto.
  * Movimientos.
  * Estado de disponibilidad de fondos.

* **Independencia:** mantiene la responsabilidad sobre la administración de los fondos y evita que otros servicios accedan o modifiquen directamente sus datos internos.

---

### 5. Servicio de Liquidación de Pagos

* **Propósito:** completar el movimiento financiero de las transacciones que han sido aprobadas.

* **Responsabilidades:**

  * Recibir pagos aprobados.
  * Ejecutar la liquidación financiera.
  * Actualizar el estado de la transacción.
  * Gestionar los estados de liquidación.
  * Ejecutar procesos de conciliación.

* **Interfaz:** mensajería y eventos.

* **Datos:**

  * Identificador de la transacción.
  * Cuentas involucradas.
  * Monto.
  * Moneda.
  * Fecha de operación.
  * Estado de liquidación.
  * Información de conciliación.

* **Independencia:** procesa las transacciones recibidas mediante eventos sin requerir una interacción directa y permanente con el usuario.

---

### 6. Servicio de Notificaciones

* **Propósito:** comunicar al usuario los resultados y eventos relevantes relacionados con sus operaciones.

* **Responsabilidades:**

  * Enviar notificaciones.
  * Gestionar plantillas de comunicación.
  * Determinar el canal de envío.
  * Procesar notificaciones por email, SMS o push.
  * Registrar el resultado del envío.

* **Interfaz:** sistema de mensajería, como Kafka o RabbitMQ.

* **Datos:**

  * Destinatario.
  * Tipo de notificación.
  * Plantilla.
  * Mensaje.
  * Canal de comunicación.
  * Evento asociado.
  * Estado del envío.

* **Independencia:** funciona de manera desacoplada del procesamiento principal. Una falla temporal en el servicio de notificaciones no debería impedir que un pago pueda continuar su procesamiento.

---

### 7. Servicio de Auditoría

* **Propósito:** garantizar la trazabilidad de las operaciones realizadas dentro del sistema.

* **Responsabilidades:**

  * Registrar eventos relevantes.
  * Mantener información de auditoría.
  * Registrar usuario, fecha, operación y resultado.
  * Asociar eventos con identificadores de transacción.
  * Facilitar consultas posteriores para seguimiento y trazabilidad.

* **Interfaz:** eventos, mensajería y almacenamiento centralizado.

* **Datos:**

  * Eventos.
  * Timestamps.
  * Identificadores de transacción.
  * Identificador de usuario.
  * Operación realizada.
  * Resultado.
  * Información de trazabilidad.

* **Independencia:** recibe eventos generados por los demás servicios y registra la información de auditoría sin intervenir directamente en el procesamiento de las transacciones.

---

### Principios aplicados en la definición de servicios

La definición de los servicios sigue los siguientes principios de diseño SOA:

* **Separación de responsabilidades:** cada servicio se enfoca en una capacidad de negocio específica.
* **Bajo acoplamiento:** los servicios interactúan mediante interfaces y contratos definidos, evitando dependencias directas sobre las implementaciones internas.
* **Alta cohesión:** las funcionalidades relacionadas se mantienen dentro del mismo servicio.
* **Independencia:** cada servicio administra sus propias responsabilidades y datos.
* **Granularidad adecuada:** se evita crear servicios excesivamente pequeños o servicios que concentren múltiples responsabilidades.
* **Interoperabilidad:** los servicios utilizan mecanismos estándar como REST/HTTPS y mensajería.
* **Trazabilidad:** las operaciones pueden ser rastreadas mediante identificadores de transacción y eventos de auditoría.
* **Ausencia de dependencias circulares:** las responsabilidades y relaciones entre servicios se mantienen claramente delimitadas.
