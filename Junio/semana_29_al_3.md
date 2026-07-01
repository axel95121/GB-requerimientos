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

# Requerimiento Backend — Tipo de documento en registro de cliente

**Fecha:** 2026-07-01  
**Prioridad:** Alta  
**Alcance:** Registro y gestión de datos de persona para clientes marketplace (módulo personas/clientes).

---

## 1) Contexto actual

En el frontend, el registro de cliente reutiliza los campos de persona y actualmente decide validaciones por **tipo de persona**:

- `type = 1` Persona natural
- `type = 2` Persona jurídica

Hoy no existe un campo explícito de tipo de documento. El número de documento (`personId`) se interpreta implícitamente según `type`:

- Natural => cédula
- Jurídica => RUC

Con el nuevo requerimiento se necesita soportar también **pasaporte** en el registro de cliente.

---

## 2) Objetivo

Incorporar un campo explícito `documentType` para separar correctamente:

1. Tipo de persona (natural/jurídica).
2. Tipo de documento (cédula/RUC/pasaporte).
3. Número de documento (`personId`).

Esto permite extender reglas de validación sin romper la lógica ya desplegada.

---

## 3) Requerimiento funcional

### 3.1 Nuevo campo obligatorio en persona

Agregar en entidad `persona`:

- `documentType`: `cedula | ruc | pasaporte`

### 3.2 Reglas de consistencia entre tipo de persona y tipo de documento

Reglas propuestas:

1. Si `type = 1` (natural), se permite `cedula` o `pasaporte`.
2. Si `type = 2` (jurídica), se permite solo `ruc`.
3. Si llega combinación inválida, responder error de validación (`400`) con mensaje claro.

### 3.3 Reglas mínimas de validación de número de documento (`personId`)

1. `cedula`: 10 dígitos numéricos.
2. `ruc`: 13 dígitos numéricos y validaciones vigentes de RUC.
3. `pasaporte`: alfanumérico, longitud entre 6 y 20 (normalizado a mayúsculas, sin espacios al inicio/fin).

> Nota: estas reglas pueden ajustarse según normativa interna, pero se recomienda establecerlas en backend como fuente de verdad.

### 3.4 Impacto en entidad cliente

Sí existe impacto en `cliente` cuando ese modelo almacena datos documentales
propios (por ejemplo `num_document`) o cuando sus respuestas son usadas por
pantallas administrativas/reportes sin expandir la relación de persona.

Decisión recomendada:

1. Mantener `persona` como fuente de verdad de identidad (`personId`, `documentType`, `type`).
2. En `cliente`, evitar duplicar la lógica de validación documental.
3. Si `cliente` mantiene campos denormalizados por compatibilidad (ej. `num_document`), sincronizarlos desde `persona`.

Regla de consistencia sugerida:

1. Si existe `cliente.personId` (ObjectId/ref), su documento efectivo proviene de `persona`.
2. Si existe `cliente.num_document` por legado, debe coincidir con `persona.personId`.
3. Para nuevas integraciones, priorizar lectura desde `persona` o exponer campos normalizados en la respuesta de cliente.

---

## 4) Contrato API propuesto

## 4.1 Crear persona

**POST** `/personas`

Request:

```json
{
  "personId": "A1234567",
  "documentType": "pasaporte",
  "type": 1,
  "name": "Juan",
  "lastName": "Perez",
  "businessName": "",
  "email": "juan@mail.com",
  "address": "Quito",
  "telephoneNumber": "0999999999"
}
```

Response exitosa:

```json
{
  "response": true,
  "data": {
    "_id": "...",
    "personId": "A1234567",
    "documentType": "pasaporte",
    "type": 1
  }
}
```

## 4.2 Actualizar persona

**PUT** `/personas/:id`

- Debe aceptar y persistir `documentType`.
- Debe revalidar consistencia `type + documentType + personId`.

## 4.3 Filtrar/buscar personas

**POST** `/personas/filter`

Compatibilidad requerida:

1. Mantener filtros actuales por `personId`.
2. Permitir filtro opcional por `documentType` para evitar ambigüedad futura.

Ejemplo:

```json
{
  "personId": "A1234567",
  "documentType": "pasaporte"
}
```

## 4.4 Respuesta de clientes (compatibilidad)

Si el endpoint de clientes devuelve campos documentales directos, incluir
también el tipo de documento para no romper vistas que no expanden persona.

Opciones válidas:

1. Expandir `personId` como objeto e incluir `personId.documentType`.
2. Exponer campo plano en cliente (ej. `documentType`) sincronizado con persona.

Ejemplo recomendado de salida:

```json
{
  "_id": "client_001",
  "name": "Juan",
  "personId": {
    "_id": "person_001",
    "personId": "A1234567",
    "documentType": "pasaporte",
    "type": 1
  },
  "num_document": "A1234567"
}
```

---

## 5) Retrocompatibilidad (no romper lógica actual)

Para evitar ruptura en registros existentes y en pantallas que aún no envían `documentType`:

1. En create/update, si no llega `documentType`, inferir temporalmente:
   - `type = 1` => `documentType = cedula`
   - `type = 2` => `documentType = ruc`
2. Responder siempre `documentType` en payload de salida.
3. Marcar la inferencia como comportamiento de transición y planificar deprecación.

---

## 6) Migración de datos

Crear script de migración para registros históricos en colección de personas:

1. Si `documentType` no existe y `type = 1` => set `documentType = cedula`.
2. Si `documentType` no existe y `type = 2` => set `documentType = ruc`.
3. Registrar conteos de documentos migrados y excepciones.

---

## 7) Errores esperados (estándar)

Cuando falle validación, retornar `400` con estructura consistente:

```json
{
  "response": false,
  "message": "documentType no válido para el tipo de persona",
  "errorCode": "PERSON_DOCUMENT_TYPE_INVALID"
}
```

Códigos sugeridos:

1. `PERSON_DOCUMENT_TYPE_REQUIRED`
2. `PERSON_DOCUMENT_TYPE_INVALID`
3. `PERSON_DOCUMENT_NUMBER_INVALID`
4. `PERSON_TYPE_DOCUMENT_MISMATCH`

---

## 8) Criterios de aceptación

1. Backend acepta y persiste `documentType` en create/update de persona.
2. Backend valida combinaciones `type + documentType`.
3. Backend valida formato de `personId` según `documentType`.
4. Endpoints devuelven `documentType` en respuestas de persona.
5. Registros antiguos quedan con `documentType` poblado por migración.
6. Flujo actual sin `documentType` sigue funcionando durante transición.
7. Endpoints de clientes mantienen consistencia documental con persona (sin divergencias entre `num_document` y `personId`).

---

## 9) Estrategia recomendada de implementación (por fases)

### Fase 1 — Backend compatible

1. Agregar campo `documentType` en schema de personas.
2. Implementar validaciones y fallback por inferencia.
3. Desplegar sin exigir cambios inmediatos de frontend.

### Fase 2 — Frontend ajustado

1. Exponer selector de documento en registro cliente.
2. Enviar `documentType` explícito en payload de persona.
3. Ajustar validaciones locales por tipo de documento.

### Fase 3 — Endurecimiento

1. Ejecutar migración completa.
2. Monitorear tráfico y errores.
3. Remover inferencia y volver `documentType` estrictamente requerido.

---

## 10) Riesgos y mitigación

1. Riesgo: duplicados por tratar mismo `personId` con distinto tipo.
   Mitigación: definir unicidad compuesta recomendada (`personId + documentType`).

2. Riesgo: clientes existentes sin `documentType`.
   Mitigación: fallback + migración por lotes + métricas.

3. Riesgo: regresión en checkout por datos históricos.
   Mitigación: respuesta de API siempre normalizada incluyendo `documentType`.

4. Riesgo: divergencia entre datos de persona y cliente por duplicación documental.
  Mitigación: definir una sola fuente de verdad (`persona`) y sincronización controlada de campos legado en cliente.

---

## 11) Decisión de diseño recomendada

Mantener separados:

1. `type` (natural/jurídica) para lógica tributaria/comercial.
2. `documentType` (cédula/RUC/pasaporte) para validación de identidad.

No reemplazar `type` por `documentType`; agregarlo como dimensión adicional.
Eso minimiza impacto y preserva el comportamiento actual.

Adicionalmente, tratar a `persona` como fuente de verdad y a `cliente` como
consumidor de esa identidad (con denormalización solo por compatibilidad).
