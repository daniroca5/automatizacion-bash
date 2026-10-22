# Automatización con Bash

Scripts para **auditar logs**, hacer **copias de seguridad** y consultar **usuarios** en Linux, programados con **cron**. Salen de la asignatura *Implantación de Sistemas Operativos* (1.º de ASIR), en la parte de scripting y tareas programadas.

<p align="center"><img src="img/auditor-logs-ejecucion.png" width="760" alt="Ejecución real de auditor_logs.sh"></p>

## La idea: scripts que funcionen sin nadie mirando

Un script que va bien a mano puede fallar en cron. El motivo es que cron arranca con un **entorno mínimo**: otro `PATH`, otro directorio de trabajo y sin terminal. Por eso todos los scripts siguen las mismas reglas:

- **Cabecera común** con los códigos de salida (`0` correcto, `2` no existe, `3` permisos, `4` uso incorrecto) y las funciones `log`, `err` y `usage`. Así cada error queda registrado con fecha.
- **Modo estricto**: `set -euo pipefail`.
  - `-u` corta si una variable está vacía. Evita el clásico `rm -rf "$DIR"/*` con `$DIR` sin definir.
  - `pipefail` detecta que falla `mysqldump` aunque `gzip` termine bien.
- **Rutas absolutas** y `PATH` fijado al principio del script.
- **Bloqueo** con `mkdir` (es atómico) y **`trap`** para limpiarlo al salir. Así no se solapan dos ejecuciones lanzadas por cron.
- **Idempotencia**: ejecutarlo dos veces no debe estropear nada, y los logs se rotan o vacían para que no crezcan sin control.

## `auditor_logs.sh`: auditoría de ficheros de log

Recibe un directorio, una extensión y, si se quiere, un patrón. Por cada fichero cuenta las líneas y los bytes, y mira si contiene el patrón. Al final da un resumen y lo deja en su propio log. La captura de arriba es una ejecución real: primero con datos de prueba y después sin argumentos, donde sale el uso y el código `4`.

```bash
patron="${3:-}"                      # tercer argumento opcional (seguro con set -u)
for f in "$dir_logs"/*"$ext"; do
  [[ -f "$f" ]] || continue          # guardia: solo ficheros reales
  lineas=$(wc -l < "$f")
  bytes=$(stat -c %s -- "$f")
  if [[ -n "$patron" ]] && grep -q -- "$patron" "$f"; then
    ((total_con_patron++))
  fi
  ((total_ficheros++)); ((total_lineas += lineas)); ((total_bytes += bytes))
done
```

## Copias de seguridad programadas

Un script de copia empaqueta y comprime con fecha en el nombre (`backup_AAAAMMDD.tar.gz`), así que cada día deja un fichero distinto:

```bash
fecha=$(date +%Y%m%d)
tar -czf "backup_$fecha.tar.gz" *.sh
```

<p align="center"><img src="img/backup-ejecucion.png" width="560" alt="Ejecución real del script de copia"></p>

Y se programa en cron con ruta absoluta, enviando la salida y los errores a un log:

```cron
# m  h  dom mes dow  comando
30   2  *   *   *    /home/dani/scripts/backup.sh >> /home/dani/logs/backup.log 2>&1
```

## Usuarios

Consultas sobre `/etc/passwd` con `grep` y `cut` (por ejemplo, la shell de un usuario) y un informe de quién tiene sesión abierta (`who`), redirigido a un fichero con fecha.

## Lo que me llevo

Que la parte difícil de un script **no es que funcione, sino que falle bien**: con un código de salida que se pueda interpretar, un registro de lo que pasó y sin dejar el sistema a medias.

---

Daniel Roca Martínez · [Currículum](https://daniroca5.gamer.gd/cv/) · [LinkedIn](https://www.linkedin.com/in/daniel-r-m-02b369164/)
