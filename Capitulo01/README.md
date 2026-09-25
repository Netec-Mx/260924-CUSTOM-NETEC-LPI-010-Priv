# Laboratorio 7: Creación de un Servicio Persistente, Seguro y Monitoreado tras un Boot Personalizado

## Metadatos

| Duration | Complexity | Bloom level |
|---|---|---|
| 223 minutos | Difícil | Crear |

## Descripción General

En este laboratorio se construirá un daemon Python persistente administrado por `systemd` en la máquina virtual `srv-lab`. El servicio expondrá un endpoint HTTP de salud en `127.0.0.1:9080`, se ejecutará con un usuario de mínimo privilegio y aplicará mecanismos básicos de aislamiento de `systemd`.

También se configurará acceso remoto seguro por OpenSSH, se desplegará Prometheus Node Exporter en la interfaz Host-Only y se creará una unidad de validación que registre evidencia del arranque. Todas las actividades se realizan exclusivamente en la VM `srv-lab`; no se deben ejecutar fallos, cambios de boot ni configuraciones destructivas sobre el sistema anfitrión.

## Objetivos de Aprendizaje

- [ ] Crear un daemon Python persistente controlado por una unidad `systemd`.
- [ ] Aplicar endurecimiento básico con usuario no privilegiado, restricciones de escritura y capacidades vacías.
- [ ] Configurar autenticación SSH mediante claves públicas y restringir el usuario `opsremote`.
- [ ] Instalar y validar Prometheus Node Exporter 1.8.2 en la dirección Host-Only del servidor.
- [ ] Crear una unidad `oneshot` que valide red, kernel y servicio al finalizar el arranque.

## Prerrequisitos

Conocimientos requeridos:

- Uso de Bash, redirecciones, permisos POSIX y comandos `sudo`.
- Administración básica de usuarios, grupos, procesos y servicios con `systemctl`.
- Edición de archivos con `nano`, `vim` o `micro`.
- Fundamentos de direccionamiento IPv4, puertos TCP y autenticación SSH.
- Comprensión básica de PID, daemon, kernel, proceso, servicio y unidad `systemd`.

Acceso requerido:

- VM persistente `srv-lab` con Ubuntu Server 24.04.2 LTS.
- Usuario administrativo `labadmin` con permisos `sudo`.
- Consola de VirtualBox disponible para recuperación de red o boot.
- Acceso NAT temporal para descargar Node Exporter.
- VM `desk-lab` o estación administrativa con cliente SSH y `curl`.
- No actualizar la distribución, kernel, `systemd` ni paquetes base durante este laboratorio.

## Entorno de Laboratorio

| Componente | Valor requerido |
|---|---|
| VM de servidor | `srv-lab` |
| Sistema operativo | Ubuntu Server 24.04.2 LTS |
| Kernel esperado | `6.8.0-55-generic` |
| Usuario administrativo | `labadmin` |
| Usuario del servicio | `telemetrysvc` |
| Usuario remoto restringido | `opsremote` |
| Interfaz NAT | `enp0s3`, DHCP |
| Interfaz Host-Only | `enp0s8`, `192.168.56.20/24` |
| Servicio Python | `telemetry-api.service`, `127.0.0.1:9080` |
| Node Exporter | `192.168.56.20:9100` |
| Directorio de aplicaciones | `/opt/linux-essentials` |
| Datos escribibles | `/var/lib/linux-essentials/telemetry` |
| Evidencias de boot | `/var/log/linux-essentials` |

Inicie sesión en la consola de `srv-lab` como `labadmin` y confirme la identidad del sistema antes de modificar configuraciones:

```bash
hostnamectl
uname -r
ip -br address
ps -p 1 -o pid,ppid,comm,args
systemctl --version | head -n 1
```

**Salida esperada:** el hostname es `srv-lab`, PID 1 corresponde a `systemd`, el kernel coincide con `6.8.0-55-generic`, `enp0s3` tiene una dirección DHCP y `enp0s8` será configurada con `192.168.56.20/24`.

> **Precaución:** mantenga abierta la consola de VirtualBox mientras cambie la red o OpenSSH. No dependa exclusivamente de una sesión SSH durante estos cambios.

## Instrucciones Paso a Paso

### Paso 1: Verificar la base del servidor y configurar la red persistente

**Objetivo:** confirmar la arquitectura basada en `systemd` y establecer la red esperada para la administración Host-Only.

**Instrucciones:**

1. Revise los servicios activos y compruebe que `systemd` administra el sistema:

   ```bash
   systemctl list-units --type=service --state=running
   systemctl status systemd-journald --no-pager
   ```

2. Inspeccione los archivos Netplan existentes:

   ```bash
   sudo ls -la /etc/netplan/
   sudo cat /etc/netplan/*.yaml
   ```

3. Cree una copia de seguridad fuera de `/etc/netplan`:

   ```bash
   sudo install -d -m 0700 /root/netplan-backup-lab
   sudo cp -a /etc/netplan/*.yaml /root/netplan-backup-lab/
   ```

4. Desde la consola de VirtualBox, reemplace la configuración Netplan por una definición controlada. Elimine únicamente los archivos YAML activos de Netplan y cree uno nuevo:

   ```bash
   sudo rm -f /etc/netplan/*.yaml

   sudo tee /etc/netplan/01-lab.yaml > /dev/null <<'EOF'
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s3:
         dhcp4: true
       enp0s8:
         dhcp4: false
         addresses:
           - 192.168.56.20/24
   EOF

   sudo chmod 600 /etc/netplan/01-lab.yaml
   ```

5. Pruebe la configuración. Confirme en la consola cuando Netplan solicite aceptación:

   ```bash
   sudo netplan try
   ```

6. Si la red sigue disponible y la interfaz Host-Only recibe la dirección esperada, aplique definitivamente la configuración:

   ```bash
   sudo netplan apply
   ip -br address show enp0s3 enp0s8
   ip route
   ```

7. Cree la estructura persistente de directorios del laboratorio:

   ```bash
   sudo install -d -o root -g root -m 0755 \
     /opt/linux-essentials \
     /opt/linux-essentials/telemetry \
     /opt/linux-essentials/infra \
     /opt/linux-essentials/infra/netplan \
     /opt/linux-essentials/infra/systemd

   sudo install -d -o telemetrysvc -g telemetrysvc -m 0750 \
     /var/lib/linux-essentials/telemetry
   ```

   Si el usuario `telemetrysvc` todavía no existe, este último comando fallará temporalmente. Continúe con el siguiente paso y repítalo después de crear el usuario.

8. Guarde una copia de referencia del archivo Netplan dentro de la infraestructura del laboratorio:

   ```bash
   sudo install -m 0644 /etc/netplan/01-lab.yaml \
     /opt/linux-essentials/infra/netplan/01-lab.yaml
   ```

**Salida esperada:**

```text
enp0s3    UP    <dirección DHCP de NAT>/...
enp0s8    UP    192.168.56.20/24
```

La tabla de rutas debe mantener una ruta por defecto a través de `enp0s3`. No debe configurarse una puerta de enlace en `enp0s8` salvo instrucción explícita del instructor.

**Verificación:**

```bash
ping -c 2 192.168.56.20
ping -c 2 1.1.1.1
getent hosts github.com
```

El primer comando debe responder localmente. Los dos últimos dependen de que NAT esté disponible.

---

### Paso 2: Crear la identidad de servicio y la aplicación Python

**Objetivo:** crear un usuario sin inicio de sesión interactivo y una aplicación HTTP que publique un estado JSON de salud.

**Instrucciones:**

1. Cree el grupo y usuario de sistema `telemetrysvc`. El usuario no debe tener directorio de inicio ni shell interactiva:

   ```bash
   sudo groupadd --system telemetrysvc 2>/dev/null || true

   sudo useradd \
     --system \
     --gid telemetrysvc \
     --no-create-home \
     --home-dir /nonexistent \
     --shell /usr/sbin/nologin \
     telemetrysvc 2>/dev/null || true
   ```

2. Verifique la identidad creada:

   ```bash
   getent passwd telemetrysvc
   id telemetrysvc
   ```

3. Cree el directorio de datos escribible por el servicio:

   ```bash
   sudo install -d -o telemetrysvc -g telemetrysvc -m 0750 \
     /var/lib/linux-essentials/telemetry
   ```

4. Cree la aplicación Python en `/opt/linux-essentials/telemetry/app.py`:

   ```bash
   sudo tee /opt/linux-essentials/telemetry/app.py > /dev/null <<'PYTHON'
   #!/usr/bin/env python3
   import json
   import os
   import platform
   import time
   from datetime import datetime, timezone
   from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

   HOST = "127.0.0.1"
   PORT = 9080
   STATE_FILE = "/var/lib/linux-essentials/telemetry/last-start.txt"
   START_TIME = time.time()


   def write_start_marker():
       timestamp = datetime.now(timezone.utc).isoformat()
       with open(STATE_FILE, "w", encoding="utf-8") as state_file:
           state_file.write(f"started_at={timestamp}\n")
           state_file.write(f"pid={os.getpid()}\n")


   class HealthHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           if self.path not in ("/", "/health"):
               self.send_response(404)
               self.send_header("Content-Type", "application/json")
               self.end_headers()
               self.wfile.write(b'{"status":"not_found"}\n')
               return

           payload = {
               "status": "ok",
               "service": "telemetry-api",
               "pid": os.getpid(),
               "uptime_seconds": round(time.time() - START_TIME, 3),
               "kernel": platform.release(),
           }

           response = (json.dumps(payload) + "\n").encode("utf-8")
           self.send_response(200)
           self.send_header("Content-Type", "application/json")
           self.send_header("Content-Length", str(len(response)))
           self.end_headers()
           self.wfile.write(response)

       def log_message(self, format_string, *args):
           print(
               "%s - - [%s] %s"
               % (self.client_address[0], self.log_date_time_string(), format_string % args),
               flush=True,
           )


   if __name__ == "__main__":
       write_start_marker()
       print(f"telemetry-api listening on http://{HOST}:{PORT}", flush=True)
       server = ThreadingHTTPServer((HOST, PORT), HealthHandler)
       server.serve_forever()
   PYTHON
   ```

5. Aplique permisos seguros. La aplicación será propiedad de `root`; el usuario del daemon solo necesita leerla y ejecutarla mediante Python:

   ```bash
   sudo chown -R root:root /opt/linux-essentials/telemetry
   sudo chmod 0755 /opt/linux-essentials/telemetry
   sudo chmod 0644 /opt/linux-essentials/telemetry/app.py
   ```

6. Compruebe la sintaxis Python sin iniciar todavía el daemon:

   ```bash
   sudo -u telemetrysvc /usr/bin/python3 -m py_compile \
     /opt/linux-essentials/telemetry/app.py
   ```

**Salida esperada:**

```text
telemetrysvc:x:<UID>:<GID>::/nonexistent:/usr/sbin/nologin
```

El comando `py_compile` no debe producir errores.

**Verificación:**

```bash
sudo -u telemetrysvc sh -c 'echo prueba > /var/lib/linux-essentials/telemetry/write-test.txt'
sudo cat /var/lib/linux-essentials/telemetry/write-test.txt
sudo rm -f /var/lib/linux-essentials/telemetry/write-test.txt
```

La escritura debe funcionar únicamente en el directorio de estado asignado al servicio.

---

### Paso 3: Crear y endurecer la unidad telemetry-api.service

**Objetivo:** implementar un servicio persistente administrado por `systemd` con recuperación automática y aislamiento básico.

**Instrucciones:**

1. Cree la unidad local de `systemd`:

   ```bash
   sudo tee /etc/systemd/system/telemetry-api.service > /dev/null <<'EOF'
   [Unit]
   Description=API local de salud para Linux Essentials
   Documentation=file:///opt/linux-essentials/telemetry/app.py
   Wants=network-online.target
   After=network-online.target

   [Service]
   Type=simple
   User=telemetrysvc
   Group=telemetrysvc
   WorkingDirectory=/opt/linux-essentials/telemetry
   ExecStart=/usr/bin/python3 -u /opt/linux-essentials/telemetry/app.py
   Restart=on-failure
   RestartSec=5
   TimeoutStartSec=15
   TimeoutStopSec=15

   CPUQuota=20%
   MemoryMax=128M
   TasksMax=64
   LimitNOFILE=1024

   NoNewPrivileges=true
   ProtectSystem=strict
   ProtectHome=true
   PrivateTmp=true
   ReadWritePaths=/var/lib/linux-essentials/telemetry

   CapabilityBoundingSet=
   AmbientCapabilities=
   LockPersonality=true
   PrivateDevices=true
   ProtectControlGroups=true
   ProtectKernelModules=true
   ProtectKernelTunables=true
   RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
   RestrictSUIDSGID=true
   SystemCallArchitectures=native
   UMask=0077

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

2. Copie la unidad al directorio de infraestructura para conservar una referencia administrativa:

   ```bash
   sudo install -m 0644 /etc/systemd/system/telemetry-api.service \
     /opt/linux-essentials/infra/systemd/telemetry-api.service
   ```

3. Recargue las definiciones de `systemd`, habilite el inicio automático e inicie el servicio:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now telemetry-api.service
   ```

4. Revise el estado, el PID principal y los registros recientes:

   ```bash
   systemctl status telemetry-api.service --no-pager
   systemctl show telemetry-api.service -p MainPID -p ActiveState -p SubState
   sudo journalctl -u telemetry-api.service -n 20 --no-pager
   ```

5. Consulte el endpoint desde el propio servidor:

   ```bash
   curl -fsS http://127.0.0.1:9080/health
   ```

6. Compruebe que el puerto está limitado al loopback y no está publicado en la interfaz Host-Only:

   ```bash
   sudo ss -ltnp | grep ':9080'
   curl --connect-timeout 2 http://192.168.56.20:9080/health || true
   ```

7. Ejecute el analizador de exposición de seguridad de `systemd`:

   ```bash
   systemd-analyze security telemetry-api.service
   ```

   El resultado no tiene que ser perfecto. El propósito es revisar qué controles se aplicaron y reconocer que el servicio no debe tener permisos innecesarios.

**Salida esperada:**

```text
Active: active (running)
```

La consulta HTTP debe devolver un JSON similar al siguiente:

```json
{"status":"ok","service":"telemetry-api","pid":1234,"uptime_seconds":2.1,"kernel":"6.8.0-55-generic"}
```

El puerto debe aparecer como `127.0.0.1:9080`, no como `0.0.0.0:9080` ni como `192.168.56.20:9080`.

**Verificación:**

```bash
systemctl is-enabled telemetry-api.service
systemctl is-active telemetry-api.service
sudo cat /var/lib/linux-essentials/telemetry/last-start.txt
```

Los estados esperados son `enabled` y `active`.

---

### Paso 4: Validar la recuperación automática del daemon

**Objetivo:** confirmar que `systemd` supervisa el daemon y lo reinicia cuando finaliza de forma anómala.

**Instrucciones:**

1. Obtenga y guarde el PID actual del proceso principal:

   ```bash
   OLD_PID="$(systemctl show -p MainPID --value telemetry-api.service)"
   echo "PID anterior: ${OLD_PID}"
   ```

2. Simule una terminación anómala con `SIGKILL`. No use `systemctl stop`, porque una detención solicitada por el administrador no debe activar `Restart=on-failure`:

   ```bash
   sudo kill -KILL "${OLD_PID}"
   ```

3. Espere más tiempo que `RestartSec=5` y revise el estado:

   ```bash
   sleep 7
   systemctl status telemetry-api.service --no-pager
   ```

4. Obtenga el nuevo PID y consulte nuevamente el endpoint:

   ```bash
   NEW_PID="$(systemctl show -p MainPID --value telemetry-api.service)"
   echo "PID nuevo: ${NEW_PID}"

   curl -fsS http://127.0.0.1:9080/health
   sudo journalctl -u telemetry-api.service -n 30 --no-pager
   ```

5. Confirme que el archivo de estado fue renovado por la nueva instancia:

   ```bash
   sudo cat /var/lib/linux-essentials/telemetry/last-start.txt
   ```

**Salida esperada:**

El servicio debe volver a `active (running)`. El nuevo PID debe ser diferente al PID almacenado antes de enviar `SIGKILL`.

Los registros normalmente incluirán una secuencia similar a:

```text
Main process exited, code=killed, status=9/KILL
Scheduled restart job, restart counter is 1.
Started API local de salud para Linux Essentials.
```

**Verificación:**

```bash
test "${OLD_PID}" != "${NEW_PID}" && echo "Recuperación automática validada"
systemctl is-active telemetry-api.service
```

---

### Paso 5: Configurar acceso remoto seguro mediante OpenSSH

**Objetivo:** exigir autenticación por clave pública y restringir el usuario de automatización `opsremote`.

**Instrucciones:**

1. Desde `desk-lab` o la estación administrativa, genere una clave para `labadmin` si aún no existe. No genere ni copie la clave privada al servidor:

   ```bash
   ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_labadmin -C "labadmin@srv-lab"
   ```

2. Muestre la clave pública generada y cópiela:

   ```bash
   cat ~/.ssh/id_ed25519_labadmin.pub
   ```

3. En `srv-lab`, cree el directorio SSH de `labadmin` y agregue la clave pública. Sustituya el texto de ejemplo por la clave pública real:

   ```bash
   install -d -m 0700 ~/.ssh

   cat >> ~/.ssh/authorized_keys <<'EOF'
   ssh-ed25519 AAAA_REEMPLAZAR_CON_LA_CLAVE_PUBLICA_DE_LABADMIN labadmin@srv-lab
   EOF

   chmod 0600 ~/.ssh/authorized_keys
   ```

4. Cree el usuario de automatización restringido:

   ```bash
   sudo groupadd --system opsremote 2>/dev/null || true

   sudo useradd \
     --system \
     --gid opsremote \
     --create-home \
     --home-dir /home/opsremote \
     --shell /bin/bash \
     opsremote 2>/dev/null || true

   sudo install -d -o opsremote -g opsremote -m 0700 /home/opsremote/.ssh
   ```

5. Cree un comando controlado que será el único comando disponible para la clave de `opsremote`:

   ```bash
   sudo tee /usr/local/sbin/opsremote-status > /dev/null <<'EOF'
   #!/usr/bin/env bash
   set -eu

   echo "hostname=$(hostname)"
   echo "kernel=$(uname -r)"
   echo "telemetry-api=$(systemctl is-active telemetry-api.service)"
   echo "node-exporter=$(systemctl is-active node-exporter.service 2>/dev/null || true)"
   EOF

   sudo chown root:root /usr/local/sbin/opsremote-status
   sudo chmod 0755 /usr/local/sbin/opsremote-status
   ```

6. Desde la estación administrativa, genere una clave específica para automatización si es necesario:

   ```bash
   ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_opsremote -C "opsremote@srv-lab"
   cat ~/.ssh/id_ed25519_opsremote.pub
   ```

7. En `srv-lab`, agregue la clave pública de automatización con restricciones. Sustituya el contenido de la clave por el valor real:

   ```bash
   sudo tee /home/opsremote/.ssh/authorized_keys > /dev/null <<'EOF'
   restrict,command="/usr/local/sbin/opsremote-status" ssh-ed25519 AAAA_REEMPLAZAR_CON_LA_CLAVE_PUBLICA_DE_OPSREMOTE opsremote@srv-lab
   EOF

   sudo chown opsremote:opsremote /home/opsremote/.ssh/authorized_keys
   sudo chmod 0600 /home/opsremote/.ssh/authorized_keys
   ```

8. Cree una configuración local de OpenSSH. Primero mantenga abierta la consola de VirtualBox y una sesión administrativa activa:

   ```bash
   sudo tee /etc/ssh/sshd_config.d/60-linux-essentials.conf > /dev/null <<'EOF'
   PubkeyAuthentication yes
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin no

   Match User opsremote
       AuthenticationMethods publickey
       PermitTTY no
       AllowTcpForwarding no
       X11Forwarding no
       PermitTunnel no
   EOF
   ```

9. Valide la sintaxis antes de recargar OpenSSH:

   ```bash
   sudo sshd -t
   sudo systemctl reload ssh.service
   sudo systemctl status ssh.service --no-pager
   ```

10. Desde `desk-lab`, valide primero el acceso de `labadmin` mediante clave:

   ```bash
   ssh -i ~/.ssh/id_ed25519_labadmin labadmin@192.168.56.20
   ```

11. Valide el usuario restringido. El servidor debe ejecutar el comando forzado y no abrir una shell interactiva:

   ```bash
   ssh -i ~/.ssh/id_ed25519_opsremote opsremote@192.168.56.20
   ```

**Salida esperada:**

El acceso de `labadmin` debe abrir una sesión normal autenticada con clave pública. El acceso de `opsremote` debe devolver datos similares a:

```text
hostname=srv-lab
kernel=6.8.0-55-generic
telemetry-api=active
node-exporter=inactive
```

En este punto `node-exporter` todavía puede aparecer como `inactive`; se iniciará en el siguiente paso.

**Verificación:**

Desde `desk-lab`, intente una autenticación por contraseña y confirme que falla:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no \
  labadmin@192.168.56.20
```

El resultado esperado es una denegación de autenticación. No elimine ni sobrescriba las claves privadas de las estaciones administrativas.

---

### Paso 6: Instalar y operar Prometheus Node Exporter 1.8.2

**Objetivo:** desplegar Node Exporter como daemon de mínimo privilegio, limitado al puerto `192.168.56.20:9100`.

**Instrucciones:**

1. Confirme la arquitectura. Este procedimiento está diseñado para `amd64`:

   ```bash
   uname -m
   ```

   Debe mostrar `x86_64`.

2. Cree el usuario de sistema para Node Exporter:

   ```bash
   sudo groupadd --system node_exporter 2>/dev/null || true

   sudo useradd \
     --system \
     --gid node_exporter \
     --no-create-home \
     --home-dir /nonexistent \
     --shell /usr/sbin/nologin \
     node_exporter 2>/dev/null || true
   ```

3. Descargue el binario y el archivo de sumas de verificación sin actualizar paquetes del sistema:

   ```bash
   export NODE_EXPORTER_VERSION="1.8.2"
   cd /tmp

   curl -fLO "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/sha256sums.txt"
   curl -fLO "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"

   grep "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz$" sha256sums.txt | sha256sum -c -
   ```

4. Instale el binario en una ruta administrada:

   ```bash
   tar -xzf "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"

   sudo install -d -o root -g root -m 0755 /opt/linux-essentials/node-exporter
   sudo install -o root -g root -m 0755 \
     "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter" \
     /opt/linux-essentials/node-exporter/node_exporter
   ```

5. Cree la unidad de servicio:

   ```bash
   sudo tee /etc/systemd/system/node-exporter.service > /dev/null <<'EOF'
   [Unit]
   Description=Prometheus Node Exporter 1.8.2
   Wants=network-online.target
   After=network-online.target

   [Service]
   Type=simple
   User=node_exporter
   Group=node_exporter
   ExecStart=/opt/linux-essentials/node-exporter/node_exporter --web.listen-address=192.168.56.20:9100
   Restart=on-failure
   RestartSec=5

   NoNewPrivileges=true
   ProtectSystem=full
   ProtectHome=true
   PrivateTmp=true
   CapabilityBoundingSet=
   AmbientCapabilities=
   LockPersonality=true
   PrivateDevices=true
   ProtectKernelTunables=true
   RestrictSUIDSGID=true
   SystemCallArchitectures=native

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

6. Guarde una copia de la unidad en la infraestructura persistente:

   ```bash
   sudo install -m 0644 /etc/systemd/system/node-exporter.service \
     /opt/linux-essentials/infra/systemd/node-exporter.service
   ```

7. Habilite e inicie el servicio:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now node-exporter.service
   systemctl status node-exporter.service --no-pager
   ```

8. Confirme que el puerto está asociado solamente a la interfaz Host-Only:

   ```bash
   sudo ss -ltnp | grep ':9100'
   curl -fsS http://192.168.56.20:9100/metrics | head -n 20
   ```

9. Desde `desk-lab`, consulte métricas remotas:

   ```bash
   curl -fsS http://192.168.56.20:9100/metrics | grep -E '^node_(cpu|memory|filesystem)' | head
   ```

**Salida esperada:**

La validación SHA-256 debe indicar:

```text
node_exporter-1.8.2.linux-amd64.tar.gz: OK
```

El socket debe mostrar una asociación similar a:

```text
LISTEN 0 4096 192.168.56.20:9100 ...
```

**Verificación:**

```bash
systemctl is-enabled node-exporter.service
systemctl is-active node-exporter.service
curl -fsS http://192.168.56.20:9100/metrics | grep '^node_exporter_build_info'
```

La métrica `node_exporter_build_info` debe informar la versión `1.8.2`.

---

### Paso 7: Crear la validación persistente de arranque

**Objetivo:** registrar una evidencia local de disponibilidad de red, estado del daemon y versión del kernel durante cada arranque.

**Instrucciones:**

1. Cree el directorio de evidencias y el script de validación:

   ```bash
   sudo install -d -o root -g root -m 0755 /var/log/linux-essentials

   sudo tee /usr/local/sbin/boot-validation.sh > /dev/null <<'EOF'
   #!/usr/bin/env bash
   set -u

   LOG_FILE="/var/log/linux-essentials/boot-validation.log"

   {
     echo "=== boot-validation $(date --iso-8601=seconds) ==="
     echo "hostname=$(hostname)"
     echo "kernel=$(uname -r)"

     if ip -4 addr show dev enp0s3 scope global | grep -q 'inet '; then
       echo "nat_network=available"
     else
       echo "nat_network=unavailable"
     fi

     if ip -4 addr show dev enp0s8 | grep -q '192.168.56.20/24'; then
       echo "hostonly_network=available"
     else
       echo "hostonly_network=unavailable"
     fi

     echo "telemetry_api=$(systemctl is-active telemetry-api.service || true)"
     echo "node_exporter=$(systemctl is-active node-exporter.service || true)"
     echo
   } >> "${LOG_FILE}"
   EOF

   sudo chown root:root /usr/local/sbin/boot-validation.sh
   sudo chmod 0755 /usr/local/sbin/boot-validation.sh
   ```

2. Cree la unidad `boot-validation.service`:

   ```bash
   sudo tee /etc/systemd/system/boot-validation.service > /dev/null <<'EOF'
   [Unit]
   Description=Validación de red, kernel y servicios al finalizar el arranque
   Wants=network-online.target telemetry-api.service node-exporter.service
   After=network-online.target telemetry-api.service node-exporter.service

   [Service]
   Type=oneshot
   ExecStart=/usr/local/sbin/boot-validation.sh
   RemainAfterExit=yes

   [Install]
   WantedBy=multi-user.target
   EOF
   ```

3. Conserve una copia de la unidad:

   ```bash
   sudo install -m 0644 /etc/systemd/system/boot-validation.service \
     /opt/linux-essentials/infra/systemd/boot-validation.service
   ```

4. Recargue unidades, habilite la validación y ejecútela una vez manualmente:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable boot-validation.service
   sudo systemctl start boot-validation.service
   ```

5. Revise su estado y las evidencias registradas:

   ```bash
   systemctl status boot-validation.service --no-pager
   sudo tail -n 20 /var/log/linux-essentials/boot-validation.log
   ```

6. Analice la cadena de arranque de los servicios relevantes:

   ```bash
   systemd-analyze critical-chain telemetry-api.service
   systemd-analyze critical-chain boot-validation.service
   systemd-analyze blame | head -n 20
   ```

7. Reinicie controladamente desde la consola o desde una sesión SSH ya validada:

   ```bash
   sudo reboot
   ```

8. Después del reinicio, vuelva a iniciar sesión y revise las evidencias:

   ```bash
   systemctl is-active telemetry-api.service
   systemctl is-active node-exporter.service
   systemctl is-active boot-validation.service

   sudo tail -n 20 /var/log/linux-essentials/boot-validation.log
   ```

**Salida esperada:**

El log debe contener un bloque comparable al siguiente:

```text
=== boot-validation 2026-... ===
hostname=srv-lab
kernel=6.8.0-55-generic
nat_network=available
hostonly_network=available
telemetry_api=active
node_exporter=active
```

**Verificación:**

```bash
systemctl is-enabled boot-validation.service
sudo journalctl -b -u boot-validation.service --no-pager
sudo test -s /var/log/linux-essentials/boot-validation.log && echo "Evidencia de boot disponible"
```

---

### Paso 8: Consolidar evidencias y documentar el estado operativo

**Objetivo:** reunir la evidencia técnica mínima para laboratorios posteriores y confirmar que los daemons funcionan como servicios persistentes.

**Instrucciones:**

1. Genere un informe local resumido de servicios y sockets:

   ```bash
   sudo tee /var/log/linux-essentials/lab-01-00-01-summary.txt > /dev/null <<'EOF'
   Laboratorio: 01-00-01
   Fecha de generación:
   EOF

   {
     date --iso-8601=seconds
     echo
     echo "=== Hostname ==="
     hostnamectl
     echo
     echo "=== Kernel ==="
     uname -r
     echo
     echo "=== Direcciones IP ==="
     ip -br address
     echo
     echo "=== Servicios ==="
     systemctl --no-pager --full status telemetry-api.service node-exporter.service boot-validation.service
     echo
     echo "=== Puertos ==="
     ss -ltnp
     echo
     echo "=== Validación HTTP local ==="
     curl -fsS http://127.0.0.1:9080/health
     echo
     echo "=== Validación Node Exporter ==="
     curl -fsS http://192.168.56.20:9100/metrics | grep '^node_exporter_build_info'
   } | sudo tee -a /var/log/linux-essentials/lab-01-00-01-summary.txt > /dev/null
   ```

2. Proteja las evidencias para evitar modificaciones accidentales por usuarios no administrativos:

   ```bash
   sudo chown root:root /var/log/linux-essentials/lab-01-00-01-summary.txt
   sudo chmod 0640 /var/log/linux-essentials/lab-01-00-01-summary.txt
   ```

3. Revise los archivos persistentes creados:

   ```bash
   sudo find /opt/linux-essentials -maxdepth 3 -type f -printf '%p\n' | sort
   sudo find /etc/systemd/system -maxdepth 1 \
     \( -name 'telemetry-api.service' -o -name 'node-exporter.service' -o -name 'boot-validation.service' \) \
     -printf '%p\n'
   sudo ls -l /var/log/linux-essentials/
   ```

4. Si el instructor solicita una instantánea, apague o deje estable la VM y cree una instantánea de VirtualBox con un nombre descriptivo, por ejemplo:

   ```text
   lab-01-00-01-servicio-persistente-validado
   ```

**Salida esperada:**

Debe existir el archivo:

```text
/var/log/linux-essentials/lab-01-00-01-summary.txt
```

También deben existir las tres unidades locales de `systemd`, los directorios de aplicación e infraestructura y los registros de arranque.

**Verificación:**

```bash
sudo cat /var/log/linux-essentials/lab-01-00-01-summary.txt
systemctl list-unit-files | grep -E 'telemetry-api|node-exporter|boot-validation'
```

## Validación y Pruebas

Ejecute la siguiente matriz de validación final desde `srv-lab` y, cuando se indique, desde `desk-lab`.

| Prueba | Comando | Resultado esperado |
|---|---|---|
| PID 1 | `ps -p 1 -o pid,comm,args` | PID 1 corresponde a `systemd`. |
| Servicio Python | `systemctl is-active telemetry-api.service` | `active`. |
| Inicio automático | `systemctl is-enabled telemetry-api.service` | `enabled`. |
| Salud local | `curl -fsS http://127.0.0.1:9080/health` | JSON con `"status": "ok"`. |
| Exposición limitada | `sudo ss -ltnp \| grep ':9080'` | Escucha solo en `127.0.0.1:9080`. |
| Recuperación | Enviar `SIGKILL` al PID y esperar 7 segundos | Nuevo PID y servicio `active`. |
| Node Exporter | `curl -fsS http://192.168.56.20:9100/metrics \| grep node_exporter_build_info` | Métrica con versión `1.8.2`. |
| Puerto Node Exporter | `sudo ss -ltnp \| grep ':9100'` | Escucha en `192.168.56.20:9100`. |
| SSH de labadmin | `ssh -i ~/.ssh/id_ed25519_labadmin labadmin@192.168.56.20` | Inicio por clave pública. |
| SSH restringido | `ssh -i ~/.ssh/id_ed25519_opsremote opsremote@192.168.56.20` | Ejecuta solo `opsremote-status`. |
| Boot personalizado | `sudo tail -n 20 /var/log/linux-essentials/boot-validation.log` | Registra kernel, red y servicios activos. |
| Cadena de arranque | `systemd-analyze critical-chain boot-validation.service` | Muestra dependencia posterior a red y servicios. |

Como evidencia adicional, consulte los registros del arranque actual:

```bash
sudo journalctl -b -u telemetry-api.service --no-pager
sudo journalctl -b -u node-exporter.service --no-pager
sudo journalctl -b -u boot-validation.service --no-pager
```

## Solución de Problemas

### Problema 1: telemetry-api.service falla con errores de permisos o no puede crear last-start.txt

**Síntomas:**

```text
telemetry-api.service: Failed with result 'exit-code'
PermissionError: [Errno 13] Permission denied
```

o el endpoint no responde en `127.0.0.1:9080`.

**Causa:** el directorio `/var/lib/linux-essentials/telemetry` no pertenece a `telemetrysvc`, o la unidad tiene `ProtectSystem=strict` sin una ruta declarada en `ReadWritePaths`.

**Corrección:**

```bash
sudo install -d -o telemetrysvc -g telemetrysvc -m 0750 \
  /var/lib/linux-essentials/telemetry

sudo systemctl daemon-reload
sudo systemctl restart telemetry-api.service

systemctl status telemetry-api.service --no-pager
sudo journalctl -u telemetry-api.service -n 30 --no-pager
```

Confirme que la unidad conserva esta línea:

```ini
ReadWritePaths=/var/lib/linux-essentials/telemetry
```

### Problema 2: se pierde el acceso SSH después de aplicar la configuración de OpenSSH

**Síntomas:**

```text
Permission denied (publickey).
```

o el servicio SSH no recarga debido a un error de sintaxis.

**Causa:** clave pública mal copiada, permisos inseguros en `~/.ssh`, configuración inválida en `/etc/ssh/sshd_config.d/60-linux-essentials.conf`, o intento de acceso por contraseña después de deshabilitar `PasswordAuthentication`.

**Corrección:**

Use la consola de VirtualBox y valide la configuración:

```bash
sudo sshd -t
sudo ls -ld /home/labadmin/.ssh
sudo ls -l /home/labadmin/.ssh/authorized_keys
sudo cat /home/labadmin/.ssh/authorized_keys
```

Corrija permisos y recargue el servicio:

```bash
sudo chown -R labadmin:labadmin /home/labadmin/.ssh
sudo chmod 0700 /home/labadmin/.ssh
sudo chmod 0600 /home/labadmin/.ssh/authorized_keys

sudo systemctl reload ssh.service
sudo systemctl status ssh.service --no-pager
```

Verifique desde la estación administrativa que usa explícitamente la clave correcta:

```bash
ssh -i ~/.ssh/id_ed25519_labadmin -o IdentitiesOnly=yes labadmin@192.168.56.20
```

## Limpieza

Este laboratorio produce componentes persistentes requeridos por laboratorios posteriores. No elimine los siguientes elementos:

- `/opt/linux-essentials/telemetry/app.py`
- `/opt/linux-essentials/infra/`
- `/var/lib/linux-essentials/telemetry/`
- `/var/log/linux-essentials/`
- `telemetry-api.service`
- `node-exporter.service`
- `boot-validation.service`
- Usuarios `telemetrysvc`, `node_exporter` y `opsremote`
- Configuración SSH basada en claves públicas.

Elimine únicamente archivos temporales de descarga utilizados durante la instalación de Node Exporter:

```bash
rm -rf /tmp/node_exporter-1.8.2.linux-amd64
rm -f /tmp/node_exporter-1.8.2.linux-amd64.tar.gz
rm -f /tmp/sha256sums.txt
```

Confirme que los servicios continúan activos después de la limpieza:

```bash
systemctl is-active telemetry-api.service
systemctl is-active node-exporter.service
systemctl is-active boot-validation.service
```

## Resumen

En este laboratorio se implementó un daemon Python administrado por `systemd`, ejecutado como `telemetrysvc` y expuesto únicamente mediante `127.0.0.1:9080`. Se aplicaron controles de seguridad como `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`, límites de CPU, memoria, tareas y capacidades vacías.

También se configuró autenticación SSH por clave pública, se restringió el usuario de automatización `opsremote`, se desplegó Node Exporter 1.8.2 en `192.168.56.20:9100` y se registró evidencia del arranque con `boot-validation.service`. La VM `srv-lab` queda preparada como base persistente para los laboratorios posteriores de red, automatización, contenedores, monitoreo y recuperación operacional.

Recursos de consulta:

- `man systemd.service`
- `man systemd.exec`
- `man systemctl`
- `man journalctl`
- `man systemd-analyze`
- `man sshd_config`
- Documentación de Prometheus Node Exporter: <https://github.com/prometheus/node_exporter>
- Documentación de systemd: <https://www.freedesktop.org/software/systemd/man/latest/>
