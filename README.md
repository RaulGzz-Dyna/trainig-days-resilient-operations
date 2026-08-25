# 🎓 Training Days 2026 - Resilient Operations with Dynatrace

Este repositorio recopila las queries y los pasos de los laboratorios del curso **Resilient Operations with Dynatrace**, parte de los **Training Days 2026**. La idea es tener, en un solo lugar, las queries DQL en orden (listas para copiar/pegar) junto con las acciones de UI que acompañan cada paso de las guías oficiales, para poder repasar o repetir los ejercicios sin depender de las capturas de pantalla del curso.

El curso está organizado en tres laboratorios principales, cada uno documentado en su propio archivo dentro de la carpeta [`labs/`](labs):

- [**🕵️ Lab 1 - Investigation**](labs/lab-1-investigation.md): uso de Grail y Security Investigator para reconstruir, a partir de logs, la causa raíz de un incidente (errores 503, fallas de heartbeat, reinicio del pod en Kubernetes) y reportar los hallazgos en Notebooks.
- [**🛡️ Lab 2 - Vulnerabilities**](labs/lab-2-vulnerabilities.md): identificación y análisis de vulnerabilidades en el entorno monitoreado.
- [**⚔️ Lab 3 - Attacks**](labs/lab-3-attacks.md): detección y análisis de ataques sobre el entorno.

Cada laboratorio documenta las queries en bloques de código `dql` y los pasos de UI (clics, colores de nodo, pivots, etc.) en texto explicativo, en el mismo orden en que aparecen en la guía.
