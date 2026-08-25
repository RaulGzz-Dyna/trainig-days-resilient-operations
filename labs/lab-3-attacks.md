# ⚔️ Lab 3 - Attacks

## 📋 Resumen de la app

En este laboratorio hacemos un repaso de la app de **Attacks / Application Security** de Dynatrace, que muestra los intentos de explotación detectados en tiempo real sobre nuestros servicios (Runtime Attack Protection). A diferencia del Lab 2 (que analiza vulnerabilidades conocidas, presentes o no explotadas), aquí trabajamos sobre `security.events` filtrando por `event.type == "DETECTION_FINDING"`: cada registro es un intento de ataque real detectado y bloqueado/observado en runtime.

Los campos clave que usaremos:

- `entry_point.url.path`: el endpoint/URL por el que entró el intento de ataque (dónde nos están atacando).
- `finding.title`: la descripción/tipo de la vulnerabilidad o técnica de ataque detectada (qué nos están haciendo).
- `actor.ips`: la IP de origen del ataque (quién nos está atacando).

Seguimos trabajando sobre el **mismo dashboard** de Lab 2, agregando una nueva sección dedicada a estos hallazgos de explotación (Exploit Findings).

## 💥 3.1 - Exploit Findings (queries)

### 🏷️ 1. Título de sección (tile de texto con DQL)

Query para una tile de texto/markdown usada solo como separador visual entre secciones del dashboard:

```dql
data record(a="Exploit Findings")
```

### 🚪 2. Top 5 Entry Points

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| summarize Total = COUNT(), by:{entry_point.url.path}
| limit 5
| sort Total desc
```

### 📝 3. Top 5 Findings (descripciones de las vulnerabilidades)

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| summarize Total = count(), by:{finding.title}
| limit 10
| sort Total desc
```

### 🌍 4. Top 5 Actor IPs detectadas

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| summarize Total = count(), by:{actor.ips}
| limit 10
| sort Total desc
```

## 📈 3.2 - Evolución de Exploit Findings (maketimeseries)

Mismas tres gráficas anteriores, pero reemplazando `summarize` por `maketimeseries` para verlas evolucionar en el tiempo.

### 🚪 5. Top 5 Entry Points (serie temporal)

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| maketimeseries Total = COUNT(), by:{entry_point.url.path}
| limit 5
| sort Total desc
```

### 📝 6. Top 5 Findings (serie temporal)

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| maketimeseries Total = count(), by:{finding.title}
| limit 5
| sort Total desc
```

### 🌍 7. Top 5 Actor IPs (serie temporal)

```dql
fetch security.events
| filter event.type == "DETECTION_FINDING"
| maketimeseries Total = count(), by:{actor.ips}
| limit 5
| sort Total desc
```
