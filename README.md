# Stack de Observabilidad — Infraestructura como Código

Stack completo de monitoreo con Prometheus, Loki, Grafana, Alloy, node-exporter y cAdvisor, más aplicaciones instrumentadas (backend + frontend Express). Todo definido como código en `docker-compose.yml`.

## Inicio rápido

```bash
git clone https://github.com/lkongc1/iac_semana10.git
cd iac_semana10
docker compose up -d --build
```

## Servicios y URLs

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Frontend | http://localhost:8080 | Hello World + botones de tráfico/carga |
| Backend (API) | http://localhost:3001 | `/api/hello`, `/metrics`, `/load` |
| Grafana | http://localhost:3000 | Dashboards y alarmas (admin/admin) |
| Prometheus | http://localhost:9090 | Recolección de métricas |
| Loki | http://localhost:3100 | Almacenamiento de logs |
| Alloy (UI) | http://localhost:12345 | Estado del recolector de logs |
| cAdvisor | http://localhost:8081 | Métricas por contenedor |
| node-exporter | http://localhost:9100/metrics | Métricas del host |

## Componentes

| Componente | Rol | Señal |
|-----------|-----|-------|
| **Prometheus** | Recolecta y almacena métricas | Métricas (time series) |
| **Loki** | Almacena logs etiquetados | Logs |
| **Alloy** | Recolecta logs de contenedores → Loki | Recolección |
| **Grafana** | Dashboards + alarmas | Visualización |
| **node-exporter** | Expone métricas del host (CPU, RAM, disco) | Exportación |
| **cAdvisor** | Expone métricas por contenedor | Exportación |

## Configuraciones

- **Datasources** (Prometheus + Loki) provisionados automáticamente en Grafana.
- **Logs** etiquetados por Alloy con `tier=application` (backend/frontend) o `tier=infrastructure` (resto del stack).
- **Métricas** scrapeadas cada 5 segundos desde apps, node-exporter y cAdvisor.

## Guía completa

Ver [DEPLOY.md](./DEPLOY.md) para instrucciones detalladas de despliegue, configuración de dashboards, alarmas y solución de problemas.

## Reset

```bash
docker compose down -v   # borra también dashboards y alarmas creados
```
