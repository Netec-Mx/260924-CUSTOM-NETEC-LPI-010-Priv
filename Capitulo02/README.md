# Laboratorio 8: Rescate de un Servidor Inundado de Logs y con Boot Corrupto

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 131 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción General

En este laboratorio se recuperará la máquina virtual `srv-lab` de dos fallos controlados: saturación de almacenamiento causada por mensajes excesivos en el journal y un fallo de arranque provocado por una entrada inválida en `/etc/fstab`. Se aplicará un método de diagnóstico basado en evidencias: observar síntomas, delimitar el incidente, identificar el componente causante, contenerlo, recuperar el espacio y establecer controles persistentes.

El resultado final será una máquina `srv-lab` operativa, con límites persistentes para `systemd-journald`, una configuración válida de montajes y la verificación de continuidad de los servicios creados en laboratorios anteriores. Al finalizar, se conservará una instantánea denominada `lab08-recuperado`.

## Objetivos de Aprendizaje

- [ ] Identificar el origen de una saturación de almacenamiento mediante `df`, `du`, `find`, `journalctl` y `systemctl`.
- [ ] Contener una inundación de logs, reducir el journal y establecer límites persistentes para `systemd-journald`.
- [ ] Recuperar el arranque de Linux tras un error deliberado en `/etc/fstab`.
- [ ] Utilizar la consola de VirtualBox, GRUB y `emergency.target` para reparar un sistema que no inicia normalmente.
- [ ] Verificar que `telemetry-api.service`, `node_exporter.service` y `boot-validation.service` continúan funcionando después de la recuperación.

## Prerrequisitos

Debe contar con los siguientes conocimientos y accesos:

- Conocimiento práctico de `journalctl`, `systemctl`, `df`, `du`, `find`, `mount`, `findmnt` y edición segura de archivos de configuración.
- Comprensión básica de unidades de `systemd`, prioridades de logs y persistencia del journal.
- Conocimiento de la estructura de `/etc/fstab` y de los riesgos de modificarlo.
- Acceso a Oracle VM VirtualBox 7.0.22.
- Acceso a la consola local de la máquina virtual `srv-lab`, incluido el menú GRUB.
- Usuario administrativo `labadmin` con privilegios `sudo`.
- Instantánea funcional `lab07-operativo` disponible antes de comenzar.
- No realizar los fallos de este laboratorio sobre el sistema anfitrión ni sobre una máquina distinta de `srv-lab`.

## Entorno de Laboratorio

### Máquinas virtuales y red

| Elemento | Configuración |
|---|---|
| Servidor | `srv-lab` |
| Sistema operativo | Ubuntu Server 24.04.2 LTS |
| Kernel esperado | `6.8.0-55-generic` |
| Usuario administrativo | `labadmin` |
| Usuario de servicio | `telemetrysvc` |
| Interfaz NAT | `enp0s3`, DHCP |
| Interfaz Host-Only | `enp0s8`, `192.168.56.20/24` |
| Telemetry API | `127.0.0.1:9080` |
| Node Exporter | `192.168.56.20:9100` |
| Nginx | `192.168.56.20:8080` |

### Servicios que deben conservarse

| Servicio | Propósito esperado |
|---|---|
| `telemetry-api.service` | Servicio Python local en `127.0.0.1:9080` |
| `node_exporter.service` | Exportador de métricas en `192.168.56.20:9100` |
| `boot-validation.service` | Validación creada en el Laboratorio 1 |
| `log-flood.service` | Servicio temporal controlado que genera eventos `LAB08-FLOOD` |

### Rutas relevantes

| Ruta | Uso |
|---|---|
| `/opt/linux-essentials` | Aplicaciones y repositorios del curso |
| `/opt/linux-essentials/telemetry` | Aplicación Python de telemetría |
| `/opt/linux-essentials/infra` | Definiciones YAML e infraestructura |
| `/var/lib/linux-essentials/telemetry` | Datos escribibles de telemetría |
| `/var/log/linux-essentials` | Evidencias de arranque |
| `/var/log/journal` | Journal persistente, si está habilitado |
| `/etc/systemd/journald.conf.d/10-lab08-limits.conf` | Límites persistentes del journal |
| `/etc/fstab` | Configuración de sistemas de archivos |

### Preparación inicial

1. En VirtualBox, confirme que la máquina `srv-lab` está apagada.
2. Restaure o clone la instantánea `lab07-operativo`.
3. Inicie `srv-lab` y acceda mediante la consola de VirtualBox como `labadmin`.
4. Ejecute las comprobaciones iniciales:

```bash
hostnamectl
uname -r
ip -brief address
df -hT /
systemctl --version
```

5. Confirme que el host, kernel e interfaz Host-Only coinciden con el diseño del laboratorio:

```bash
hostname
ip -4 addr show enp0s8
```

La salida debe incluir, como mínimo:

```text
srv-lab
inet 192.168.56.20/24
```

> **Advertencia de seguridad:** los fallos de inundación de logs y de `/etc/fstab` deben ocurrir únicamente en `srv-lab`, dentro de VirtualBox. No copie los comandos de preparación del instructor a un equipo anfitrión, servidor productivo o estación personal.

## Instrucciones Paso a Paso

### Paso 1: Establecer la línea base y preservar evidencias

**Objetivo:** confirmar que el sistema inicia correctamente antes del incidente, registrar el estado de servicios críticos y crear un punto de recuperación.

**Instrucciones:**

1. Compruebe el usuario y el nombre del host:

```bash
whoami
hostname
```

2. Registre la fecha y la hora del inicio de la investigación:

```bash
date --iso-8601=seconds
timedatectl status
```

3. Cree el directorio local de evidencias si aún no existe:

```bash
sudo install -d -m 0750 -o root -g adm /var/log/linux-essentials
```

4. Guarde una línea base del espacio disponible, el journal y los servicios relevantes:

```bash
{
  echo "=== FECHA ==="
  date --iso-8601=seconds
  echo
  echo "=== ESPACIO EN / ==="
  df -hT /
  echo
  echo "=== USO DEL JOURNAL ==="
  journalctl --disk-usage
  echo
  echo "=== SERVICIOS ==="
  systemctl --no-pager --full status \
    telemetry-api.service \
    node_exporter.service \
    boot-validation.service
} | sudo tee /var/log/linux-essentials/lab08-baseline.txt >/dev/null
```

5. Revise el archivo generado:

```bash
sudo less /var/log/linux-essentials/lab08-baseline.txt
```

6. En VirtualBox, cree una instantánea manual denominada:

```text
lab08-antes-incidentes
```

7. No continúe hasta que el instructor confirme que el fallo controlado de inundación de logs está preparado o activo.

**Salida esperada:**

- `hostname` muestra `srv-lab`.
- La raíz `/` dispone inicialmente de espacio libre.
- Los tres servicios del Laboratorio 1 aparecen como `active (running)` o según el estado operativo definido por el instructor.
- Existe el archivo `/var/log/linux-essentials/lab08-baseline.txt`.

**Verificación:**

```bash
sudo test -s /var/log/linux-essentials/lab08-baseline.txt && echo "Evidencia inicial creada"
systemctl is-active telemetry-api.service node_exporter.service boot-validation.service
```

---

### Paso 2: Observar y delimitar la saturación causada por logs

**Objetivo:** detectar los síntomas de falta de espacio y determinar si el journal es una fuente principal del consumo.

**Instrucciones:**

1. Cuando el instructor indique que `log-flood.service` está activo, observe el espacio libre de la raíz:

```bash
df -hT /
```

2. Ejecute el mismo comando cada 15 a 30 segundos durante dos minutos. Si se requiere observación continua, utilice:

```bash
watch -n 5 'df -hT /; echo; journalctl --disk-usage'
```

3. Detenga `watch` con `Ctrl+C`.

4. Determine qué directorios de primer nivel consumen más espacio:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

5. Inspeccione específicamente el directorio de logs:

```bash
sudo du -xhd1 /var/log 2>/dev/null | sort -h
sudo du -sh /var/log/journal 2>/dev/null
```

6. Consulte la ocupación reportada por `systemd-journald`:

```bash
sudo journalctl --disk-usage
```

7. Localice archivos grandes bajo `/var/log` sin cruzar sistemas de archivos:

```bash
sudo find /var/log -xdev -type f -size +10M -printf '%10s %p\n' 2>/dev/null | sort -n
```

8. Revise los eventos recientes y busque la etiqueta identificable del laboratorio:

```bash
sudo journalctl --since "-15 min" --no-pager | grep -F "LAB08-FLOOD" | tail -n 20
```

9. Consulte exclusivamente la unidad responsable:

```bash
sudo systemctl status log-flood.service --no-pager --full
sudo journalctl -u log-flood.service --since "-15 min" --no-pager | tail -n 30
```

10. Documente los hallazgos en un archivo de evidencias. Sustituya los valores entre corchetes por los observados:

```bash
sudo tee /var/log/linux-essentials/lab08-log-analysis.txt >/dev/null <<'EOF'
Incidente: saturación controlada por logs.
Síntoma observado: [por ejemplo, poco espacio libre o errores de escritura].
Componente generador identificado: log-flood.service.
Patrón de log identificado: LAB08-FLOOD.
Fuente principal de almacenamiento: [journal persistente, /var/log/journal u otra].
Evidencia usada: df -hT, du, find, journalctl --disk-usage, journalctl -u.
Hipótesis: el crecimiento de mensajes emitidos por log-flood.service está incrementando
el tamaño del journal hasta comprometer el espacio disponible en el sistema de archivos raíz.
EOF
```

**Salida esperada:**

- `df -hT /` muestra una reducción significativa del espacio libre o uso muy alto.
- `journalctl --disk-usage` reporta una ocupación elevada.
- `journalctl -u log-flood.service` contiene mensajes con `LAB08-FLOOD`.
- `du` identifica `/var/log/journal` como una ubicación relevante.
- La unidad `log-flood.service` aparece activa durante el incidente.

**Verificación:**

```bash
sudo journalctl -u log-flood.service --since "-15 min" --no-pager | grep -q "LAB08-FLOOD" \
  && echo "Generador de logs confirmado"

sudo test -s /var/log/linux-essentials/lab08-log-analysis.txt \
  && echo "Análisis documentado"
```

---

### Paso 3: Contener la fuente de la inundación de logs

**Objetivo:** detener el componente causante antes de eliminar o reducir evidencia almacenada.

**Instrucciones:**

1. Detenga inmediatamente el servicio generador:

```bash
sudo systemctl stop log-flood.service
```

2. Evite que el servicio vuelva a iniciarse de forma accidental durante la recuperación:

```bash
sudo systemctl disable log-flood.service
sudo systemctl mask log-flood.service
```

3. Verifique el estado de la unidad:

```bash
sudo systemctl status log-flood.service --no-pager --full
```

4. Confirme que no continúan apareciendo mensajes nuevos. Ejecute la consulta dos veces, dejando aproximadamente 20 segundos entre ellas:

```bash
sudo journalctl -u log-flood.service --since "-2 min" --no-pager | tail -n 10
```

5. Compruebe si existen procesos asociados a la unidad:

```bash
sudo systemctl show log-flood.service \
  -p ActiveState \
  -p SubState \
  -p MainPID \
  -p ExecMainStatus
```

6. Registre la hora de contención:

```bash
sudo tee -a /var/log/linux-essentials/lab08-log-analysis.txt >/dev/null <<EOF

Contención aplicada:
Fecha y hora: $(date --iso-8601=seconds)
Acción: systemctl stop, disable y mask de log-flood.service.
Resultado: se detuvo la fuente de mensajes LAB08-FLOOD antes de reducir el journal.
EOF
```

> **Importante:** detener la fuente es prioritario. Ejecutar `journalctl --vacuum-size` mientras el servicio continúa inundando el journal puede liberar espacio solo temporalmente; el crecimiento se reanudará inmediatamente.

**Salida esperada:**

- `log-flood.service` aparece como `inactive (dead)` o `failed`, pero no como activo.
- La propiedad `MainPID` es `0`.
- La unidad aparece enmascarada para impedir reinicios accidentales.

**Verificación:**

```bash
systemctl is-active log-flood.service
systemctl is-enabled log-flood.service
```

La primera consulta debe devolver `inactive` o `failed`; la segunda debe indicar `masked`.

---

### Paso 4: Reducir el journal y establecer límites persistentes

**Objetivo:** recuperar capacidad de almacenamiento de forma controlada y prevenir que el journal vuelva a consumir todo el disco.

**Instrucciones:**

1. Registre el uso del journal antes de la reducción:

```bash
sudo journalctl --disk-usage
df -hT /
```

2. Reduzca el journal persistente hasta un tamaño máximo de 200 MiB:

```bash
sudo journalctl --vacuum-size=200M
```

3. Vuelva a comprobar el consumo y el espacio disponible:

```bash
sudo journalctl --disk-usage
df -hT /
```

4. Cree el directorio de configuración adicional para `systemd-journald`:

```bash
sudo install -d -m 0755 /etc/systemd/journald.conf.d
```

5. Cree el archivo de límites persistentes. Estos límites reservan espacio libre y acotan el crecimiento del journal del sistema:

```bash
sudo tee /etc/systemd/journald.conf.d/10-lab08-limits.conf >/dev/null <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=200M
SystemKeepFree=300M
MaxRetentionSec=7day
EOF
```

6. Verifique el contenido y permisos del archivo:

```bash
sudo cat /etc/systemd/journald.conf.d/10-lab08-limits.conf
sudo ls -l /etc/systemd/journald.conf.d/10-lab08-limits.conf
```

7. Reinicie `systemd-journald` para cargar la configuración:

```bash
sudo systemctl restart systemd-journald
```

8. Compruebe el estado del servicio de journal:

```bash
sudo systemctl status systemd-journald.service --no-pager --full
```

9. Revise mensajes de error o advertencia relacionados con `journald`:

```bash
sudo journalctl -u systemd-journald.service -p warning..alert --since "-5 min" --no-pager
```

10. Confirme que el journal mantiene uso acotado:

```bash
sudo journalctl --disk-usage
df -hT /
```

11. Guarde una evidencia de la política implementada:

```bash
{
  echo "=== CONFIGURACIÓN DE LÍMITES ==="
  cat /etc/systemd/journald.conf.d/10-lab08-limits.conf
  echo
  echo "=== USO POSTERIOR DEL JOURNAL ==="
  journalctl --disk-usage
  echo
  echo "=== ESPACIO POSTERIOR EN / ==="
  df -hT /
} | sudo tee /var/log/linux-essentials/lab08-journal-recovery.txt >/dev/null
```

**Salida esperada:**

- `journalctl --vacuum-size=200M` elimina archivos archivados antiguos cuando es necesario.
- `df -hT /` muestra recuperación de espacio libre.
- `systemd-journald.service` permanece `active (running)`.
- El archivo `/etc/systemd/journald.conf.d/10-lab08-limits.conf` contiene los límites persistentes.
- El journal utiliza almacenamiento persistente bajo `/var/log/journal`, si esa ruta está habilitada en la imagen del laboratorio.

**Verificación:**

```bash
sudo grep -E '^(Storage|SystemMaxUse|SystemKeepFree|MaxRetentionSec)=' \
  /etc/systemd/journald.conf.d/10-lab08-limits.conf

sudo journalctl --disk-usage
df -h /
```

---

### Paso 5: Validar la recuperación tras el incidente de logs

**Objetivo:** confirmar que la recuperación del almacenamiento no afectó los servicios operativos del Laboratorio 1.

**Instrucciones:**

1. Compruebe el estado de los servicios persistentes:

```bash
sudo systemctl status \
  telemetry-api.service \
  node_exporter.service \
  boot-validation.service \
  --no-pager --full
```

2. Verifique que la API de telemetría responde localmente:

```bash
curl --fail --silent --show-error http://127.0.0.1:9080/ || true
```

3. Si el servicio expone una ruta de salud definida en el Laboratorio 1, pruébela:

```bash
curl --fail --silent --show-error http://127.0.0.1:9080/health || true
```

4. Consulte el puerto de telemetría:

```bash
sudo ss -ltnp | grep -E ':9080\b' || true
```

5. Consulte el puerto de Node Exporter:

```bash
sudo ss -ltnp | grep -E '192\.168\.56\.20:9100\b' || true
```

6. Realice una consulta de métricas desde el propio servidor:

```bash
curl --fail --silent http://192.168.56.20:9100/metrics | head -n 10
```

7. Revise errores recientes de las unidades importantes:

```bash
sudo journalctl \
  -u telemetry-api.service \
  -u node_exporter.service \
  -u boot-validation.service \
  -p err \
  --since "-30 min" \
  --no-pager
```

**Salida esperada:**

- Los servicios requeridos aparecen activos.
- `telemetry-api.service` escucha en `127.0.0.1:9080`.
- `node_exporter.service` escucha en `192.168.56.20:9100`.
- No aparecen errores nuevos relacionados con falta de espacio en los servicios verificados.

**Verificación:**

```bash
systemctl is-active telemetry-api.service
systemctl is-active node_exporter.service
systemctl is-active boot-validation.service

curl --fail --silent http://127.0.0.1:9080/ >/dev/null \
  && echo "Telemetry API disponible"
```

---

### Paso 6: Analizar el fallo de arranque provocado por `/etc/fstab`

**Objetivo:** identificar la entrada inválida que impedirá el arranque normal y preparar una corrección basada en evidencia.

**Instrucciones:**

1. El instructor agregará una entrada deliberadamente inválida y no crítica para el punto de montaje:

```text
/mnt/lab08-missing
```

2. Antes de reiniciar, inspeccione `/etc/fstab` y localice la línea marcada para el laboratorio:

```bash
sudo nl -ba /etc/fstab
```

3. Busque específicamente el punto de montaje afectado:

```bash
sudo grep -n -C 2 '/mnt/lab08-missing' /etc/fstab
```

4. Guarde una copia forense de la configuración actual antes de cambiar nada:

```bash
sudo cp -a /etc/fstab /var/log/linux-essentials/fstab-lab08-failure.txt
sudo chmod 0640 /var/log/linux-essentials/fstab-lab08-failure.txt
```

5. Valide la configuración sin reiniciar:

```bash
sudo findmnt --verify --verbose
```

6. Pruebe todos los montajes definidos en `fstab`. Se espera un error asociado al punto de montaje inválido:

```bash
sudo mount -a
```

7. Registre la salida de la validación:

```bash
sudo findmnt --verify --verbose 2>&1 | \
  sudo tee /var/log/linux-essentials/lab08-fstab-verify-before.txt
```

8. No corrija todavía la entrada. El propósito es practicar el rescate desde el arranque controlado.

9. Reinicie la máquina:

```bash
sudo reboot
```

**Salida esperada:**

- `findmnt --verify --verbose` y/o `mount -a` informa un problema asociado con `/mnt/lab08-missing`, un UUID inválido o un dispositivo inexistente.
- Tras reiniciar, el arranque normal no se completa y el sistema puede entrar en modo de emergencia o mostrar un mensaje de fallo de montaje.
- La máquina no debe quedar inutilizable permanentemente; la consola de VirtualBox y GRUB permiten recuperarla.

**Verificación:**

Antes del reinicio, confirme que existe la evidencia:

```bash
sudo test -s /var/log/linux-essentials/fstab-lab08-failure.txt \
  && echo "Copia de fstab preservada"

sudo test -s /var/log/linux-essentials/lab08-fstab-verify-before.txt \
  && echo "Validación previa registrada"
```

---

### Paso 7: Arrancar temporalmente en modo de emergencia desde GRUB

**Objetivo:** utilizar la consola de VirtualBox y un parámetro temporal de GRUB para obtener acceso de recuperación sin modificar permanentemente la configuración de arranque.

**Instrucciones:**

1. Mantenga abierta la ventana de consola de VirtualBox para `srv-lab`.

2. Durante el arranque, muestre el menú GRUB:
   - En sistemas BIOS, pulse repetidamente `Shift` después del inicio de la VM.
   - En sistemas UEFI, pulse repetidamente `Esc`.
   - Si no aparece el menú, reinicie la VM y vuelva a intentarlo.

3. Seleccione la entrada normal de Ubuntu, pero **no** pulse todavía `Enter`.

4. Pulse `e` para editar temporalmente la entrada de GRUB.

5. Localice la línea que comienza por `linux`. Al final de esa línea agregue un espacio y el parámetro:

```text
systemd.unit=emergency.target
```

6. No elimine otros parámetros existentes, como `root=`, `ro`, `quiet` o `splash`, salvo instrucción expresa del instructor.

7. Inicie con la modificación temporal usando `Ctrl+X` o `F10`.

8. Si el sistema solicita autenticación administrativa o acceso de mantenimiento, utilice las credenciales aprobadas por el instructor.

9. Si `emergency.target` no permite una consola de recuperación debido a una política local de credenciales, vuelva a GRUB y use el modo de recuperación de Ubuntu desde:

```text
Advanced options for Ubuntu
```

Seleccione el kernel correspondiente y después la opción de consola raíz de recuperación.

10. Como último recurso autorizado exclusivamente en la VM aislada del laboratorio, agregue temporalmente a la línea `linux`:

```text
rw init=/bin/bash
```

Este método omite controles normales de autenticación; úselo solo con autorización del instructor y únicamente en la consola local de VirtualBox. Nunca debe usarse en un sistema expuesto o productivo.

**Salida esperada:**

- El sistema inicia en una consola mínima de recuperación.
- No se inicia la sesión multiusuario normal.
- El administrador dispone de una consola local desde la que puede inspeccionar y reparar `/etc/fstab`.

**Verificación:**

Ejecute:

```bash
mount | grep ' on / '
systemctl get-default 2>/dev/null || true
cat /proc/cmdline
```

La salida de `/proc/cmdline` debe incluir el parámetro temporal utilizado, por ejemplo:

```text
systemd.unit=emergency.target
```

---

### Paso 8: Reparar `/etc/fstab` y validar los montajes

**Objetivo:** corregir la entrada inválida de `/etc/fstab`, comprobar la sintaxis y asegurar que todos los montajes configurados pueden procesarse correctamente.

**Instrucciones:**

1. Compruebe si la raíz está montada en modo solo lectura:

```bash
findmnt -no TARGET,OPTIONS /
```

2. Si la salida contiene `ro`, remonte la raíz en modo lectura-escritura:

```bash
mount -o remount,rw /
```

3. Compruebe nuevamente las opciones de montaje:

```bash
findmnt -no TARGET,OPTIONS /
```

4. Inspeccione la entrada problemática con números de línea:

```bash
nl -ba /etc/fstab
grep -n -C 2 '/mnt/lab08-missing' /etc/fstab
```

5. Cree una copia de seguridad adicional antes de editar:

```bash
cp -a /etc/fstab /etc/fstab.lab08-emergency.bak
```

6. Edite el archivo con `nano`:

```bash
nano /etc/fstab
```

7. Corrija la entrada inválida usando una de estas acciones aprobadas:
   - Elimine la línea incorrecta si el montaje no debe existir.
   - Comente la línea colocando `#` al inicio si se conserva solo como referencia del incidente.
   - Sustituya el UUID, dispositivo o punto de montaje por datos válidos, únicamente si el instructor proporcionó una configuración correcta.

8. Para este laboratorio, la resolución esperada es comentar o eliminar la entrada deliberadamente inválida para `/mnt/lab08-missing`. No agregue la opción `nofail` como forma de ocultar el problema; el objetivo es restaurar una configuración correcta.

9. Guarde el archivo y salga de `nano`:
   - `Ctrl+O`, `Enter` para guardar.
   - `Ctrl+X` para salir.

10. Revise el resultado:

```bash
grep -n -C 2 '/mnt/lab08-missing' /etc/fstab || true
```

11. Valide la configuración de montajes:

```bash
findmnt --verify --verbose
```

12. Ejecute la prueba de montaje sin reiniciar:

```bash
mount -a
```

13. Compruebe el código de retorno del último comando:

```bash
echo $?
```

14. Registre la configuración corregida y la validación:

```bash
{
  echo "=== FECHA DE RECUPERACIÓN ==="
  date --iso-8601=seconds
  echo
  echo "=== FSTAB CORREGIDO ==="
  cat /etc/fstab
  echo
  echo "=== VALIDACIÓN ==="
  findmnt --verify --verbose
} > /var/log/linux-essentials/lab08-fstab-recovered.txt 2>&1
```

**Salida esperada:**

- La raíz queda montada con permisos de escritura durante la reparación.
- La referencia inválida a `/mnt/lab08-missing` ya no es una entrada activa de `/etc/fstab`.
- `findmnt --verify --verbose` no reporta errores críticos.
- `mount -a` termina sin errores y `echo $?` devuelve `0`.

**Verificación:**

```bash
sudo findmnt --verify --verbose
sudo mount -a
echo "Código de mount -a: $?"

sudo test -s /var/log/linux-essentials/lab08-fstab-recovered.txt \
  && echo "Evidencia de recuperación creada"
```

---

### Paso 9: Reiniciar normalmente y validar la recuperación integral

**Objetivo:** confirmar que el sistema vuelve a iniciar en modo normal, que el journal conserva sus límites y que los servicios previos sobreviven al proceso de recuperación.

**Instrucciones:**

1. Si inició mediante `emergency.target`, reinicie normalmente:

```bash
systemctl reboot
```

2. Si utilizó `init=/bin/bash`, después de reparar correctamente `fstab`, ejecute:

```bash
exec /sbin/init
```

Si el sistema no completa el arranque, reinicie desde VirtualBox. No deje permanentemente el parámetro `init=/bin/bash`; los cambios hechos en GRUB son temporales y desaparecen en el siguiente arranque.

3. Inicie sesión como `labadmin` cuando el sistema arranque normalmente.

4. Confirme el arranque actual y revise eventos de montaje:

```bash
journalctl -b --no-pager | grep -Ei 'mount|fstab|failed|dependency' | tail -n 50
```

5. Confirme que no se inició en modo de emergencia:

```bash
systemctl is-system-running
systemctl get-default
```

6. Compruebe que la raíz tiene espacio libre y que el journal está dentro de la política esperada:

```bash
df -hT /
journalctl --disk-usage
```

7. Verifique que el archivo de límites persiste:

```bash
sudo cat /etc/systemd/journald.conf.d/10-lab08-limits.conf
```

8. Compruebe los servicios obligatorios:

```bash
sudo systemctl is-active telemetry-api.service
sudo systemctl is-active node_exporter.service
sudo systemctl is-active boot-validation.service
```

9. Verifique los puertos esperados:

```bash
sudo ss -ltnp | grep -E '127\.0\.0\.1:9080|192\.168\.56\.20:9100|192\.168\.56\.20:8080'
```

10. Compruebe la API de telemetría y Node Exporter:

```bash
curl --fail --silent --show-error http://127.0.0.1:9080/ >/dev/null \
  && echo "telemetry-api.service responde"

curl --fail --silent http://192.168.56.20:9100/metrics | head -n 5
```

11. Registre el estado final:

```bash
{
  echo "=== RECUPERACIÓN FINAL LAB08 ==="
  date --iso-8601=seconds
  echo
  echo "=== ESTADO DEL SISTEMA ==="
  systemctl is-system-running
  echo
  echo "=== ESPACIO ==="
  df -hT /
  echo
  echo "=== JOURNAL ==="
  journalctl --disk-usage
  echo
  echo "=== SERVICIOS ==="
  systemctl is-active telemetry-api.service
  systemctl is-active node_exporter.service
  systemctl is-active boot-validation.service
  echo
  echo "=== VALIDACIÓN FSTAB ==="
  findmnt --verify --verbose
} | sudo tee /var/log/linux-essentials/lab08-final-validation.txt >/dev/null
```

12. En VirtualBox, cree una instantánea de recuperación denominada:

```text
lab08-recuperado
```

**Salida esperada:**

- El sistema inicia normalmente sin intervención de GRUB.
- `systemctl is-system-running` devuelve `running` o `degraded` solo si existe una condición conocida y documentada no relacionada con el laboratorio.
- `findmnt --verify --verbose` no informa errores de configuración.
- Los servicios de telemetría, métricas y validación de arranque permanecen disponibles.
- La instantánea `lab08-recuperado` queda creada.

**Verificación:**

```bash
systemctl is-active telemetry-api.service node_exporter.service boot-validation.service
sudo findmnt --verify --verbose
sudo journalctl --disk-usage
sudo test -s /var/log/linux-essentials/lab08-final-validation.txt \
  && echo "Recuperación integral documentada"
```

## Validación y Pruebas

Complete la siguiente lista de comprobación antes de dar por finalizado el laboratorio.

| Validación | Comando | Resultado esperado |
|---|---|---|
| Host correcto | `hostname` | `srv-lab` |
| Espacio recuperado | `df -hT /` | Espacio libre disponible en `/` |
| Journal acotado | `journalctl --disk-usage` | Uso coherente con `SystemMaxUse=200M` |
| Política persistente | `cat /etc/systemd/journald.conf.d/10-lab08-limits.conf` | Archivo presente con cuatro directivas |
| Fuente de flood detenida | `systemctl is-enabled log-flood.service` | `masked` |
| Fstab válido | `sudo findmnt --verify --verbose` | Sin errores críticos |
| Montajes válidos | `sudo mount -a` | Código de retorno `0` |
| Arranque recuperado | `systemctl is-system-running` | `running` |
| Telemetría activa | `systemctl is-active telemetry-api.service` | `active` |
| Node Exporter activo | `systemctl is-active node_exporter.service` | `active` |
| Validación de boot activa | `systemctl is-active boot-validation.service` | `active` |
| API local disponible | `curl -f http://127.0.0.1:9080/` | Respuesta HTTP satisfactoria |
| Métricas disponibles | `curl -f http://192.168.56.20:9100/metrics` | Texto de métricas Prometheus |
| Evidencias almacenadas | `sudo ls -lh /var/log/linux-essentials/` | Archivos `lab08-*` presentes |

Ejecute una validación compacta final:

```bash
set -e

hostname
df -hT /
journalctl --disk-usage
sudo findmnt --verify --verbose
sudo mount -a

systemctl is-active telemetry-api.service
systemctl is-active node_exporter.service
systemctl is-active boot-validation.service

curl --fail --silent http://127.0.0.1:9080/ >/dev/null
curl --fail --silent http://192.168.56.20:9100/metrics >/dev/null

echo "Validación final de Lab 08 completada correctamente."
```

## Solución de Problemas

### Problema 1: `journalctl --vacuum-size=200M` no libera suficiente espacio o el sistema sigue sin espacio

**Síntomas:**

- `df -h /` continúa mostrando uso cercano al 100 %.
- `journalctl --disk-usage` disminuye poco o no disminuye.
- Los servicios registran mensajes nuevos incluso después de ejecutar el vacuum.
- Existen archivos grandes en `/var/log` que no pertenecen al journal.

**Causa probable:**

El generador `log-flood.service` no fue detenido por completo, el journal aún contiene archivos activos que no se pueden eliminar inmediatamente o el consumo principal corresponde a otra fuente, como archivos tradicionales de `rsyslog`, logs de aplicaciones o archivos eliminados que siguen abiertos por un proceso.

**Corrección:**

```bash
sudo systemctl stop log-flood.service
sudo systemctl mask log-flood.service

sudo journalctl --disk-usage
sudo journalctl --rotate
sudo journalctl --vacuum-size=200M

sudo du -xhd1 /var/log 2>/dev/null | sort -h
sudo find /var/log -xdev -type f -size +10M -printf '%10s %p\n' 2>/dev/null | sort -n
sudo lsof +L1 2>/dev/null | head -n 30

df -hT /
```

Revise los archivos identificados antes de eliminarlos. No borre manualmente archivos internos de `/var/log/journal`; utilice `journalctl --rotate` y `journalctl --vacuum-size` para conservar la consistencia del journal.

### Problema 2: Después de corregir `/etc/fstab`, el sistema vuelve a entrar en modo de emergencia

**Síntomas:**

- El arranque se detiene nuevamente por un montaje fallido.
- `findmnt --verify --verbose` informa errores.
- `mount -a` falla con mensajes sobre UUID, dispositivo, tipo de sistema de archivos o punto de montaje.
- La raíz se inicia en modo solo lectura durante la recuperación.

**Causa probable:**

La entrada inválida no fue eliminada o comentada correctamente, se introdujo un error de sintaxis al editar `/etc/fstab`, el punto de montaje no existe o se modificó una entrada válida distinta de la entrada del laboratorio.

**Corrección:**

1. Vuelva a arrancar temporalmente desde GRUB con:

```text
systemd.unit=emergency.target
```

2. Remonte la raíz con escritura:

```bash
mount -o remount,rw /
```

3. Compare la configuración actual con la copia preservada:

```bash
diff -u /var/log/linux-essentials/fstab-lab08-failure.txt /etc/fstab || true
```

4. Revise las líneas y asegure que la entrada de `/mnt/lab08-missing` está comentada o eliminada:

```bash
nl -ba /etc/fstab
grep -n -C 2 '/mnt/lab08-missing' /etc/fstab || true
```

5. Compruebe que los puntos de montaje válidos existen:

```bash
awk '!/^[[:space:]]*#/ && NF >= 2 {print $2}' /etc/fstab | while read -r mountpoint; do
  [ "$mountpoint" = "none" ] || [ "$mountpoint" = "swap" ] || ls -ld "$mountpoint"
done
```

6. Valide antes de reiniciar:

```bash
findmnt --verify --verbose
mount -a
```

7. Reinicie solo cuando ambos comandos terminen sin errores críticos.

## Limpieza

1. Mantenga enmascarado el servicio de inundación para evitar que vuelva a ejecutarse accidentalmente:

```bash
sudo systemctl mask log-flood.service
sudo systemctl is-enabled log-flood.service
```

2. No elimine las evidencias generadas. Deben conservarse para revisión técnica:

```bash
sudo ls -lh /var/log/linux-essentials/
```

3. Mantenga el archivo de política de journal:

```bash
sudo cat /etc/systemd/journald.conf.d/10-lab08-limits.conf
```

4. Confirme que no queda una entrada activa para el montaje ficticio:

```bash
grep -n '/mnt/lab08-missing' /etc/fstab || echo "No hay entrada activa de lab08-missing"
```

5. En VirtualBox, confirme que existe la instantánea `lab08-recuperado`.

6. No actualice la distribución, kernel, `systemd`, Docker, Netplan ni otros paquetes durante la limpieza. La VM recuperada será la línea base para los laboratorios posteriores de automatización YAML.

## Resumen

En este laboratorio se aplicó un procedimiento de recuperación basado en evidencias. Se identificó que `log-flood.service` emitía mensajes `LAB08-FLOOD`, se relacionó el crecimiento de esos eventos con el consumo del journal y se detuvo la fuente antes de liberar espacio. Posteriormente, se utilizaron `journalctl --vacuum-size`, `SystemMaxUse`, `SystemKeepFree` y `MaxRetentionSec` para recuperar capacidad y prevenir recurrencias.

También se recuperó un fallo de arranque causado por una entrada inválida en `/etc/fstab`. Mediante la consola de VirtualBox, GRUB y `emergency.target`, se remontó la raíz con escritura, se corrigió la configuración y se validó con `findmnt --verify --verbose` y `mount -a`. Finalmente, se comprobó la continuidad de `telemetry-api.service`, `node_exporter.service` y `boot-validation.service`, y se creó la instantánea `lab08-recuperado`.

Recursos de consulta local:

```bash
man journalctl
man journald.conf
man systemd-journald.service
man fstab
man findmnt
man mount
man systemctl
```
