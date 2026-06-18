# Sistema de Monitoreo de Cadena de Frío

Sistema IoT para monitoreo de temperatura en cámaras de frío de farmacias y distribuidoras. Arquitectura en capas con bases de datos PostgreSQL, API REST y dashboard web.

## Estructura

```
TB2_Redes/
├── Capa_de_Datos/       → github.com/Redes-Camaras-De-Frio/Capa-De-Datos
├── Capa_de_Aplicacion/  → github.com/Redes-Camaras-De-Frio/Capa-De-Aplicacion
├── docker-compose.yml   # Orquestador principal
└── docs/
```

## Clonar

```bash
git clone --recurse-submodules git@github.com:Redes-Camaras-De-Frio/Raiz.git
```

Si ya clonaste sin submódulos:

```bash
git submodule update --init --recursive
```

## Inicio rápido

```bash
docker-compose up -d
```

| Servicio | URL |
|---|---|
| API REST | http://localhost:3000 |
| Swagger UI | http://localhost:3000/api-docs |
| PostgreSQL | localhost:5433 |
| pgAdmin | http://localhost:5050 |

## Capas

| Capa | Estado | Tecnología |
|---|---|---|
| Datos | ✅ Activa | PostgreSQL 15 + pgAdmin |
| Aplicación | ✅ Activa | Node.js + Express + JWT |
| Presentación | ⏳ Pendiente | HTML/CSS/JS |
