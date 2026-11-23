# Informe Técnico — LAB-01: Bitácoras y Auditoría del Dato

## 1. Despliegue y situación inicial
El servidor se desplegó mediante Docker (`postgres:16`, contenedor `dba-u3-pg`) sobre la base de datos `comercial`. En su estado inicial, `logging_collector` estaba en `off`, `log_connections` en `off` y `log_statement` en `none`. Aunque existían controles de acceso de la Unidad II (que restringían a `app_ventas` a `SELECT` y `UPDATE`), el servidor no registraba conexiones ni sentencias, imposibilitando demostrar quién alteró el RFC de María García (`id = 3`) o cuál era su valor original.

## 2. Configuración aplicada
Se utilizó `ALTER SYSTEM` (escribiendo en `postgresql.auto.conf`) para garantizar portabilidad:
* `logging_collector = on` *(Requirió reinicio)*: Captura `stderr` en archivos rotativos dentro de `$PGDATA/log/`.
* `log_connections = on` y `log_disconnections = on` *(Recarga)*: Registra inicio, autenticación, autorización y cierre de sesión con duración.
* `log_line_prefix = '%m [%p] user=%u db=%d app=%a client=%h '` *(Recarga)*: Añade marca temporal UTC, PID, rol de sesión, base de datos, aplicación cliente e IP de origen.
* `log_statement = 'mod'` *(Recarga, tras probar `ddl`)*: Registra definiciones de estructura, cambios de privilegios y sentencias de modificación de datos (`INSERT`, `UPDATE`, `DELETE`).

## 3. Eventos observados
Se verificaron e interpretaron los 6 tipos de eventos auditables:
1. **Configuración:** Registro automático en el PID principal (`parameter "log_statement" changed to "mod"`) tras ejecutar `pg_reload_conf()`.
2. **Acceso:** Intentos fallidos (`FATAL: password authentication failed`) para `app_ventas` y `usuario_inexistente`, además de conexiones exitosas (`connection authorized`) desde `127.0.0.1`.
3. **Privilegio:** Ejecución de `REVOKE UPDATE` a `soporte_ana` y `GRANT UPDATE` a `app_ventas`.
4. **Estructura:** Creación de columna `segmento` (`ALTER TABLE clientes ADD COLUMN...`).
5. **Datos:** Ejecución de `UPDATE clientes SET telefono...` por `app_ventas` desde `app-ventas-web`.
6. **Error:** Denegación de borrado (`ERROR: permission denied for table clientes`) registrada automáticamente gracias a `log_min_error_statement = error`.

## 4. Auditoría del dato
Se creó el esquema privado `auditoria` (sin privilegios para `PUBLIC`) y la tabla `auditoria.cambios_clientes` almacenando `valores_antes` y `valores_despues` en formato `JSONB`. 
* **Papel de `SECURITY DEFINER`:** Permite que la función `auditoria.registrar_cambio_cliente()` inserte registros de auditoría cuando `app_ventas` modifica `clientes`, sin otorgar a `app_ventas` permisos de escritura o borrado sobre la tabla de auditoría. Se fijó `SET search_path = pg_catalog, public` para evitar secuestro de esquema.
* **Papel de `session_user`:** Dentro de una función `SECURITY DEFINER`, `current_user` devuelve al propietario (`postgres`). Usar `session_user` garantiza registrar al usuario real que inició la sesión (`app_ventas`).
* **Resultado del incidente:** Al actualizar el RFC de `id = 3`, la consulta `CROSS JOIN LATERAL jsonb_each` demostró exactamente que el campo `rfc` cambió de `GAMA780912XYZ` a `GAMA780912ZZZ` por `app_ventas`.

## 5. Limitaciones
* **Lecturas (`SELECT`):** Ni `log_statement = 'mod'` ni los disparadores (`FOR EACH ROW`) registran consultas de lectura; `soporte_ana` leyó el RFC sin dejar rastro en estas capas (requeriría `pgaudit` o `log_statement = 'all'`).
* **Volumen:** Un `UPDATE` masivo sin `WHERE` genera 1 línea en la bitácora del servidor, pero `N` filas en `auditoria.cambios_clientes`, impactando almacenamiento e I/O en tablas de millones de filas.
* **Administradores con control total:** El superusuario `postgres` pudo desactivar el trigger (`DISABLE TRIGGER`), alterar un dato sin dejar rastro en `cambios_clientes` y reactivarlo. Aunque la bitácora del servidor registró el `ALTER TABLE`, un administrador de sistema operativo también podría alterar el archivo `.log` local.

## 6. Riesgos de la bitácora
* **Exposición de secretos:** Ejecutar `CREATE ROLE ... PASSWORD '...'` con `log_statement` activo escribe la contraseña en texto plano en el log. Debe usarse `\password` (que envía hash SCRAM) y sanitizar logs antes de exportarlos.
* **Pérdida y acceso no autorizado:** Si el contenedor se destruye o los archivos rotan sin respaldo externo, se pierde la evidencia. Asimismo, el acceso a `dba.bitacora` debe restringirse exclusivamente a roles auditores/DBA.

## 7. Cronología y conclusión (Respuestas a Dirección)

### Cronología de Eventos
| # | Fecha/Hora (UTC) | Usuario | Cliente / App | Tipo | Qué ocurrió | ¿Exitoso? | Fuente |
|---|---|---|---|---|---|---|---|
| 1 | T1 | `postgres` | `[local]` / `psql` | Configuración | Activación de `log_connections`, `log_line_prefix` y `log_statement='ddl'` | Sí | `dba.bitacora` (PID 1) |
| 2 | T2 | `app_ventas` | `127.0.0.1` / `[unknown]` | Acceso | Intento de conexión con contraseña errónea | No (`FATAL`) | `dba.bitacora` |
| 3 | T3 | `postgres` | `[local]` / `psql` | Privilegio / Estructura | `GRANT UPDATE` a `app_ventas` y `ADD COLUMN segmento` | Sí | `dba.bitacora` |
| 4 | T4 | `app_ventas` | `127.0.0.1` / `app-ventas-web` | Datos | Cambio de teléfono en `id=3` (`log_statement='mod'`) | Sí | `dba.bitacora` |
| 5 | T5 | `soporte_ana` | `127.0.0.1` / `psql` | Error | Intento de `DELETE FROM clientes WHERE id=5` | No (`ERROR`) | `dba.bitacora` |
| 6 | T6 | `app_ventas` | `127.0.0.1` / `app-ventas-web` | Datos | Cambio de RFC en `id=3` (`GAMA780912XYZ` -> `GAMA780912ZZZ`) | Sí | `auditoria.cambios_clientes` (`id_evento=1`) |
| 7 | T7 | `app_ventas` | `127.0.0.1` / `app-ventas-web` | Error | Intento de borrar `auditoria.cambios_clientes` y desactivar trigger | No (`ERROR`) | `dba.bitacora` |
| 8 | T8 | `postgres` | `[local]` / `psql` | Estructura / Datos | Desactivación de trigger, cambio de RFC a `GAMA780912GHI` y reactivación | Sí | `dba.bitacora` |

### Respuestas a Dirección (Basadas estrictamente en la evidencia)
1. **¿Quién hizo el cambio de RFC?**
   * *Primera vez (Actividad 3):* **No hay evidencia** (la bitácora y el disparador aún no estaban configurados).
   * *Segunda vez (Actividad 7):* **No hay evidencia** del cambio de RFC (`log_statement` estaba en `ddl`, solo registró conexión/desconexión de `app_ventas`).
   * *Tercera vez (Actividad 12):* El rol **`app_ventas`** (`id_evento = 1`).
   * *Cuarta vez (Actividad 14):* El rol **`postgres`** (registrado en `dba.bitacora` tras desactivar el trigger).
2. **¿Cuándo?** En las marcas de tiempo UTC exactas registradas en la columna `momento` de `auditoria.cambios_clientes` (para el evento 1) y el prefijo `%m` de `dba.bitacora` (para el evento de `postgres`). Para las Actividades 3 y 7 **no hay evidencia** de hora de ejecución.
3. **¿Desde qué aplicación o equipo?** Para el cambio de `app_ventas`: desde la IP **`127.0.0.1`** declarando la aplicación **`app-ventas-web`**. Para el cambio de `postgres`: conexión **`[local]`** mediante **`psql`**.
4. **¿Hubo intentos de acceso fallidos antes del cambio?** Sí. La bitácora registra intentos fallidos (`FATAL: password authentication failed`) previos desde `127.0.0.1` tanto para el usuario `app_ventas` como para `usuario_inexistente`.
5. **¿Qué valor tenía el RFC antes del cambio?** Para el cambio auditado de `app_ventas` (`id_evento = 1`), el valor anterior era **`GAMA780912XYZ`** (y pasó a `GAMA780912ZZZ`). Para la Actividad 3 **no hay evidencia** del valor previo.

---

## Preguntas de Reflexión
1. **¿Por qué la auditoría no puede configurarse después de un incidente para investigarlo?** Porque los eventos que ocurren mientras el registro está apagado no se almacenan en disco ni en memoria histórica; la auditoría es preventiva en su activación y forense en su lectura, no retroactiva.
2. **¿Por qué conviene que la bitácora la administre el propio servidor (`logging_collector = on`) y no el mecanismo de despliegue?** Porque estandariza la ubicación, rotación y consulta de los registros mediante funciones SQL internas (`pg_current_logfile()`, `pg_read_file()`), independizando la evidencia de si el motor corre en Linux, Windows o Docker.
3. **¿Por qué el prefijo de línea (`log_line_prefix`) es tan importante como el nivel de `log_statement`?** Porque `log_statement` registra *qué* consulta se ejecutó, pero sin los marcadores `%u`, `%d`, `%h` y `%m` del prefijo es imposible vincular esa sentencia con *quién* la ejecutó, *cuándo* y *desde qué dirección IP*.
4. **¿Qué diferencia práctica hay entre registrar la sentencia y registrar el cambio en el dato?** Registrar la sentencia muestra la intención SQL (`UPDATE clientes SET rfc = ... WHERE ciudad = 'Culiacán'`), pero no revela cuántas filas afectó ni qué valores tenían antes. Registrar el cambio en el dato (`OLD` y `NEW` en `JSONB`) permite reconstruir y restaurar el estado exacto de cada registro modificado.
5. **¿Qué evidencia ofrece `pg_stat_activity` que no ofrece la bitácora, y viceversa?** `pg_stat_activity` ofrece una fotografía en tiempo real de las sesiones activas, bloqueos y consultas en curso en ese instante, pero desaparece al terminar la sesión. La bitácora ofrece el historial cronológico persistente de lo que ya ocurrió y de los intentos fallidos que nunca llegaron a abrir sesión.
6. **Si el administrador puede cambiar `log_statement` a `none` y desactivar el disparador, ¿quién audita al administrador?** En un entorno empresarial, se aplica segregación de funciones: las bitácoras se envían en tiempo real mediante `syslog` o agentes recolectores hacia un servidor externo centralizado (SIEM) administrado por el equipo de Seguridad de la Información, al cual el DBA no tiene acceso de escritura ni borrado.
