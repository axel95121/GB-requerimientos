# Requerimiento: Soporte de Monto Parcial en Pago Deuna

**Fecha:** 23 de marzo de 2026  
**Solicitado por:** Equipo Frontend (App Order GB97)  
**Dirigido a:** Equipo Backend  
**Prioridad:** Alta  
**Endpoint afectado:** `POST /deuna/request-payment`

---

## Contexto

Actualmente, al crear una orden con método de pago **Deuna**, el flujo funciona de la siguiente manera:

1. El usuario selecciona el método de pago "Deuna" en la pantalla de creación de orden.
2. Se habilita un campo de texto donde el usuario **ingresa el monto que desea cobrar** (puede ser parcial, es decir, menor o igual al total de la orden).
3. La orden se crea exitosamente vía `POST /orders` incluyendo:
   - `totalPayed`: el monto ingresado por el usuario.
   - `payment`: array con el detalle del pago, incluyendo `amount` con el monto ingresado.
4. Post-creación, se invoca `POST /deuna/request-payment` para generar el enlace/deeplink de pago en Deuna.

### Problema

El endpoint `POST /deuna/request-payment` actualmente solo recibe:

```json
{
  "id_order": "ORDER_ID"
}
```

El backend toma internamente el **total de la orden** (`total`) como monto a cobrar en Deuna, **ignorando por completo** el monto parcial que el usuario estableció desde el frontend (`totalPayed` / `payment[].amount`).

Esto significa que sin importar qué monto ingrese el usuario en el campo de pago, **Deuna siempre genera un cobro por el 100% del valor de la orden**.

---

## Requerimiento

Necesitamos que el endpoint `POST /deuna/request-payment` acepte un **nuevo campo en el request body** que permita al frontend especificar el monto exacto a cobrar en Deuna.

### Request Body Propuesto

```json
{
  "id_order": "ORDER_ID",
  "amount": 150.50
}
```

| Campo      | Tipo     | Requerido | Descripción                                                                                       |
|------------|----------|-----------|---------------------------------------------------------------------------------------------------|
| `id_order` | `string` | Sí        | ID de la orden (campo existente, sin cambios).                                                    |
| `amount`   | `number` | No*       | Monto a cobrar en Deuna. Si no se envía o es `null`, mantener el comportamiento actual (cobrar el total de la orden). |

> \* Se sugiere que sea opcional para mantener **retrocompatibilidad** con versiones anteriores del frontend que no envíen este campo.

---

## Reglas de Negocio Sugeridas

1. **Validación de rango:** El `amount` debe ser mayor a `0` y menor o igual al `total` de la orden.
2. **Retrocompatibilidad:** Si `amount` no se incluye en el body, el backend debe seguir usando el total de la orden como monto a cobrar (comportamiento actual).
3. **Precisión decimal:** El campo `amount` debe soportar hasta 2 decimales (consistente con el formato monetario de la app).

---

## Cambios en el Frontend (para referencia)

Una vez implementado el cambio en el backend, el frontend ajustará la llamada en `ApiService.requestDeUnaPayment()` de:

```dart
// Actual
final body = jsonEncode({'id_order': ticketId});
```

A:

```dart
// Propuesto
final body = jsonEncode({
  'id_order': ticketId,
  'amount': paymentAmount,  // Monto ingresado por el usuario
});
```

---

## Respuesta Esperada

No se requieren cambios en la estructura de la respuesta. El response actual es suficiente:

```json
{
  "transactionId": "...",
  "deeplink": "...",
  ...
}
```

La única diferencia es que el deeplink/enlace de pago generado por Deuna debería reflejar el `amount` enviado en lugar del total de la orden.

---

## Criterios de Aceptación

- [ ] El endpoint `POST /deuna/request-payment` acepta el campo `amount` en el request body.
- [ ] Cuando se envía `amount`, Deuna genera el cobro por ese monto específico.
- [ ] Cuando **no** se envía `amount`, el comportamiento sigue siendo el actual (cobro por el total de la orden).
- [ ] Se valida que `amount > 0` y `amount <= total de la orden`.
- [ ] Se retorna un error claro si `amount` no cumple las validaciones.
