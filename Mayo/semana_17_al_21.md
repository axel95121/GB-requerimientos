# Requerimientos Backend — Semana 17 de mayo de 2026

**Fecha:** 17 de mayo de 2026  
**Solicitado por:** Frontend (App Delivery)  
**Dirigido a:** Backend  
**Sprint:** Semana del 17 al 23 de mayo de 2026

---

## Índice

1. [Login con ID de usuario explícito (social login)](#1-login-con-id-de-usuario-explícito-social-login)
2. [Populate del campo `rol` en el endpoint de filtro de usuarios](#2-populate-del-campo-rol-en-el-endpoint-de-filtro-de-usuarios)
3. [Integración Payphone — Cajita de Pagos (móvil vía WebView)](#3-integración-payphone--cajita-de-pagos-móvil-vía-webview)
4. [Corrección: campo `plan` nulo en beneficios al canjear código](#4-corrección-campo-plan-nulo-en-beneficios-al-canjear-código)
5. [Corrección: beneficio del referente no acreditado al canjear](#5-corrección-beneficio-del-referente-no-acreditado-al-canjear-un-código-de-referido)
6. [Completar array `control` — campos faltantes por plan](#6-completar-array-control--campos-faltantes-por-plan)

---

## 1. Login con ID de usuario explícito (social login)

### Contexto

Cuando un usuario inicia sesión con Google o Apple y tiene **múltiples cuentas** asociadas al mismo correo electrónico (y por tanto al mismo `google_id` / `apple_id`), el frontend muestra una pantalla de selección de cuenta. Una vez que el usuario elige la cuenta deseada, el frontend actualiza ese usuario con el `google_id` y luego vuelve a llamar al endpoint de login con solo el `google_id`.

**El problema actual:** el backend devuelve siempre el primer documento que coincide con el `google_id`, que puede ser una cuenta distinta a la seleccionada (por ejemplo, una cuenta sin organización asignada). Esto genera un loop de error porque el login falla por `organizationId: null`.

### Solución requerida

Permitir que el body del endpoint `POST /user/login` acepte un campo `_id` opcional. Cuando se envíe junto con `google_id` o `apple_id`, el backend debe usar el `_id` para identificar el documento exacto a autenticar, en lugar de buscar por el campo social únicamente.

**Método:** `POST`  
**Ruta:** `/user/login`

---

#### Caso actual (sin cambios — debe seguir funcionando)

**Body:**
```json
{
  "google_id": "YNymeAFhiXfIkikY420WcdFPOqm2"
}
```

**Comportamiento actual:** busca el primer usuario con ese `google_id`.

---

#### Caso nuevo (con `_id` explícito)

**Body:**
```json
{
  "google_id": "YNymeAFhiXfIkikY420WcdFPOqm2",
  "_id": "69bd4aa152651da7385bbeab"
}
```

**Comportamiento esperado:**
1. Buscar el usuario por `_id` = `"69bd4aa152651da7385bbeab"`.
2. Verificar que el `google_id` del documento coincida con el enviado (validación de seguridad).
3. Si coincide, proceder con el login normalmente y devolver el token.
4. Si no coincide, responder con error `401`.

El mismo comportamiento aplica para `apple_id`.

**Body equivalente con Apple:**
```json
{
  "apple_id": "apple.uid.xxxx",
  "_id": "69bd4aa152651da7385bbeab"
}
```

---

#### Respuesta esperada (sin cambios)

```json
{
  "response": true,
  "message": "Login Exitoso.",
  "data": {
    "userId": "69bd4aa152651da7385bbeab",
    "userName": "BrandonTU",
    "token": "...",
    "organizationId": { ... },
    "rol": { ... },
    "sucursal": { ... }
  }
}
```

#### Respuesta en caso de `_id` y `google_id` no coincidentes

```json
{
  "response": false,
  "message": "Las credenciales no corresponden al usuario indicado."
}
```

---

### Notas adicionales

- El campo `_id` es **opcional**. Si no se envía, el comportamiento actual no cambia.
- Esta modificación no afecta el login por `userName`/`password` ni ningún otro método.

---

## 2. Populate del campo `rol` en el endpoint de filtro de usuarios

### Contexto

El endpoint `POST /user/filter` devuelve actualmente el campo `rol` como un **ObjectId** en string (sin expandir), por ejemplo:

```json
{
  "_id": "69146750efe984c419075e35",
  "userName": "AlexanderTubon",
  "rol": "68af8b11879b470f2e59922f",
  ...
}
```

El frontend necesita conocer el **nombre y tipo del rol** (`rol.rol`, `rol.type`) de cada usuario en la pantalla de selección de cuenta y en la gestión de usuarios, para poder mostrar información útil al usuario final (ej.: "Administrador", "Transportista").

### Solución requerida

Agregar un **populate del campo `rol`** en el endpoint `POST /user/filter`, de modo que el objeto devuelto incluya los datos completos del rol en lugar del ObjectId.

**Método:** `POST`  
**Ruta:** `/user/filter`

---

#### Respuesta actual (campo `rol` sin populate)

```json
{
  "_id": "69146750efe984c419075e35",
  "userName": "AlexanderTubon",
  "rol": "68af8b11879b470f2e59922f",
  "userActive": "inactivo"
}
```

#### Respuesta esperada (campo `rol` con populate)

```json
{
  "_id": "69146750efe984c419075e35",
  "userName": "AlexanderTubon",
  "rol": {
    "_id": "68af8b11879b470f2e59922f",
    "type": "gb97",
    "rol": "super-admin",
    "module": [],
    "createdAt": "2025-08-27T22:47:45.150Z",
    "updatedAt": "2025-08-27T22:47:45.150Z"
  },
  "userActive": "inactivo"
}
```

---

### Notas adicionales

- Este populate ya se realiza en otros endpoints del sistema (por ejemplo, `GET /user/:id`), por lo que el patrón es consistente con la arquitectura existente.
- No se requiere ningún cambio en el body ni en los filtros del endpoint; únicamente el populate del campo `rol` en la consulta de Mongoose.

---

## 3. Integración Payphone — Cajita de Pagos (móvil vía WebView)

### Contexto

La **Cajita de Pagos de Payphone** está diseñada exclusivamente para entornos web (requiere dominio con SSL, SDK JS/CSS inyectado en HTML y un Bearer Token que no puede exponerse en la app). Por esto, la app móvil **no puede integrar el SDK directamente**.

La solución implementada en el frontend es la misma que se usa para Datafast: cargar una **URL del backend** dentro de un `WebView`. El backend sirve la página HTML con el SDK de Payphone ya configurado, manteniendo el token seguro en el servidor. Cuando el pago termina, Payphone redirige a la URL de respuesta, el WebView intercepta esa navegación y la app llama al backend para confirmar la transacción.

### Flujo completo

```
App (Flutter/WebView)              Backend GB97                   Payphone
─────────────────────────────────────────────────────────────────────────────
1. Crear orden ─────────────────▶  Retorna orderId (clientTransactionId)
2. GET /payphone/checkout?... ──▶  Sirve HTML + SDK Payphone configurado
3. Usuario completa pago ──────────────────────────────────────────────────▶
4. Payphone redirige a URL de respuesta (?id=...&clientTransactionId=...) ◀─
5. WebView intercepta redirect (NavigationDecision.prevent)
6. POST /payphone/confirm ──────▶  Reenvía a Payphone con Bearer Token
7. App muestra éxito / cancelado / error ◀──────────────────────────────────
```

> ⚠️ Si el backend no ejecuta la confirmación dentro de los **5 minutos** posteriores al pago, Payphone revierte la transacción automáticamente.

---

### Endpoint 1 — Servir la cajita de pagos

**Método:** `GET`  
**Ruta:** `/payphone/checkout`

El backend debe devolver una página HTML completa que incluya el SDK de Payphone (JS + CSS) configurado con los parámetros de la transacción.

#### Query parameters recibidos del frontend

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `clientTransactionId` | `string` | ✅ | ID único de la orden (máx. 50 chars) |
| `amount` | `int` | ✅ | Monto total **en centavos** (ej.: $3.15 → `315`) |

#### Respuesta esperada

`Content-Type: text/html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script type="module" src="https://cdn.payphonetodoesposible.com/box/v2.0/payphone-payment-box.js"></script>
  <link href="https://cdn.payphonetodoesposible.com/box/v2.0/payphone-payment-box.css" rel="stylesheet">
</head>
<body>
  <script>
    window.addEventListener('DOMContentLoaded', () => {
      new PPaymentButtonBox({
        token: 'TU_TOKEN_PAYPHONE',           // ← guardado de forma segura en el backend
        clientTransactionId: '{clientTransactionId}',
        amount: {amount},                      // en centavos
        amountWithoutTax: {amount},            // ajustar según aplique IVA
        currency: 'USD',
        storeId: 'TU_STOREID',
        reference: 'Pago orden #{clientTransactionId}',
        lang: 'es',
        defaultMethod: 'card'
      }).render('pp-button');
    });
  </script>
  <div id="pp-button"></div>
</body>
</html>
```

> ℹ️ Los valores monetarios deben expresarse como **enteros en centavos**:  
> `amount = amountWithoutTax + amountWithTax + tax + service + tip`  
> Ejemplo: $3.15 USD → `amount: 315`

#### URL de respuesta (configurar en Payphone Developer)

La URL de respuesta registrada en Payphone Developer debe apuntar al backend, por ejemplo:

```
https://api-prod-general.gb97.ec/payphone/response
```

Payphone redirigirá a esta URL con los parámetros:

```
https://api-prod-general.gb97.ec/payphone/response?id=23178284&clientTransactionId=ORDEN-001
```

El backend puede dejar que esta URL sea simplemente interceptable por el WebView (es decir, no necesita servir contenido real en `/payphone/response`; el WebView detecta la navegación hacia esa URL, la intercepta y llama a `/payphone/confirm`).

---

### Endpoint 2 — Confirmar la transacción

**Método:** `POST`  
**Ruta:** `/payphone/confirm`

Recibe los parámetros de la URL de respuesta y reenvía la confirmación a la API de Payphone usando el Bearer Token guardado en el backend.

#### Body recibido del frontend

```json
{
  "id": 23178284,
  "clientTxId": "ORDEN-001"
}
```

#### Llamada que el backend hace a Payphone

```
POST https://paymentbox.payphonetodoesposible.com/api/confirm
Authorization: Bearer TU_TOKEN_PAYPHONE
Content-Type: application/json

{
  "id": 23178284,
  "clientTxId": "ORDEN-001"
}
```

#### Respuesta que el backend debe devolver al frontend

El backend reenvía directamente el JSON de Payphone:

```json
{
  "statusCode": 3,
  "transactionStatus": "Approved",
  "authorizationCode": "W23178284",
  "clientTransactionId": "ORDEN-001",
  "transactionId": 23178284,
  "amount": 315,
  "cardBrand": "Mastercard Produbanco/Promerica",
  "lastDigits": "XX17",
  "cardType": "Credit",
  "email": "cliente@mail.com",
  "currency": "USD",
  "date": "2026-05-18T11:57:26.367",
  "reference": "Pago orden #ORDEN-001"
}
```

> `statusCode: 3` = Aprobado · `statusCode: 2` = Cancelado

En caso de error de Payphone, reenviar el JSON de error con el mismo status HTTP:

```json
{
  "message": "La transacción no existe, verifique que el identificador enviado sea correcto.",
  "errorCode": 20
}
```

---

### Consideraciones de seguridad

- El **Bearer Token de Payphone nunca debe enviarse al frontend** ni incluirse en la URL de checkout. Debe almacenarse como variable de entorno en el backend.
- El `storeId` de Payphone también debe configurarse en el backend.
- El dominio registrado en Payphone Developer debe ser el dominio del backend (`api-prod-general.gb97.ec`), no el de la app móvil.
- El backend debe registrar la URL de respuesta en su configuración de Payphone Developer: `https://api-prod-general.gb97.ec/payphone/response`.

### Constantes usadas en el frontend

```dart
// lib/src/api/constants.dart
static const String payphoneUrl         = '$apiUrl/payphone';
static const String payphoneCheckoutUrl = '$payphoneUrl/checkout';
static const String payphoneConfirmUrl  = '$payphoneUrl/confirm';
```

---

## 4. Corrección: campo `plan` nulo en beneficios al canjear código

### Contexto

Al canjear un código de referido o activación, el endpoint `POST /referrals/redeem` devuelve actualmente el campo `plan: null` dentro de cada objeto de `applied_benefits`:

```json
{
  "success": true,
  "message": "Código canjeado exitosamente",
  "applied_benefits": [
    {
      "id": "6a0a897aa51d961fed8dc6c4",
      "description": "30 días gratis",
      "days": 30,
      "plan": null
    }
  ],
  "new_expiration_date": "2026-06-17T04:52:28.463Z"
}
```

La `new_expiration_date` se actualiza correctamente (los días se suman), pero **no se activa ningún plan de suscripción** sobre la organización. El usuario queda con días extendidos pero sin un plan asignado, por lo que no obtiene los beneficios funcionales del plan (límites de órdenes, facturas, sucursales, etc.).

### Causa raíz

El documento del beneficio `6a0a897aa51d961fed8dc6c4` tiene el campo `plan` vacío en la base de datos. Al hacer populate o serialización en el response, se devuelve `null`.

### Correcciones requeridas

#### 4.1 — Asignar el plan al beneficio en la base de datos

Actualizar el documento del beneficio `6a0a897aa51d961fed8dc6c4` para que el campo `plan` apunte al ObjectId del **Plan Emprendedor** (`6a07121ed84a9a5c2ccc4491`):

```json
// Documento beneficio — campo a corregir
{
  "_id": "6a0a897aa51d961fed8dc6c4",
  "description": "30 días gratis",
  "days": 30,
  "plan": "6a07121ed84a9a5c2ccc4491"   // ← Plan Emprendedor
}
```

#### 4.2 — Activar el plan en la organización al canjear

Al ejecutar `POST /referrals/redeem`, si el beneficio tiene un `plan` asignado, el backend debe:

1. Obtener la organización del usuario autenticado.
2. Si la organización **no tiene suscripción activa** o tiene el plan Freemium: asignar el `plan` del beneficio como suscripción activa.
3. Si la organización **ya tiene una suscripción activa** de igual o mayor nivel: mantener el plan actual y únicamente extender `fecha_expiracion` sumando los `days` del beneficio.
4. Actualizar `fecha_expiracion` sumando los `days` desde la fecha actual (o desde la fecha de expiración vigente, si es posterior a hoy).

#### 4.3 — Respuesta esperada corregida

```json
{
  "success": true,
  "message": "Código canjeado exitosamente",
  "applied_benefits": [
    {
      "id": "6a0a897aa51d961fed8dc6c4",
      "description": "30 días gratis",
      "days": 30,
      "plan": "Emprendedor"
    }
  ],
  "new_expiration_date": "2026-06-17T04:52:28.463Z"
}
```

> ℹ️ El campo `plan` en el response puede ser el nombre del plan (string) o el ObjectId; el frontend acepta ambos formatos.

---

## 5. Corrección: beneficio del referente no acreditado al canjear un código de referido

### Contexto

Cuando el usuario A canjea el código de referido del usuario B, el usuario B (referente/benefactor) **no recibe ningún beneficio**. Sus estadísticas de referidos permanecen en `canjeados: 0` luego del canje:

```json
// Stats del referente (usuario B) — obtenidas desde GET /referrals/my-code
{
  "code": "GB-C3BKN0",
  "stats": { "referidos": 0, "canjeados": 0, "libres": 0 }
}
```

El requerimiento original ([ver `requerimientos_backend_referidos.md`](./requerimientos_backend_referidos.md)) especificaba que al canjear un código de referido, **ambas partes reciben su beneficio correspondiente**:

- **Usuario que canjea (referido):** recibe los `benefits_for_referred`.
- **Dueño del código (referente):** recibe los `benefits_for_referrer`.

### Corrección requerida

Al procesar `POST /referrals/redeem`:

1. Identificar al dueño del código canjeado (campo `owner` o equivalente en el documento del código).
2. Aplicar los `benefits_for_referrer` a la organización del dueño del código:
   - Sumar los `days` a su `fecha_expiracion`.
   - Si el beneficio tiene `plan` y el dueño no tiene suscripción activa, activar ese plan.
3. Incrementar el contador `canjeados` en las estadísticas del referente.
4. El response del endpoint no necesita cambiar (ya refleja solo los beneficios del usuario que canjea); el crédito al referente es un efecto secundario interno.

### Comportamiento esperado tras el canje

```json
// Stats del referente (usuario B) — DESPUÉS del canje
{
  "code": "GB-C3BKN0",
  "stats": { "referidos": 1, "canjeados": 1, "libres": 0 }
}
```

### Notas adicionales

- Los beneficios del referente configurados en `GET /referrals/benefits` → `benefits_for_referrer` deben usarse como fuente de verdad.
- Si el referente ya tiene suscripción activa de mayor nivel, solo sumar los días (no degradar el plan).
- Este procesamiento debe ser atómico o manejarse con reintentos para evitar que un error en el crédito al referente impida el canje del usuario principal.

---

## 6. Completar array `control` — campos faltantes por plan

### Contexto

La app consume el campo `control` del objeto suscripción para conocer los límites operativos de cada plan y aplicarlos en tiempo real. El frontend persiste estos límites en SharedPreferences y los valida en cada pantalla relevante (órdenes, facturas, proformas, sucursales, catálogos, etc.).

Actualmente el array `control` incluye los tipos `Ordenes`, `Factura`, `Sucursal`, `Proformas` y `Catalogo`. Sin embargo, existen **dos restricciones definidas en los planes que no se están enviando aún**:

| `tipo` esperado por el frontend | Clave en SharedPreferences | Descripción |
|---|---|---|
| `Usuarios` | `usersControl` | Número máximo de usuarios (staff) de la organización |
| `Clientes Documentos` | `clientDocsControl` | Archivos adjuntos máximos por cliente |
| `Items` | `itemFeaturesControl` | Funcionalidades avanzadas de ítems habilitadas |

> ℹ️ Los nombres `Clientes Documentos` e `Items` son los **exactamente esperados por el frontend** tal como está el código actual. Si el backend ya los persiste con nombres distintos (`ClienteDocs`, `ItemFeatures` u otro), por favor confirmar para alinear.

---

### 6.1 — Nuevo campo: `Usuarios`

**Descripción:** límite máximo de usuarios activos que puede tener la organización (admins + vendedores + transportistas sumados).

**Valores por plan:**

| Plan | `cantidad` |
|---|---|
| GRATUIDAD | `"1"` |
| EMPRENDEDOR | `"2"` |
| PREMIUM | `"5"` |
| PRO + MARKETPLACE | `"999999"` |

**Comportamiento en el frontend:** al intentar crear un nuevo usuario en la pantalla de gestión de staff, la app consultará `usersControl` y bloqueará la acción si el conteo actual de usuarios activos de la organización alcanza o supera ese límite.

---

### 6.2 — Confirmar / añadir campos `Clientes Documentos` e `Items`

El frontend ya lee y persiste estos dos campos desde el array `control`. Lo que se solicita es verificar que el backend los incluya en todos los planes con los valores correctos:

**`Clientes Documentos`** — número de archivos que se pueden adjuntar por cliente:

| Plan | `cantidad` |
|---|---|
| GRATUIDAD | `"0"` |
| EMPRENDEDOR | `"0"` |
| PREMIUM | `"3"` |
| PRO + MARKETPLACE | `"999999"` |

**`Items`** — habilita funcionalidades avanzadas de ítems (variantes, atributos adicionales, etc.):

| Plan | `cantidad` |
|---|---|
| GRATUIDAD | `"0"` |
| EMPRENDEDOR | `"0"` |
| PREMIUM | `"0"` |
| PRO + MARKETPLACE | `"999999"` |

> ℹ️ `"0"` indica que la funcionalidad no está disponible; `"999999"` indica ilimitado/habilitado.

---

### 6.3 — Estructura completa esperada del array `control`

A partir de esta semana, el frontend espera recibir el array `control` con **los 7 campos** en el orden indicado (el orden importa como fallback):

```json
"control": [
  { "tipo": "Ordenes",             "cantidad": "400"    },
  { "tipo": "Factura",             "cantidad": "300"    },
  { "tipo": "Sucursal",            "cantidad": "1"      },
  { "tipo": "Proformas",           "cantidad": "999999" },
  { "tipo": "Clientes Documentos", "cantidad": "3"      },
  { "tipo": "Items",               "cantidad": "0"      },
  { "tipo": "Catalogo",            "cantidad": "10"     },
  { "tipo": "Usuarios",            "cantidad": "5"      }
]
```

> Ejemplo para el **Plan Premium**. Ajustar los valores según la tabla de cada plan.

---

### 6.4 — Tabla de valores completa por plan

| `tipo` | GRATUIDAD | EMPRENDEDOR | PREMIUM | PRO + MARKETPLACE |
|---|---|---|---|---|
| `Ordenes` | `"30"` | `"80"` | `"400"` | `"999999"` |
| `Factura` | `"24"` | `"50"` | `"300"` | `"999999"` |
| `Sucursal` | `"1"` | `"1"` | `"1"` | `"999999"` |
| `Proformas` | `"50"` | `"150"` | `"999999"` | `"999999"` |
| `Clientes Documentos` | `"0"` | `"0"` | `"3"` | `"999999"` |
| `Items` | `"0"` | `"0"` | `"0"` | `"999999"` |
| `Catalogo` | `"1"` | `"3"` | `"10"` | `"999999"` |
| `Usuarios` | `"1"` | `"2"` | `"5"` | `"999999"` |

---

### Notas adicionales

- Todos los `cantidad` se envían como **string** (no número), tal como está actualmente.
- El valor `"999999"` representa **ilimitado** en toda la plataforma.
- Este cambio afecta los documentos de todos los planes en la colección de suscripciones de MongoDB. Se requiere una **migración de datos** para añadir los campos faltantes a los planes existentes.
- El frontend ya tiene el modelo `ControlLimit` y el método `getLimit(tipo)` preparados para leer estos nuevos campos sin requerir cambios de código del lado del cliente.

