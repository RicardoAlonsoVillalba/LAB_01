cat << 'EOF' > README.md
# LAB-01 — Bitácoras y auditoría del dato en PostgreSQL

## Información del Despliegue
* **Motor:** PostgreSQL 16.x (Imagen oficial Docker `postgres:16`)
* **Contenedor:** `dba-u3-pg` (Puerto `5432` mapeado a `127.0.0.1`)
* **Base de datos:** `comercial`
* **Autenticación TCP:** Ajustada de `trust` a `scram-sha-256` en `pg_hba.conf`

## Estructura del Repositorio
* `sql/`: Scripts reproducibles de configuración del colector, vista de auditoría, roles, tabla `auditoria.cambios_clientes`, función disparadora y consulta de diferencias `JSONB`.
* `evidencias/`: Documentación paso a paso de las 17 actividades, bitácora exportada y sanitizada (`bitacora-lab01.log`) y su control de integridad (`bitacora-lab01.sha256`).
* `informe.md`: Reporte ejecutivo técnico, cronología del incidente y respuestas de reflexión.
