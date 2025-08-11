# Bucacar API – Diseño Reflexivo de Endpoints

**Autor:** Joan Sebastián Omaña Suarez
**Fecha:** 11 de agosto de 2025

---

## 1) ¿Qué es un endpoint?

Un **endpoint** es una URL específica de la API que representa una operación sobre un recurso (p. ej., `/v1/rides/{id}/complete`) y define **método HTTP**, **ruta**, **parámetros** y **contratos de entrada/salida**.

## 2) Endpoint público vs privado

* **Público:** No requiere autenticación (p. ej., `GET /status`).
* **Privado:** Requiere token (p. ej., `GET /v1/users/me`) y puede restringirse por **rol** o **scopes**.

## 3) Datos confidenciales de un usuario

Información personal identificable: teléfono, email, documento, dirección exacta, token de pago, histórico de ubicación, credenciales. Solo se exponen bajo consentimiento y con mínimos necesarios.

## 4) Importancia de elegir bien los métodos HTTP

Los métodos comunican intención y habilitan **idempotencia**, **caché**, **seguridad** y **semántica** clara: `GET` (lectura), `POST` (creación/acción), `PUT/PATCH` (actualización), `DELETE` (eliminación).

## 5) Información que requiere autenticación

Cualquier dato personal, gestión de viajes, estados operativos, métodos de pago, ratings y vistas administrativas.

## 6) Seguridad de la ubicación

* **Frecuencia y precisión limitadas** (throttling, degradación de precisión cuando no sea necesario).
* **Canales seguros** (TLS 1.2+), **tokens de corto plazo** y **firma**.
* **Reglas de acceso** por rol y minimización de datos históricos.

## 7) Si no hay conductores disponibles

Responder con **202 Accepted**, tambien se puede responder un un json especificando el error, en este caso que no hay disponibilidad

## 8) Identificación de recursos principales

`users`, `drivers`, `vehicles`, `rides`, `payments`, `payment-methods`, `ratings`, `locations`, `admin/*`, `price-rules`.

## 9) Beneficios de versionar (`/v1`)

Permite evolucionar la API sin romper clientes existentes, facilitando **cambios incompatibles**, pruebas A/B y deprecaciones ordenadas.

## 10) Documentar errores

Evita ambigüedad, acelera debugging y mejora el desarrollo. Todo endpoint debe especificar **códigos HTTP**, **cuerpo de error** y **posibles causas**.

---

## 2. Roles y permisos

* **Pasajero**

  * Registrar/gestionar perfil y métodos de pago
  * Solicitar/cancelar viaje, ver cotización y seguimiento
  * Calificar conductor y ver recibos
* **Conductor**

  * Gestionar disponibilidad y vehículo
  * Aceptar/iniciar/finalizar viaje
  * Recibir pagos (vía liquidaciones) y ver ratings
* **Administrador**

  * Moderar usuarios, ver métricas, gestionar reglas de precios, auditoría

---

## 3. Recursos principales

* `auth`, `users`, `drivers`, `vehicles`, `rides`, `payments`, `payment-methods`, `ratings`, `locations`, `admin`, `price-rules`, `health`.

---

## 4. Convenciones generales

* **Base URL:** `https://api.bucacar.com/v1`
* **Auth:** `Authorization: Bearer <JWT>` (scopes y roles).
* **Idempotencia:** En acciones críticas (`POST /rides`, `/payments/*`) usar `Idempotency-Key`.
* **Paginación:** `page`, `limit` (máx. 100), `sort`, `order`.
* **Filtrado por fecha:** `from`, `to` (ISO 8601).
* **Formato errores:**

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {"...": "..."},
    "correlationId": "uuid"
  }
}
```

---

## 5. Tabla de endpoints

| #  | Método | Ruta                          | Descripción                                    | Parámetros                                                             | Auth               |
| -- | ------ | ----------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------- | ------------------ |
| 1  | GET    | `/status`                     | Salud pública de la API                        | —                                                                      | Público            |
| 2  | POST   | `/auth/register`              | Registro de usuario (rol `passenger`/`driver`) | **Body:** `name`, `email`, `phone`, `password`, `role`                 | Público            |
| 3  | POST   | `/auth/login`                 | Login y entrega de JWT                         | **Body:** `email/phone`, `password`                                    | Público            |
| 4  | POST   | `/auth/refresh`               | Renovación de token                            | **Body:** `refreshToken`                                               | Público            |
| 5  | POST   | `/auth/logout`                | Revoca sesión                                  | **Headers:** `Authorization`                                           | Token              |
| 6  | GET    | `/users/me`                   | Perfil del usuario autenticado                 | —                                                                      | Token              |
| 7  | PATCH  | `/users/me`                   | Actualizar perfil                              | **Body:** `name`, `photo`, `preferred_payment_method`                  | Token              |
| 8  | GET    | `/users/{id}`                 | Ver otro usuario (restricciones)               | **Path:** `id`                                                         | Admin              |
| 9  | POST   | `/drivers/vehicle`            | Registrar/actualizar vehículo del conductor    | **Body:** `plate`, `brand`, `model`, `year`, `color`                   | Conductor          |
| 10 | PATCH  | `/drivers/availability`       | Cambiar estado (online/offline)                | **Body:** `available: boolean`                                         | Conductor          |
| 11 | GET    | `/drivers/nearby`             | Listar conductores cercanos                    | **Query:** `lat`, `lng`, `radius` (m)                                  | Token              |
| 12 | POST   | `/locations/driver-heartbeat` | Actualizar ubicación en tiempo real            | **Body:** `lat`, `lng`, `accuracy`, `heading`                          | Conductor          |
| 13 | GET    | `/locations/driver/{id}/last` | Última ubicación del conductor (minimizada)    | **Path:** `id`                                                         | Token (scoped)     |
| 14 | POST   | `/rides/quote`                | Cotiza tarifa estimada                         | **Body:** `origin`, `destination`, `service_type`, `promo_code?`       | Token              |
| 15 | POST   | `/rides`                      | Crear solicitud de viaje                       | **Body:** `origin`, `destination`, `payment_method_id`, `preferences?` | Token + Idemp.     |
| 16 | GET    | `/rides/{id}`                 | Obtener detalle del viaje                      | **Path:** `id`                                                         | Token (partes)     |
| 17 | POST   | `/rides/{id}/accept`          | Conductor acepta viaje                         | **Path:** `id`                                                         | Conductor          |
| 18 | POST   | `/rides/{id}/start`           | Inicia viaje                                   | **Path:** `id`                                                         | Conductor          |
| 19 | POST   | `/rides/{id}/complete`        | Completa viaje (dispara cobro)                 | **Path:** `id`                                                         | Conductor          |
| 20 | PATCH  | `/rides/{id}/cancel`          | Cancela viaje (reglas)                         | **Path:** `id`; **Body:** `reason_code`                                | Pasajero/Conductor |
| 21 | GET    | `/rides`                      | Listar viajes (historial/filtros)              | **Query:** `status`, `role`, `from`, `to`, `page`, `limit`             | Token              |
| 22 | GET    | `/rides/{id}/timeline`        | Eventos del viaje (audit log)                  | **Path:** `id`                                                         | Token              |
| 23 | POST   | `/payment-methods`            | Agregar método de pago                         | **Body:** `paymentToken` (PCI safe)                                    | Token              |
| 24 | GET    | `/payment-methods`            | Listar métodos de pago                         | —                                                                      | Token              |
| 25 | DELETE | `/payment-methods/{id}`       | Eliminar método de pago                        | **Path:** `id`                                                         | Token              |
| 26 | POST   | `/payments/authorize`         | Pre-autorización (previa a inicio)             | **Body:** `rideId`, `amount`                                           | Token              |
| 27 | POST   | `/payments/capture`           | Captura al completar viaje                     | **Body:** `rideId`, `amountFinal`                                      | Token              |
| 28 | POST   | `/payments/refund`            | Reembolso parcial/total                        | **Body:** `rideId`, `amount`, `reason`                                 | Admin              |
| 29 | GET    | `/payments/receipts/{rideId}` | Recibo/Factura del viaje                       | **Path:** `rideId`                                                     | Token              |
| 30 | POST   | `/ratings`                    | Calificar viaje                                | **Body:** `rideId`, `to` (driver/passenger), `score`, `comment?`       | Token              |
| 31 | GET    | `/ratings/driver/{id}`        | Ver rating agregado de conductor               | **Path:** `id`                                                         | Token              |
| 32 | GET    | `/admin/dashboard/stats`      | KPIs operativos                                | **Query:** `from`, `to`                                                | Admin              |
| 33 | PATCH  | `/admin/users/{id}/status`    | Suspender/banear usuario                       | **Path:** `id`; **Body:** `status`                                     | Admin              |
| 34 | GET    | `/admin/rides`                | Listado administrativo de viajes               | **Query:** `status`, `userId`, `driverId`                              | Admin              |
| 35 | GET    | `/admin/settlements/payouts`  | Liquidaciones a conductores                    | **Query:** `cycle`                                                     | Admin              |
| 36 | POST   | `/admin/price-rules`          | Crear/actualizar reglas de precio              | **Body:** `city`, `base`, `per_km`, `per_min`, `surge?`                | Admin              |


---

## 6. Flujos de uso

### Flujo A: Solicitud → Aceptación → Finalización

1. **Pasajero** cotiza: `POST /rides/quote` con origen/destino.
2. Solicita: `POST /rides` (con `Idempotency-Key`).
3. **Sistema** busca conductor cercano; **Conductor** acepta: `POST /rides/{id}/accept`.
4. Conductor inicia: `POST /rides/{id}/start` (se verifica `payments/authorize`).
5. Conductor completa: `POST /rides/{id}/complete` → `payments/capture` → `receipts/{rideId}`.
6. Ambas partes pueden **calificar**: `POST /ratings`.

### Flujo B: Cancelación y devolución parcial

1. Viaje en estado `accepted` → **Pasajero** cancela: `PATCH /rides/{id}/cancel` con `reason_code`.
2. Motor de tarifas aplica **penalidad** si corresponde.
3. **Admin/PSP** procesa **refund**: `POST /payments/refund` (monto = penalidad lógica).
4. Se registra en `timeline` y emite recibo actualizado.

### Flujo C: Gestión de ubicación y disponibilidad

1. **Conductor** pasa `online`: `PATCH /drivers/availability`.
2. Envía `driver-heartbeat` cada N segundos.
3. **Pasajero** visualiza ETA basado en `drivers/nearby` (precisión degradada).
4. Durante el viaje, solo **pasajero asignado** accede a ubicación en vivo del conductor.

---

## 7. Decisiones de diseño y justificación

* **Versionado `/v1`**: facilita cambios breaking en el futuro.
* **RBAC + Scopes**: rutas segregadas por rol (p. ej., `/admin/*`).
* **Idempotencia**: evita duplicados en solicitudes y cobros.
* **Privacidad de ubicación**: compartir solo lo necesario; retención corta y anonimización en histórico.
* **Pagos en dos fases**: `authorize` → `capture` reduce fraudes y discrepancias.
* **Timeline auditable**: cada transición de estado queda registrada.
* **Errores explícitos**: contrato de error consistente acelera soporte.
* **Observabilidad**: `correlationId` por petición.

---

## 8. Manejo de errores

**Códigos comunes:**

* `200 OK`, `201 Created`, `202 Accepted`
* `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`
* `429 Too Many Requests`, `500 Internal Server Error`, `503 Service Unavailable`

**Ejemplos (5+)**

1. `401 Unauthorized` – Token inválido/expirado al acceder `GET /users/me`.
2. `403 Forbidden` – Usuario pasajero intenta `POST /rides/{id}/accept`.
3. `404 Not Found` – `rideId` inexistente en `GET /rides/{id}`.
4. `409 Conflict` – No hay conductores disponibles al `POST /rides` (retornar mensaje y sugerencias).
5. `422 Unprocessable Entity` – Tarjeta rechazada en `POST /payments/authorize`.
6. `429 Too Many Requests` – Exceso de `driver-heartbeat`.
7. `503 Service Unavailable` – PSP caído durante `payments/capture` (reintentos con backoff).

---

## 9. Propuestas de mejora

1. **Notificaciones en tiempo real**: permitir que la app muestre en vivo dónde está el conductor y cuándo cambia el estado del viaje.
2. **Precios dinámicos**: ajustar el precio según la zona, hora y demanda usando cálculos automáticos.
3. **Seguridad y confianza**: verificar la identidad y antecedentes de conductores, y añadir un botón de emergencia con ubicación.
4. **Promociones y descuentos**: ofrecer cupones y ofertas especiales para ciertos usuarios o momentos.
5. **Documentación para desarrolladores**: tener una guía clara y ejemplos para que otros puedan conectar sus apps fácilmente a la API.

---
