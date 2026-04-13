# Requerimientos Backend — Pagos a Plazos, Notificaciones y Revisión de Ítems

**Fecha:** 13 de abril de 2026  
**Solicitado por:** Frontend (App Delivery)  
**Dirigido a:** Equipo Backend

---

## 1. Notificación por correo electrónico al cambiar el estado de una cotización/proforma

### Contexto

Las cotizaciones (endpoint `/cotizaciones`) poseen un campo `status` que indica su estado actual:

| Valor de `status` | Significado               |
|--------------------|---------------------------|
| `1`                | Solicitud (pendiente)     |
| `2`                | Proforma vigente / consolidada |
| `0`                | Rechazada                 |

Adicionalmente existe un campo `statusMessage` de tipo `String` que contiene un mensaje descriptivo asociado al cambio de estado (por ejemplo: motivo de rechazo, observaciones, detalle adicional, etc.).

### Requerimiento

Cuando el backend detecte un **update** en el campo `status` de un registro de cotización, debe:

1. **Enviar un correo electrónico al cliente** asociado a dicha cotización, notificándole sobre el cambio de estado.
2. El correo debe incluir:
   - **Datos del cliente** (nombre, correo, etc.) que ya existen en el registro de la cotización.
   - **Estado nuevo** de la cotización (en texto legible, ej. "Su proforma ha sido aprobada", "Su solicitud ha sido rechazada", etc.).
   - **Detalle adicional (`statusMessage`)**: incluir el texto contenido en el campo `statusMessage` como parte del cuerpo del correo. Este campo será actualizado siempre desde el frontend antes de realizar el cambio de estado, por lo que nunca debería llegar vacío. Sin embargo, si por alguna razón llega vacío, se sugiere omitir esa sección o poner un texto genérico.

### Campos relevantes del registro de cotización

```json
{
  "status": 0 | 1 | 2,
  "statusMessage": "Texto descriptivo del cambio / motivo",
  "isOrder": true | false,
  "validDate": "2026-04-20T00:00:00.000Z",
  "date": "2026-04-13T00:00:00.000Z",
  "expirationDays": 30,
  // ... datos del cliente incluidos en el registro
}
```

### Endpoint de referencia

- **Base URL (producción):** `https://api-prod-general.gb97.ec`
- **Recurso:** `/cotizaciones`
- **Filtro:** `/cotizaciones/filter` (POST)

### Notas

- El Front End se compromete a **siempre actualizar el campo `statusMessage`** al momento de cambiar el `status`, para evitar falsos positivos o correos sin contexto.
- La notificación debe dispararse **únicamente** cuando exista un cambio real en el campo `status` (no en cada PUT/PATCH general del registro).

---

## 2. Creación del campo `installmentPlan` en el modelo de órdenes

### Contexto

Desde el frontend se contempla la funcionalidad de **pagos a plazos** para órdenes, pero actualmente el backend **no cuenta con el campo `installmentPlan`** en el modelo/esquema de órdenes, por lo que no es posible persistir ni enviar esta información.

### Requerimiento

Necesitamos que se agregue un nuevo campo `installmentPlan` al modelo de órdenes en el backend. Sugerimos que sea de tipo **`Object` (documento embebido / JSON libre)** para que desde el frontend podamos definir y evolucionar su estructura interna de forma iterativa sin requerir cambios en el backend por cada ajuste.

### Tipo sugerido

```
installmentPlan: Object | null    (por defecto: null)
```

- Cuando la orden **no** utilice pagos a plazos, el campo será `null` o no estará presente.
- Cuando la orden **sí** utilice pagos a plazos, el frontend enviará un objeto JSON con las propiedades del plan.

### Estructura inicial de referencia (manejada desde frontend)

A continuación se muestra la estructura que el frontend enviará inicialmente dentro de este campo. **No es necesario que el backend valide ni interprete estas propiedades internas por ahora**, solo que las persista y las devuelva tal cual al consultarlas:

```json
{
  "installmentPlan": {
    "installmentCount": 3,
    "frequency": "monthly",
    "firstPaymentDate": "2026-05-13T00:00:00.000Z",
    "downPayment": 100.50,
    "installmentAmount": 125.25,
    "totalAmount": 476.25,
    "installments": [
      {
        "number": 1,
        "amount": 125.25,
        "dueDate": "2026-05-13T00:00:00.000Z",
        "status": "pending",
        "paidDate": null,
        "paidAmount": null
      }
    ]
  }
}
```

### Acciones requeridas

1. **Agregar el campo `installmentPlan`** (tipo `Object`, nullable) al esquema/modelo de órdenes.
2. **Aceptar el campo** en los endpoints de creación (`POST`) y actualización (`PUT`/`PATCH`) de órdenes.
3. **Devolver el campo** en las respuestas de consulta de órdenes (GET, GET by ID, filter).
4. No es necesario crear endpoints adicionales ni validaciones internas sobre la estructura del objeto por ahora.

### Notas

- Una vez que la funcionalidad esté estable y estandarizada desde el frontend, coordinaremos con el backend para definir un esquema formal, validaciones y los requerimientos de notificaciones asociados (ver punto 3).
- Este campo es **prerequisito** para el sistema de recordatorios descrito en la sección siguiente.

---

## 3. Notificaciones de recordatorio para pagos a plazos (cuotas)

### Contexto

Una vez implementado el campo `installmentPlan` (punto 2), el sistema soportará **pagos a plazos** para órdenes. La estructura del plan de pagos contendrá la información necesaria para gestionar recordatorios automáticos.

### Estructura del plan de pagos (referencia)

```json
{
  "installmentPlan": {
    "installmentCount": 3,
    "frequency": "monthly",
    "firstPaymentDate": "2026-05-13T00:00:00.000Z",
    "downPayment": 100.50,
    "installmentAmount": 125.25,
    "totalAmount": 476.25,
    "installments": [
      {
        "number": 1,
        "amount": 125.25,
        "dueDate": "2026-05-13T00:00:00.000Z",
        "status": "pending",
        "paidDate": null,
        "paidAmount": null
      },
      {
        "number": 2,
        "amount": 125.25,
        "dueDate": "2026-06-13T00:00:00.000Z",
        "status": "pending",
        "paidDate": null,
        "paidAmount": null
      },
      {
        "number": 3,
        "amount": 125.25,
        "dueDate": "2026-07-13T00:00:00.000Z",
        "status": "pending",
        "paidDate": null,
        "paidAmount": null
      }
    ]
  }
}
```

### Frecuencias soportadas

| Valor de `frequency` | Descripción |
|------------------------|-------------|
| `weekly`              | Semanal     |
| `biweekly`            | Quincenal (cada 15 días) |
| `monthly`             | Mensual     |

### Estados de cada cuota (`installments[].status`)

| Valor     | Significado |
|-----------|-------------|
| `pending` | Pendiente de pago |
| `paid`    | Pagada      |
| `overdue` | Vencida     |

### Requerimiento

El backend debe implementar un **sistema de recordatorios automáticos** que notifique al usuario deudor sobre sus próximas cuotas mediante:

1. **Notificación push** (FCM o el servicio que se maneje).
2. **Correo electrónico** al usuario asociado a la orden.

> **Dependencia:** Este requerimiento depende de la implementación del punto 2 (creación del campo `installmentPlan`).

### Casos y reglas de notificación sugeridos

#### A) Recordatorio anticipado (antes del vencimiento)

| Frecuencia    | Cuándo notificar                          | Ejemplo                                                  |
|---------------|-------------------------------------------|----------------------------------------------------------|
| `monthly`     | **7 días antes** de la fecha de vencimiento | Cuota vence el 13/05 → notificar el 06/05               |
| `biweekly`    | **3 días antes** de la fecha de vencimiento | Cuota vence el 28/04 → notificar el 25/04               |
| `weekly`      | **1 día antes** de la fecha de vencimiento  | Cuota vence el 20/04 → notificar el 19/04               |

**Mensaje sugerido:**  
> "Recordatorio: Su cuota #[número] por $[monto] vence el [fecha]. Por favor, realice su pago a tiempo."

#### B) Notificación el mismo día de vencimiento

Si el día de vencimiento (`dueDate`) llega y la cuota **aún tiene `status: "pending"`** (es decir, no se ha registrado pago):

- Enviar notificación push + correo indicando que **la cuota vence hoy**.

**Mensaje sugerido:**  
> "Su cuota #[número] por $[monto] vence hoy [fecha]. Si ya realizó el pago, puede ignorar este mensaje."

#### C) Notificación de cuota vencida (post-vencimiento)

Si la fecha de vencimiento ya pasó y la cuota sigue en `status: "pending"`:

- Marcar automáticamente la cuota como `overdue`.
- Enviar notificación push + correo indicando que la cuota está **vencida**.
- **Sugerencia:** repetir la notificación cada cierto intervalo (ej. cada 3 días) mientras permanezca impaga, hasta un máximo razonable.

**Mensaje sugerido:**  
> "Aviso: Su cuota #[número] por $[monto] está vencida desde el [fecha]. Por favor, regularice su pago a la brevedad."

#### D) Consideraciones adicionales

- **Cuota inicial (enganche/`downPayment`):** Si existe un `downPayment > 0`, este ya se registra como pago al momento de crear la orden, por lo que no requiere recordatorio.
- **Verificación de pago:** Antes de enviar cualquier notificación, el backend debe verificar en la orden si la cuota ya fue pagada (`status: "paid"` o si `paidDate` no es null).
- **Horario de envío:** Idealmente enviar las notificaciones en horario laboral (ej. entre 8:00 AM y 6:00 PM hora local del negocio).
- **No duplicar notificaciones:** Si ya se envió un recordatorio anticipado para una cuota específica, no volver a enviar el mismo tipo de recordatorio.

### Datos del usuario deudor

Los datos del usuario (nombre, correo, device token para push) se obtienen del registro de la orden y/o del usuario asociado a la misma.

---

## 4. Revisión del endpoint de ítems — Tab mostrando resultados vacíos

### Problema

En la pantalla de creación de órdenes (pestaña/tab de ítems), los ítems se muestran vacíos con el mensaje **"No existen registros"**, a pesar de que deberían existir ítems registrados.

### Endpoint

- **URL:** `POST https://api-prod-general.gb97.ec/items-variant/filter`

### Headers

```json
{
  "Content-Type": "application/json",
  "Authorization": "<token del usuario>",
  "Accept-Language": "es"
}
```

### Body actual que se está enviando

**Con filtro de grupo:**
```json
{
  "sucursal._id": "<branch-uuid>",
  "group._id": "<group-id>",
  "dataStatus": 1
}
```

**Sin filtro de grupo:**
```json
{
  "sucursal._id": "<branch-uuid>",
  "dataStatus": 1
}
```

### Solicitud al backend

1. **Verificar** que el endpoint `POST /items-variant/filter` responda correctamente cuando recibe el body con filtros (`sucursal._id`, `group._id`, `dataStatus`).
2. **Confirmar** la estructura esperada del body para este endpoint y que funcione correctamente.
3. Si hubo un cambio en la estructura del filtro o en el nombre de los campos, por favor **informar los campos correctos** para actualizar el frontend.

---

## Resumen de acciones requeridas

| #  | Tema                              | Tipo de acción        | Prioridad | Dependencia |
|----|-----------------------------------|-----------------------|-----------|-------------|
| 1  | Notificación por correo al cambiar status de cotización | Nuevo desarrollo | Alta | — |
| 2  | Creación del campo `installmentPlan` en órdenes         | Nuevo desarrollo | Alta | — |
| 3  | Recordatorios de pago a plazos (push + correo)          | Nuevo desarrollo | Alta | Punto 2 |
| 4  | Revisión endpoint `/items-variant/filter`                | Bugfix / Revisión | Alta | — |

