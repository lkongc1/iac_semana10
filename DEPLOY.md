# Guía de Despliegue — Stack de Observabilidad

Stack de monitoreo con Prometheus, Loki, Alloy y Grafana, más aplicaciones instrumentadas (backend + frontend Express). Todo definido como Infraestructura como Código.

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
                 ▼
          ┌──────────────┐
          │   Grafana    │
          │    :3000     │
          │ (dashboard   │
          │ + alarmas)   │
          └──────────────┘

┌──────────┐    ┌──────────┐
│  Alloy   │───▶│   Loki   │
│  :12345  │    │  :3100   │
│(recolect.│    │ (logs)   │
│ de logs) │    └──────────┘
└──────────┘
```

- **Prometheus**: recibe métricas de las apps vía `/metrics` y las almacena.
- **Loki**: almacena logs etiquetados que recibe de Alloy.
- **Alloy**: recolecta logs de todos los contenedores vía el socket de Docker y los etiqueta por `tier` (`application` o `infrastructure`).
- **Grafana**: dashboards y alarmas. Las fuentes de datos (Prometheus + Loki) se aprovisionan como código al iniciar.

## Prerrequisitos

| Herramienta | Versión mínima | Verificación |
|-------------|---------------|--------------|
| Docker | 24+ | `docker --version` |
| Docker Compose | 2.20+ | `docker compose version` |
| Puertos libres | 3000, 3001, 3100, 8080, 9090, 12345 | `ss -tlnp` |

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

La primera ejecución descarga imágenes y construye las apps. Esperá 1–2 minutos hasta que todos los contenedores estén healthy:

```bash
docker compose ps
```

Deberías ver 6 servicios con estado `Up`:

```
lab-backend        Up
lab-frontend       Up
lab-prometheus     Up
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

#### Panel 1: Peticiones HTTP por segundo (backend)

- Fuente: **Prometheus**
- Query:
  ```
  rate(http_requests_total{job="backend"}[1m])
  ```
- Visualización: **Time series**
- Título: *"Peticiones HTTP/s — Backend"*

#### Panel 2: Latencia de peticiones (backend)

- Fuente: **Prometheus**
- Query:
  ```
  rate(http_request_duration_seconds_sum{job="backend"}[1m]) / rate(http_request_duration_seconds_count{job="backend"}[1m])
  ```
- Visualización: **Time series**
- Unit: **seconds (s)**
- Título: *"Latencia promedio — Backend"*

#### Panel 3: CPU del proceso backend

- Fuente: **Prometheus**
- Query:
  ```
  rate(process_cpu_seconds_total{job="backend"}[1m]) * 100
  ```
- Visualización: **Time series**
- Unit: **Percent (0–100)**
- Threshold: **50** (rojo)
- Título: *"CPU proceso backend (%)"*

#### Panel 4: Logs de aplicación

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

#### Panel 5: Logs de infraestructura

- Fuente: **Loki**
- Visualización: **Logs**
- Query:
  ```
  {tier="infrastructure"}
  ```
- Título: *"Logs de infraestructura"*

Guardá el dashboard con **Save dashboard** (ícono arriba a la derecha).

## Paso 6: Configurar alarma de CPU en backend

1. **Alerting → Alert rules → New alert rule**.
2. Nombre: `CPU backend elevada`.
3. Query A (Prometheus):
   ```
   rate(process_cpu_seconds_total{job="backend"}[1m]) * 100
   ```
4. Condición: expresión **Threshold** con **IS ABOVE `50`**.
5. Evaluation interval: `10s`. Pending period: `30s`.
6. Etiqueta: `severity = warning`.
7. Guardar con **Save rule and exit**.

## Paso 7: Probar la alarma

1. En el frontend, pulsá **"Generar carga de CPU (30s)"** (o `curl "http://localhost:3001/load?seconds=60"`).
2. Observá en **Alerting → Alert rules** cómo la regla pasa de `Normal` → `Pending` → `Firing`.
3. Cuando la carga termina, la CPU baja y la alarma vuelve a `Normal`.

## Paso 8: Cerrar el ciclo — alarma → log

El backend tiene un endpoint `/alerts` que recibe webhooks de Grafana Alerting y los registra como logs estructurados. Esto permite visualizar el recorrido completo: **métrica cruza umbral → se dispara alarma → la alarma produce un log → el log aparece en el dashboard**.

1. En Grafana, andá a **Alerting → Contact points → New contact point**.
2. Nombre: `backend-webhook`.
3. Integration: **Webhook**.
4. URL: `http://backend:3001/alerts`.
5. Guardá con **Save contact point**.
6. Volvé a **Alerting → Notification policies**, editá la política por defecto y asignale el contact point `backend-webhook`.

Ahora cuando la alarma se dispare, el backend recibe el webhook y escribe un log con `level=ERROR` (si es `firing`) o `level=INFO` (si es `resolved`). Ese log aparece automáticamente en el panel **"Logs de aplicación"** que configuraste en el Paso 5.

Para verificarlo: dispará la alarma, andá al panel de logs de aplicación y filtrá:

```
{tier="application"} | json | msg="grafana_alert_received"
```

Verás la entrada con `alert_status: firing` y el nombre de la regla. Así se cierra el ciclo: métrica → alarma → webhook → log → dashboard.

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
| Alarma no se dispara | Sin carga suficiente | Esperar a que `process_cpu_seconds_total` acumule datos, luego generar carga |
| Loki 503 | Inicializando el ring | Esperar 2–3 minutos, reintentar |

## Variables de entorno

| Variable | Default | Descripción |
|----------|---------|-------------|
| `BACKEND_URL` | `http://backend:3001` | URL del backend para el proxy del frontend |
| `GF_SECURITY_ADMIN_USER` | `admin` | Usuario de Grafana |
| `GF_SECURITY_ADMIN_PASSWORD` | `admin` | Contraseña de Grafana |
