# trainig-days-resilient-operations

## Lab 1.1 - Find Logs From Grail (queries)

Queries en orden, listas para copiar/pegar siguiendo los pasos de la guia.

### 1. Fetch logs del cluster

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
```

### 2. Agregar `summarize` (Log Sources)

```dql
| summarize count(), by: { k8s.cluster.name, k8s.container.name, dt.process.name }
```

### 3. Fetch logs de ISTIO (quitar el `summarize` y filtrar por `istio-proxy`)

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
```

### 4. Extraer `response_code` del JSON

```dql
| parse content, "json{int:response_code}(flat=true)"
```

### 5. Crear serie temporal con `makeTimeseries`

```dql
| makeTimeseries count(default: 0), by: response=toString(response_code), interval: 1m
```

### 6. Query completa (Response code distribution)

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
| parse content, "json{int:response_code}(flat=true)"
| makeTimeseries count(default: 0), by: response=toString(response_code), interval: 1m
```

### 7. Filtrar solo respuestas 503 (quitar `makeTimeseries` antes de agregar esto)

```dql
| filter response_code == 503
```

### 8. Obtener el primer timestamp

```dql
| summarize timestamp = min(timestamp)
```

### 9. Query final (What happened before the first 503?)

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
| parse content, "json{int:response_code}(flat=true)"
| filter response_code == 503
| summarize timestamp = min(timestamp)
```

## Lab 1.2 - What Happened Before the First 503 (queries)

Continuación del Lab 1.1. Aparece un nuevo campo virtual `timestamp_diff` en la tabla de resultados, que indica la diferencia de tiempo entre el campo timestamp y el timestamp de referencia (por ahora, 0 ms).

### 1. Quitar las últimas dos líneas (filtro de 503 y `summarize`) y filtrar eventos anteriores al primer 503

Se puede agregar el filtro de timestamp haciendo clic derecho sobre el valor del timestamp en la tabla de resultados y eligiendo **Timestamp filters → Earlier than**, o usando el menú **Reference time → Earlier than** en la parte superior de la Investigation.

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
| parse content, "json{int:response_code}(flat=true)"
| filter timestamp < toTimestamp("2025-02-14T05:32:01.000000000Z")
```

Ejecutar la query. Muchos de los eventos resultan en 200 (OK) o 401 (Unauthorized), que no son interesantes por ahora. Cambiar el color de este nodo a **azul** (clic derecho sobre el nodo → Color → azul) para referencia futura.

### 2. Filtrar los códigos 200 y 401

Mantener presionada la tecla Ctrl (Cmd en Mac), seleccionar los valores 401 y 200 en la tabla de resultados, clic derecho → Filter. Esto agrega automáticamente:

```dql
| filter in(response_code, {200, 401})
```

Agregar `not` para excluir esos registros:

```dql
| filter not in(response_code, {200, 401})
```

Ejecutar la query. Si no hay resultados, cambiar el color del nodo a **naranja** para referencia futura.

### Response Latency - extraer duración y generar métricas

Volver al nodo azul creado antes. Cambiar el `parse` para extraer también `duration` de los logs de istio:

```dql
| parse content, "json{string:response_code, int:duration}(flat=true)"
```

Generar métricas de latencia con `makeTimeseries`:

```dql
| makeTimeseries {avg(duration), max(duration)}, interval:1m
```

Query final (response latency chart):

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
| parse content, "json{string:response_code, int:duration}(flat=true)"
| filter timestamp < toTimestamp("2025-02-14T05:32:01.000000000Z")
| makeTimeseries {avg(duration), max(duration)}, interval:1m
```

Renombrar el nodo como **"response latency chart"** (clic derecho → Rename). Nada fuera de lo normal en la latencia.

### Events Based on `start_time` Field

El campo `timestamp` representa el fin de la transacción (cuando se recibió la respuesta). Como la solicitud sospechosa pudo haber tardado mucho en responder, puede aparecer en los logs después del primer 503 si se ordena por `timestamp`. Para verificar esto, hay que filtrar usando el campo `start_time` en vez de `timestamp`.

Volver al primer nodo **naranja**. Modificar el patrón DPL del `parse` para extraer `start_time` como timestamp:

```dql
| parse content, "json{int:response_code, timestamp('yyyy-MM-ddTHH:mm:ss.SZ'):start_time}(flat=true)"
```

Cambiar el filtro de `timestamp` a `start_time`. Query final:

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.container.name == "istio-proxy"
| parse content, "json{int:response_code, timestamp('yyyy-MM-ddTHH:mm:ss.SZ'):start_time}(flat=true)"
| filter start_time > toTimestamp("2025-02-14T05:32:01.000000000Z")
| filter not in(response_code, {200, 401})
```

Ejecutar la query. Esta vez aparece un resultado: un único evento con código **500 (Server Error)**. La columna `timestamp_diff` muestra que este error 500 se escribió en los logs más de 20 segundos después del primer 503.

Se puede inspeccionar el payload JSON completo con clic derecho sobre el campo `content` → **View field details**.

Cambiar el color del nodo a **verde** para referencia futura.

### What Was Executed With the Request? (Distributed Tracing)

Para investigar qué ocurrió durante esta solicitud, usamos Dynatrace Distributed Tracing junto con el `trace_id` adjunto al evento de log.

1. Clic derecho sobre el valor de `trace_id` en la tabla de resultados → **Add to evidence list → Create a new list**. Nombrar la lista **"errors trace"**.
2. Buscar todos los logs relacionados con ese `trace_id`. Hay 4 formas de hacerlo (se documentan todas, se usa la segunda):
   - **Filter → content field** desde el menú de la Evidence list, que agrega todos los elementos de la lista al comando `filter` de la query.
   - **Pivot query by evidence** desde el menú de la Evidence list, para definir una nueva query donde se usan los valores de evidencia (crea nuevos nodos, cada uno con su propio valor de evidencia). *(Método usado en este paso.)*
   - Clic derecho sobre el registro en los resultados → **Pivot query by → Trace ID**, que crea un nuevo nodo de query bajo el nodo actual.
   - Clic derecho sobre el `trace_id` en la tabla → **Pivot query by → Use custom dimension**, para definir una query personalizada con los valores elegidos.

## Lab 1.3 - Pivot From Evidence Values (queries)

### 1. Pivot query by evidence (usando la lista "errors trace")

Navegar hasta el primer nodo (el de todos los logs del cluster). Desde el menú de la Evidence list **"errors trace"**, elegir **Pivot query by evidence**. En el modal de pivoting, en vez del comando `search` por defecto, usar un `filter`:

```dql
| filter trace_id == "$value"
```

Query final construida:

```dql
fetch logs
| filter trace_id == "$value"
```

El macro `$value` se reemplaza por cada valor de la Evidence list (por ahora solo hay uno; si se agregan más valores, se generan más nodos de query).

Seleccionar **Pivot 2 queries** para ejecutar. Se crean nuevas ramas bajo el primer nodo, una por cada valor de `trace_id`.

Se puede eliminar el nodo vacío que tiene `trace_id == "demo"` (era solo demostrativo). Con el nodo restante (el `trace_id` real):

- Cambiar su color a **morado**.
- Activar el modo multilínea en la columna `content` (menú Column, arriba de la tabla) para leer mejor logs largos como stack traces.
- Alternativamente, abrir **Field Details** desde el menú contextual del campo y navegar entre registros con las flechas del teclado.
- Renombrar el nodo a **"Failing request"**.

Al comparar con el log de ejemplo de un POST request esperado (sección "Background"), este registro es distinto — en particular el stack trace, que revela irregularidades en el heartbeat que dejaron al sistema en un estado inesperado.

### What Else Did the Application Do After This?

Pasos para ver qué hizo la aplicación después de este evento:

1. Quitar el último `filter` de `trace_id` de la query.
2. Filtrar por el mismo pod: clic derecho sobre el valor de `k8s.pod.name` → **Filter for**.
3. Filtrar por el mismo container: clic derecho sobre el valor de `k8s.container.name` → **Filter for**.
4. Ver solo eventos posteriores: clic derecho sobre el `timestamp` del primer registro → **Timestamp filters → Later than**.
5. Agregar `sort` por `timestamp` ascendente.

Query final:

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter k8s.pod.name == "heartbeat-matcher-service-78f6c784c9-2g77v"
| filter k8s.container.name == "heartbeat-matcher-service"
| filter timestamp > toTimestamp("2025-02-14T05:32:00.000000000Z")
| sort timestamp
```

Ejecutar la query. Resultado: el servicio recibió un comando de shutdown (graceful) y la aplicación se reinició:

```
Commencing graceful shutdown. Waiting for active requests to complete.
```

¿Por qué ocurrió esto? Habrá que seguir investigando en otros logs.

## Lab 1.4 - What Did Kubernetes Do at That Time (queries)

### 1. Filtrar los logs del control plane de K8S ("Linux System")

Copiar las últimas dos líneas (`filter timestamp` y `sort`) de la query anterior. Ir al segundo nodo gris ("Log sources"), quitar la línea de `summarize`, hacer clic derecho sobre el valor **"Linux System"** en la columna `dt.process.name` → **Filter for**, y pegar al final las líneas copiadas.

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter dt.process.name == "Linux System"
| filter timestamp > toTimestamp("2025-02-14T05:32:00.000000000Z")
| sort timestamp
```

Ejecutar la query. Los logs muestran que los heartbeats/health checks empezaron a fallar después de la request, y el control plane de K8S reinició el contenedor de forma graceful al marcarlo como unhealthy:

```
Killing container with a grace period
```

**Secuencia de eventos reconstruida:**

1. Llega una request que produce un error y causa problemas en el servicio.
2. Por el error, aumenta la carga del servidor y empiezan a fallar los health check endpoints.
3. Por los heartbeats fallidos, K8S decide reiniciar el contenedor y envía la señal de "kill container".
4. La señal reinicia el servicio de forma graceful.
5. Las requests que llegan durante el reinicio devuelven código **503** (service not available).

### 2. Encontrar el fin del ciclo de reinicio

Buscar el primer evento exitoso después del reinicio (indica el fin de la secuencia de reinicio). Clic derecho sobre su `timestamp` → **Timestamp filter → Earlier than**:

```dql
| filter timestamp < toTimestamp("2025-02-06T05:35:01.000000000Z")
```

Ejecutar la query. Cambiar el color del nodo a **neón** y renombrarlo a **"Restart Cycle"**.

### What Was The Request That Killed It All? (análisis del Heartbeat Fragment)

Volver al primer nodo **morado** para analizar el fragmento de heartbeat que se envió, posible origen del problema.

1. Filtrar solo el registro que contiene el "Heartbeat Fragment": seleccionar ese texto en el campo `content`, clic derecho → **Filter for**.

```dql
| filter contains(content, "Heartbeat Fragment")
```

2. Extraer el bitmap del heartbeat como un array (hasta 100 elementos de 1 carácter):

```dql
| parse content, "LD 'Heartbeat Fragment: ' array{ LD{1}:a }{,100}:binary"
```

3. Generar una key por cada valor del bitmap usando `iIndex()`:

```dql
| fields heartbeats = record(value = binary[], key = concat("position_", iIndex()))
```

4. Usar `expand` para separar los elementos del array en registros individuales y `fieldsFlatten` para separar los campos del record:

```dql
| expand heartbeats
| fieldsFlatten heartbeats
```

5. Quitar el campo `heartbeats` y dejar solo `key` y `value`:

```dql
| fieldsRemove heartbeats
```

Query final:

```dql
fetch logs
| filter k8s.cluster.name == "prod.cupid.cluster"
| filter contains(trace_id, "e0f28a67b2854b1fa8442d9df9f3deeb")
| filter contains(content, "Heartbeat Fragment")
| parse content, "LD 'Heartbeat Fragment: ' array{ LD{1}:a }{,100}:binary"
| fields heartbeats = record(value = binary[], key = concat("position_",iIndex()))
| expand heartbeats
| fieldsFlatten heartbeats
| fieldsRemove heartbeats
```

Los resultados no dicen mucho directamente en el Security Investigator; se usará Notebooks para darles mejor formato. Cambiar el color del nodo a **dorado (Gold)** y renombrarlo a **"Heartbeats"**.

### Report Your Investigation Results in Notebooks

Para generar un reporte, seleccionar (Ctrl/Cmd + clic) los siguientes nodos y elegir **Download nodes as → Notebooks document** (marcar el checkbox **Results** para incluir resultados, luego **Download**):

- 2º nodo gris — "Log sources"
- 4º nodo gris — "Response code distribution"
- 2º nodo azul — "Response latency chart"
- 1º nodo morado — request que causó el error
- Nodo neón — reinicio de la aplicación
- Nodo dorado — bitmap de heartbeats

### Illustrate Your Report

1. Abrir **Notebooks**, elegir **Upload** y seleccionar el documento descargado desde el Security Investigator.
2. Ajustar el tamaño de la tabla de resultados de "log sources" para que llene el contenido.
3. Cambiar las visualizaciones de "response code distribution" y "response latency chart" a **line chart** y ocultar el input (**Hide the input**).
4. Ajustar el tamaño de todas las secciones para que el reporte se vea prolijo; agregar secciones de markup donde haga falta para completar la narrativa de la investigación.
5. En la sección del Heartbeat Fragment, abrir **Options** → elegir visualización **Honeycomb** → en la paleta de colores, elegir **Fireplace**.


Se deberia de ver algo similar a esta imagen con los cambios aplicados
<img width="1254" height="526" alt="image" src="https://github.com/user-attachments/assets/d713fee5-de4b-47a8-a598-22b4510d98a9" />
