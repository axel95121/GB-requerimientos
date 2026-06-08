# Requerimientos Backend — Semana 8 al 12 de junio 2026
**Origen:** Integración de pasarela de pagos Payphone  
**Prioridad:** Alta — funcionalidad de cobro requerida por flujo de negocio  
**Formato:** Acción concreta que debe tomar el backend, impacto si no se resuelve y contrato esperado con el frontend.

---

## 🟡 PENDIENTE — Implementar esta semana

### BE-PAY-01 — Endpoint para generación de link de pago vía Payphone API Link
**Origen:** [Documentación oficial Payphone API Link](https://docs.payphone.app/api-link)  
**Prioridad:** Alta — el frontend no puede iniciar el flujo de cobro sin este endpoint

**Contexto:**  
El backend ya tiene las credenciales de Payphone configuradas (`token` y `storeId`) y la guía de Datafast implementada. Este requerimiento es el paso natural siguiente: exponer un endpoint propio que internamente llame a la API Link de Payphone y devuelva al frontend el link de cobro generado.

**Acción requerida (BE):**  
Crear un endpoint que reciba los datos del pago, llame a la API Link de Payphone (`POST https://pay.payphonetodoesposible.com/api/Links`) con las credenciales ya almacenadas en el servidor, y retorne el link resultante.

```
POST /pagos/generar-link
Authorization: Bearer <token_usuario_app>
Content-Type: application/json

Body:
{
  "amount": 1150,               // Total en centavos (ej: $11.50 → 1150). REQUERIDO
  "amountWithTax": 1000,        // Subtotal gravado en centavos. Opcional si se usa amountWithoutTax
  "tax": 150,                   // IVA en centavos. Opcional
  "amountWithoutTax": 0,        // Subtotal sin impuesto. Opcional
  "currency": "USD",            // Siempre "USD". REQUERIDO
  "reference": "string",        // Descripción del pago (máx. 100 chars). Opcional
  "clientTransactionId": "string", // ID único por transacción (máx. 15 chars). REQUERIDO — generarlo en BE
  "oneTime": true               // true = link de un solo uso. Opcional (recomendado true)
}
```

> **Nota sobre `clientTransactionId`:** El backend debe generarlo internamente (no recibirlo del frontend) para garantizar unicidad. Se puede construir con fecha-hora más un sufijo aleatorio truncado a 15 caracteres, por ejemplo: `"260612-143022A1"`.

> **Nota sobre montos:** Todos los valores monetarios van en **centavos enteros** (multiplicar dólares × 100). El campo `amount` debe ser igual a la suma de `amountWithTax + amountWithoutTax + tax`. Al menos uno de los campos de desglose debe estar presente.

**Llamada interna que hará el backend a Payphone:**

```http
POST https://pay.payphonetodoesposible.com/api/Links
Authorization: Bearer <TOKEN_PAYPHONE_ALMACENADO_EN_ENV>
Content-Type: application/json

{
  "amount": 1150,
  "amountWithTax": 1000,
  "tax": 150,
  "currency": "USD",
  "reference": "Pago de suscripción",
  "clientTransactionId": "260612-143022A1",
  "storeId": "<STORE_ID_ALMACENADO_EN_ENV>",
  "oneTime": true
}
```

**Respuesta esperada de Payphone (éxito):**  
Un string con la URL del formulario de pago:
```
"https://payp.page.link/aYu55"
```

**Respuesta que el backend debe devolver al frontend:**

```json
// 200 OK
{
  "paymentUrl": "https://payp.page.link/aYu55",
  "clientTransactionId": "260612-143022A1"
}
```

```json
// 400 Bad Request — si Payphone rechaza la solicitud
{
  "message": "No se pudo generar el link de pago",
  "detail": "<mensaje de error de Payphone>"
}
```

**Consideraciones de seguridad e implementación:**
- El `token` y `storeId` de Payphone deben leerse desde variables de entorno (ya configuradas), nunca hardcodeados.
- El `clientTransactionId` debe guardarse en la BD vinculado al registro del pago para poder rastrearlo y, si aplica, hacer reversos el mismo día (hasta las 20:00).
- El link **no debe embeberse en un iframe**; el frontend debe abrirlo en una pestaña nueva del navegador (`target="_blank"`).
- El endpoint tiene un límite de 30 solicitudes POST/minuto por parte de Payphone; no se requiere manejo especial por ahora, pero documentarlo.
- Una vez generado el link, Payphone no envía respuesta de vuelta al sistema al completarse el pago (a menos que se configure webhook de Notificación Externa, lo cual queda fuera del alcance de esta semana).

**Criterio de aceptación:**  
El frontend puede llamar a `POST /pagos/generar-link` y recibir una URL válida que, al abrirse en el navegador, muestre el formulario de pago de Payphone con el monto correcto.
