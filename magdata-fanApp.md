# Requerimientos de Configuración CORS — FanApp Web
**Fecha:** 27 de abril de 2026  
**Para:** Equipo de Backend / Infraestructura  
**De:** Equipo de Desarrollo Mobile/Web  
**Asunto:** Configuración obligatoria de CORS para el funcionamiento de la versión Web de FanApp

---

## Contexto

La aplicación **FanApp** está siendo habilitada para plataforma **Web** (Flutter Web). Al ejecutarse en el navegador, todas las solicitudes HTTP quedan sujetas a la política de **Same-Origin Policy**, lo que requiere que los servidores externos incluyan los headers CORS apropiados.

Actualmente, todas las peticiones al API y a los recursos en S3 son **bloqueadas por el navegador**, impidiendo el funcionamiento de la aplicación web.

---

## Origen que debe ser permitido

| Entorno | Origen |
|---|---|
| Desarrollo local | `http://localhost:*` (cualquier puerto) |
| Producción (pendiente de definir) | `https://<dominio-de-produccion-web>` |

> **Nota:** Recomendamos configurar desde ahora el dominio de producción web definitivo.

---

## Problema 1 — API REST: `api-fanapp.magdata.com.ec`

### Endpoints afectados (confirmados en consola)

| Método | Endpoint | Error |
|---|---|---|
| `POST` | `/ticket/filter` | `net::ERR_FAILED 401 (Unauthorized)` + CORS |
| `GET` | `/persona/` | `net::ERR_FAILED 401 (Unauthorized)` + CORS |

> Todos los demás endpoints del API están igualmente bloqueados. La corrección debe aplicarse **a nivel global del servidor**, no endpoint por endpoint.

### Headers enviados por el cliente Flutter Web

```
Content-Type: application/json
Authorization: <jwt-token>
```

> El token se envía directamente en el header `Authorization` sin prefijo `Bearer`.

### Causa raíz

El servidor **no retorna** el header `Access-Control-Allow-Origin` en las respuestas, ni responde correctamente al preflight `OPTIONS`. El navegador bloquea la petición antes de procesarla, lo que produce también el error **401** (el request nunca llega con el token porque el preflight falla primero).

### Configuración requerida en Express.js

Instalar el paquete `cors` si no está instalado:

```bash
npm install cors
```

Aplicar la configuración **antes de todas las rutas**:

```javascript
const cors = require('cors');

const corsOptions = {
  origin: [
    'http://localhost', // desarrollo (cualquier puerto)
    /^http:\/\/localhost:\d+$/,  // regex para cubrir cualquier puerto local
    'https://<dominio-de-produccion-web>', // REEMPLAZAR con el dominio real
  ],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'customToken',
  ],
  credentials: true,
  optionsSuccessStatus: 200, // Para compatibilidad con browsers legacy
};

app.use(cors(corsOptions));

// Responder explícitamente a todas las peticiones OPTIONS (preflight)
app.options('*', cors(corsOptions));
```

> **Importante:** El header `Authorization` debe estar explícitamente listado en `allowedHeaders` para que el navegador permita enviarlo en las peticiones.

---

## Problema 2 — AWS S3: `files-gear-dev.s3.us-east-2.amazonaws.com`

### Síntoma

Las imágenes de logos de equipos se descargan exitosamente (`200 OK`), pero el navegador **bloquea la lectura** del response porque el bucket no incluye los headers CORS:

```
Access to image at 'https://files-gear-dev.s3.us-east-2.amazonaws.com/...' 
from origin 'http://localhost:56326' has been blocked by CORS policy: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

Imágenes afectadas (confirmadas):
- `upload/San Antonio/images-teams/San Antonio.png`
- `upload/Cuenca Jrs/images-teams/Diseño sin título.png`
- `upload/Independiente Juniors/images-teams/Independiente Jrs.png`
- `upload/Quito FC/images-teams/Quito FC.png`
- `upload/El Nacional/images-teams/Nacional.png`
- `upload/Liga de Portoviejo/images-teams/Liga Portoviejo.png`
- `upload/Deportivo Santo Domingo/images-teams/Deportivo Santo Domingo.png`

### Configuración requerida en AWS S3

En la consola de AWS → S3 → Bucket `files-gear-dev` → **Permissions** → **Cross-origin resource sharing (CORS)**, pegar la siguiente política JSON:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedOrigins": [
      "http://localhost",
      "http://localhost:*",
      "https://<dominio-de-produccion-web>"
    ],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

> **Nota AWS:** Para cubrir `localhost` con cualquier puerto no es posible usar wildcard parcial en S3. Se recomienda agregar `"http://localhost:*"` aunque S3 solo acepta valores exactos. La alternativa más simple para desarrollo es agregar `"*"` temporalmente en `AllowedOrigins` durante el periodo de QA, y restringirlo al dominio de producción antes del go-live.

---

## Resumen de acciones requeridas

| # | Responsable | Acción | Prioridad |
|---|---|---|---|
| 1 | Backend | Configurar middleware CORS en servidor Express (`api-fanapp.magdata.com.ec`) con soporte para `Authorization` header y preflight `OPTIONS` | **Alta** |
| 2 | Infraestructura / DevOps | Configurar política CORS en bucket S3 `files-gear-dev` para `GET`/`HEAD` | **Alta** |
| 3 | Ambos | Reemplazar `localhost` por el dominio web de producción definitivo una vez se defina | Media |

---

## Validación

Una vez realizados los cambios, verificar en la consola del navegador que:

1. Las imágenes de los equipos cargan correctamente en la pantalla de selección de equipo.
2. Los endpoints `/ticket/filter` y `/persona/` retornan datos sin errores CORS.
3. No aparece ningún error `ERR_FAILED` relacionado a CORS en la consola.

Para pruebas rápidas sin afectar producción, el equipo de frontend puede usar el flag `--disable-web-security` en Chrome, pero esto **no es una solución permanente** y solo es válido para desarrollo local del equipo de frontend.

---

*Ante cualquier duda, contactar al equipo de desarrollo mobile/web.*
