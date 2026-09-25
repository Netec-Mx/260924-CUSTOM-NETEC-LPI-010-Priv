# Laboratorio 10: Despliegue Multi-Entorno y Recuperación ante Caídas de X11/Wayland

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 123 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción General

En este laboratorio se despliega en `desk-lab` la infraestructura declarativa previamente creada en `srv-lab`, utilizando perfiles de Docker Compose para los entornos de desarrollo y producción. Se compara el comportamiento de sesiones GNOME sobre Wayland y Xorg, identificando los componentes implicados: GDM, GNOME Shell/Mutter, compositor, sesión de usuario y servicios systemd. Finalmente, se simula de forma reversible un fallo de sesión gráfica limitado al usuario `labadmin`, se recupera el entorno desde una TTY y se comprueba que la administración remota y los contenedores continúan disponibles.

> **Advertencia de seguridad:** ejecute los fallos controlados únicamente dentro de la VM `desk-lab`. No deshabilite GDM, no modifique archivos de sesión y no realice pruebas de recuperación en el sistema anfitrión ni en `srv-lab`.

## Objetivos de Aprendizaje

- [ ] Copiar y desplegar el repositorio YAML de infraestructura desde `srv-lab` hacia `desk-lab` mediante SSH/SCP.
- [ ] Ejecutar perfiles `dev` y `prod` de Docker Compose usando archivos de entorno separados y los puertos locales `8081` y `8082`.
- [ ] Identificar y comparar sesiones GNOME Wayland y GNOME on Xorg mediante `loginctl`, variables de entorno, GDM y registros del sistema.
- [ ] Diagnosticar y recuperar un fallo controlado de inicio de sesión gráfico utilizando una TTY, `journalctl`, `systemctl` y una configuración reversible de usuario.
- [ ] Documentar un procedimiento operativo que mantenga disponibles la administración SSH y los servicios de contenedores durante una incidencia gráfica.

## Prerrequisitos

Conocimientos requeridos:

- Uso de comandos Linux, permisos, rutas y edición básica de archivos.
- Uso de `systemctl`, `journalctl`, `loginctl` y acceso a TTY mediante `Ctrl+Alt+F3`.
- Comprensión básica de YAML, Docker Compose, perfiles y archivos `.env`.
- Comprensión de la diferencia entre GDM, GNOME, Wayland, X11/Xorg y una sesión de usuario.

Accesos y estado requerido:

- VM `srv-lab` disponible en el estado `lab09-yaml-operativo`.
- VM `desk-lab` con Ubuntu Desktop 24.04.2 LTS, GNOME 46.0, GDM 46.2 y Docker Engine 27.5.1.
- Adaptadores de red configurados según la convención del curso:
  - `srv-lab`: `192.168.56.20/24` en `enp0s8`.
  - `desk-lab`: `192.168.56.30/24` en `enp0s8`.
  - `enp0s3`: NAT mediante DHCP en ambas VMs.
- Usuario administrativo `labadmin` disponible en ambas máquinas.
- Acceso a la consola de VirtualBox para `desk-lab`.
- Servicio SSH operativo en ambas VMs.
- Repositorio `/opt/linux-essentials/infra` disponible en `srv-lab`.
- El usuario `labadmin` debe poder ejecutar Docker en `desk-lab`, normalmente mediante pertenencia al grupo `docker`.

## Entorno de Laboratorio

| Componente | `srv-lab` | `desk-lab` |
|---|---|---|
| Sistema operativo | Ubuntu Server 24.04.2 LTS | Ubuntu Desktop 24.04.2 LTS |
| Hostname | `srv-lab` | `desk-lab` |
| IP Host-Only | `192.168.56.20` | `192.168.56.30` |
| Usuario administrativo | `labadmin` | `labadmin` |
| Servicio gráfico | No aplica | GNOME 46.0 con GDM 46.2 |
| Docker Compose | Infraestructura origen | Perfiles `dev` y `prod` |
| Puerto de desarrollo local | No aplica | `127.0.0.1:8081` |
| Puerto de producción local | No aplica | `127.0.0.1:8082` |
| Endpoint productivo del servidor | `http://192.168.56.20:8080/health` | Consumido desde `desk-lab` |

Ejecute inicialmente en `desk-lab`:

```bash
hostnamectl
ip -brief address
docker --version
docker compose version
systemctl is-active ssh
systemctl is-active gdm3
```

Cree un directorio de evidencias de la práctica. Los registros del incidente gráfico se conservarán localmente para revisión:

```bash
sudo install -d -m 0755 -o root -g adm /var/log/linux-essentials
sudo touch /var/log/linux-essentials/lab10-evidencias.log
sudo chmod 0640 /var/log/linux-essentials/lab10-evidencias.log
```

> **Nota:** no ejecute actualizaciones de distribución, kernel, systemd, GNOME, GDM, Docker o paquetes durante este laboratorio.

## Instrucciones Paso a Paso

### Paso 1: Verificar el estado inicial y proteger la VM

**Objetivo:** Confirmar que `desk-lab` y `srv-lab` están accesibles, que las direcciones IP coinciden con la topología del curso y que existe un punto de retorno antes de provocar el incidente gráfico.

**Instrucciones:**

1. En VirtualBox, compruebe que las VMs persistentes se denominan exactamente `srv-lab` y `desk-lab`.

2. Cree una instantánea de `desk-lab` desde VirtualBox con un nombre descriptivo, por ejemplo:

   ```text
   lab10-antes-fallo-grafico
   ```

3. En una terminal de `desk-lab`, compruebe identidad, interfaces y conectividad con el servidor:

   ```bash
   hostnamectl --static
   ip -brief address
   ping -c 2 192.168.56.20
   ```

4. Verifique que los servicios críticos de continuidad están activos:

   ```bash
   systemctl is-active ssh
   systemctl is-active docker
   systemctl is-active gdm3
   ```

5. Registre la información inicial:

   ```bash
   {
     echo "===== LAB10: ESTADO INICIAL $(date --iso-8601=seconds) ====="
     hostnamectl
     ip -brief address
     systemctl is-active ssh docker gdm3
     docker compose version
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- El hostname es `desk-lab`.
- La interfaz `enp0s8` posee `192.168.56.30/24`.
- El ping a `192.168.56.20` responde.
- `ssh`, `docker` y `gdm3` aparecen como `active`.

**Verificación:**

```bash
test "$(hostnamectl --static)" = "desk-lab" && echo "Hostname correcto"
ip -brief address show enp0s8
systemctl --no-pager --full status ssh docker gdm3
```

---

### Paso 2: Copiar el repositorio de infraestructura mediante SSH/SCP

**Objetivo:** Transferir el repositorio YAML desde `srv-lab` a `desk-lab` sin editar los manifiestos declarativos de Docker Compose.

**Instrucciones:**

1. Desde `desk-lab`, pruebe primero el acceso SSH al servidor:

   ```bash
   ssh labadmin@192.168.56.20 'hostnamectl --static; ls -ld /opt/linux-essentials/infra'
   ```

2. Cree el directorio de destino en `desk-lab` y asígnele propiedad a `labadmin`:

   ```bash
   sudo install -d -m 0755 -o labadmin -g labadmin /opt/linux-essentials
   ```

3. Copie el repositorio mediante SCP. Si ya existe una copia anterior, consérvela como respaldo antes de reemplazarla:

   ```bash
   if [ -d /opt/linux-essentials/infra ]; then
     mv /opt/linux-essentials/infra "/opt/linux-essentials/infra.backup.$(date +%Y%m%d-%H%M%S)"
   fi

   scp -r labadmin@192.168.56.20:/opt/linux-essentials/infra /opt/linux-essentials/
   ```

4. Ajuste propietario y permisos básicos del repositorio copiado:

   ```bash
   sudo chown -R labadmin:labadmin /opt/linux-essentials/infra
   find /opt/linux-essentials/infra -type d -exec chmod 0755 {} \;
   find /opt/linux-essentials/infra -type f -exec chmod 0644 {} \;
   ```

5. Inspeccione los archivos transferidos sin mostrar secretos completos de archivos `.env`:

   ```bash
   cd /opt/linux-essentials/infra
   find . -maxdepth 3 -type f | sort
   find . -maxdepth 2 -type f \( -name 'compose*.yaml' -o -name 'compose*.yml' \) -print
   ```

6. Registre la procedencia y el contenido general del repositorio:

   ```bash
   {
     echo "===== LAB10: REPOSITORIO COPIADO $(date --iso-8601=seconds) ====="
     find /opt/linux-essentials/infra -maxdepth 2 -type f | sort
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- SSH solicita contraseña o usa la clave configurada para `labadmin`.
- El directorio `/opt/linux-essentials/infra` existe en `desk-lab`.
- Se observan uno o más archivos Compose y los archivos de entorno autorizados por el Laboratorio 3.

**Verificación:**

```bash
cd /opt/linux-essentials/infra
test -d . && echo "Repositorio disponible"
find . -maxdepth 2 -type f -name 'compose*.y*ml' -print
```

---

### Paso 3: Revisar perfiles y preparar los archivos de entorno autorizados

**Objetivo:** Confirmar que los servicios usan perfiles Docker Compose y configurar solamente los valores de entorno permitidos para los puertos locales de desarrollo y producción.

**Instrucciones:**

1. Sitúese en el repositorio:

   ```bash
   cd /opt/linux-essentials/infra
   ```

2. Identifique el archivo Compose principal. En los comandos siguientes se asumirá `compose.yaml`; si el repositorio usa otro nombre, reemplace el argumento `-f compose.yaml` por el archivo correspondiente:

   ```bash
   ls -l compose.yaml compose.yml 2>/dev/null
   ```

3. Revise servicios, perfiles, redes, volúmenes y referencias a variables sin modificar el YAML:

   ```bash
   grep -nE 'profiles:|ports:|environment:|\$\{' compose.yaml
   docker compose -f compose.yaml config --profiles
   ```

4. Localice los archivos de entorno permitidos:

   ```bash
   find . -maxdepth 2 -type f \( -name '.env*' -o -name '*dev*.env' -o -name '*prod*.env' \) -print
   ```

5. Edite únicamente los archivos de entorno autorizados por el repositorio del Laboratorio 3. El resultado funcional requerido es el siguiente:

   | Entorno | Dirección publicada | Puerto requerido |
   |---|---:|---:|
   | Desarrollo | `127.0.0.1` | `8081` |
   | Producción | `127.0.0.1` | `8082` |

   Si el repositorio utiliza variables como `HOST_BIND`, `HOST_PORT`, `BIND_ADDRESS` o equivalentes, asigne los valores correspondientes. Un ejemplo típico de archivos permitidos sería:

   ```dotenv
   # .env.dev
   HOST_BIND=127.0.0.1
   HOST_PORT=8081
   ```

   ```dotenv
   # .env.prod
   HOST_BIND=127.0.0.1
   HOST_PORT=8082
   ```

   Edite con:

   ```bash
   nano .env.dev
   nano .env.prod
   ```

   > No agregue puertos `8080`, `9080` o `9100` en `desk-lab`. Estos puertos están reservados según la topología del curso.

6. Valide sintácticamente la renderización de cada entorno. Sustituya los nombres de archivo si en su repositorio son distintos:

   ```bash
   docker compose -f compose.yaml --env-file .env.dev --profile dev config > /tmp/lab10-dev-rendered.yaml
   docker compose -f compose.yaml --env-file .env.prod --profile prod config > /tmp/lab10-prod-rendered.yaml

   grep -nE '127\.0\.0\.1:8081|127\.0\.0\.1:8082|8081:|8082:' /tmp/lab10-*-rendered.yaml
   ```

**Salida esperada:**

- La configuración renderizada no presenta errores de sintaxis.
- El perfil `dev` publica exclusivamente el puerto local `127.0.0.1:8081`.
- El perfil `prod` publica exclusivamente el puerto local `127.0.0.1:8082`.
- No se modifican los archivos YAML de Compose.

**Verificación:**

```bash
git -C /opt/linux-essentials/infra status --short 2>/dev/null || true
grep -RInE '8080|9080|9100' .env.dev .env.prod 2>/dev/null || true
```

La salida de `git status` debe mostrar únicamente cambios en los archivos de entorno permitidos, si el repositorio está bajo control de versiones.

---

### Paso 4: Desplegar los perfiles de desarrollo y producción

**Objetivo:** Ejecutar perfiles independientes de Docker Compose en `desk-lab` usando nombres de proyecto diferentes para evitar colisiones entre los entornos.

**Instrucciones:**

1. Compruebe que `labadmin` puede utilizar Docker sin `sudo`:

   ```bash
   docker ps
   ```

2. Inicie el perfil de desarrollo. Se emplea el proyecto `le-dev` para aislar sus contenedores, redes y nombres:

   ```bash
   cd /opt/linux-essentials/infra

   docker compose \
     -p le-dev \
     -f compose.yaml \
     --env-file .env.dev \
     --profile dev \
     up -d
   ```

3. Inicie el perfil de producción usando el proyecto `le-prod`:

   ```bash
   docker compose \
     -p le-prod \
     -f compose.yaml \
     --env-file .env.prod \
     --profile prod \
     up -d
   ```

4. Revise el estado de ambos proyectos:

   ```bash
   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev ps
   docker compose -p le-prod -f compose.yaml --env-file .env.prod --profile prod ps
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
   ```

5. Verifique que los sockets locales están asociados a Docker:

   ```bash
   ss -ltnp | grep -E ':8081|:8082'
   ```

6. Pruebe los endpoints de la aplicación. Si el manifiesto del Laboratorio 3 define otra ruta de salud, use la ruta declarada en dicho manifiesto:

   ```bash
   curl -i http://127.0.0.1:8081/health
   curl -i http://127.0.0.1:8082/health
   ```

7. Guarde una evidencia resumida:

   ```bash
   {
     echo "===== LAB10: DOCKER MULTI-ENTORNO $(date --iso-8601=seconds) ====="
     docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
     ss -ltn | grep -E ':8081|:8082' || true
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- Los contenedores de `le-dev` y `le-prod` aparecen en estado `Up` o `running`.
- El puerto `8081` está publicado únicamente en `127.0.0.1`.
- El puerto `8082` está publicado únicamente en `127.0.0.1`.
- Las solicitudes HTTP responden con un código de éxito, normalmente `200 OK`.

**Verificación:**

```bash
curl --fail --silent --show-error http://127.0.0.1:8081/health
curl --fail --silent --show-error http://127.0.0.1:8082/health
docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev ps
docker compose -p le-prod -f compose.yaml --env-file .env.prod --profile prod ps
```

---

### Paso 5: Validar la conectividad hacia el servicio productivo de srv-lab

**Objetivo:** Confirmar que `desk-lab` puede administrar y verificar el entorno productivo remoto de `srv-lab` sin reutilizar ni alterar los puertos locales de la estación de escritorio.

**Instrucciones:**

1. Desde `desk-lab`, pruebe el endpoint publicado por Nginx en `srv-lab`:

   ```bash
   curl -i http://192.168.56.20:8080/health
   ```

2. Compruebe que la conexión se realiza hacia la dirección Host-Only correcta:

   ```bash
   ip route get 192.168.56.20
   ```

3. Valide el acceso SSH remoto y el estado de la infraestructura del servidor:

   ```bash
   ssh labadmin@192.168.56.20 '
     hostnamectl --static
     docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
     systemctl is-active nginx
   '
   ```

4. Registre el resultado HTTP sin incluir datos sensibles:

   ```bash
   {
     echo "===== LAB10: VALIDACION REMOTA SRV-LAB $(date --iso-8601=seconds) ====="
     curl --silent --show-error --fail http://192.168.56.20:8080/health
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- `curl` recibe una respuesta satisfactoria desde `192.168.56.20:8080`.
- La ruta hacia el servidor usa la interfaz Host-Only, normalmente `enp0s8`.
- El servicio Nginx del servidor está activo.

**Verificación:**

```bash
curl --fail --silent --show-error http://192.168.56.20:8080/health
ssh labadmin@192.168.56.20 'systemctl is-active nginx'
```

---

### Paso 6: Identificar la sesión gráfica actual y recopilar la línea base de GDM

**Objetivo:** Determinar qué tipo de sesión gráfica está activa y relacionarla con GDM, `loginctl`, el compositor y los registros del sistema.

**Instrucciones:**

1. Abra una terminal dentro de la sesión gráfica actual de `desk-lab`.

2. Consulte el tipo de sesión y la pantalla gráfica asociada:

   ```bash
   echo "XDG_SESSION_TYPE=$XDG_SESSION_TYPE"
   echo "DISPLAY=$DISPLAY"
   echo "WAYLAND_DISPLAY=$WAYLAND_DISPLAY"
   ```

3. Liste las sesiones conocidas por `logind`:

   ```bash
   loginctl list-sessions
   loginctl session-status
   ```

4. Identifique específicamente la sesión de `labadmin`:

   ```bash
   loginctl list-sessions --no-legend
   loginctl show-user labadmin
   ```

5. Consulte el estado del gestor de pantalla:

   ```bash
   systemctl status gdm3 --no-pager --full
   systemctl status display-manager --no-pager --full
   ```

6. Examine los registros recientes de GDM durante el arranque actual:

   ```bash
   sudo journalctl -b -u gdm3 --no-pager -n 80
   ```

7. Inspeccione el hardware gráfico y los dispositivos DRM disponibles:

   ```bash
   lspci -k | grep -A 3 -Ei 'vga|3d|display'
   ls -l /dev/dri/ 2>/dev/null || true
   ```

8. Guarde la línea base en el archivo de evidencias:

   ```bash
   {
     echo "===== LAB10: LINEA BASE GRAFICA $(date --iso-8601=seconds) ====="
     echo "XDG_SESSION_TYPE=$XDG_SESSION_TYPE"
     echo "DISPLAY=$DISPLAY"
     echo "WAYLAND_DISPLAY=$WAYLAND_DISPLAY"
     loginctl list-sessions
     systemctl is-active gdm3
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- `XDG_SESSION_TYPE` muestra `wayland` o `x11`.
- `gdm3` y `display-manager` aparecen activos.
- `loginctl` muestra una sesión gráfica asociada a `labadmin`.
- Los registros de GDM no muestran errores persistentes que impidan el inicio de sesión.

**Verificación:**

```bash
printf 'Tipo de sesión: %s\n' "$XDG_SESSION_TYPE"
loginctl session-status
systemctl is-active gdm3
```

---

### Paso 7: Comparar GNOME Wayland y GNOME on Xorg

**Objetivo:** Iniciar de forma documentada ambos tipos de sesión desde GDM y observar sus variables de entorno y diferencias operativas básicas.

**Instrucciones:**

1. En la sesión actual, guarde cualquier trabajo abierto y cierre sesión desde el menú de GNOME.

2. En la pantalla de GDM, seleccione el usuario `labadmin`.

3. Antes de introducir la contraseña, use el icono de engranaje de GDM y seleccione:

   ```text
   GNOME
   ```

   Esta opción normalmente inicia una sesión Wayland cuando el controlador gráfico y GDM lo permiten.

4. Inicie sesión y abra una terminal. Ejecute:

   ```bash
   echo "Tipo de sesión: $XDG_SESSION_TYPE"
   echo "Display X11: ${DISPLAY:-no definido}"
   echo "Display Wayland: ${WAYLAND_DISPLAY:-no definido}"
   loginctl session-status
   ```

5. Registre la evidencia de Wayland:

   ```bash
   {
     echo "===== LAB10: SESION WAYLAND $(date --iso-8601=seconds) ====="
     echo "XDG_SESSION_TYPE=$XDG_SESSION_TYPE"
     echo "DISPLAY=${DISPLAY:-no definido}"
     echo "WAYLAND_DISPLAY=${WAYLAND_DISPLAY:-no definido}"
     loginctl session-status
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

6. Cierre sesión nuevamente.

7. En GDM, seleccione `labadmin`, pulse el icono de engranaje y elija:

   ```text
   GNOME on Xorg
   ```

8. Inicie sesión y ejecute:

   ```bash
   echo "Tipo de sesión: $XDG_SESSION_TYPE"
   echo "Display X11: ${DISPLAY:-no definido}"
   echo "Display Wayland: ${WAYLAND_DISPLAY:-no definido}"
   ps -ef | grep -E '[X]org|[X]wayland|[m]utter'
   ```

9. Registre la evidencia de Xorg:

   ```bash
   {
     echo "===== LAB10: SESION XORG $(date --iso-8601=seconds) ====="
     echo "XDG_SESSION_TYPE=$XDG_SESSION_TYPE"
     echo "DISPLAY=${DISPLAY:-no definido}"
     echo "WAYLAND_DISPLAY=${WAYLAND_DISPLAY:-no definido}"
     ps -ef | grep -E '[X]org|[X]wayland|[m]utter' || true
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

10. Mantenga la sesión `GNOME on Xorg` activa para el siguiente paso, ya que el fallo controlado se aplicará de forma específica a esta modalidad.

**Salida esperada:**

- En GNOME Wayland, `XDG_SESSION_TYPE=wayland`; puede existir `WAYLAND_DISPLAY=wayland-0`.
- En GNOME on Xorg, `XDG_SESSION_TYPE=x11`; normalmente `DISPLAY=:0` y existe un proceso `Xorg`.
- GNOME continúa utilizando Mutter como componente central de la sesión, pero la capa de protocolo gráfico cambia entre Wayland y X11.

**Verificación:**

```bash
echo "$XDG_SESSION_TYPE"
test "$XDG_SESSION_TYPE" = "x11" && echo "Sesión Xorg preparada para el incidente"
```

---

### Paso 8: Preparar la continuidad administrativa antes del incidente

**Objetivo:** Confirmar que SSH, Docker y Docker Compose permanecen operables sin depender de la interfaz gráfica.

**Instrucciones:**

1. Desde una segunda terminal o desde `srv-lab`, abra una sesión SSH hacia `desk-lab`:

   ```bash
   ssh labadmin@192.168.56.30
   ```

2. Dentro de la sesión SSH a `desk-lab`, ejecute comprobaciones de continuidad:

   ```bash
   hostnamectl --static
   systemctl is-active ssh
   systemctl is-active docker
   cd /opt/linux-essentials/infra
   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev ps
   docker compose -p le-prod -f compose.yaml --env-file .env.prod --profile prod ps
   ```

3. Mantenga esta sesión SSH abierta durante el incidente gráfico. Será el canal remoto de administración.

4. Desde la sesión gráfica Xorg de `desk-lab`, verifique que los contenedores responden antes de simular el fallo:

   ```bash
   curl --fail --silent --show-error http://127.0.0.1:8081/health
   curl --fail --silent --show-error http://127.0.0.1:8082/health
   ```

5. Registre el plan de continuidad mínimo:

   ```bash
   {
     echo "===== LAB10: CONTINUIDAD PRE-INCIDENTE $(date --iso-8601=seconds) ====="
     echo "Canal remoto: ssh labadmin@192.168.56.30"
     echo "Consola local: Ctrl+Alt+F3"
     echo "Infraestructura: /opt/linux-essentials/infra"
     echo "Desarrollo: 127.0.0.1:8081"
     echo "Produccion: 127.0.0.1:8082"
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- La sesión SSH a `desk-lab` funciona mientras la sesión gráfica está activa.
- Docker y los dos proyectos Compose son administrables desde SSH.
- Los endpoints locales de ambos perfiles responden antes del incidente.

**Verificación:**

Desde la sesión SSH:

```bash
systemctl is-active ssh docker
docker ps
curl --fail --silent --show-error http://127.0.0.1:8081/health
curl --fail --silent --show-error http://127.0.0.1:8082/health
```

---

### Paso 9: Simular un fallo reversible de sesión gráfica Xorg

**Objetivo:** Deshabilitar temporalmente GDM y crear una configuración de usuario reversible que provoque el retorno de una sesión Xorg a GDM, sin modificar kernel, GRUB, `fstab` ni configuraciones globales de GNOME.

**Instrucciones:**

1. Desde la sesión SSH abierta o desde una terminal de `desk-lab`, deshabilite temporalmente GDM. Esto terminará la sesión gráfica activa; por ello, asegúrese de tener la sesión SSH disponible o acceso a la consola de VirtualBox:

   ```bash
   sudo systemctl disable --now gdm3
   ```

2. Confirme que GDM quedó detenido y deshabilitado:

   ```bash
   systemctl is-active gdm3
   systemctl is-enabled gdm3
   ```

3. Cree una configuración de usuario limitada a `labadmin`. El archivo `~/.xsession` se utiliza para este incidente exclusivamente bajo una sesión Xorg y finalizará inmediatamente con error controlado:

   ```bash
   cat > /home/labadmin/.xsession <<'EOF'
   # Fallo controlado del Laboratorio 10.
   # Este archivo se elimina durante la recuperación.
   logger -t lab10-xsession "Inicio Xorg bloqueado intencionalmente para prueba de recuperación"
   exit 1
   EOF

   chown labadmin:labadmin /home/labadmin/.xsession
   chmod 0700 /home/labadmin/.xsession
   ```

4. Registre una huella del archivo de fallo y el momento de la simulación:

   ```bash
   {
     echo "===== LAB10: INYECCION DE FALLO XORG $(date --iso-8601=seconds) ====="
     ls -l /home/labadmin/.xsession
     sha256sum /home/labadmin/.xsession
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

5. Reactive GDM:

   ```bash
   sudo systemctl enable --now gdm3
   ```

6. En la consola de VirtualBox, espere la pantalla de inicio de sesión de GDM.

7. Seleccione `labadmin`, elija explícitamente **GNOME on Xorg** en el engranaje e intente iniciar sesión.

8. Observe el resultado. La sesión debe fallar de forma controlada y regresar a la pantalla de GDM, o cerrar inmediatamente después de autenticarse.

9. No reinicie la VM. Pulse `Ctrl+Alt+F3` para acceder a una TTY.

**Salida esperada:**

- `gdm3` queda temporalmente detenido y luego vuelve a estar habilitado y activo.
- El archivo `/home/labadmin/.xsession` existe y pertenece únicamente a `labadmin`.
- La autenticación en `GNOME on Xorg` no completa una sesión gráfica funcional y regresa a GDM.
- La VM sigue accesible mediante TTY y SSH.

**Verificación:**

En TTY o SSH:

```bash
systemctl is-active gdm3
systemctl is-enabled gdm3
ls -l /home/labadmin/.xsession
ssh labadmin@192.168.56.30 'hostnamectl --static'
```

---

### Paso 10: Diagnosticar el fallo desde TTY y recuperar el entorno gráfico

**Objetivo:** Recopilar evidencias usando herramientas de consola, identificar la configuración de usuario responsable, revertirla y restaurar la sesión gráfica sin afectar los contenedores.

**Instrucciones:**

1. En la TTY abierta con `Ctrl+Alt+F3`, inicie sesión como `labadmin`.

2. Confirme que está en una consola de texto y no en una sesión gráfica:

   ```bash
   tty
   echo "XDG_SESSION_TYPE=${XDG_SESSION_TYPE:-no definido}"
   loginctl list-sessions
   ```

3. Revise el estado de GDM:

   ```bash
   sudo systemctl status gdm3 --no-pager --full
   ```

4. Revise los eventos de GDM del arranque actual:

   ```bash
   sudo journalctl -b -u gdm3 --no-pager -n 120
   ```

5. Busque la marca del incidente introducida por `logger`:

   ```bash
   sudo journalctl -b -t lab10-xsession --no-pager
   ```

6. Consulte eventos de la cuenta `labadmin` en el journal del sistema:

   ```bash
   sudo journalctl -b _UID="$(id -u labadmin)" --no-pager -n 120
   ```

7. Inspeccione la configuración específica del usuario y confirme el contenido problemático:

   ```bash
   ls -la /home/labadmin/.xsession
   cat /home/labadmin/.xsession
   ```

8. Guarde una evidencia diagnóstica antes de revertir el cambio:

   ```bash
   {
     echo "===== LAB10: DIAGNOSTICO EN TTY $(date --iso-8601=seconds) ====="
     tty
     systemctl status gdm3 --no-pager --full
     journalctl -b -u gdm3 --no-pager -n 80
     journalctl -b -t lab10-xsession --no-pager
     ls -l /home/labadmin/.xsession
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

9. Elimine el archivo de configuración reversible que provoca el fallo:

   ```bash
   rm -f /home/labadmin/.xsession
   ```

10. Restablezca propietario correcto del directorio personal y reactive GDM para asegurar una sesión limpia:

   ```bash
   sudo chown labadmin:labadmin /home/labadmin
   sudo systemctl enable gdm3
   sudo systemctl restart gdm3
   ```

11. Compruebe que GDM está activo:

   ```bash
   systemctl is-active gdm3
   systemctl is-enabled gdm3
   ```

12. Regrese a la consola gráfica con `Ctrl+Alt+F2` o `Ctrl+Alt+F1`, según la asignación de TTY mostrada por su VM.

13. Inicie sesión como `labadmin`. Primero pruebe **GNOME on Xorg** para confirmar que el fallo específico fue corregido. Después, si el tiempo lo permite, cierre sesión y pruebe también **GNOME** para validar Wayland.

**Salida esperada:**

- La TTY permite recopilar registros y administrar el sistema aunque falle la interfaz gráfica.
- `journalctl` muestra la marca `lab10-xsession` o eventos coherentes con el cierre de la sesión Xorg.
- El archivo `/home/labadmin/.xsession` deja de existir.
- `gdm3` queda `enabled` y `active`.
- `labadmin` puede iniciar sesión de nuevo en GNOME on Xorg y/o GNOME Wayland.

**Verificación:**

Desde la TTY o SSH:

```bash
test ! -e /home/labadmin/.xsession && echo "Configuración de fallo eliminada"
systemctl is-enabled gdm3
systemctl is-active gdm3
```

Desde una terminal abierta tras recuperar el escritorio:

```bash
echo "$XDG_SESSION_TYPE"
loginctl session-status
```

---

### Paso 11: Confirmar la continuidad de Docker Compose tras la recuperación

**Objetivo:** Demostrar que el incidente de GDM y la recuperación de la sesión gráfica no interrumpieron los servicios Docker ni la administración remota.

**Instrucciones:**

1. Desde la sesión gráfica recuperada o desde SSH, compruebe el estado de Docker:

   ```bash
   systemctl is-active docker
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
   ```

2. Revise ambos proyectos Compose:

   ```bash
   cd /opt/linux-essentials/infra

   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev ps
   docker compose -p le-prod -f compose.yaml --env-file .env.prod --profile prod ps
   ```

3. Valide los endpoints locales:

   ```bash
   curl --fail --silent --show-error http://127.0.0.1:8081/health
   curl --fail --silent --show-error http://127.0.0.1:8082/health
   ```

4. Valide nuevamente el endpoint remoto de `srv-lab`:

   ```bash
   curl --fail --silent --show-error http://192.168.56.20:8080/health
   ```

5. Compruebe que SSH sigue disponible desde `srv-lab` hacia `desk-lab`:

   ```bash
   ssh labadmin@192.168.56.30 '
     systemctl is-active ssh docker gdm3
     curl --fail --silent --show-error http://127.0.0.1:8081/health
     curl --fail --silent --show-error http://127.0.0.1:8082/health
   '
   ```

6. Registre el estado final:

   ```bash
   {
     echo "===== LAB10: ESTADO FINAL $(date --iso-8601=seconds) ====="
     systemctl is-active ssh docker gdm3
     docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
     curl --silent --show-error --fail http://127.0.0.1:8081/health
     curl --silent --show-error --fail http://127.0.0.1:8082/health
     curl --silent --show-error --fail http://192.168.56.20:8080/health
   } | sudo tee -a /var/log/linux-essentials/lab10-evidencias.log
   ```

**Salida esperada:**

- `ssh`, `docker` y `gdm3` están activos.
- Los proyectos `le-dev` y `le-prod` continúan en ejecución.
- Los puertos `8081` y `8082` continúan respondiendo localmente.
- El endpoint `192.168.56.20:8080/health` continúa disponible.
- La recuperación gráfica no modificó los archivos YAML ni interrumpió la operación de los contenedores.

**Verificación:**

```bash
systemctl is-active ssh docker gdm3
curl --fail --silent --show-error http://127.0.0.1:8081/health
curl --fail --silent --show-error http://127.0.0.1:8082/health
curl --fail --silent --show-error http://192.168.56.20:8080/health
```

## Validación y Pruebas

Complete la siguiente lista de validación final:

- [ ] `desk-lab` tiene configurada la dirección `192.168.56.30/24` en `enp0s8`.
- [ ] `desk-lab` alcanza `srv-lab` mediante `ping` y `curl` hacia `http://192.168.56.20:8080/health`.
- [ ] El repositorio `/opt/linux-essentials/infra` fue copiado desde `srv-lab` usando SSH/SCP.
- [ ] Los manifiestos YAML de Compose no fueron modificados durante el laboratorio.
- [ ] Solamente se modificaron los archivos de entorno autorizados para configurar `8081` y `8082`.
- [ ] El perfil `dev` responde en `127.0.0.1:8081`.
- [ ] El perfil `prod` responde en `127.0.0.1:8082`.
- [ ] Se registraron evidencias de una sesión GNOME Wayland y una sesión GNOME on Xorg.
- [ ] Se identificó el estado de GDM mediante `systemctl status gdm3` y `journalctl -u gdm3`.
- [ ] El fallo controlado se limitó al archivo de usuario `/home/labadmin/.xsession`.
- [ ] La recuperación se realizó desde TTY o SSH sin reiniciar la VM.
- [ ] `gdm3` quedó habilitado y activo después de la recuperación.
- [ ] Docker Compose continuó disponible durante y después del incidente.
- [ ] El archivo de evidencias existe y contiene información del laboratorio.

Ejecute esta prueba consolidada:

```bash
echo "=== Servicios base ==="
systemctl is-active ssh docker gdm3

echo "=== Puertos locales ==="
ss -ltn | grep -E ':8081|:8082'

echo "=== Salud de perfiles locales ==="
curl --fail --silent --show-error http://127.0.0.1:8081/health
echo
curl --fail --silent --show-error http://127.0.0.1:8082/health
echo

echo "=== Salud remota srv-lab ==="
curl --fail --silent --show-error http://192.168.56.20:8080/health
echo

echo "=== Estado de GDM ==="
systemctl is-enabled gdm3
systemctl is-active gdm3

echo "=== Evidencias ==="
sudo ls -lh /var/log/linux-essentials/lab10-evidencias.log
```

El runbook de continuidad resultante debe contener, como mínimo, esta secuencia operativa:

1. Confirmar si la indisponibilidad afecta únicamente a la sesión gráfica o también a SSH, Docker y red.
2. Mantener una sesión SSH administrativa activa hacia `desk-lab` cuando sea posible.
3. Usar `Ctrl+Alt+F3` para acceder a una TTY si GDM o GNOME no permiten iniciar sesión.
4. Revisar `systemctl status gdm3`, `journalctl -b -u gdm3` y los registros del usuario afectado.
5. Revisar primero configuraciones reversibles del usuario antes de modificar componentes globales.
6. Revertir la configuración identificada y reiniciar únicamente `gdm3` si resulta necesario.
7. Comprobar la continuidad de Docker mediante `docker compose ... ps` y las rutas `/health`.
8. Conservar evidencias en `/var/log/linux-essentials/lab10-evidencias.log`.

## Solución de Problemas

### Problema 1: El perfil Docker Compose no publica 8081 o 8082, o `docker compose config` muestra variables vacías

**Síntomas:**

- `docker compose ... up -d` crea contenedores, pero `curl http://127.0.0.1:8081/health` falla.
- `docker compose config` muestra puertos inesperados, una cadena vacía o valores como `:8081`.
- `docker ps` no muestra publicaciones en `127.0.0.1:8081` o `127.0.0.1:8082`.

**Causa probable:**

Se utilizó un archivo `.env` distinto al esperado, el nombre de una variable no coincide con el declarado en `compose.yaml`, o se levantó un perfil sin la opción `--env-file` correspondiente.

**Corrección:**

1. Identifique las variables exactas utilizadas por el manifiesto:

   ```bash
   cd /opt/linux-essentials/infra
   grep -n '\${' compose.yaml
   ```

2. Revise el archivo de entorno autorizado sin revelar secretos innecesarios:

   ```bash
   grep -nE 'BIND|HOST|PORT|PROFILE' .env.dev .env.prod
   ```

3. Renderice cada perfil antes de reiniciarlo:

   ```bash
   docker compose -f compose.yaml --env-file .env.dev --profile dev config
   docker compose -f compose.yaml --env-file .env.prod --profile prod config
   ```

4. Detenga y levante nuevamente solo el proyecto afectado:

   ```bash
   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev down
   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev up -d
   ```

5. Confirme las publicaciones:

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Ports}}'
   ss -ltn | grep -E ':8081|:8082'
   ```

### Problema 2: GDM está activo, pero GNOME on Xorg vuelve inmediatamente a la pantalla de inicio de sesión

**Síntomas:**

- `systemctl is-active gdm3` muestra `active`.
- La autenticación en GDM parece correcta, pero la pantalla vuelve a mostrar el formulario de inicio de sesión.
- Wayland puede funcionar mientras Xorg falla, o ambas sesiones pueden verse afectadas por una configuración de usuario.

**Causa probable:**

Permanece una configuración de usuario inválida, como `/home/labadmin/.xsession`, permisos incorrectos en el directorio personal o un archivo de sesión residual creado durante la simulación.

**Corrección:**

1. Acceda mediante `Ctrl+Alt+F3` o SSH.

2. Compruebe los registros y la configuración del usuario:

   ```bash
   sudo journalctl -b -u gdm3 --no-pager -n 120
   sudo journalctl -b -t lab10-xsession --no-pager
   ls -la /home/labadmin/.xsession
   ```

3. Elimine el archivo de fallo controlado y corrija la propiedad del directorio personal:

   ```bash
   rm -f /home/labadmin/.xsession
   sudo chown labadmin:labadmin /home/labadmin
   ```

4. Reinicie GDM sin reiniciar toda la VM:

   ```bash
   sudo systemctl enable gdm3
   sudo systemctl restart gdm3
   ```

5. Vuelva a intentar iniciar sesión. Si el fallo ocurrió en Xorg, pruebe primero **GNOME on Xorg** para verificar que la recuperación fue efectiva y, después, pruebe **GNOME** para validar Wayland.

## Limpieza

1. Compruebe que no permanece el archivo de fallo controlado:

   ```bash
   test ! -e /home/labadmin/.xsession && echo "Archivo de fallo eliminado"
   ```

2. Asegure que GDM inicia automáticamente en futuros arranques:

   ```bash
   sudo systemctl enable gdm3
   sudo systemctl start gdm3
   systemctl is-enabled gdm3
   systemctl is-active gdm3
   ```

3. Conserve el repositorio y el archivo de evidencias para laboratorios posteriores:

   ```bash
   ls -ld /opt/linux-essentials/infra
   sudo ls -lh /var/log/linux-essentials/lab10-evidencias.log
   ```

4. Si el instructor requiere liberar recursos al finalizar, detenga los entornos locales sin eliminar archivos YAML, archivos de entorno ni volúmenes no especificados:

   ```bash
   cd /opt/linux-essentials/infra

   docker compose -p le-dev -f compose.yaml --env-file .env.dev --profile dev down
   docker compose -p le-prod -f compose.yaml --env-file .env.prod --profile prod down
   ```

5. Si los entornos deben permanecer disponibles para la siguiente práctica, no ejecute el paso anterior. Documente explícitamente que `le-dev` y `le-prod` permanecen activos.

6. Mantenga la instantánea `lab10-antes-fallo-grafico` hasta que el instructor confirme la validación del laboratorio. No restaure la instantánea si la recuperación se completó correctamente, ya que se perderían las evidencias y la configuración válida generada.

## Resumen

En este laboratorio se desplegaron los perfiles declarativos `dev` y `prod` de Docker Compose en `desk-lab`, usando puertos locales diferenciados y archivos de entorno independientes. Se comprobó que Wayland y Xorg son tipos de sesión distintos gestionados por GDM, identificables mediante `XDG_SESSION_TYPE`, `loginctl`, procesos gráficos y registros de `gdm3`.

La recuperación demostró que una caída de sesión gráfica no implica necesariamente una caída del sistema ni de los servicios de contenedores. El acceso mediante TTY y SSH permitió diagnosticar una configuración de usuario reversible, eliminarla, restablecer GDM y verificar la continuidad operacional de Docker Compose y del endpoint productivo de `srv-lab`.

Las evidencias principales del laboratorio se conservan en:

```text
/var/log/linux-essentials/lab10-evidencias.log
```

Los componentes operativos esenciales para el runbook son:

```text
Consola local: Ctrl+Alt+F3
Acceso remoto: ssh labadmin@192.168.56.30
Gestor gráfico: systemctl status gdm3
Registros: journalctl -b -u gdm3
Infraestructura YAML: /opt/linux-essentials/infra
Desarrollo local: http://127.0.0.1:8081/health
Producción local: http://127.0.0.1:8082/health
Servicio remoto: http://192.168.56.20:8080/health
```
