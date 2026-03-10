
# Requerimiento: API de Favoritos — Proyecto Marketplace

**Semana:** 9 al 13 de Marzo  
**Proyecto:** GB97 — Marketplace  
**Tipo:** Backend

---

## 1. Descripción General

Se requiere desarrollar una API REST en el backend que permita a los usuarios guardar, consultar y eliminar productos de su lista de favoritos dentro del Proyecto Marketplace.

---

## 2. Modelo de Datos

### Colección / Tabla: `favorites`

| Campo        | Tipo       | Descripción                                      | Requerido |
|--------------|------------|--------------------------------------------------|-----------|
| `_id`        | ObjectId   | Identificador único del registro de favorito     | Auto      |
| `userId`     | ObjectId   | Referencia al ID del usuario                     | ✅        |
| `itemId`     | ObjectId   | Referencia al ID del producto/item               | ✅        |
| `createdAt`  | Date       | Fecha en que se guardó el favorito               | Auto      |

> **Nota:** La combinación de `userId` + `itemId` debe ser única (un usuario no puede guardar el mismo producto dos veces).

---

## 3. Endpoints Requeridos

### 3.1 Guardar un Favorito

- **Método:** `POST`
- **URL:** `/api/favorites`
- **Descripción:** Guarda un producto en la lista de favoritos del usuario.

**Body (JSON):**
```json
{
  "userId": "<ID del usuario>",
  "itemId": "<ID del producto>"
}
```

**Respuesta exitosa (201):**
```json
{
  "message": "Favorito guardado exitosamente",
  "data": {
    "_id": "<ID del favorito>",
    "userId": "<ID del usuario>",
    "itemId": "<ID del producto>",
    "createdAt": "2025-03-10T00:00:00.000Z"
  }
}
```

**Errores posibles:**
- `400` — Faltan campos requeridos (`userId` o `itemId`)
- `409` — El producto ya está en favoritos del usuario
- `500` — Error interno del servidor

---

### 3.2 Consultar Favoritos de un Usuario (Populados)

- **Método:** `GET`
- **URL:** `/api/favorites/:userId`
- **Descripción:** Retorna la lista de favoritos de un usuario con los datos del usuario y del producto populados.

**Parámetros de ruta:**
- `:userId` — ID del usuario cuyos favoritos se desean consultar

**Respuesta exitosa (200):**
```json
{
  "message": "Favoritos obtenidos exitosamente",
  "data": [
    {
      "_id": "<ID del favorito>",
      "userId": {
        // TODO: Completar con los campos de la referencia de Usuario
        // Ejemplo: "_id", "name", "email", etc.
      },
      "itemId": {
        // TODO: Completar con los campos de la referencia de Item/Producto
        // Ejemplo: "_id", "title", "price", "image", etc.
      },
      "createdAt": "2025-03-10T00:00:00.000Z"
    }
  ]
}
```

**Errores posibles:**
- `400` — `userId` inválido o no proporcionado
- `404` — Usuario no encontrado
- `500` — Error interno del servidor

---

### 3.3 Eliminar un Favorito

- **Método:** `DELETE`
- **URL:** `/api/favorites/:favoriteId`
- **Descripción:** Elimina un registro de favorito por su ID.

**Parámetros de ruta:**
- `:favoriteId` — ID del registro de favorito a eliminar

**Respuesta exitosa (200):**
```json
{
  "message": "Favorito eliminado exitosamente"
}
```

**Errores posibles:**
- `404` — Favorito no encontrado
- `500` — Error interno del servidor

---

## 5. Validaciones Requeridas

- [ ] `userId` debe ser un ID válido y existente en la base de datos.
- [ ] `itemId` debe ser un ID válido y existente en la base de datos.
- [ ] No permitir duplicados: un mismo usuario no puede guardar el mismo item dos veces.
- [ ] Al consultar, siempre retornar los datos populados (no solo los IDs).