# Requerimientos Backend — Semana 27 de abril de 2026

**Fecha:** 27 de abril de 2026  
**Solicitado por:** Frontend (App Delivery)  
**Dirigido a:** Equipo Backend  
**Sprint:** Semana del 27 de abril al 3 de mayo de 2026

---

## Índice

1. [Endpoint de tracking points + visualización de ruta](#1-endpoint-de-tracking-points--visualización-de-ruta)
2. [Campo de dirección de entrega en el modelo de delivery](#2-campo-de-dirección-de-entrega-en-el-modelo-de-delivery)
3. [Integración Payphone (nuevo método de pago)](#3-integración-payphone-nuevo-método-de-pago)

---

## 1. Endpoint de tracking points + visualización de ruta

### Contexto

Esta semana se prioriza poner en producción el **endpoint exclusivo para subir puntos de tracking** (`PUT /delivery-tracking/:id/locations`) — el cual fue diseñado como endpoint de batch — y, de manera complementaria, asegurar que la API soporte la **reconstrucción completa de la ruta trazada** para su visualización en mapa.

### 1.1 Endpoint prioritario: subida de tracking points (batch)

> Este endpoint ya fue especificado en el documento previo. Se re-incluye aquí como tarea prioritaria para confirmar su implementación esta semana.

**Método:** `POST`  
**Ruta:** `/delivery-tracking/locations`  
**Autenticación:** Requerida (token del transportista)

**Body:**

```json
{
    "_id" : "delivery-tracking-id",
  "locations": [
    {
      "lat": -0.1810,
      "lng": -78.4680,
      "timestamp": "2026-04-27T09:01:00.000Z",
      "accuracy": 8.2,
      "speed": 40.5,
      "heading": 175.0
    },
    {
      "lat": -0.1825,
      "lng": -78.4695,
      "timestamp": "2026-04-27T09:01:30.000Z",
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
    "_id" : "delivery-tracking-id",
    "pointsAdded": 2,
    "lastLocation": {
      "lat": -0.1825,
      "lng": -78.4695,
      "timestamp": "2026-04-27T09:01:30.000Z"
    }
  }
}
```

**Comportamiento:**
1. Agregar los puntos al array `trackingPoints` del documento de tracking.
2. Actualizar `lastLocation` con el punto de mayor `timestamp`.
3. Actualizar `updatedAt`.

**Validaciones:**
- El tracking debe estar en estado `active`.
- El `carrierId` del tracking debe corresponder al usuario autenticado (extraído del JWT - opcional pero fuerte).
- Los timestamps no deben ser futuros por más de 1 minuto (tolerancia de sincronización).
- Máximo **100 puntos por petición** para evitar abuso.

---

### 1.2 Endpoint para obtener la ruta completa trazada

Para que la app pueda dibujar la ruta recorrida por el transportista en el mapa (polyline), se necesita recuperar el historial completo de `trackingPoints`.

**Método:** `GET`  
**Ruta:** `/delivery-tracking/?includeHistory=true&historyLimit=500`  
**Autenticación:** Requerida

**Respuesta esperada (con historial completo):**

```json
{
  "response": true,
  "data": {
    "_id": "uuid-del-tracking",
    "orderId": "uuid-de-la-orden",
    "carrierId": "uuid-del-transportista",
    "status": "completed",
    "startedAt": "2026-04-27T09:00:00.000Z",
    "endedAt": "2026-04-27T10:30:00.000Z",
    "lastLocation": {
      "lat": -0.2010,
      "lng": -78.5020,
      "timestamp": "2026-04-27T10:30:00.000Z",
      "accuracy": 6.0,
      "speed": 0.0,
      "heading": 90.0,
      "address": "Av. República y Diego de Almagro, Quito"
    },
    "trackingPoints": [
      {
        "lat": -0.1807,
        "lng": -78.4678,
        "timestamp": "2026-04-27T09:00:00.000Z",
        "accuracy": 10.5,
        "speed": 35.2,
        "heading": 180.0
      }
    ],
    "routeSummary": {
      "originLat": -0.1807,
      "originLng": -78.4678,
      "destinationLat": -0.2010,
      "destinationLng": -78.5020,
      "totalPoints": 187,
      "totalDistanceMeters": 14320
    },
    "metadata": {
      "totalDistance": 14320,
      "estimatedArrival": null
    }
  }
}
```

> `routeSummary` puede calcularse en el momento de la consulta (no necesariamente almacenarse) y sirve para que el front pueda encuadrar el mapa automáticamente sin recorrer todos los puntos del array.

**Nota sobre el parámetro `historyLimit`:**
- El límite por defecto puede mantenerse en `200` para consultas generales.
- Para la visualización de ruta completa, el frontend enviará `historyLimit=500` (o el máximo que el backend soporte sin degradación de rendimiento).
- Si el historial supera el límite, el backend puede devolver una muestra uniforme (distribución homogénea a lo largo del tiempo) en lugar de truncar desde el final.

---

### 1.3 Relación entre el punto de origen y el punto de destino

Con la implementación del campo `deliveryAddress` (ver sección 2), se puede establecer claramente:

| Punto       | Fuente                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------ |
| **Origen**  | Primer `trackingPoint` (coordenada GPS del transportista al iniciar la ruta)                     |
| **Destino** | `deliveryAddress` del documento de delivery (dirección del cliente, geocodificada si es posible) |

Esto permite al frontend mostrar una ruta desde el origen hasta el destino final sobre el mapa.

---

## 2. Campo de dirección de entrega en el modelo de delivery

### Contexto

Actualmente, cuando se registra una entrega en la app, existen tres tipos:

| Índice | Tipo                         | Descripción                                               |
| ------ | ---------------------------- | --------------------------------------------------------- |
| `0`    | Delivery (servicio propio)   | El transportista lleva el paquete a domicilio             |
| `1`    | Entrega personal (en tienda) | El cliente recoge en sucursal — **no requiere dirección** |
| `2`    | Transporte externo           | Courrier / mensajería externa                             |

Para los tipos `0` (delivery propio) y `2` (transporte externo), el paquete se traslada a una **dirección del cliente**. Esta dirección actualmente **no se registra** en el documento de delivery, lo que impide:
- Conocer el punto de destino para trazar la ruta en el mapa.
- Tener un registro de adónde fue entregado el paquete.

### 2.1 Requerimiento

Agregar el campo `deliveryAddress` al modelo de documentos de delivery para registrar la dirección completa de entrega cuando el tipo **no** es entrega personal (tipo `1`).

### 2.2 Cambios en el modelo de Delivery

**Campo a agregar:**

```json
{
  "deliveryAddress": {
    "fullAddress": "Av. República y Diego de Almagro N35-17, Quito",
    "reference": "Edificio azul, piso 3, oficina 302",
    "city": "Quito",
    "province": "Pichincha",
    "lat": -0.2010,
    "lng": -78.5020
  }
}
```

| Campo                         | Tipo             | Requerido                          | Descripción                                                   |
| ----------------------------- | ---------------- | ---------------------------------- | ------------------------------------------------------------- |
| `deliveryAddress`             | `Object \| null` | No                                 | Nulo para tipo `1` (personal). Requerido para tipos `0` y `2` |
| `deliveryAddress.fullAddress` | `String`         | Sí (si `deliveryAddress` presente) | Dirección completa en texto                                   |
| `deliveryAddress.reference`   | `String?`        | No                                 | Referencias adicionales (piso, apto, punto de referencia)     |
| `deliveryAddress.city`        | `String?`        | No                                 | Ciudad de entrega                                             |
| `deliveryAddress.province`    | `String?`        | No                                 | Provincia de entrega                                          |
| `deliveryAddress.lat`         | `Double?`        | No                                 | Latitud geocodificada (puede llenarse desde el frontend)      |
| `deliveryAddress.lng`         | `Double?`        | No                                 | Longitud geocodificada (puede llenarse desde el frontend)     |

### 2.3 Acciones requeridas en el backend

1. **Agregar el campo `deliveryAddress`** (tipo `Object`, nullable) al esquema/modelo de deliveries.
2. **Aceptarlo en los endpoints de creación** (`POST`) y actualización (`PUT`/`PATCH`) de deliveries.
3. **Devolverlo en las respuestas de consulta** (GET individual, GET by order, filter).
4. No se requiere validación de formato de dirección por ahora (el frontend enviará texto libre).

### 2.4 Relación con el tracking GPS

Cuando `deliveryAddress.lat` y `deliveryAddress.lng` están presentes, el backend puede usarlos como referencia del punto final en `routeSummary` (ver sección 1.2). Si no están presentes, el punto final se infiere del último `trackingPoint`.

---

## 3. Integración Payphone (nuevo método de pago)

### Contexto

1. El usuario selecciona "Payphone" como método de pago al crear una orden.
2. El frontend invoca el endpoint del backend para iniciar el pago.
3. El backend crea la transacción en Payphone y devuelve un **token o URL de pago**.
4. El frontend abre la interfaz de Payphone (webview o deeplink) para que el usuario complete el pago con su tarjeta.
5. Payphone notifica al backend sobre el resultado (webhook) y/o el frontend consulta el estado final.
6. El backend actualiza el estado del pago en la orden y notifica al frontend.

> La integración en backend es análoga a la ya existente con **DeUna** (`POST /deuna/request-payment`). Se sugiere seguir el mismo patrón arquitectónico.

