# Rediseno de Roles y Gestion de Sensores (SaaS de Monitoreo)

Este documento detalla la redefinicion estrategica de los roles del sistema y el flujo para la gestion de camaras y sensores por parte de los operadores de sede. Esto alinea la implementacion del software con un modelo SaaS enfocado estrictamente en **monitoreo**.

---

## 1. Definicion y Simplificacion de Roles

Se elimina por completo el rol de `tecnico` de la plataforma para simplificar el flujo y adaptarlo a un modelo de negocio donde la plataforma solo expone la informacion y alertas, dejando las acciones de reparacion fisica en manos del cliente.

| Rol | Alcance | Descripcion |
|---|---|---|
| **`admin`** (Global) | **Acceso Total (Multi-tenant)** | Crea usuarios operadores, da de alta nuevas sedes (clientes), configura camaras base y tiene visibilidad global del sistema. |
| **`operador`** (Cliente de Sede) | **Acceso Aislado (Single/Multi-sede)** | Monitorea el dashboard de su sede, visualiza alertas, las resuelve manualmente y **gestiona sus propios sensores y camaras** dentro de las sedes que tiene asignadas. |

---

## 2. Gestion Autonoma de Sensores por el Operador

Para permitir que el cliente de la sede reemplace sensores defectuosos o anada camaras, el rol `operador` adquiere permisos de creacion, actualizacion y desactivacion, **pero estrictamente restringido a las sedes que tiene asignadas**.

### 2.1 Regla de Seguridad de Aislamiento
El backend interceptara cualquier peticion de creacion/modificacion para validar que la camara o el sensor modificado pertenezca a una de las sedes a las que el operador tiene acceso.

* **Ejemplo de intento de vulneracion:** 
  Un operador asignado a la sede `1` (Farmacia Central) envia un `POST` para agregar un sensor a la camara `5` (que pertenece a la sede `2` - Distribuidora Norte). 
  * **Comportamiento esperado:** El backend detecta que la camara `5` pertenece a la sede `2`. Como el operador no tiene la sede `2` en su lista (`usuario_sede`), el middleware `verificarAccesoSede` (o el servicio correspondiente) bloquea la peticion retornando un error `403 Forbidden`.

---

## 3. Estrategia de "Soft Delete" (Desactivacion de Sensores)

**Queda estrictamente prohibido realizar un borrado fisico (`DELETE`) de un sensor.**

### 3.1 Justificacion
En el monitoreo de cadena de frio (medicamentos, vacunas), la trazabilidad historica de las lecturas es critica por temas de auditorias sanitarias y legales. Eliminar fisicamente un sensor borra por cascada sus lecturas historicas asociadas, lo cual rompe la integridad del sistema.

### 3.2 Flujo de Reemplazo (Soft Delete + Adicion)
Cuando un sensor falla fisicamente y el operador de la sede lo reemplaza:

1. **Desactivacion:** 
   El operador "elimina" el sensor en la interfaz. El backend ejecuta un `soft delete`:
   ```sql
   UPDATE sensores SET activo = false WHERE id = $1;
   ```
2. **Historial intacto:** 
   Todas las lecturas previas de ese sensor siguen existiendo en la tabla `lecturas`. No se pierde la trazabilidad de la camara de frio.
3. **Registro del nuevo sensor:** 
   El operador registra el nuevo dispositivo fisico insertando una nueva fila:
   ```sql
   INSERT INTO sensores (camara_id, tipo, unidad, activo) VALUES ($1, $2, $3, true);
   ```
   Esto genera un nuevo `id` de sensor, el cual empezara a recibir lecturas limpias a partir de ese momento.

---

## 4. Impacto en Matriz de Accesos (API REST)

Los permisos definidos en la guia de implementacion se reestructuran eliminando al tecnico y otorgando los permisos adecuados al operador.

| Recurso / Accion | `admin` | `operador` | `tecnico` (ELIMINADO) |
|---|---|---|---|
| **Sedes** | | | |
| `GET /api/sedes` (Listar) | ✅ (Ver todas) | ✅ (Ver sus sedes) | ❌ |
| `POST / PUT / DELETE` | ✅ | ❌ | ❌ |
| **Camaras** | | | |
| `GET /api/camaras` | ✅ (Ver todas) | ✅ (Ver sus camaras) | ❌ |
| `POST /api/camaras` (Crear) | ✅ | ✅ (Solo en sus sedes) | ❌ |
| `PUT /api/camaras/:id` (Editar) | ✅ | ✅ (Solo en sus sedes) | ❌ |
| `DELETE /api/camaras/:id` (Eliminar) | ✅ | ❌ (Solo admin borra) | ❌ |
| **Sensores** | | | |
| `GET /api/sensores` | ✅ (Ver todos) | ✅ (Ver sus sensores) | ❌ |
| `POST /api/sensores` (Crear)| ✅ | ✅ (Solo en sus sedes) | ❌ |
| `PUT /api/sensores/:id` (Editar) | ✅ | ✅ (Solo en sus sedes) | ❌ |
| `DELETE /api/sensores/:id` (Eliminar/Soft)| ✅ | ✅ (Soft Delete: pone `activo=false`) | ❌ |
| **Alertas** | | | |
| `PATCH /api/alertas/:id/resolver` | ✅ | ✅ (Solo de sus sedes) | ❌ |

---

## 5. Cambios requeridos en la Base de Datos y Backend

### 5.1 Base de Datos (SQL)
- **Eliminar Seed Tecnico:** En `02_seed.sql`, remover cualquier insercion de usuario con rol `'tecnico'`.
- **Restriccion de Rol:** Si existe un constraint `CHECK` para el campo `rol` en la tabla `usuarios`, actualizarlo para permitir unicamente `admin` y `operador`.

### 5.2 Middlewares y Controladores (`Capa_de_Aplicacion`)
- **Remover logica de tecnico:** Limpiar `requiereRol('tecnico')` de todas las rutas y controladores.
- **Modificar Controlador de Eliminacion de Sensores:** 
  En `sensorController.js` (o en su correspondiente servicio), el metodo para `DELETE /api/sensores/:id` ejecutado por un **Operador** debe llamar a la funcion de **desactivacion (Soft Delete)**, mientras que un **Admin** podria, bajo su responsabilidad, hacer un borrado fisico si se creo por error (opcional, se recomienda que ambos hagan soft delete).
- **Proteger Endpoints de Escritura para Operador:**
  En los controladores/servicios de creacion (`POST`) y modificacion (`PUT`) de Camaras y Sensores, validar que el `sede_id` asociado pertenezca a `req.usuario.sedes` cuando el rol sea `'operador'`.

---

## 6. Siguiente Paso Recomendado

1. Modificar el script de base de datos (`01_schema.sql` y `02_seed.sql`) para remover las referencias al tecnico y asegurar los roles validos.
2. Actualizar las politicas de autorizacion en las rutas de la `Capa_de_Aplicacion`.
3. Implementar el endpoint de "Soft Delete" en `sensorService.js` reemplazando la consulta de eliminacion fisica por un `UPDATE sensores SET activo = false`.

---

## 7. Plan de Implementacion (Pendiente)

A continuacion se detalla la checklist de cambios a realizar en el codigo para completar el rediseno:

### 7.1 Base de Datos ✅ (Ya aplicado)
- [x] `01_schema.sql` — CHECK de `rol` ya solo permite `'admin'` y `'operador'`.
- [x] `02_seed.sql` — Ya no inserta usuario con rol `'tecnico'`.

### 7.2 Controladores
- [x] `authController.js:14` — Eliminar `'tecnico'` del array `rolesValidos`.
- [x] `camaraController.js:25-45` — Validar que el `sede_id` del body pertenezca a las sedes del operador en `crear`.
- [x] `camaraController.js:47-65` — Validar que la camara a editar pertenezca a las sedes del operador en `actualizar`.
- [x] `sensorController.js:25-47` — Validar que la `camara_id` pertenezca a las sedes del operador en `crear`.

### 7.3 Servicios
- [x] `sensorService.js:72-75` — Reemplazar `DELETE FROM sensores` por `UPDATE sensores SET activo = false`.
- [x] `sensorService.js:43-51` — Agregar validacion de pertenencia de sede en `crearSensor` _(se maneja desde el controlador)_.

### 7.4 Rutas (Autorizacion)
- [x] `sensorRoutes.js:100` — Cambiar `requiereRol('admin', 'tecnico')` a `requiereRol('admin', 'operador')` en POST.
- [x] `sensorRoutes.js:138` — Cambiar `requiereRol('admin', 'tecnico')` a `requiereRol('admin', 'operador')` en PUT.
- [x] `sensorRoutes.js:159` — Cambiar `requiereRol('admin')` a `requiereRol('admin', 'operador')` en DELETE.
- [x] `camaraRoutes.js:109` — Cambiar `requiereRol('admin')` a `requiereRol('admin', 'operador')` en POST.
- [x] `camaraRoutes.js:148` — Cambiar `requiereRol('admin', 'tecnico')` a `requiereRol('admin', 'operador')` en PUT.
- [x] `camaraRoutes.js:169` — DELETE se mantiene solo para `admin` _(sin cambios)_.

### 7.5 Middleware
- [x] `auth.js` — `verificarAccesoSede` ya funciona correctamente. No requiere cambios.

---

## 8. Progreso

| Etapa | Estado |
|---|---|
| Schema SQL | ✅ Completo |
| Seed SQL | ✅ Completo |
| Rutas (autorizacion) | ✅ Completo |
| Soft Delete en sensorService | ✅ Completo |
| Validacion de aislamiento en controladores | ✅ Completo |
| Auth controller (roles validos) | ✅ Completo |
| Documentacion actualizada | ✅ Completo |
