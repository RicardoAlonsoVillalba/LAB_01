Actividad 2 — Que el servidor administre su propia bitácora
Estudiante: Ricardo Alonso Villalba

Comando ejecutado:
ALTER SYSTEM SET logging_collector = on;
-- Tras reiniciar el servicio:
CREATE SCHEMA dba;
REVOKE ALL ON SCHEMA dba FROM PUBLIC;
CREATE VIEW dba.bitacora AS
SELECT t.n AS num, t.linea
FROM regexp_split_to_table(pg_read_file(pg_current_logfile()), E'\n')
WITH ORDINALITY AS t(linea, n)
WHERE t.linea <> '';

Resultado obtenido:
Respuestas e Interpretación¿Por qué logging_collector requiere reinicio y log_statement solo recarga? Porque logging_collector inicia un proceso secundario dedicado en el arranque del servidor para capturar stderr, mientras que log_statement solo ajusta el filtro de escritura de los procesos activos.   
¿Qué ventaja tiene que la bitácora se lea con SQL y no con una herramienta del SO? ¿Qué riesgo introduce? Permite consultar y filtrar registros de forma estándar sin depender del acceso al sistema operativo ni de cómo se desplegó el contenedor. El riesgo es que pg_read_file() puede leer cualquier archivo accesible para el usuario del sistema postgres si se otorgan permisos indebidos.   
¿Quién debería poder consultar dba.bitacora? Exclusivamente los superusuarios (DBA) o roles específicos de auditoría interna autorizados.
