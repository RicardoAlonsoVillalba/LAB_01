# Actividad 1 — Revisar la configuración de registro por defecto
**Estudiante:** Ricardo Alonso Villalba

## Comando ejecutado
```sql
SELECT name, setting, source FROM pg_settings
WHERE name IN ('log_destination', 'logging_collector', 'log_directory', 'log_filename',
'log_connections', 'log_disconnections', 'log_line_prefix',
'log_statement', 'log_min_error_statement', 'log_timezone') ORDER BY name;


Respuestas e InterpretaciónCon esta configuración, ¿se registra quién se conecta? No, porque el parámetro log_connections se encuentra en off por defecto.
¿Se registran las sentencias UPDATE? No, porque log_statement tiene el valor none.
¿La línea de bitácora incluye el usuario o la dirección IP del cliente? No, el prefijo por defecto (%m [%p] ) solo incluye la marca temporal y el identificador del proceso (PID).
¿En qué zona horaria se registran los eventos?
¿Por qué importa al reconstruir una cronología? Se registran en Etc/UTC. Importa porque permite correlacionar registros entre servidores, contenedores y aplicaciones sin desfases por horarios locales o cambios de horario.   Con logging_collector = off, ¿quién decide dónde termina la bitácora? El mecanismo con el que se desplegó el servidor (en este caso, Docker captura la salida stderr), no PostgreSQL.
