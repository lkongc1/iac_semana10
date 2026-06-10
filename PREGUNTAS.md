# PASO 13: PREGUNTAS A RESPONDER

## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?

Porque Prometheus y Loki resuelven problemas distintos.
- Prometheus está diseñado para recolectar y consultar métricas expuestas a través de endpoints.
- Loki es un sistema de agregación de logs. Recibe líneas de log y las indexa únicamente por etiquetas.

## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

- Reproducibilidad y consistencia.
- Reducción de errores.

## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

1. CPU del host refleja el uso total de CPU de toda la máquina.
2. CPU del contenedor mide el consumo de los procesos dentro del contenedor y puede estar afectada por límites de CPU.

## 4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?

1. Evaluation Interval: es la frecuencia con la que Prometheus evalúa la expresión de la regla.
2. Pending period: es el tiempo mínimo que la condición de alerta debe mantenerse verdadera de forma continua para que la alerta cambie de estado.
