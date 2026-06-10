# Paso 13: Preguntas a responder

## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

Porque Prometheus y Loki resuelven problemas distintos.

- **Prometheus** está diseñado para recolectar y consultar métricas numéricas (time series) expuestas a través de endpoints como `/metrics`. Almacena pares timestamp-valor y permite hacer agregaciones, rates, histogramas, etc.
- **Loki** es un sistema de agregación de logs. Recibe líneas de log y las indexa únicamente por etiquetas (labels), sin indexar el contenido completo como haría Elasticsearch. Esto lo hace eficiente en almacenamiento y compatible con el ecosistema Grafana.

Las métricas te dicen **qué** está pasando (CPU al 90%, 50 errores/segundo). Los logs te dicen **por qué** está pasando (stack trace, mensaje de error, contexto de la petición). Son señales complementarias.

## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

- **Reproducibilidad y consistencia**: todo el equipo obtiene exactamente la misma configuración sin depender de que alguien recuerde hacer clic en la UI. Si el stack se destruye y se recrea, los datasources vuelven a aparecer automáticamente.
- **Reducción de errores**: elimina tipeos manuales de URLs, nombres de datasource o UIDs. La configuración se versiona y se revisa como cualquier otro cambio de infraestructura.
- **Infraestructura como Código**: la observabilidad se trata con la misma disciplina que el resto de la infraestructura. No hay "configuración fantasma" que solo existe en la cabeza de quien la creó.

## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

1. **CPU del host** refleja el uso total de CPU de toda la máquina (todos los procesos, todos los contenedores, el kernel). Puede estar al 20% mientras un contenedor específico está al 100% de su límite asignado.
2. **CPU del contenedor** mide el consumo de los procesos dentro de ese contenedor y puede estar afectada por límites de CPU (cgroups). Un contenedor con límite de 0.5 cores puede mostrar 100% aunque el host esté al 10%.

Para alertar sobre una aplicación concreta, usaría la métrica **del contenedor** (o del proceso), porque:
- Refleja exactamente el recurso que consume esa aplicación.
- No se ve afectada por lo que hagan otros procesos en el host.
- Permite detectar problemas específicos de esa app sin ruido del resto del sistema.

## 4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?

1. **Evaluation Interval**: es la frecuencia con la que Grafana/Prometheus evalúa la expresión de la regla de alerta. Si está en 10s, cada 10 segundos se ejecuta la query y se verifica si la condición se cumple.
2. **Pending Period**: es el tiempo mínimo que la condición de alerta debe mantenerse verdadera de forma **continua** para que la alerta cambie de estado `Pending` a `Firing`. Si la condición deja de cumplirse antes de que termine el pending period, la alerta vuelve a `Normal` sin haberse disparado nunca.

El pending period evita falsas alarmas por picos momentáneos: un pico de CPU de 2 segundos no dispara la alerta si el pending period es de 30 segundos.
