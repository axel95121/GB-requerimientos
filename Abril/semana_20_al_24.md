# Requerimientos Backend — Semana del 20 al 24 de Abril

**Fecha:** 20 de abril de 2026  
**Solicitado por:** Frontend (App Delivery)  
**Dirigido a:** Backend  
**Sprint:** Semana del 20 al 24 de abril de 2026

**Enfoque principal:** Track de entregas por GPS

---

## Contexto y Problema Actual

Actualmente, el seguimiento de entregas se realiza mediante un **registro manual de estados** por parte del transportista. Cada vez que el transportista actualiza el estado de una orden (En tránsito, Intento de entrega, Cancelada, Completada), se capturan sus coordenadas GPS y se almacenan dentro del array `log` de la orden:

```json
{
  "status": 5,
  "fecha": "2026-04-20T14:30:00.000Z",
  "lugar": "Quito, Av. Amazonas, 170150",
  "lat": -0.1807,
  "lng": -78.4678,
  "comentario": "En camino al destino"
}
```

**Problema:** Este modelo solo registra la ubicación del transportista en los **momentos puntuales** en que cambia el estado de la entrega. El cliente que espera su paquete **no puede conocer la ubicación del transportista en tiempo real** entre un cambio de estado y otro, lo cual genera incertidumbre y una mala experiencia de usuario.

### Flujo de estados de entrega actual

| Valor de `status` | Significado              |
|--------------------|--------------------------|
| `4`                | Asignado a transportista |
| `5`                | En tránsito              |
| `6`                | Intento de entrega       |
| `7`                | Entrega cancelada        |
| `8`                | Entrega completada       |

---

## 1. Nuevo recurso: Tracking GPS (`/delivery-tracking`)

### Descripción

Se requiere crear un **nuevo recurso/colección** para almacenar los puntos de ubicación GPS que el transportista envía periódicamente mientras tiene una entrega activa (status `5` — En tránsito).

### Endpoint base sugerido

- **Base URL (producción):** `https://api-prod-general.gb97.ec`
- **Recurso:** `/delivery-tracking`

### Modelo de datos sugerido: `DeliveryTracking`

```json
{
  "_id": "uuid",
  "orderId": "uuid-de-la-orden",
  "deliveryId": "uuid-del-delivery",
  "carrierId": "uuid-del-transportista",
  "organizationId": "uuid-de-la-organizacion",
  "branchId": "uuid-de-la-sucursal",
  "status": "active",
  "startedAt": "2026-04-20T14:00:00.000Z",
  "endedAt": null,
  "lastLocation": {
    "lat": -0.1807,
    "lng": -78.4678,
    "timestamp": "2026-04-20T15:30:00.000Z",
    "accuracy": 10.5,
    "speed": 35.2,
    "heading": 180.0,
    "address": "Av. Amazonas y Naciones Unidas, Quito"
  },
  "trackingPoints": [
    {
      "lat": -0.1807,
      "lng": -78.4678,
      "timestamp": "2026-04-20T14:00:00.000Z",
      "accuracy": 10.5,
      "speed": 35.2,
      "heading": 180.0
    }
  ],
  "metadata": {
    "totalDistance": 12500,
    "estimatedArrival": "2026-04-20T16:00:00.000Z"
  },
  "createdAt": "2026-04-20T14:00:00.000Z",
  "updatedAt": "2026-04-20T15:30:00.000Z"
}
```

### Descripción de campos

| Campo              | Tipo          | Descripción |
|--------------------|---------------|-------------|
| `_id`              | `String (UUID)` | Identificador único del registro de tracking |
| `orderId`          | `String (UUID)` | ID de la orden asociada |
| `deliveryId`       | `String (UUID)` | ID del delivery asociado |
| `carrierId`        | `String (UUID)` | ID del transportista (usuario) |
| `organizationId`   | `String (UUID)` | ID de la organización |
| `branchId`         | `String (UUID)` | ID de la sucursal |
| `status`           | `String`        | Estado del tracking: `active`, `paused`, `completed`, `cancelled` |
| `startedAt`        | `DateTime`      | Fecha/hora en que se inició el tracking |
| `endedAt`          | `DateTime?`     | Fecha/hora en que finalizó el tracking (null si está activo) |
| `lastLocation`     | `Object`        | Última ubicación conocida del transportista (snapshot rápido) |
| `trackingPoints`   | `Array[Object]` | Historial completo de puntos GPS registrados |
| `metadata`         | `Object`        | Datos calculados (distancia, ETA, etc.) |
| `createdAt`        | `DateTime`      | Fecha de creación del registro |
| `updatedAt`        | `DateTime`      | Fecha de última actualización |

### Descripción de `trackingPoints[]`

| Campo       | Tipo      | Descripción |
|-------------|-----------|-------------|
| `lat`       | `Double`  | Latitud |
| `lng`       | `Double`  | Longitud |
| `timestamp` | `DateTime`| Fecha/hora del punto GPS |
| `accuracy`  | `Double?` | Precisión en metros del GPS |
| `speed`     | `Double?` | Velocidad en km/h (si disponible) |
| `heading`   | `Double?` | Dirección en grados (0-360, si disponible) |

### Estados del tracking

| Valor        | Significado |
|--------------|-------------|
| `active`     | El transportista está en ruta, enviando ubicaciones activamente |
| `paused`     | El tracking está pausado temporalmente (ej. parada técnica) |
| `completed`  | La entrega fue completada exitosamente |
| `cancelled`  | La entrega fue cancelada |

---

## 2. Endpoints requeridos

### 2.1 Iniciar tracking — `POST /delivery-tracking`

Se invoca cuando el transportista cambia el estado de la orden a **En tránsito (status 5)**. Crea un nuevo registro de tracking.

**Body:**

```json
{
  "orderId": "uuid-de-la-orden",
  "deliveryId": "uuid-del-delivery",
  "carrierId": "uuid-del-transportista",
  "organizationId": "uuid-de-la-organizacion",
  "branchId": "uuid-de-la-sucursal",
  "lastLocation": {
    "lat": -0.1807,
    "lng": -78.4678,
    "timestamp": "2026-04-20T14:00:00.000Z",
    "accuracy": 10.5
  }
}
```

**Respuesta esperada:**

```json
{
  "response": true,
  "data": {
    "_id": "uuid-generado",
    "status": "active",
    "startedAt": "2026-04-20T14:00:00.000Z"
  }
}
```

**Validaciones:**
- El `carrierId` debe corresponder al token de autenticación.
- La orden debe estar en status `5` (En tránsito).
- No debe existir otro tracking `active` para la misma orden.

---

### 2.2 Actualizar ubicación (batch) — `PUT /delivery-tracking/:id/locations`

El transportista envía **un lote de puntos GPS** acumulados en un intervalo (ej. cada 15-30 segundos el frontend acumula, y cada 1-2 minutos envía el batch). Esto optimiza el consumo de red y batería.

**Body:**

```json
{
  "locations": [
    {
      "lat": -0.1810,
      "lng": -78.4680,
      "timestamp": "2026-04-20T14:01:00.000Z",
      "accuracy": 8.2,
      "speed": 40.5,
      "heading": 175.0
    },
    {
      "lat": -0.1825,
      "lng": -78.4695,
      "timestamp": "2026-04-20T14:01:30.000Z",
      "accuracy": 7.8,
      "speed": 38.0,
      "heading": 180.0
    }
  ]
}
```

**Respuesta esperada:**

```json
{
  "response": true,
  "data": {
    "pointsAdded": 2,
    "lastLocation": {
      "lat": -0.1825,
      "lng": -78.4695,
      "timestamp": "2026-04-20T14:01:30.000Z"
    }
  }
}
```

**Comportamiento esperado:**
1. Agregar los puntos al array `trackingPoints`.
2. Actualizar el campo `lastLocation` con el punto más reciente (mayor `timestamp`).
3. Actualizar el campo `updatedAt`.

**Validaciones:**
- El tracking debe estar en estado `active`.
- El `carrierId` del tracking debe corresponder al usuario autenticado.
- Los timestamps de las ubicaciones deben ser válidos (no futuros por más de 1 minuto como tolerancia).
- Máximo **100 puntos por petición** para evitar abuso.

### 2.3 Finalizar tracking — `PUT /delivery-tracking`

Se invoca cuando el transportista cambia el estado de la orden a **Completada (8)** o **Cancelada (7)**.

**Body:**

```json
{
  "status": "completed",
  "lastLocation": {
    "lat": -0.2010,
    "lng": -78.5020,
    "timestamp": "2026-04-20T16:00:00.000Z"
  }
}
```

**Comportamiento esperado:**
1. Cambiar `status` a `completed` o `cancelled`.
2. Registrar `endedAt` con la fecha actual.
3. Agregar la última ubicación a `trackingPoints`.
4. Actualizar `lastLocation`.
---

## 3. Seguridad y privacidad de datos GPS

### 4.1 Control de acceso

- **Transportista:** Solo puede escribir/actualizar tracking de entregas asignadas a él.
- **Admin / Operador de la organización:** Puede consultar cualquier tracking de su organización.
- **Cliente destinatario:** Solo puede consultar (lectura) el tracking de sus propias órdenes al tener disponible el id del tracking (por el momento LIBRE para acceso del perfil de INVITADO, luego se debe privatizar el uso cuando se implemente el Marketplace).

### 4.2 Almacenamiento seguro

- Los datos GPS son **datos sensibles de ubicación personal**. Se recomienda:
  - Almacenar con **encriptación en reposo** (encryption at rest) en la base de datos.
  - Transmisión siempre sobre **HTTPS/TLS**.
  - No exponer coordenadas exactas del transportista en endpoints públicos.

### 4.3 Política de retención de datos

- Los `trackingPoints` detallados (historial granular) deben tener una **política de retención**:
  - **Trackings activos:** conservar todos los puntos.
  - **Trackings completados/cancelados:** conservar el historial detallado por **30 días** después de finalizado, luego se puede reducir a un resumen (punto inicio, puntos intermedios clave, punto final).
  - **Después de 90 días:** eliminar los `trackingPoints` granulares, conservar únicamente `lastLocation`, `startedAt`, `endedAt` y `metadata` como registro histórico.

### 4.4 Límites y rate limiting

- **Máximo de escritura:** 100 puntos por minuto por transportista (para evitar abuso/ataques).
- **Máximo de lectura:** 60 peticiones por minuto por usuario en el endpoint de consulta (polling).

---

## 4. Integración con el flujo actual de órdenes

### Relación con el campo `log` existente

El sistema de tracking GPS **no reemplaza** el sistema actual de log por estados. Ambos coexisten:

| Sistema          | Propósito | Frecuencia | Datos |
|------------------|-----------|------------|-------|
| `log` (existente) | Registro de cambios de estado del pedido | Puntual (al cambiar estado) | status, fecha, lugar, lat, lng, comentario |
| `delivery-tracking` (nuevo) | Seguimiento GPS continuo del transportista | Periódico (cada 15-30 seg) | lat, lng, timestamp, accuracy, speed, heading |

### Vinculación automática

Cuando el frontend invoque `PUT /ordenes/update/:id` para cambiar el status a `5` (En tránsito), **inmediatamente después** creará el registro de tracking vía `POST /delivery-tracking`. De la misma forma, al cambiar a status `7` u `8`, finalizará el tracking vía `PUT /delivery-tracking/:id/finish`.

---

## 5. Campos adicionales sugeridos en la orden

Para vincular fácilmente la orden con su tracking activo, se sugiere agregar un campo opcional al modelo de órdenes:

```json
{
  "trackingId": "uuid-del-delivery-tracking | null"
}
```

- Se setea cuando se crea el tracking (`POST /delivery-tracking`).
- Se consulta para acceso rápido sin necesidad de buscar por `orderId`.

### Acción requerida

Agregar el campo `trackingId` (tipo `String`, nullable, por defecto `null`) al modelo de órdenes, aceptarlo en los endpoints de actualización (`PUT`/`PATCH`) y devolverlo en las respuestas.

---

## 6. Otras tareas

### Integración con Payphone

Se esperaría disponer de el o los endpoints para su integral implementación.

### App Textil

Hay un error que está ocurriendo al descargar el documento de las órdenes de un cliente, es una actividad de prioridad baja, por favor revisar cuando se podría verificar un momento. 
