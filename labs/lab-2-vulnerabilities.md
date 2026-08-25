# 🛡️ Lab 2 - Vulnerabilities

## 📋 Resumen de la app

En este laboratorio hacemos un repaso de la app de **Vulnerabilities** de Dynatrace antes de construir un **Dashboard**. La app consume los eventos del bucket `default_securityevents_builtin` dentro de `security.events`, donde Dynatrace registra cada hallazgo de seguridad detectado (CVEs de terceros, código propio, etc.) junto con su evaluación de riesgo.

Cada vulnerabilidad se identifica de forma única con `vulnerability.display_id`, y Dynatrace la clasifica con distintos atributos que usaremos en las queries de este lab:

- `vulnerability.risk.level`: nivel de riesgo (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`).
- `vulnerability.davis_assessment.exposure_status`: si el proceso vulnerable está expuesto a la red pública (`PUBLIC_NETWORK`).
- `vulnerability.davis_assessment.exploit_status`: si existe un exploit disponible para la vulnerabilidad (`AVAILABLE`).
- `vulnerability.davis_assessment.vulnerable_function_status`: si la función vulnerable del código está realmente en uso (`IN_USE`).
- `vulnerability.davis_assessment.data_assets_status`: si hay data assets sensibles alcanzables desde el proceso vulnerable (`Reachable`).

Estas evaluaciones de Davis (exposición, exploit, función vulnerable y data assets) son las que permiten priorizar qué vulnerabilidades atacar primero, en lugar de tratarlas todas por igual solo según su severidad (CVSS).

## 📊 2.1 - Conteo de vulnerabilidades por severidad (queries)

Queries en orden, listas para copiar/pegar, una por cada gráfica del dashboard.

### 🔴 1. Conteo de vulnerabilidades - Critical

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter vulnerability.risk.level == "CRITICAL"
| filter isNotNull(vulnerability.references.cve)
| filter isNotNull(vulnerability.display_id)
| dedup vulnerability.display_id
| summarize count()
```

### 🟠 2. Conteo de vulnerabilidades - High

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter vulnerability.risk.level == "HIGH"
| filter isNotNull(vulnerability.references.cve)
| filter isNotNull(vulnerability.display_id)
| dedup vulnerability.display_id
| summarize count()
```

### 🟡 3. Conteo de vulnerabilidades - Medium

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter vulnerability.risk.level == "MEDIUM"
| filter isNotNull(vulnerability.references.cve)
| filter isNotNull(vulnerability.display_id)
| dedup vulnerability.display_id
| summarize count()
```

### 🟢 4. Conteo de vulnerabilidades - Low

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter vulnerability.risk.level == "LOW"
| filter isNotNull(vulnerability.references.cve)
| filter isNotNull(vulnerability.display_id)
| dedup vulnerability.display_id
| summarize count()
```

### 🖼️ 5. Bonus - tile de imagen

Agregar una tile de tipo **Image** al dashboard (sin query DQL) y usar la siguiente URL:

```
https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTBkFV1H7cwY376IU9YmlVvA81sY6AONiJhWg&s
```

### 🌐 6. Conteo de vulnerabilidades con Public Network Exposure

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter isNotNull(vulnerability.references.cve)
| dedup vulnerability.display_id
| filter isNotNull(vulnerability.display_id)
| filter vulnerability.davis_assessment.exposure_status == "PUBLIC_NETWORK"
| summarize count()
```

### 💣 7. Conteo de vulnerabilidades con Exploit Available

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter isNotNull(vulnerability.references.cve)
| dedup vulnerability.display_id
| filter isNotNull(vulnerability.display_id)
| filter vulnerability.davis_assessment.exploit_status == "AVAILABLE"
| summarize count()
```

### ⚙️ 8. Conteo de Vulnerable Functions (in use)

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter isNotNull(vulnerability.references.cve)
| dedup vulnerability.display_id
| filter isNotNull(vulnerability.display_id)
| filter vulnerability.davis_assessment.vulnerable_function_status == "IN_USE"
| summarize count()
```

### 🗄️ 9. Conteo de Data Assets Reachable

```dql
fetch security.events
| filter dt.system.bucket == "default_securityevents_builtin"
| filter isNotNull(vulnerability.references.cve)
| dedup vulnerability.display_id
| filter isNotNull(vulnerability.display_id)
| filter vulnerability.davis_assessment.data_assets_status == "Reachable"
| summarize count()
```

<img width="1791" height="725" alt="image" src="https://github.com/user-attachments/assets/8a16199a-b213-41e9-bcee-fae2aaf943aa" />


## 🧬 2.2 - Desglose de vulnerabilidades

Segunda sección del dashboard: un título visual, una variable dinámica de severidad y las gráficas de detalle/agrupación que la usan.

### 🏷️ 10. Título de sección (tile de texto con DQL)

Query para una tile de texto/markdown usada solo como separador visual entre secciones del dashboard:

```dql
data record(a="Vulnerabilidades por grupos")
```

<img width="691" height="751" alt="image" src="https://github.com/user-attachments/assets/a6ffe3ce-e15b-4e93-b77b-c7a51631f88a" />

### 🎚️ 11. Variable dinámica `$Severity`

Crear una variable del dashboard (menú de variables, arriba del dashboard) con estas propiedades:

- Nombre: `Severity`
- Tipo: **List**
- Valores: `CRITICAL,HIGH,MEDIUM,LOW`

Esta variable se usa luego para filtrar dinámicamente las vulnerabilidades por severidad desde los controles del dashboard.

### 📄 12. Detalle de vulnerabilidades (usando `$Severity`)

```dql
fetch security.events, scanLimitGBytes:1
| filter dt.system.bucket == "default_securityevents_builtin"
| filter vulnerability.resolution.status == "OPEN"
     and vulnerability.mute.status != "MUTED"
| filter in(vulnerability.davis_assessment.level, $Severity)
| fields vulnerability.display_id, event.status, vulnerability.risk.level, vulnerability.title, vulnerability.davis_assessment.score, vulnerability.description
| sort vulnerability.davis_assessment.score desc
```

### 💻 13. Desglose por tecnología afectada

```dql
fetch security.events
| dedup vulnerability.display_id
| filter event.kind == "SECURITY_EVENT"
| summarize count(), by:{vulnerability.technology}
| sort `count()` desc
| filter isNotNull(vulnerability.technology)
```

### 🖥️ 14. Conteo de vulnerabilidades por entidades afectadas

```dql
fetch security.events
| fields Entidades_Afectadas = affected_entity.affected_processes.names
| filter Entidades_Afectadas != "null"
| summarize Total = count(), by:{Entidades_Afectadas}
| sort Total desc
```

### 🏆 15. Top 10 vulnerabilidades activas

```dql
fetch security.events
| dedup vulnerability.display_id
| summarize Total = count(), by:{vulnerability.title}
| sort Total desc
| limit 10
```

<img width="1797" height="718" alt="image" src="https://github.com/user-attachments/assets/be9ecb8d-3a17-425c-8392-c4e95cf41a4d" />

