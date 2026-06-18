# PC3 — Organización del proyecto

## 1. Definición del caso de estudio

El proyecto propone una solución IoT para el monitoreo de cámaras de frío en una farmacia central y en tres grandes cámaras de frío que representan a los distribuidores o sedes remotas. El objetivo es asegurar la conservación de medicamentos y productos sensibles a la temperatura, registrando lecturas periódicas, generando alertas y permitiendo el seguimiento desde una sala de control.

### Alcance del servicio
- Monitoreo continuo de temperatura y, de ser necesario, humedad.
- Registro histórico de las mediciones para consultas posteriores.
- Generación de alertas cuando se detecten valores fuera del rango permitido.
- Supervisión desde una sala de administración con acceso a dashboard.
- Comunicación entre la farmacia central y las cámaras de frío remotas mediante la red diseñada en Cisco Packet Tracer.

### Cobertura del servicio
La solución se orienta a un entorno académico que simula una pequeña red empresarial compuesta por:
- **Una farmacia central**, donde se ubica la sala de atención y la sala de monitoreo.
- **Tres cámaras de frío distribuidas**, que representan a los distribuidores o sedes externas.
- **Una red de comunicación interna** que conecta los dispositivos de monitoreo con el servidor central.

### Usuarios del sistema
#### Usuarios directos
- Administrador de la farmacia.
- Personal de monitoreo.
- Personal técnico.
- Personal de atención al cliente o mesa de ayuda.

#### Usuarios potenciales
- Otras farmacias o boticas que necesiten monitorear su cadena de frío.
- Distribuidores de productos farmacéuticos.
- Entidades o supervisores que requieran reportes de trazabilidad.

### Clientes concurrentes y demanda
La solución considera que varias personas pueden consultar el sistema al mismo tiempo desde dispositivos de administración como PC, laptop, tablet o smartphone. En la práctica, esto se traduce en un entorno de baja o media concurrencia, suficiente para pruebas académicas, pero con la lógica necesaria para escalar después.

---

## 2. Propuesta de arquitectura local con Docker

Para mantener el proyecto simple y fácil de implementar, el servidor central se representará de forma local mediante **Docker**. Esto permite simular una arquitectura por capas sin necesidad de montar varios servidores físicos.

La idea es usar **un solo servidor físico o virtual** dentro de la sala de monitoreo, y dentro de él ejecutar **tres contenedores** separados:

### 2.1 Capa de presentación
Contenedor encargado de la interfaz visual del sistema.

**Funciones:**
- Mostrar dashboards.
- Visualizar temperatura en tiempo real.
- Ver alertas y estados de cada cámara de frío.
- Permitir acceso desde la sala de atención o monitoreo.

**Tecnologías sugeridas:**
- HTML
- CSS
- Bootstrap
- JavaScript

### 2.2 Capa de aplicación
Contenedor encargado de la lógica de negocio y la API.

**Funciones:**
- Recibir datos enviados por los sensores o por el gateway.
- Validar información recibida.
- Procesar alertas cuando la temperatura salga del rango.
- Exponer endpoints para el frontend.
- Conectarse con la base de datos.

**Tecnologías sugeridas:**
- Node.js
- Express

### 2.3 Capa de datos
Contenedor encargado del almacenamiento persistente.

**Funciones:**
- Guardar lecturas históricas.
- Guardar registros de alertas.
- Almacenar usuarios y accesos básicos.
- Conservar información para reportes y auditorías.

**Tecnologías sugeridas:**
- PostgreSQL

### 2.4 Justificación de esta decisión
Esta propuesta es adecuada para un proyecto académico porque:
- Simplifica la implementación.
- Mantiene separadas las responsabilidades.
- Permite explicar claramente la arquitectura.
- Facilita el despliegue local.
- Representa un entorno real de forma ordenada.

---

## 3. Estructura mínima recomendada del entorno

### Sala de atención / farmacia central
- 1 PC de administración o atención al cliente.
- Acceso al dashboard.
- Consulta de alertas y estado del sistema.

### Sala de monitoreo / administración
- 1 PC de monitoreo.
- 1 servidor central representado con Docker.
- Acceso principal a la aplicación y a la base de datos.

### Cámaras de frío / distribuidores
- Sensores IoT.
- Microcontroladores o gateways.
- Sin necesidad de PC ni servidor adicional.

---

## 4. Tareas pendientes para esta entregable

### 4.1 Definir la capa de datos
- Diseñar el modelo de base de datos.
- Identificar tablas mínimas.
- Definir campos, tipos de datos y relaciones.
- Determinar qué información se almacenará por cada lectura.

### 4.2 Preparar el despliegue con Docker
- Crear el `Dockerfile` para cada capa.
- Configurar `docker-compose`.
- Conectar frontend, backend y base de datos.
- Verificar la comunicación entre contenedores.

### 4.3 Implementar la lógica básica
- Crear endpoints para recibir y consultar datos.
- Guardar lecturas de temperatura.
- Registrar alertas.
- Mostrar datos en el dashboard.

### 4.4 Integrar la red del caso de estudio
- Mantener la farmacia central como nodo principal.
- Conectar las cámaras de frío a la red diseñada.
- Validar el flujo de información entre sedes y servidor.

### 4.5 Documentar el informe
- Redactar el caso de estudio.
- Describir la arquitectura del servidor.
- Explicar el uso de Docker.
- Incluir conclusiones y referencias en formato APA.

---

## 5. Próximo paso sugerido

Lo siguiente será definir la **estructura de la base de datos** y luego pasarla a una implementación simple con PostgreSQL dentro de Docker.

