# Requerimientos Backend — Semana 1 de junio 2026
**Origen:** Bitácora de estabilización pre-producción (C1–C12, M1–M25, L1–L11)  
**Prioridad:** De mayor a menor impacto en negocio, seguridad e integridad de datos  
**Formato:** Cada ítem indica la acción concreta que debe tomar el backend, el impacto si no se resuelve, y el contrato esperado con el frontend.

---

## 🔴 CRÍTICOS — Resolver antes de cualquier despliegue

### BE-00 — Suscripción expira en 5–30 minutos por duración hardcodeada en el cliente
**Origen:** `subscriptions_screen.dart` + `my_profile_screen.dart`  
**Impacto si no se resuelve:** Todos los usuarios que compran un plan son degradados automáticamente al plan Freemium sin un control real del tiempo de suscripción.

**Problema actual:**  
El frontend calcula la fecha de expiración localmente de manera errónea:

**Acción requerida (BE):**  
Al procesar `updateSubscription` (`PUT /organizaciones/:id` con `{ "subscription": subscriptionId }`), el backend debe calcular y almacenar la fecha de expiración real según el tipo de plan, y devolverla en la respuesta de `GET /usuarios/perfil` bajo el campo `subscriptionEnd`:

```json
// Respuesta de GET /usuarios/perfil  →  data.organization.subscription
{
  "subscription": {
    "name": "Plan Premium",
    "planCode": "premium_monthly",
    "subscriptionEnd": "2026-07-01T00:00:00.000Z",   // ← campo requerido
    "subscriptionStart": "2026-06-01T00:00:00.000Z"
  }
}
```

---

### BE-01 — Endpoint de registro unificado y atómico
**Origen:** C1  
**Impacto si no se resuelve:** Los usuarios quedan permanentemente bloqueados si cualquiera de los 5 pasos del registro actual falla a mitad de camino (organización creada sin usuario, RUC duplicado que impide reintento).

**Problema actual:**  
El frontend realiza 5 llamadas independientes en secuencia:
1. `POST /organizaciones` → crea la org
2. `POST /sucursales` → crea la sucursal
3. `GET /personas?filter=...` → busca persona
4. `POST /personas` o `PUT /personas/:id` → crea/actualiza persona
5. `POST /usuarios` → crea el usuario

Si el paso 2, 3, 4 o 5 falla, los recursos del paso anterior quedan como huérfanos en BD. En el reintento, el paso 1 falla por RUC duplicado y el usuario queda bloqueado.

**Acción requerida:**  
Crear un único endpoint de registro atómico:

```
POST /usuarios/registro-completo
Authorization: (ninguna — endpoint público)
Content-Type: application/json

Body:
{
  "organizacion": {
    "nombre": "string",
    "ruc": "string",
    "tipo": "string",
    ...
  },
  "sucursal": {
    "nombre": "string",
    "direccion": "string",
    "telefono": "string",
    ...
  },
  "persona": {
    "nombre": "string",
    "apellido": "string",
    "dni": "string",
    ...
  },
  "usuario": {
    "userName": "string",
    "password": "string",
    "email": "string",
    ...
  }
}

Response exitoso (HTTP 201):
{
  "response": true,
  "data": {
    "userId": "string",
    "token": "string",
    "organizationId": "string",
    "branchId": "string",
    "personId": "string"
  }
}

Response error (HTTP 200 o 4xx):
{
  "response": false,
  "message": "El RUC ya está registrado."
}
```

El backend garantiza atomicidad (rollback completo si cualquier paso interno falla).

**Coordinación FE:** El frontend eliminará las 5 llamadas y usará este único endpoint. Una vez disponible, avisa al frontend para hacer el cambio.


### BE-04 — Incluir `userId` en la respuesta de "primer login"
**Origen:** C4  
**Impacto si no se resuelve:** Cuando el backend detecta primer login y retorna `response: false`, el frontend no puede extraer el `userId`. El `updateUser` de primer login se ejecuta con `_id: ''` (o con el UUID de una sesión anterior de otro usuario), actualizando el usuario incorrecto.

**Problema actual:**  
El backend retorna algo como:
```json
{ "response": false, "message": "primer login" }
```
El frontend interpreta `response: false` como error y no guarda el `userId`.

**Acción requerida:**  
Cambiar la respuesta de "primer login" para que siempre incluya el `userId`:
```json
{
  "response": true,
  "primer_login": true,
  "data": {
    "userId": "string",
    "token": "string",
    "role": "string"
    ...
  },
  "message": "primer login"
}
```

Alternativamente (si no se puede cambiar el contrato), confirmación de si la respuesta actual ya incluye `userId` en algún campo de `data` para que el frontend lo lea directamente.

**Coordinación FE:** El frontend necesita saber exactamente el campo y la ruta del `userId` en la respuesta de "primer login" para leerlo correctamente.

---

## 🟡 IMPORTANTES — Resolver esta semana si es posible


### BE-07 — Registro y deduplicación de compras (fase 1) + verificación completa (fase 2)
**Origen:** M17  
**Impacto si no se resuelve:** La verificación está completamente desactivada (`payload.isNotEmpty` siempre retorna `true`). Cualquier compra manipulada o un mismo receipt enviado N veces es aceptado.

**Problema actual:**
```dart
// subscriptions_screen.dart
final valid = payload.isNotEmpty;   // ← siempre true, sin pasar por backend
```

---

#### Fase 1 — Esta semana (complejidad: BAJA)

No requiere integración con Apple/Google. El backend solo almacena el token de compra y rechaza duplicados. Esto elimina los dos ataques más comunes: reutilizar el mismo receipt y activar la suscripción sin haber comprado nada.

**Acción BE:**  
Crear la colección/tabla `purchase_tokens` con campos `{ token, productId, organizationId, platform, usedAt }` y un índice único sobre `token`.

```
POST /suscripciones/registrar-compra
Authorization: Bearer <token_usuario>

Body:
{
  "purchaseToken": "string",     // token de Google Play o transactionId de Apple
  "productId": "string",         // ver tabla de IDs abajo
  "platform": "android" | "ios",
  "organizationId": "string"
}
```

**IDs de producto reales por plataforma:**

| Plan | Android `productId` | iOS `productId` |
|---|---|---|
| Emprendedor mensual | `medium_monthly` | `gb97_emprendedor_mensual` |
| Emprendedor anual | `medium_annual` | `gb97_emprendedor_anual` |
| Premium mensual | `premium_monthly` | `gb97_premium_mensual` |
| Premium anual | `premium_annual` | `gb97_premium_anual` |
| Pro + Marketplace mensual | `pro_marketplace_monthly` | `gb97_pro_market_mensual` |
| Pro + Marketplace anual | `pro_marketplace_annual` | `gb97_pro_market_anual` |

```
Response válido (primera vez que se ve este token):
{
  "response": true,
  "valid": true,
  "message": "Compra registrada."
}

Response rechazado (token ya usado):
{
  "response": true,
  "valid": false,
  "message": "Esta compra ya fue procesada anteriormente."
}
```

**Acción FE:**  
Descomentar la llamada al backend y usar este endpoint. Solo activar el plan localmente si `valid == true`.

**Lo que protege esta fase:**
- ✅ Previene reutilización del mismo receipt (replay attack)
- ✅ Previene doble activación accidental
- ✅ El backend tiene registro de todas las compras para auditoría
- ❌ No verifica autenticidad del token contra Apple/Google (eso es fase 2)

---

#### Fase 2 — Sprint siguiente (complejidad: ALTA)

Una vez que la fase 1 está en producción, agregar verificación real contra las stores sin cambiar el contrato del endpoint (el frontend no necesita modificarse).

**Acción BE (interna, sin cambios de contrato):**  
Antes de insertar el token en la BD, hacer una llamada de validación:

- **Android:** `GET https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{pkg}/purchases/subscriptions/{productId}/tokens/{token}` — requiere service account en Google Cloud Console con permiso `androidpublisher`.
- **iOS:** `POST https://buy.itunes.apple.com/verifyReceipt` con `{ "receipt-data": base64, "password": shared_secret }` — el `shared_secret` se obtiene en App Store Connect → Tu app → Suscripciones → Shared Secret.

Si la store responde que el token no es válido → retornar `valid: false` antes de almacenar.

**Prerrequisitos para fase 2:**
- Google: service account con rol "Financial data viewer" en Play Console
- Apple: shared secret generado en App Store Connect (5 minutos de configuración)

**Coordinación FE:** El contrato del endpoint no cambia entre fase 1 y fase 2. El frontend no necesita modificarse al pasar de una fase a la otra.

---

### BE-08 — Auto-asignar suscripción Freemium sin que el frontend envíe un ID de MongoDB
**Origen:** M3  
**Impacto si no se resuelve:** Si el ID `'67b38e2a72bf455d09a041fc'` cambia en la BD (migración, reset), todos los registros nuevos fallarán o asignarán la suscripción incorrecta hasta un nuevo release de la app.

**Problema actual:**
```dart
// register_screen2.dart
'subscription': '67b38e2a72bf455d09a041fc'  // Plan Freemium — ID hardcodeado
```

**Acción requerida:**  
El endpoint de creación de organización (o el nuevo BE-01) debe asignar automáticamente el plan Freemium por defecto sin necesitar que el frontend lo envíe. Si se necesita un identificador, usar un código estable:
```json
"subscriptionPlan": "freemium"   // string estable, no ObjectId
```
Y el backend resuelve el `_id` interno.

---

### BE-09 — Resolver IDs de planes de suscripción desde código de plan, no ObjectId
**Origen:** M8  
**Impacto si no se resuelve:** Si la BD migra o se regeneran los documentos de planes, ningún cliente puede actualizar su suscripción hasta un nuevo release de app.

**Problema actual (4 IDs hardcodeados en el cliente):**
```dart
// api_service.dart — updateSubscription
subscriptionId = '6a07121ed84a9a5c2ccc4491'; // Emprendedor
subscriptionId = '6818c9efa558c757e143514f'; // Premium
subscriptionId = '6a071254d84a9a5c2ccc4493'; // Pro + Marketplace
subscriptionId = '67b38e2a72bf455d09a041fc'; // Freemium
```

**Acción requerida:**  
El endpoint `PUT /suscripciones/:orgId` debe aceptar un código de plan en lugar del ObjectId:

```
PUT /suscripciones/:organizationId
Authorization: Bearer <token_usuario>

Body:
{
  "planCode": "emprendedor" | "premium" | "pro_marketplace" | "freemium",
  "purchaseToken": "string"   // para validación
}
```

El backend mapea `planCode` → `subscriptionId` internamente.

---

### BE-13 — Endpoint de inserción masiva de clientes desde Excel
**Origen:** M25  
**Impacto si no se resuelve:** La importación masiva realiza 1 llamada HTTP por cada cliente en secuencia. Para 100 clientes son 100 round-trips sin batching. El proceso puede durar minutos y si se interrumpe (app al fondo, timeout) deja datos parcialmente importados sin posibilidad de recuperación.

**Acción requerida:**

```
POST /clientes/bulk
Authorization: Bearer <token_usuario>

Body:
{
  "organizationId": "string",
  "clients": [
    {
      "name": "string",
      "type": 1 | 2,
      "ruc": "string",
      ...
    },
    ...
  ]
}

Response:
{
  "response": true,
  "results": [
    { "index": 0, "success": true, "clientId": "string" },
    { "index": 1, "success": false, "error": "RUC duplicado" },
    ...
  ],
  "totalCreated": 98,
  "totalFailed": 2
}
```

El backend procesa cada cliente y retorna el resultado por ítem, permitiendo al frontend mostrar cuáles fallaron.

---

## 🔵 MEJORAS — Incluir si el tiempo lo permite

---

### BE-14 — Enviar credenciales iniciales por email en lugar de mostrarlas en pantalla
**Origen:** M2  
**Impacto:** La contraseña inicial `username + año` es predecible y se muestra en texto claro. Un observador puede verla.

**Acción requerida:**  
Al crear el usuario en BE-01 (o en el registro actual), el backend envía un correo con las credenciales de acceso al email registrado. El frontend mostrará: *"Tu cuenta ha sido creada. Te hemos enviado las credenciales a tu correo."*

---

### BE-15 — Rate limiting en endpoint de reenvío de código de verificación
**Origen:** L2  
**Impacto:** Sin límite de reintentos, el botón "Reenviar código" puede usarse para spam de correos.

**Acción requerida:**  
Implementar rate limiting de 1 reenvío por email cada 60 segundos. Retornar `429 Too Many Requests` con el tiempo restante si se excede:
```json
{
  "response": false,
  "message": "Por favor espera 45 segundos antes de solicitar un nuevo código.",
  "retryAfterSeconds": 45
}
```

---

### BE-16 — Diferenciación entre "persona no encontrada" y "error de búsqueda"
**Origen:** L1  
**Impacto:** Si la búsqueda de personas falla por error de red y el frontend recibe `response: false`, interpreta que la persona no existe y crea una nueva, generando duplicados.

**Acción requerida:**  
Estandarizar el contrato de búsqueda de personas:
- **No encontrado (éxito vacío):** `{ "response": true, "data": [] }`
- **Error real:** `{ "response": false, "message": "Error interno." }`

El frontend puede entonces diferenciar entre "no existe → crear" vs "error → reintentar".

---

### BE-17 — Endpoint atómico para crear persona + cliente en un solo paso
**Origen:** L10  
**Impacto:** Si `insertPerson` tiene éxito pero `insertClient` falla, la persona queda huérfana en BD. El reintento puede crear duplicados.

**Acción requerida:**

```
POST /clientes/con-persona
Authorization: Bearer <token_usuario>

Body:
{
  "persona": { "nombre": "...", "dni": "...", ... },
  "cliente": { "organizacionId": "...", ... }
}

Response exitoso:
{
  "response": true,
  "data": { "clientId": "string", "personId": "string" }
}
```

Operación atómica: o se crean ambos registros o ninguno.

---

## Resumen de prioridades

| # | ID | Severidad | Acción backend | Complejidad estimada |
|---|---|---|---|---|
| 0 | BE-00 | 🔴 CRÍTICO | Calcular y retornar `subscriptionEnd` real desde el backend (suscripción expira por decisión del front actualmente) | Baja |
| 1 | BE-01 | 🔴 CRÍTICO | Endpoint registro unificado atómico | Alta |
| 2 | BE-04 | 🔴 CRÍTICO | Incluir `userId` en respuesta de primer login | Baja |
| 3 | BE-07 | 🟡 IMPORTANTE | Registro + deduplicación de compras (**fase 1** esta semana, verificación Apple/Google en fase 2) | Baja (fase 1) / Alta (fase 2) |
| 4 | BE-08 | 🟡 IMPORTANTE | Auto-asignar Freemium sin ObjectId del cliente | Baja |
| 5 | BE-09 | 🟡 IMPORTANTE | Resolver IDs de planes desde código estable | Baja |
| 6 | BE-13 | 🟡 IMPORTANTE | Endpoint de inserción masiva de clientes | Media |
| 7 | BE-14 | 🔵 MEJORA | Enviar credenciales iniciales por email | Media |
| 8 | BE-15 | 🔵 MEJORA | Rate limiting en reenvío de código de verificación | Baja |
| 9 | BE-16 | 🔵 MEJORA | Diferenciar "no encontrado" vs "error" en búsqueda personas | Baja |
| 10 | BE-17 | 🔵 MEJORA | Endpoint atómico crear persona + cliente | Media |