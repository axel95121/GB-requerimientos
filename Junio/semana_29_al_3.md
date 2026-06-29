# Requerimientos Backend — Semana 29 de junio 2026

**Alcance:** Estabilización de Crashlytics (timeouts en lambdas), corrección del envío de correo en proformas, validación de nuevos campos tributarios de organización, validación de nombres únicos en ítems/combos/extras, nuevo campo `barcodes` en ítems variantes, y trazabilidad de acciones de usuario (auditoría).
**Contexto:** Esta semana se agrupan hallazgos de monitoreo en producción (Crashlytics) junto con ajustes pendientes de los módulos de organización, catálogo y auditoría que quedaron abiertos tras los últimos cambios de frontend.

> **Convenciones:**
> - 🔴 **Crítico** — bloqueante para el flujo principal
> - 🟡 **Coordinado** — FE + BE deben alinear contrato antes de implementar
> - 🟢 **Independiente** — BE puede completar sin esperar al FE

---

## 🔴 BE-01 — Revisión de errores por timeout en Crashlytics

**Contexto:**
El panel de Crashlytics reporta errores en producción para la app Android (`com.gb97.order`). Es necesario que el equipo de backend revise specíficamente los reportes asociados a **timeouts**, dado que estos suelen originarse en las lambdas/funciones del backend (tiempos de respuesta excedidos, cold starts, conexiones a base de datos colgadas, etc.) y no en el cliente.

**Acción requerida:**
1. Acceder al panel:
   ```
   https://console.firebase.google.com/u/0/project/delivery-40b09/crashlytics/app/android:com.gb97.order
   ```
2. Filtrar y priorizar los reportes cuyo stack trace o mensaje indique `timeout`, `SocketTimeoutException`, `TimeoutException` o similares.
3. Identificar qué endpoints/lambdas están asociados a esos timeouts (correlacionar con logs de CloudWatch/servidor en el mismo rango de fecha/hora del crash).
4. Aplicar las correcciones de estabilización correspondientes: aumentar timeout de lambda, optimizar queries lentas, agregar reintentos/backoff donde aplique, o ajustar el tamaño de memoria/concurrencia de la función.
5. Reportar al FE qué endpoints fueron afectados para validar si el timeout configurado en el cliente (HTTP client) es coherente con el nuevo SLA del backend.

**Resultado esperado:** Reducción medible de los crashes por timeout en el próximo corte de Crashlytics.

---

## 🔴 BE-02 — Corrección de envío de correo al emitir proformas

**Contexto:**
Al enviar una **orden**, el correo llega correctamente al cliente. Al enviar una **proforma**, el correo no se está emitiendo, aunque la operación responde como exitosa en la app.

**Acción requerida:**
1. Revisar el servicio de envío de correo (`/email` o el endpoint que use el flujo de proformas) y comparar la implementación contra el flujo de órdenes, que sí funciona.
2. Confirmar si la proforma usa un payload, plantilla o endpoint diferente al de órdenes, y si ese código está realmente disparando el envío (puede ser un fallo silencioso: la API responde `response: true` pero el correo no se encola/envía).
3. Validar que el remitente, plantilla y datos del destinatario (email del cliente) se estén resolviendo correctamente para proformas.
4. Agregar logging o manejo de error explícito si el envío de correo falla, en lugar de responder éxito sin haber enviado el correo.

**Referencia FE:** El frontend invoca `ApiService.sendEmail` (`lib/src/api/api_service.dart:1956`) tanto para órdenes como para proformas — se requiere confirmar si el backend distingue el tipo de documento y arma el contenido del correo correctamente en cada caso.

---

## 🟡 BE-03 — Validación de nuevos campos tributarios de organización

**Contexto:**
Se agregaron nuevos campos al formulario de "Mi Comercio" (`lib/src/ui/organization/my_organization_screen.dart`) para capturar información tributaria de la organización. Se necesita que el backend valide y persista estos campos al actualizar la organización.

**Endpoint de referencia:**
```
PUT https://api-prod-textil.gb97.ec/organizaciones/:id
```

**Payload actual enviado por el FE (`updateOrganization`):**
```json
{
  "_id": "<organizationId>",
  "organizationName": "string",
  "organizationAlias": "string",
  "organizationEmail": "string",
  "organizationTelephone": "string",
  "organizationCellphone": "string",
  "organizationAddress": "string",
  "codigo_iva": 4,
  "iva": 15,
  "obligatorio_contabilidad": true,
  "type_taxpayer": "RIMPE" | "REGIMEN GENERAL" | "CONTRIBUYENTE ESPECIAL",
  "isWithholdingAgent": true,
  "special_taxpayer_number": "string"
}
```

**Acción requerida:**
1. Confirmar que el endpoint acepta y persiste los campos nuevos: `obligatorio_contabilidad` (boolean), `type_taxpayer` (enum de 3 valores), `isWithholdingAgent` (boolean), `special_taxpayer_number` (string).
2. Validar reglas de negocio en backend (no solo en el FE):
   - `type_taxpayer` debe ser uno de los 3 valores permitidos.
   - `special_taxpayer_number` es obligatorio únicamente cuando `type_taxpayer === 'CONTRIBUYENTE ESPECIAL'`; en cualquier otro caso debe poder ir vacío.
3. Confirmar el nombre exacto de cada campo en el modelo de organización en base de datos (el FE está usando snake_case para los campos tributarios y camelCase para el resto, replicando lo ya existente en el endpoint — verificar consistencia).
4. Confirmar que `GET` de organización (`ApiService.getOneOrganizationFilter`) devuelve estos mismos campos para que el FE pueda precargarlos (ya se está leyendo `obligatorio_contabilidad`, `type_taxpayer`, `special_taxpayer_number`, `isWithholdingAgent` desde la respuesta).

**Coordinación FE (BE-03 ↔ FE-03):** Si el backend requiere otro nombre de campo o tipo de dato distinto al listado arriba, avisar para ajustar el payload del FE antes de pasar a QA.

---

## 🟡 BE-04 — Validación de nombre único en ítems, combos y extras

**Contexto:**
Actualmente es posible registrar dos ítems, combos o extras con el mismo nombre dentro de la misma organización/sucursal, lo cual genera confusión en catálogo y reportes. Se requiere una validación de unicidad de nombre, pero **acotada por organización y sucursal** para evitar falsos negativos (es decir, el mismo nombre debe poder repetirse entre organizaciones o sucursales distintas, pero no dentro de la misma).

**Endpoints de referencia:**
```
POST/PUT https://api-prod-general.gb97.ec/items-variant
POST/PUT https://api-prod-general.gb97.ec/combo
POST/PUT https://api-prod-general.gb97.ec/extra
```

**Acción requerida:**
1. Antes de crear (`POST`) o actualizar (`PUT`) un ítem/combo/extra, validar en backend que no exista otro registro **activo** con el mismo nombre (recomendado: comparación case-insensitive y con `trim()`) para la misma combinación de `organization` + `branch` (o `sucursal`, según el nombre real del campo en el modelo).
2. En `PUT` (edición), excluir el propio `_id` del registro de la validación, para no bloquear el guardado de un registro contra sí mismo.
3. Definir si la validación debe considerar registros eliminados/inactivos (lógicamente borrados) como "libres" para reutilizar el nombre, o si deben seguir bloqueando — confirmar regla de negocio.
4. Responder con un mensaje claro y diferenciable cuando se rechace por duplicado, por ejemplo:
   ```json
   { "response": false, "message": "Ya existe un ítem con este nombre en esta sucursal." }
   ```
   para que el FE pueda mostrar el error específico en el formulario en lugar de un error genérico.

**Coordinación FE (BE-04 ↔ FE-04):** El FE necesita el `message`/código de error específico de duplicado para mapearlo al campo de nombre en el formulario, en vez de mostrarlo como error genérico de guardado.

---

## 🟢 BE-05 — Nuevo campo `barcodes` en ítems variantes

**Contexto:**
Se requiere soportar múltiples códigos de barra por producto (por ejemplo, el mismo producto con distintas presentaciones o empaques que comparten variante pero tienen códigos de barra distintos).

**Endpoint de referencia:**
```
POST/PUT https://api-prod-general.gb97.ec/items-variant
```

**Acción requerida:**
1. Agregar el campo `barcodes` al modelo de ítem variante: `Array<String>`.
2. Aceptar el campo en `POST` (creación) y `PUT` (actualización), permitiendo arrays vacíos `[]` (campo opcional).
3. Devolver el campo `barcodes` en las respuestas de lectura del ítem (`GET`/`filter`) para que el FE pueda precargarlo.
4. Definir si se requiere validación de unicidad de código de barras (recomendado: único por organización, para evitar que dos productos distintos compartan el mismo código) — a confirmar con negocio si aplica en esta fase o en una posterior.

**Coordinación FE (BE-05 ↔ FE-05):** El FE implementará el campo de captura/lectura de código de barras (posiblemente con escaneo de cámara) una vez confirmada la estructura de respuesta.

---

## 🟡 BE-06 — Auditoría: trazabilidad de usuario y acción por registro

**Contexto:**
Se necesita poder determinar **qué usuario realizó qué acción** sobre los registros principales de la app, para fines de soporte, control interno y resolución de disputas.

**Módulos afectados:** Clientes, Productos (ítems/variantes/combos/extras), Comercios (organización/sucursal), Órdenes, Proformas y Usuarios.

**Acción requerida:**
1. Agregar a los modelos de los módulos listados los siguientes campos de auditoría (si no existen ya):
   ```json
   {
     "created_by": "<userId>",
     "created_at": "<timestamp>",
     "updated_by": "<userId>",
     "updated_at": "<timestamp>"
   }
   ```
2. Poblar `created_by`/`updated_by` automáticamente en backend a partir del usuario autenticado (token), **no** confiar en un valor enviado por el cliente.
3. Para los módulos con mayor sensibilidad (Órdenes, Proformas, Usuarios), evaluar mantener un **historial de cambios** (log de auditoría) con: usuario, acción (`create`/`update`/`delete`/`status_change`), fecha y, de ser posible, el diff de los campos modificados — al menos para los campos críticos (estado, montos, datos de contacto).
4. Exponer un endpoint de consulta de historial por registro, por ejemplo:
   ```
   GET /:modulo/:id/history
   Response: {
     "response": true,
     "history": [
       { "user": "<userId>", "userName": "...", "action": "update", "date": "...", "changes": { "status": { "from": "pending", "to": "approved" } } }
     ]
   }
   ```
5. Confirmar con FE qué pantallas requerirán mostrar este historial (por ejemplo, detalle de orden o detalle de usuario) para priorizar qué módulo se expone primero.

**Coordinación FE (BE-06 ↔ FE-06):** El FE definirá en qué pantallas se mostrará el historial de auditoría una vez que el backend confirme la disponibilidad de los campos/endpoint por módulo.

---

## 📋 Seguimiento de sprint (semana 29 jun – 3 jul)

| ID | Descripción | Prioridad | Estado |
|---|---|---|---|
| BE-01 | Revisión de timeouts en Crashlytics y estabilización de lambdas | 🔴 Crítico | Pendiente |
| BE-02 | Corrección de envío de correo en proformas | 🔴 Crítico | Pendiente |
| BE-03 | Validación de campos tributarios de organización | 🟡 Coordinado | Pendiente |
| BE-04 | Nombre único en ítems/combos/extras por organización y sucursal | 🟡 Coordinado | Pendiente |
| BE-05 | Campo `barcodes` en ítems variantes | 🟢 Independiente | Pendiente |
| BE-06 | Auditoría: usuario y acción por registro | 🟡 Coordinado | Pendiente |
