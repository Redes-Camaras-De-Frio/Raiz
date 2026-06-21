# Aislamiento de Datos por Sede (Tenencia Múltiple)

Feature que permite que cada usuario vea únicamente los datos de las sedes a las que está asignado.

---

## 1. Modelo de datos

### 1.1 Nueva tabla `usuario_sede`

```sql
CREATE TABLE usuario_sede (
    usuario_id INT NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
    sede_id    INT NOT NULL REFERENCES sedes(id) ON DELETE CASCADE,
    PRIMARY KEY (usuario_id, sede_id)
);
```

### 1.2 Regla de negocio

| Rol | `usuario_sede` | Comportamiento |
|---|---|---|
| `admin` | Sin registros (tabla vacía para ese usuario) | Acceso total a **todas** las sedes |
| `operador` | 1 o más registros | Acceso **solo** a las sedes listadas |

### 1.3 Seed modificado

```sql
-- Admin no se inserta en usuario_sede (acceso total)

INSERT INTO usuario_sede (usuario_id, sede_id) VALUES
    (2, 1);   -- Operador Central → Farmacia Central
```

---

## 2. JWT payload

El token incluye el listado de sedes del usuario:

```json
{
  "id": 2,
  "nombre": "Operador Central",
  "email": "operador@farmacia.com",
  "rol": "operador",
  "sedes": [1]
}
```

- admin: `"sedes": null` (significa "todas")
- operador: `"sedes": [1, 3, 5]`

### 2.1 Login

`SELECT` incluye LEFT JOIN a `usuario_sede` para armar el array:

```sql
SELECT u.id, u.nombre, u.email, u.password_hash, u.rol, u.activo,
       CASE WHEN u.rol = 'admin' THEN NULL
            ELSE array_agg(us.sede_id) FILTER (WHERE us.sede_id IS NOT NULL)
       END AS sedes
FROM usuarios u
LEFT JOIN usuario_sede us ON us.usuario_id = u.id
WHERE u.email = $1
GROUP BY u.id;
```

### 2.2 Register

`POST /api/auth/register` acepta campo opcional `sedes: [1, 3]`. Solo admin puede asignar sedes.

```sql
-- Insertar en usuario_sede después de crear el usuario
INSERT INTO usuario_sede (usuario_id, sede_id) VALUES ($1, unnest($2::int[]));
```

---

## 3. Middleware `verificarAccesoSede`

Función que verifica si el usuario autenticado tiene acceso a una sede específica. Se usa en rutas que reciben `sede_id` como query param.

```javascript
function verificarAccesoSede(req, res, next) {
  const { sedes, rol } = req.usuario;
  const sedeId = req.query.sede_id ? parseInt(req.query.sede_id, 10) : null;

  // Admin tiene acceso total
  if (rol === 'admin') return next();

  // Si no pidió sede específica, se filtra por sus sedes
  if (!sedeId) return next();

  // Si pidió una sede específica, verificar que esté en su lista
  if (!sedes || !sedes.includes(sedeId)) {
    return res.status(403).json({ error: 'No tienes acceso a esta sede' });
  }

  next();
}
```

### 3.1 Uso en rutas

```javascript
// Ejemplo: GET /api/sedes?sede_id=1
router.get('/', verificarToken, sedeRoutes);
// Dentro de sedeRoutes:
router.get('/', verificarAccesoSede, sedeController.listar);
```

---

## 4. Contratos de servicios

Cada función recibe `usuario` (objeto con `rol` y `sedes`) como último parámetro.

### 4.1 SedeService

| Función | Comportamiento |
|---|---|
| `obtenerTodasLasSedes(usuario)` | admin → todas; no-admin → `WHERE id = ANY($1)` |
| `obtenerSedePorId(id, usuario)` | admin → sin filtro; no-admin → verifica que `id` esté en `usuario.sedes` |
| `crearSede(data)` | Sin cambios (solo admin) |
| `actualizarSede(id, data)` | Sin cambios (solo admin) |
| `eliminarSede(id)` | Sin cambios (solo admin) |

### 4.2 CamaraService

| Función | Comportamiento |
|---|---|
| `obtenerTodasLasCamaras(sedeId, usuario)` | si `sedeId` dado → `WHERE sede_id = $1 AND sede_id = ANY($2)`; sino → `WHERE sede_id = ANY($1)` |
| `obtenerCamaraPorId(id, usuario)` | JOIN a camaras.sede_id contra usuario.sedes |

### 4.3 SensorService

| Función | Comportamiento |
|---|---|
| `obtenerTodosLosSensores(camaraId, usuario)` | JOIN sensores → camaras, filtro por `camaras.sede_id = ANY(usuario.sedes)` |
| `obtenerSensorPorId(id, usuario)` | JOIN para verificar acceso a la sede de la cámara |

### 4.4 LecturaService

| Función | Comportamiento |
|---|---|
| `obtenerLecturas(sensorId, limite, usuario)` | JOIN lecturas → sensores → camaras, filtro por sede |
| `obtenerUltimasLecturasPorCamara(usuario)` | JOIN con filtro por `camaras.sede_id = ANY(usuario.sedes)` |

### 4.5 AlertaService

| Función | Comportamiento |
|---|---|
| `obtenerAlertas({ resuelta, limite, usuario })` | JOIN alertas → camaras, filtro por sede |
| `obtenerAlertaPorId(id, usuario)` | JOIN para verificar acceso |

### 4.6 DashboardService

| Función | Comportamiento |
|---|---|
| `obtenerResumen(usuario)` | Todos los COUNT y lecturas filtrados por `camaras.sede_id = ANY(usuario.sedes)` |

---

## 5. Controladores

Cada controlador extrae `req.usuario` y lo pasa al servicio.

```javascript
// Ejemplo: dashboardController
async function resumen(req, res) {
  const data = await dashboardService.obtenerResumen(req.usuario);
  res.json({ datos: data });
}
```

No hay cambios en la lógica de validación de entrada (se mantiene igual).

---

## 6. Rutas

Se agrega `verificarAccesoSede` donde aplique:

```javascript
// index.js
router.use('/sedes', verificarToken, verificarAccesoSede, sedeRoutes);
router.use('/camaras', verificarToken, verificarAccesoSede, camaraRoutes);
router.use('/sensores', verificarToken, verificarAccesoSede, sensorRoutes);
router.use('/lecturas', verificarToken, verificarAccesoSede, lecturaRoutes);
router.use('/alertas', verificarToken, verificarAccesoSede, alertaRoutes);
router.use('/dashboard', verificarToken, verificarAccesoSede, dashboardRoutes);
```

---

## 7. Swagger

- Agregar tabla `usuario_sede` a la descripción del esquema.
- Agregar campo `sedes` (array de ints) en la respuesta de auth/login y auth/register.
- Agregar campo `sedes` opcional en body de register.
- Actualizar descripción de endpoints GET indicando que filtran por sede del usuario.

---

## 8. Resumen de archivos a modificar

| # | Archivo | Cambio |
|---|---|---|
| 1 | `Capa_de_Datos/sql/01_schema.sql` | + tabla `usuario_sede` |
| 2 | `Capa_de_Datos/sql/02_seed.sql` | + INSERT `usuario_sede` |
| 3 | `Capa_de_Aplicacion/src/services/authService.js` | + JOIN usuario_sede en login y register |
| 4 | `Capa_de_Aplicacion/src/controllers/authController.js` | + campo `sedes` opcional en register |
| 5 | `Capa_de_Aplicacion/src/middlewares/auth.js` | + función `verificarAccesoSede` |
| 6 | `Capa_de_Aplicacion/src/routes/index.js` | + `verificarAccesoSede` |
| 7 | `Capa_de_Aplicacion/src/services/sedeService.js` | filtrar por sedes del usuario |
| 8 | `Capa_de_Aplicacion/src/services/camaraService.js` | filtrar por sedes del usuario |
| 9 | `Capa_de_Aplicacion/src/services/sensorService.js` | filtrar por sedes del usuario |
| 10 | `Capa_de_Aplicacion/src/services/lecturaService.js` | filtrar por sedes del usuario |
| 11 | `Capa_de_Aplicacion/src/services/alertaService.js` | filtrar por sedes del usuario |
| 12 | `Capa_de_Aplicacion/src/services/dashboardService.js` | filtrar por sedes del usuario |
| 13 | `Capa_de_Aplicacion/src/controllers/camaraController.js` | pasar `req.usuario` |
| 14 | `Capa_de_Aplicacion/src/controllers/sensorController.js` | pasar `req.usuario` |
| 15 | `Capa_de_Aplicacion/src/controllers/lecturaController.js` | pasar `req.usuario` |
| 16 | `Capa_de_Aplicacion/src/controllers/alertaController.js` | pasar `req.usuario` |
| 17 | `Capa_de_Aplicacion/src/controllers/dashboardController.js` | pasar `req.usuario` |
| 18 | `Capa_de_Aplicacion/src/controllers/sedeController.js` | pasar `req.usuario` |
| 19 | `Capa_de_Aplicacion/src/config/swagger.js` | documentar nuevo campo y esquema |
| 20 | `Capa_de_Datos/docs/modelo-datos.md` | agregar tabla `usuario_sede` |
| 21 | `docs/diagrama-clases.puml` (raíz o app) | agregar relación usuario_sede |

---

## 9. Orden de implementación

1. Schema BD → seed
2. Auth (JWT payload)
3. Middleware verificarAccesoSede
4. Servicios (de abajo arriba: dashboard → alerta → lectura → sensor → camara → sede)
5. Controladores
6. Rutas
7. Swagger
8. Docs
