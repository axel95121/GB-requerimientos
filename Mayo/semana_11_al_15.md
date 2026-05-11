# Requerimientos Backend 

> Fecha: 2026-05-11  
> Solicitado por: Frontend (App Delivery)  

---

## Índice

1. [Módulo de Códigos y Referidos](#1-módulo-de-códigos-y-referidos)
   - [Tipos de código](#11-tipos-de-código)
   - [Reglas de negocio](#12-reglas-de-negocio)
   - [Endpoints de referidos](#13-endpoints-de-referidos)
2. [Módulo de Planes y Suscripciones](#2-módulo-de-planes-y-suscripciones)
   - [Planes definidos](#21-planes-definidos)
   - [Límites por plan](#22-límites-por-plan)
   - [Endpoints de suscripciones](#23-endpoints-de-suscripciones)
   - [Pendientes críticos](#24-pendientes-críticos)

---

---

# 1. Módulo de Códigos y Referidos

## 1.1 Tipos de código

| Tipo | Emisor | Usos | Generación |
|------|--------|------|------------|
| `referral` | Cada usuario registrado | Ilimitado | Automática al registrarse |
| `activation` | Distribuidores / vendedores autorizados | Un solo uso por código | Mediante endpoint exclusivo (ver §1.3.6) |

## 1.2 Reglas de negocio

- El código `referral` de cada usuario **se genera automáticamente al registrarse** en la app. No es personalizable.
- **Un usuario no puede canjear su propio código** bajo ningún concepto. El backend valida comparando el `userId` del JWT con el `ownerId` del código.
- Los beneficios consisten en **días de uso adicionales de un plan de suscripción**. Si el usuario ya tiene suscripción activa, los días se suman a `fecha_expiracion`; no se reemplaza el plan.
- Los códigos `activation` son de **un solo uso** y representan una licencia individual. Al canjearse deben marcarse como usados.
- Los códigos `referral` son de **uso ilimitado** (cualquier número de usuarios puede canjearlos).

---

## 1.3 Endpoints de referidos

### 1.3.1 Obtener código de referido del usuario autenticado

```
GET /referrals/my-code
Authorization: <token>
```

**Respuesta esperada:**
```json
{
  "code": "GB-X7K2M9",
  "stats": {
    "referidos": 3,
    "canjeados": 1,
    "libres": 5
  }
}
```

> ℹ️ Las métricas `referidos`, `canjeados` y `libres` son estadísticas derivadas de la implementación del backend y sujetas a cambios según su criterio.

---

### 1.3.2 Obtener beneficios del código del usuario autenticado

```
GET /referrals/benefits
Authorization: <token>
```

**Respuesta esperada:**
```json
{
  "benefits_for_referred": [
    { "id": "1", "description": "30 días gratis en Plan Emprendedor" },
    { "id": "2", "description": "5% de descuento en tu próxima renovación" }
  ],
  "benefits_for_referrer": [
    { "id": "3", "description": "30 días adicionales por cada referido activo" }
  ]
}
```

> ℹ️ Los beneficios son globales (no dependen del plan actual del usuario). Se aplican como días adicionales sobre la suscripción activa.

---

### 1.3.3 Obtener lista de referidos recientes

```
GET /referrals/recent?page=1&limit=20
Authorization: <token>
```

**Respuesta esperada:**
```json
{
  "referrals": [
    {
      "email": "carlos@empresa.com",
      "status": "active",
      "created_at": "2026-04-01T00:00:00Z"
    },
    {
      "email": "melisa@empresa.com",
      "status": "pending",
      "created_at": "2026-04-15T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 3,
    "totalPages": 1
  }
}
```

> ℹ️ `status: "active"` = código canjeado y suscripción activa. `status: "pending"` = código canjeado pero sin activar aún.  
> ℹ️ Parámetros: `page` (default `1`), `limit` (default `20`).

---

### 1.3.4 Verificar código (antes de canjear)

```
POST /referrals/verify
Authorization: <token>
Body: { "code": "GB-X7K2M9" }
```

**Validaciones requeridas:**
- El código debe existir en la base de datos.
- Si `type = activation`: debe estar disponible (`used: false`).
- Si `type = referral`: el `userId` del JWT **no puede coincidir** con el `ownerId` del código → error `SELF_REDEEM_NOT_ALLOWED`.

**Respuesta — código válido:**
```json
{
  "valid": true,
  "code": "GB-X7K2M9",
  "type": "referral",
  "benefits": [
    { "id": "1", "description": "30 días gratis en Plan Emprendedor" }
  ]
}
```

**Respuesta — código inválido:**
```json
{
  "valid": false,
  "error": "INVALID_CODE"
}
```

**Códigos de error:**

| Código | Descripción |
|--------|-------------|
| `INVALID_CODE` | El código no existe |
| `ALREADY_USED` | Código de activación ya fue utilizado |
| `EXPIRED` | El código ha expirado |
| `SELF_REDEEM_NOT_ALLOWED` | El usuario intenta canjear su propio código |

---

### 1.3.5 Confirmar canje de código

```
POST /referrals/redeem
Authorization: <token>
Body: { "code": "GB-X7K2M9" }
```

**Comportamiento:**
- Aplica las mismas validaciones que `/verify`.
- Si el usuario tiene suscripción activa: **sumar** los días del beneficio a `fecha_expiracion`.
- Si no tiene suscripción activa: activar el plan del beneficio con la duración correspondiente.
- Si `type = activation`: marcar el código como usado (`used: true`, `used_by: userId`, `used_at: timestamp`).

**Respuesta esperada:**
```json
{
  "success": true,
  "message": "Código canjeado exitosamente",
  "applied_benefits": [
    { "id": "1", "description": "30 días gratis en Plan Emprendedor" }
  ],
  "new_expiration_date": "2026-06-11T00:00:00Z"
}
```

---

### 1.3.6 Generar códigos de activación (distribuidores / vendedores)

> Endpoint exclusivo para distribuidores y vendedores autorizados. Requiere rol con permisos especiales (`distributor` | `vendor`).

```
POST /referrals/activation-codes/generate
Authorization: <token>
```

**Body:**
```json
{
  "quantity": 10,
  "plan_id": "medium_monthly",
  "days": 30,
  "notes": "Licencias campaña mayo 2026"
}
```

**Comportamiento:**
- Genera `quantity` códigos únicos de tipo `activation`, cada uno de un solo uso.
- Devuelve la lista de códigos generados para que el distribuidor los distribuya.

**Respuesta esperada:**
```json
{
  "success": true,
  "codes": [
    { "code": "ACT-A1B2C3", "used": false },
    { "code": "ACT-D4E5F6", "used": false }
  ],
  "generated_at": "2026-05-11T00:00:00Z"
}
```

---

---

# 2. Módulo de Planes y Suscripciones

## 2.1 Planes definidos

| Plan ID (frontend) | Product IDs (IAP) | Nombre | Precio mensual | Precio anual |
|--------------------|-------------------|--------|---------------|-------------|
| `free` | — | GRATUIDAD | $0 | Sin costo |
| `entrepreneur` | `medium_monthly` / `medium_annual` | PLAN EMPRENDEDOR | $15/mes | $150/año (≡ $12/mes) |
| `premium` | `premium_monthly` / `premium_annual` | PLAN PREMIUM | $29/mes | $290/año (≡ $23/mes) |
| `pro_marketplace` | `pro_marketplace_monthly` / `pro_marketplace_annual` | PLAN PRO + MARKETPLACE | $53/mes + 2% plataforma | $530/año (≡ $42/mes) |

> ℹ️ Los planes anuales incluyen 2 meses gratis respecto al precio mensual × 12.

---

## 2.2 Límites por plan

### Tabla de límites

| Límite (`tipo` en `control`) | GRATUIDAD | EMPRENDEDOR | PREMIUM | PRO + MARKETPLACE |
|------------------------------|-----------|-------------|---------|-------------------|
| `Ordenes` | 30 | 80 | 400 | Ilimitado |
| `Factura` | 24 anuales | 50 anuales | 300 anuales | Ilimitado |
| `Sucursal` | 1 | 1 | 1 | Ilimitado |
| `Proformas` | 50 | 150 | Ilimitado | Ilimitado |
| `ClienteDocs` (archivos por cliente) | — | — | 3 | Ilimitado |
| `ItemFeatures` | — | — | — | Ilimitado |
| `Catalogo` | 1 | 3 | 10 | Ilimitado |
| Usuarios | 1 admin | 2 (admin + vendedor) | 5 (1 admin, 2 vendedores, 2 transportistas) | Ilimitado |
| Registro de entregas | 3 | Básico (foto + firma) | Completo | Completo |
| Seguimiento de mercadería | ❌ | ✅ | ✅ | ✅ |
| Pasarelas de pago | ❌ | DEUNA | DEUNA + PayPhone | DEUNA + PayPhone |
| Ruta vendedor | ❌ | ❌ | Por ubicación | Por recorrido |
| Marketplace | ❌ | ❌ | ❌ | ✅ (+ 2% comisión por venta) |

> ℹ️ `Ilimitado` debe representarse como `"999999"` en el campo `cantidad` del array `control`.

### Estructura del array `control` en MongoDB

El frontend consume el campo `control` del objeto suscripción. Se requiere que el backend lo devuelva **como array con objetos `{ tipo, cantidad }`**, usando los nombres de `tipo` exactamente como se listan abajo:

```json
"control": [
  { "tipo": "Ordenes",      "cantidad": "400"    },
  { "tipo": "Factura",      "cantidad": "300"    },
  { "tipo": "Sucursal",     "cantidad": "1"      },
  { "tipo": "Proformas",    "cantidad": "999999" },
  { "tipo": "ClienteDocs",  "cantidad": "10"     },
  { "tipo": "ItemFeatures", "cantidad": "1"      },
  { "tipo": "Catalogo",     "cantidad": "10"     }
]
```

> ⚠️ **Importante:** el frontend actualmente lee el array por **posición fija** (índice 0 al 6). Si el orden cambia, la app lee límites incorrectos. Se recomienda que el backend siempre devuelva el array en el orden indicado, o bien que el frontend se migre a lectura por `tipo` (tarea pendiente del frontend).

---

## 2.3 Endpoints de suscripciones

### 2.3.1 Activar / actualizar suscripción de la organización

Ya implementado. Se llama tras una compra IAP exitosa.

```
PUT /organizations/:organizationId
Authorization: <token>
Body: { "subscription": "<mongoId_del_plan>" }
```

**IDs de MongoDB por plan:**

| Plan | Product ID (IAP) | MongoDB ID |
|------|-----------------|------------|
| EMPRENDEDOR | `medium_monthly` / `medium_annual` | `6818c9efa558c757e143514f` |
| PREMIUM | `premium_monthly` / `premium_annual` | `6818cbada558c757e1435150` |
| PRO + MARKETPLACE | `pro_marketplace_monthly` / `pro_marketplace_annual` | ⚠️ **Pendiente** — actualmente usa el ID de PREMIUM por error |

> 🔴 **Acción requerida:** crear el documento de plan Pro+Marketplace en MongoDB y comunicar el ID al equipo frontend para actualizar [api_service.dart](../lib/src/api/api_service.dart).

---

### 2.3.2 Verificar compra IAP con Apple / Google *(pendiente implementación)*

> 🔴 **Crítico:** actualmente la verificación de compras se hace **localmente en el cliente**, lo que permite activar planes sin pagar. Se requiere implementación real en el servidor.

```
POST /subscriptions/verify-purchase
Authorization: <token>
```

**Body:**
```json
{
  "store": "google_play",
  "productId": "premium_monthly",
  "purchaseId": "GPA.xxxx-xxxx-xxxx",
  "transactionDate": "1746800000000",
  "verificationData": {
    "localVerificationData": "...",
    "serverVerificationData": "...",
    "source": "google_play"
  }
}
```

**Comportamiento esperado:**
- Validar el receipt contra la API de Apple App Store o Google Play.
- Si es válido: activar el plan en la organización y devolver el objeto de suscripción actualizado.
- Si es inválido: devolver `{ "valid": false, "error": "INVALID_RECEIPT" }`.

**Respuesta esperada (válido):**
```json
{
  "valid": true,
  "subscription": {
    "_id": "...",
    "name": "Plan Premium",
    "montly_price": 29.0,
    "yearly_price": 290.0,
    "description": "...",
    "benefits": [],
    "control": [ /* array completo con los 7 tipos */ ],
    "fecha_contrato": "2026-05-11T00:00:00Z",
    "fecha_expiracion": "2026-06-11T00:00:00Z"
  }
}
```

---

### 2.3.3 Perfil de usuario con suscripción actualizada

Tras una compra, el frontend consulta el perfil del usuario para refrescar los límites del plan. Se requiere que la respuesta incluya el objeto `organization.subscription` completo.

```
GET /users/profile   (o el endpoint equivalente de perfil)
Authorization: <token>
```

**Estructura mínima requerida en la respuesta:**
```json
{
  "response": true,
  "data": {
    "organization": {
      "_id": "...",
      "subscription": {
        "_id": "...",
        "name": "Plan Premium",
        "logo": "https://...",
        "montly_price": 29.0,
        "yearly_price": 290.0,
        "description": "Plan operativo completo con transporte.",
        "benefits": ["Beneficio 1", "Beneficio 2"],
        "control": [
          { "tipo": "Ordenes",      "cantidad": "400"    },
          { "tipo": "Factura",      "cantidad": "300"    },
          { "tipo": "Sucursal",     "cantidad": "1"      },
          { "tipo": "Proformas",    "cantidad": "999999" },
          { "tipo": "ClienteDocs",  "cantidad": "10"     },
          { "tipo": "ItemFeatures", "cantidad": "1"      },
          { "tipo": "Catalogo",     "cantidad": "10"     }
        ],
        "fecha_contrato":   "2026-05-11T00:00:00Z",
        "fecha_expiracion": "2026-06-11T00:00:00Z"
      }
    }
  }
}
```

---

### 2.3.4 Validación de límites del plan (server-side)

Cuando un usuario intenta crear un recurso que supera el límite de su plan (órdenes, facturas, proformas, catálogos, etc.), el backend debe responder con:

```
HTTP 403 Forbidden
```

```json
{
  "response": false,
  "message": "Has alcanzado el límite de órdenes de tu plan actual.",
  "error_code": "PLAN_LIMIT_EXCEEDED",
  "limit_type": "Ordenes",
  "current_limit": 30
}
```

> ℹ️ La app detecta este error y muestra el banner de "Mejora tu plan" al usuario. El campo `limit_type` permite que el frontend muestre el mensaje correcto.

---

## 2.4 Pendientes críticos

| # | Tarea | Prioridad |
|---|-------|-----------|
| 1 | Implementar verificación real de receipt IAP con Apple/Google (`POST /subscriptions/verify-purchase`) | 🔴 Alta |
| 2 | Crear documento MongoDB para Plan Pro+Marketplace y comunicar el ID al frontend | 🔴 Alta |
| 3 | Corregir duración de planes en producción: mensual = 30 días, anual = 365 días (actualmente staging usa 1 h / 24 h) | 🔴 Alta |
| 4 | Validar límites del plan en el servidor al crear recursos y devolver `403` con `PLAN_LIMIT_EXCEEDED` | 🟡 Media |
| 5 | Garantizar que el array `control` siempre se devuelva con los 7 tipos en el orden especificado | 🟡 Media |

---

---

## Resumen de decisiones — Referidos

| # | Pregunta | Respuesta |
|---|----------|-----------|
| 1 | ¿Qué significa `libres`? | Estadística derivada, definida y controlada por el backend |
| 2 | ¿Los beneficios dependen del plan? | No, son globales. Se aplican como días adicionales sobre la suscripción activa |
| 3 | ¿Un usuario puede canjear su propio código? | **No** → error `SELF_REDEEM_NOT_ALLOWED` |
| 4 | ¿El código de activación sigue el mismo flujo? | Sí, mismo flujo verify + redeem. Se genera en sección separada |
| 5 | ¿El código de referido se genera automáticamente? | **Sí**, el backend lo genera al momento del registro |
| 6 | ¿Hay límite de usos? | `referral`: ilimitado. `activation`: un solo uso por código |
| 7 | ¿Paginación en `/referrals/recent`? | **Sí**, con `page` y `limit` |
