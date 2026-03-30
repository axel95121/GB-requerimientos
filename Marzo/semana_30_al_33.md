
# Requerimientos BACKEND

**Semana:** 30 al 33 de Marzo  
**Proyecto:** Marketplace | Order GB97

---

## 1. Listado de Productos (PÚBLICO — sin Authorization)

```
GET /items-variant?page=1&limit=20
```

**Query params:**
| Param | Tipo | Requerido | Ejemplo |
|---|---|---|---|
| `page` | int | sí | `1` |
| `limit` | int | sí | `20` |
| `sortBy` | string | no | `price_asc`, `price_desc`, `name_asc`, `name_desc` |

**Headers:**
```
Content-Type: application/json
Accept-Language: es
```

**NO envía `Authorization`.** Este endpoint debe responder sin token.

**Response esperada:**
```json
{
  "response": true,
  "data": [
    {
      "_id": "abc123",
      "name": "Producto X",
      "description": "...",
      "basePrice": 25.50,
      "discount": 10,
      "stock": 50,
      "itemImages": ["https://...img1.png", "https://...img2.png"],
      "imgUrl": "https://...fallback.png",
      "group": { "name": "Camisetas" },
      "subGroup": { "name": "Manga larga" },
      "collections": [
        { "collectionId": "featured" },
        { "collectionId": "bestseller" }
      ],
      "createdAt": "2026-01-15T...",
      "updatedAt": "2026-03-20T..."
    }
  ],
  "pagination": {
    "totalPages": 5,
    "currentPage": 1,
    "totalItems": 98
  }
}
```

**Notas:**
- `stock: null` → el frontend lo interpreta como stock ilimitado (999999)
- `stock: 0` → sin stock
- `discount: 0` o `null` → sin descuento
- `discount: 10` → el frontend calcula `discountPrice = basePrice * (1 - discount/100)`
- `itemImages` es `List<String>` directo, no objetos

---

## 2. Filtrado de Productos (PÚBLICO — sin Authorization)

```
POST /items-variant/filter?page=1&limit=20&sortBy=price_asc
```

**Query params:** mismos que el GET anterior.

**Headers:**
```
Content-Type: application/json
Accept-Language: es
Authorization: <token>   ← OPCIONAL, envía si existe pero NO debe ser requerido
```

**Body — combinaciones posibles (todas opcionales):**

```json
{
  "group.name": "Camisetas",
  "basePrice": { "$gte": 10.0, "$lte": 50.0 },
  "stock": { "$gt": 0 },
  "discount": { "$gt": 0 },
  "name": "polo"
}
```

**Detalle de cada campo del body:**

| Campo | Operadores | Comportamiento esperado |
|---|---|---|
| `group.name` | string exacto | Filtrar por categoría. Valor = nombre del grupo |
| `basePrice` | `$gte`, `$lte` | Rango de precio. Pueden venir ambos o solo uno |
| `stock` | `$gt: 0` | Solo productos con stock > 0 |
| `stock` | `$gt: 0, $lt: 5` | Productos con bajo stock (1-4 unidades) |
| `discount` | `$gt: 0` | Solo productos en oferta (cualquier descuento) |
| `discount` | `$gte: N` | Productos con descuento mínimo de N% (N = 10, 20, 30, 50) |
| `name` | string | Búsqueda parcial por nombre. El backend debe hacer match parcial (regex o `$regex`) |

**Comportamiento:**
- Si el body está vacío `{}`, debe devolver todos los productos (igual que GET)
- Múltiples campos en el body se combinan con AND
- `sortBy` viene como query param, no en el body
- La paginación (`page`, `limit`) viene como query params

**Response:** misma estructura que el GET (sección 1).

---

## 3. Categorías / Grupos (PÚBLICO — sin Authorization)

```
GET /grupos
```

**El frontend necesita:** lista de grupos para mostrar como categorías seleccionables.

**Response esperada:**
```json
{
  "response": true,
  "data": [
    { "_id": "...", "name": "Camisetas" },
    { "_id": "...", "name": "Pantalones" }
  ]
}
```

**Actualmente el frontend extrae las categorías de los productos cargados.** Si este endpoint requiere token, el frontend no puede cargar categorías para usuarios no logueados. Debe ser público.

## 4. Requerimiento: `sortBy` en query param

Valores que el frontend envía en `sortBy`:

| Valor | Ordenamiento esperado |
|---|---|
| `price_asc` | `basePrice` ascendente |
| `price_desc` | `basePrice` descendente |
| `name_asc` | `name` alfabético A-Z |
| `name_desc` | `name` alfabético Z-A |
| *(vacío o ausente)* | Orden por defecto del backend |

Aplica tanto para `GET /items-variant` como para `POST /items-variant/filter`.

---

## 5. Paginación — Contrato de respuesta

El frontend lee `response.pagination` para continuar el scroll infinito.

```json
{
  "pagination": {
    "totalPages": 5,
    "currentPage": 1,
    "totalItems": 98
  }
}
```

Si `pagination` no viene en la respuesta o es `null`, el frontend asume que no hay más páginas y deja de paginar. **La ausencia de este campo invalida también el caché local.**

---

## 6. Búsqueda por `name`

Cuando el body incluye `"name": "polo"`, el backend debe hacer **búsqueda parcial** (case-insensitive). El frontend NO envía operadores regex; envía el string directo y espera que el backend lo resuelva internamente (ej. `{ $regex: "polo", $options: "i" }` en Mongo).

---

## 7. Requerimientos Facturación

**Habilitar modo DEV:** Para pruebas de Contabilidad

**Listado de modalidades de pago:** EFECTIVO, CHEQUE, TRANSFERENCIA, TARJETA DE CRÉDITO, OTROS CON UTILIZACIÓN DEL SISTEMA FINANCIERO, SIN UTILIZACIÓN DEL SISTEMA FINANCIERO.

**Habilitar array de informacion adicional:** Actualmente sólo existe un campo, y en la práctica el contador de GB97 nos indica que normalmente en los sistemas de facturación se puede habilitar más de uno.