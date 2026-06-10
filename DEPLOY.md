# Guía de Despliegue — Stack de Observabilidad

Stack completo de monitoreo con Prometheus, Loki, Alloy, Grafana, node-exporter y cAdvisor, más aplicaciones instrumentadas (backend + frontend Express). Todo definido como Infraestructura como Código.

## Arquitectura

```
┌──────────────┐     ┌──────────────┐
│   Frontend   │────▶│   Backend    │
│   :8080      │     │   :3001      │
│  (Express)   │     │  (Express)   │
└──────┬───────┘     └──────┬───────┘
       │ /metrics           │ /metrics
       ▼                    ▼
┌──────────────────────────────────────┐
│           Prometheus :9090           │
│         (métricas + alertas)         │
└────────────────┬─────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────┐ ┌──────────┐ ┌──────────┐
│  node  │ │ cAdvisor │ │ Grafana  │
│exporter│ │  :8081   │ │  :3000   │
│ :9100  │ │          │ │(dashboard│
└────────┘ └──────────┘ │+ alarmas)│
                        └──────────┘

┌──────────┐    ┌──────────┐
│  Alloy   │───▶│   Loki   │
│  :12345  │    │  :3100   │
│(recolect.│    │ (logs)   │
│ de logs) │    └──────────┘
└──────────┘
```

- **Prometheus**: recibe métricas de los exporters y de las apps vía `/metrics`.
- **Loki**: almacena logs etiquetados que recibe de Alloy.
- **Alloy**: recolecta logs de todos los contenedores vía el socket de Docker y los etiqueta por `tier` (`application` o `infrastructure`).
- **Grafana**: dashboards y alarmas. Las fuentes de datos (Prometheus + Loki) se aprovisionan como código al iniciar.

## Prerrequisitos

| Herramienta | Versión mínima | Verificación |
|-------------|---------------|--------------|
| Docker | 24+ | `docker --version` |
| Docker Compose | 2.20+ | `docker compose version` |
| Puertos libres | 3000, 3001, 3100, 8080, 8081, 9090, 9100, 12345 | `ss -tlnp` |

## Paso 1: Clonar el repositorio

```bash
git clone https://github.com/lkongc1/iac_semana10.git
cd iac_semana10
```

## Paso 2: Levantar el stack

Desde la raíz del proyecto:

```bash
docker compose up -d --build
```

La primera ejecución descarga imágenes (~500 MB) y construye las apps. Esperá 1–2 minutos hasta que todos los contenedores estén healthy:

```bash
docker compose ps
```

Deberías ver 8 servicios con estado `Up`:

```
lab-backend        Up
lab-frontend       Up
lab-prometheus     Up
lab-node-exporter  Up
lab-cadvisor       Up
lab-loki           Up
lab-alloy          Up
lab-grafana        Up
```

## Paso 3: Verificar los servicios

Abrí en el navegador:

| Servicio | URL | Qué esperar |
|----------|-----|-------------|
| Frontend | http://localhost:8080 | Página "Hello World" con dos botones |
| Backend | http://localhost:3001 | JSON `{"message":"Hello World desde el backend"}` |
| Prometheus | http://localhost:9090 | UI de Prometheus |
| Grafana | http://localhost:3000 | Login (`admin` / `admin`) |
| Alloy | http://localhost:12345 | UI de estado del recolector |
| cAdvisor | http://localhost:8081 | Métricas de contenedores |
| Node Exporter | http://localhost:9100/metrics | Métricas del host |

### Health check rápido

```bash
curl -s http://localhost:8080 | head -1
curl -s http://localhost:3001/healthz
curl -s http://localhost:3001/metrics | head -5
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:3100/ready
```

## Paso 4: Generar datos de prueba

1. Abrí http://localhost:8080
2. Pulsá varias veces **"Saludar (API)"** — cada click genera peticiones, métricas y logs.
3. Las apps emiten logs simulados cada pocos segundos (pedidos, pagos, errores).

## Paso 5: Configurar dashboards en Grafana

### 5.1 Verificar datasources

1. Entrá a Grafana (http://localhost:3000, `admin`/`admin`).
2. Andá a **Connections → Data sources**.
3. Confirmá que **Prometheus** y **Loki** aparecen con estado OK (ya vienen provisionados).

### 5.2 Crear dashboard

**Dashboard → New → New dashboard → Add visualization**.

#### Panel 1: CPU del contenedor backend

- Fuente: **Prometheus**
- Query:
  ```
  sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
  ```
- Visualización: **Time series**
- Unit: **Percent (0–100)**
- Threshold: **50** (rojo)
- Título: *"CPU contenedor backend (%)"*

#### Panel 2: CPU del host

- Fuente: **Prometheus**
- Query:
  ```
  100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
  ```
- Visualización: **Time series**
- Unit: **Percent (0–100)**
- Título: *"CPU del host (%)"*

#### Panel 3: Logs de aplicación

- Fuente: **Loki**
- Visualización: **Logs**
- Query:
  ```
  {tier="application"} | json
  ```
- Para filtrar solo errores:
  ```
  {tier="application"} | json | level="ERROR"
  ```
- Título: *"Logs de aplicación"*

#### Panel 4: Logs de infraestructura

- Fuente: **Loki**
- Visualización: **Logs**
- Query:
  ```
  {tier="infrastructure"}
  ```
- Título: *"Logs de infraestructura"*

Guardá el dashboard con **Save dashboard** (ícono arriba a la derecha).

## Paso 6: Configurar alarma de CPU > 50%

1. **Alerting → Alert rules → New alert rule**.
2. Nombre: `CPU backend > 50%`.
3. Query A (Prometheus):
   ```
   sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
   ```
4. Condición: expresión **Threshold** con **IS ABOVE `50`**.
5. Evaluation interval: `10s`. Pending period: `30s`.
6. Etiqueta: `severity = warning`.
7. Guardar con **Save rule and exit**.

## Paso 7: Probar la alarma

1. En el frontend, pulsá **"Generar carga de CPU (30s)"** (o `curl "http://localhost:3001/load?seconds=60"`).
2. Observá el panel de CPU del backend — debe superar el 50%.
3. En **Alerting → Alert rules**, verificá que la regla pase de `Normal` → `Pending` → `Firing`.
4. Cuando la carga termina, vuelve a `Normal`.

## Comandos útiles

```bash
# Levantar (reconstruir si hay cambios)
docker compose up -d --build

# Ver estado de todos los servicios
docker compose ps

# Logs de un servicio específico
docker compose logs -f grafana
docker compose logs -f backend

# Logs de todos los servicios
docker compose logs -f

# Detener el stack (conserva dashboards y datos)
docker compose down

# Detener y borrar TODO (reset completo)
docker compose down -v
```

## Solución de problemas

| Problema | Causa probable | Solución |
|----------|---------------|----------|
| Puerto en uso | Otra instancia corriendo | `docker compose down`, liberar puerto, reintentar |
| Servicio no levanta | Error de build | `docker compose logs <servicio>` |
| Sin métricas en Prometheus | Target down | http://localhost:9090/targets |
| Sin logs en Loki | Alloy sin acceso al socket | http://localhost:12345, verificar volúmenes |
| Alarma no se dispara | Nombre de contenedor incorrecto | Usar `name="lab-backend"` en la query |
| Loki 503 | Inicializando el ring | Esperar 2–3 minutos, reintentar |

## Variables de entorno

| Variable | Default | Descripción |
|----------|---------|-------------|
| `BACKEND_URL` | `http://backend:3001` | URL del backend para el proxy del frontend |
| `GF_SECURITY_ADMIN_USER` | `admin` | Usuario de Grafana |
| `GF_SECURITY_ADMIN_PASSWORD` | `admin` | Contraseña de Grafana |
