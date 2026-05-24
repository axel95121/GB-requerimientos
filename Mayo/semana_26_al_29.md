# Requerimientos Backend

**Fecha:** 23 de mayo de 2026  
**Solicitado por:** Frontend (App Delivery)  
**Dirigido a:** Backend  

---

## Contexto

El módulo de reportes de ventas de la app permite al usuario filtrar órdenes por **fecha de inicio** y **fecha fin**. Actualmente, este filtrado se realiza completamente **en el cliente (frontend)**: el endpoint `POST /orders/filter` devuelve el **histórico completo** de órdenes de la sucursal y la app descarta las que quedan fuera del rango seleccionado.

Esto genera varios problemas:

- **Rendimiento degradado:** se transfieren y procesan órdenes que no serán mostradas.
- **Inconsistencia en totales:** si el usuario no aplica un filtro de fechas, el reporte incluye toda la historia, lo que distorsiona los números del período actual.
- **Falta de confiabilidad:** en organizaciones con muchas órdenes, el volumen de datos puede causar timeouts o lentitud perceptible.

Adicional a esto, se requiere de un endpoint para aplicar un reinicio completo de usuario. Esto incluiría toda la información generada por este usuario.

---

## Requerimiento #1: Filtro de órdenes por fechas

Ampliar el endpoint `POST /orders/filter` para que acepte los campos `startDate` y `endDate` como parámetros opcionales de filtrado en el body. Cuando se reciban, el backend debe retornar **únicamente** las órdenes cuya fecha (`date`) se encuentre dentro del rango indicado (inclusive en ambos extremos).

**Método:** `POST`  
**Ruta:** `/orders/filter`  
**Autenticación:** Requerida (token del usuario)

---

## Cambio en el body del request

Los campos nuevos son **opcionales**. Todos los filtros existentes deben seguir funcionando igual si no se envían `startDate`/`endDate`.

### Ejemplo — sin filtro de fechas (comportamiento actual, sin cambios)

```json
{
  "sucursal._id": "abc123",
  "dataStatus": 1
}
```

### Ejemplo — con filtro de rango de fechas (nuevo comportamiento)

```json
{
  "sucursal._id": "abc123",
  "dataStatus": 1,
  "startDate": "2026-05-01T00:00:00.000Z",
  "endDate": "2026-05-23T23:59:59.999Z"
}
```

**Reglas:**

| Caso | Comportamiento esperado |
|---|---|
| Se envían `startDate` y `endDate` | Retornar órdenes con `date >= startDate` AND `date <= endDate` |
| Se envía solo `startDate` | Retornar órdenes con `date >= startDate` |
| Se envía solo `endDate` | Retornar órdenes con `date <= endDate` |
| No se envía ninguno | Comportamiento actual sin restricción de fecha |

---

## Formato de fechas

- Las fechas deben ser cadenas en formato **ISO 8601 UTC** (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- El frontend enviará:
  - `startDate`: inicio del día seleccionado a las `00:00:00.000Z`.
  - `endDate`: fin del día seleccionado a las `23:59:59.999Z`.

---

## Requerimiento #2: API de reinicio

Para la semana de pruebas se requiere que tengamos un API que resetee/elimine toda la información generada por un usuario. Esto incluye órdenes, proformas, facturas, datos sobre la organización y demás.