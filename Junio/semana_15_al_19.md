# Requerimientos Backend — Registro de Clientes/Visitantes con Marketplace

**Fecha:** 15 de junio de 2026  
**Alcance:** Completar el flujo de registro/login de clientes externos (rol `customer`), habilitar el acceso público a catálogos (colecciones) sin token de autorización, y garantizar que el modelo `user` soporte correctamente los campos `name` y `lastName`.  
**Contexto:** El lado del cliente/visitante ya tiene las pantallas construidas en el frontend (`CustomerRegisterScreen`, `CustomerHomeScreen`, `CustomerCatalogsScreen`, `CustomerItemsScreen`). El flujo de navegación libre (sin autenticación para browsing) está implementado, pero depende de tres ajustes en el backend para funcionar correctamente de punta a punta.

> **Convenciones:**
> - 🔴 **Crítico** — bloqueante para el flujo principal
> - 🟡 **Coordinado** — FE + BE deben alinear contrato antes de implementar
> - 🟢 **Independiente** — BE puede completar sin esperar al FE

---

## 🔴 BE-01 — Añadir campos `name` y `lastName` (String) al modelo `user`

### Problema

El frontend envía `name` y `lastName` como `String` al endpoint `POST /user/registro` para el registro de clientes externos. Si el modelo `user` en MongoDB no define estos campos, el backend los descarta silenciosamente o devuelve error de validación, impidiendo el guardado correcto del nombre visible en el perfil del cliente.

En el `CustomerRegisterScreen` del FE, el body del registro es:

```json
{
  "name": "Juan",
  "lastName": "Pérez",
  "email": "juan@example.com",
  "userName": "gb97_a3b9x1z2",
  "password": "...",
  "rol": "6978ca40a8264f82adeb800a"
}
```

### Acción requerida

Actualizar el **schema de Mongoose** (u ORM equivalente) del modelo `user` para incluir:

```js
// Schema del modelo User — campos a añadir
name: {
  type: String,
  required: false,
  trim: true,
  default: '',
},
lastName: {
  type: String,
  required: false,
  trim: true,
  default: '',
},
```

> **Nota:** Ambos campos deben ser **opcionales** (`required: false`) para no romper registros de usuarios existentes (organizaciones, administradores, etc.) que no envían estos campos.

### Respuesta esperada tras registro exitoso

```json
{
  "response": true,
  "message": "Usuario registrado exitosamente",
  "data": {
    "_id": "...",
    "name": "Juan",
    "lastName": "Pérez",
    "email": "juan@example.com",
    "userName": "gb97_a3b9x1z2",
    "rol": { "_id": "6978ca40a8264f82adeb800a", "name": "customer" }
  }
}
```

### Respuesta esperada en login (JWT payload / respuesta del `/user/login`)

El objeto `user` devuelto en el token o en el body del login debe incluir `name` y `lastName` para que el perfil del cliente los muestre correctamente:

```json
{
  "response": true,
  "token": "<jwt>",
  "user": {
    "_id": "...",
    "name": "Juan",
    "lastName": "Pérez",
    "email": "juan@example.com",
    "userName": "gb97_a3b9x1z2",
    "rol": { ... }
  }
}
```

### Impacto en el FE

- `CustomerProfileScreen` usa `SharedService.userName` para mostrar el nombre del cliente en la vista de perfil autenticada. Este campo se popula desde la respuesta del login.
- No se requiere cambio en el FE; solo necesita que el backend persista y devuelva los campos.

---

## 🔴 BE-02 — Quitar autorización de `GET /collection` y `POST /collection/filter` (catálogos)

### Problema

La pantalla `CustomerCatalogsScreen` necesita dos operaciones para funcionar correctamente sin sesión:

1. **`POST /collection/filter`** — usado por `ApiService.getAllCollections()` para listar catálogos disponibles al visitante, con posibilidad de filtrar por negocio (`sucursal._id`).
2. **`GET /collection`** — usado para obtener el listado general de colecciones.

Ambos endpoints envían el token almacenado en SharedPreferences, que para un visitante no autenticado es una **cadena vacía** (`""`). Si el middleware rechaza tokens vacíos con `401 Unauthorized`, el visitante **nunca podrá ver los catálogos**.

```dart
// Código actual en api_service.dart — getAllCollections()
final String token = prefs.getString('token') ?? '';  // ← vacío para visitantes

final Map<String, String> requestHeaders = <String, String>{
  'Content-Type': 'application/json',
  'Authorization': token,  // ← enviado vacío
};
```

### Acción requerida

**Liberar de autenticación** ambos endpoints: `GET /collection` y `POST /collection/filter`. Las opciones son (en orden de preferencia):

**Opción A (recomendada):** Mover los endpoints a rutas públicas en el router:
```js
// Antes (rutas protegidas):
router.get('/collection', authMiddleware, collectionController.getAll);
router.post('/collection/filter', authMiddleware, collectionController.filter);

// Después (rutas públicas):
router.get('/collection', collectionController.getAll);
router.post('/collection/filter', collectionController.filter);
```

**Opción B:** Hacer el middleware de auth condicional — si no hay token, continúa sin autenticar (en lugar de rechazar):
```js
// authMiddleware actualizado (modo permisivo para endpoints públicos):
if (!token || token === '') {
  req.user = null;  // usuario anónimo
  return next();
}
// ... verificar JWT normalmente
```

> Se recomienda la **Opción A** por ser la más simple y coherente con el modelo de datos (los catálogos son información pública del negocio, similar a un menú o vitrina).

### Contratos esperados

```
GET /collection
Headers: Authorization: (opcional)

POST /collection/filter
Headers: Content-Type: application/json
         Authorization: (opcional — vacío o ausente para visitantes)
Body: { "sucursal._id": "" }        ← vacío = todas las colecciones públicas
      { "sucursal._id": "<id>" }    ← filtrar por sucursal específica
      { "organization._id": "<id>" } ← filtrar por negocio/organización

Respuesta 200:
{
  "response": true,
  "data": [
    {
      "_id": "...",
      "collectionName": "Catálogo Verano 2026",
      "collectionImage": "https://...",
      "description": "...",
      "sucursal": { "_id": "...", "nombre": "..." }
    }
  ],
  "total": 12,
  "page": 1
}
```

### Impacto en el FE

Sin cambios de código en el FE. Solo se necesita que el backend acepte tokens vacíos o ausentes en ambos endpoints.

---

## 🔴 BE-03 — Quitar autorización del endpoint `POST /items-variant/filter` (ítems públicos)

### Estado actual

- ✅ `GET /items-variant` — **ya está liberado** de autenticación.
- ❌ `POST /items-variant/filter` — **requiere token**, lo que impide que visitantes puedan filtrar ítems por negocio, categoría u otros criterios desde la pantalla `CustomerItemsScreen`.

### Problema

El filtrado de ítems desde la vista del cliente/visitante (por negocio, sucursal, grupo, etc.) se realiza a través de `POST /items-variant/filter`. El método `ApiService.getAllItemsByOrg()` y similares envían el token del usuario. Para un visitante no autenticado el token está vacío, bloqueando la capacidad de filtrar.

```dart
// Código actual en api_service.dart — getAllItemsByOrg()
final String token = prefs.getString('token') ?? '';  // ← vacío para visitantes

final Map<String, String> requestHeaders = <String, String>{
  'Content-Type': 'application/json',
  'Authorization': token,  // ← enviado vacío
};
final Uri url = Uri.parse(ApiConstants.itemsFilterUrl);  // POST /items-variant/filter
```

### Acción requerida

**Liberar de autenticación** el endpoint `POST /items-variant/filter` de la misma forma que BE-02. Esto permite a visitantes filtrar el catálogo sin necesidad de sesión.

```js
// Antes (ruta protegida):
router.post('/items-variant/filter', authMiddleware, itemController.filter);

// Después (ruta pública):
router.post('/items-variant/filter', itemController.filter);
```

### Contrato esperado

```
POST /items-variant/filter
Headers: Content-Type: application/json
         Authorization: (opcional — vacío o ausente para visitantes)
Body: { "sucursal._id": "<id>" }        ← filtrar por sucursal
      { "organization._id": "<id>" }    ← filtrar por organización/negocio
      { "groups._id": "<id>" }          ← filtrar por grupo/categoría

Respuesta 200:
{
  "response": true,
  "data": [ ... array de items-variante ... ],
  "total": 150
}
```

> Si existe preocupación por exposición de datos sensibles (costos, márgenes), se puede aplicar una proyección en el controlador que excluya campos internos (`costPrice`, `internalCode`, etc.) cuando la petición no lleva token.

---

## 🟡 BE-04 — Endpoint de login devuelve `name` y `lastName` del usuario

### Problema

Cuando un cliente se autentica, el FE guarda `SharedService.userName` a partir de la respuesta del login. Si la respuesta de `POST /user/login` no incluye `name` y `lastName` en el objeto `user`, el perfil autenticado mostrará el campo en blanco o el `userName` generado aleatoriamente (`gb97_a3b9x1z2`), lo que es confuso para el usuario final.

### Acción requerida

Asegurarse de que la respuesta de `POST /user/login` incluya `name` y `lastName` como parte del objeto `user` (ver contrato en BE-01).

Si el sistema usa JWT, los claims del token también deben incluir estos campos para que el FE pueda leerlos sin hacer una llamada adicional al perfil.

---

## 🟢 BE-05 — Validaciones de unicidad en registro de clientes

### Acción requerida

El endpoint `POST /user/registro` debe validar:

1. **Email único:** Rechazar con `400` si ya existe un usuario con el mismo `email`.
2. **UserName único:** Rechazar con `400` si el `userName` generado aleatoriamente ya existe (probabilidad muy baja pero posible).

**Respuestas de error esperadas (para que el FE las muestre al usuario):**

```json
// Email duplicado:
{
  "response": false,
  "message": "Ya existe una cuenta con este correo electrónico.",
  "error": "DUPLICATE_EMAIL"
}

// UserName duplicado (el FE reintentará con otro userName aleatorio):
{
  "response": false,
  "message": "Error al generar el nombre de usuario. Intente nuevamente.",
  "error": "DUPLICATE_USERNAME"
}
```

