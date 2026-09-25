# Laboratorio 9: Automatización de Red e Infraestructura de Contenedores mediante YAML

## Metadatos

| Duration | Complexity | Bloom level |
|---|---|---|
| 123 minutos | Alta | Crear |

## Descripción General

En este laboratorio se automatiza la red persistente de `srv-lab` mediante Netplan y se crea una infraestructura reproducible de contenedores mediante Docker Compose. El servicio de telemetría recuperado en el Laboratorio 8 se encapsula en una red interna de Docker y se publica únicamente a través de un proxy Nginx en `192.168.56.20:8080`. La configuración resultante se almacena en `/opt/linux-essentials/infra` para reutilizarse posteriormente en despliegues de desarrollo y producción.

## Objetivos de Aprendizaje

- [ ] Modelar y aplicar una configuración persistente de red con Netplan en formato YAML.
- [ ] Validar una configuración de red antes de aplicarla mediante `netplan generate` y `netplan try`.
- [ ] Crear una definición Docker Compose con perfiles, redes internas, volúmenes, variables no secretas y comprobaciones de salud.
- [ ] Publicar el servicio de telemetría mediante Nginx sin exponer directamente el puerto interno `9080`.
- [ ] Documentar evidencias técnicas y procedimientos de rollback para red e infraestructura de contenedores.

## Prerrequisitos

**Conocimientos necesarios:**

- Sintaxis básica de YAML y su sensibilidad a la indentación.
- Direccionamiento IPv4, subredes, puertos TCP y resolución DNS.
- Conceptos de Docker: imagen, contenedor, red bridge, volumen y proxy inverso.
- Uso básico de `sudo`, `systemctl`, `ip`, `journalctl` y editores de texto.
- Comprensión de que los cambios de red y de políticas deben validarse antes de aplicarse.

**Acceso requerido:**

- Consola local de VirtualBox para la VM `srv-lab`. No realice la modificación inicial de Netplan desde una única sesión SSH.
- Usuario administrativo `labadmin` con privilegios `sudo`.
- VM `srv-lab` restaurada desde la instantánea `lab08-recuperado`.
- Docker Engine 27.5.1 y Docker Compose 2.32.4 ya instalados.
- Imagen local disponible: `linuxessentials/telemetry-api:1.0.0`.
- Acceso temporal a Internet por NAT en `enp0s3` para descargar la imagen `nginx:1.27.3-alpine` si aún no está disponible localmente.

> **Advertencia de seguridad:** todos los cambios se realizan exclusivamente dentro de la VM `srv-lab`. No aplique fallos, configuraciones de Netplan, cambios de montaje ni pruebas de contenedores sobre el sistema anfitrión.

## Entorno de Laboratorio

| Componente | Valor esperado |
|---|---|
| Máquina virtual | `srv-lab` |
| Sistema operativo | Ubuntu Server 24.04.2 LTS |
| Kernel | `6.8.0-55-generic` |
| Usuario administrativo | `labadmin` |
| Directorio de infraestructura | `/opt/linux-essentials/infra` |
| Interfaz NAT | `enp0s3`, DHCP |
| Interfaz Host-Only | `enp0s8`, `192.168.56.20/24` |
| Red Docker interna | `telemetry_net`, `172.28.0.0/16` |
| Volumen Docker | `telemetry_data` |
| Servicio interno | `telemetry-api:9080` |
| Publicación Nginx | `192.168.56.20:8080` |
| Imagen de telemetría | `linuxessentials/telemetry-api:1.0.0` |
| Imagen proxy | `nginx:1.27.3-alpine` |

Ejecute las siguientes comprobaciones iniciales desde la consola de `srv-lab`:

```bash
hostnamectl --static
id
ip -br address
docker --version
docker compose version
docker image inspect linuxessentials/telemetry-api:1.0.0 >/dev/null && echo "Imagen de telemetría disponible"
sudo systemctl is-active docker
sudo aa-status
sysctl net.ipv4.ip_forward
```

La salida esperada incluye el hostname `srv-lab`, las interfaces `enp0s3` y `enp0s8`, Docker activo y la confirmación de que la imagen local de telemetría existe. No modifique `net.ipv4.ip_forward`: Docker puede gestionar reglas de red necesarias para sus redes bridge.

## Instrucciones Paso a Paso

### Paso 1: Documentar el estado inicial y preparar el repositorio local

**Objetivo:** registrar el estado actual de la red y preparar una estructura persistente para los archivos YAML, evidencias y procedimientos de rollback.

**Instrucciones:**

1. Cree la estructura de directorios del laboratorio con permisos controlados:

   ```bash
   sudo install -d -o labadmin -g labadmin -m 0750 /opt/linux-essentials/infra
   sudo install -d -o labadmin -g labadmin -m 0750 /opt/linux-essentials/infra/netplan
   sudo install -d -o labadmin -g labadmin -m 0750 /opt/linux-essentials/infra/nginx
   sudo install -d -o labadmin -g labadmin -m 0750 /var/log/linux-essentials
   ```

2. Registre la configuración actual de red, rutas y archivos Netplan:

   ```bash
   {
     echo "=== Fecha UTC ==="
     date -u
     echo
     echo "=== Interfaces ==="
     ip -br address
     echo
     echo "=== Rutas ==="
     ip route
     echo
     echo "=== Estado Netplan ==="
     sudo netplan get
     echo
     echo "=== Archivos /etc/netplan ==="
     sudo ls -la /etc/netplan
   } | tee /var/log/linux-essentials/lab09-red-inicial.txt
   ```

3. Cree una copia de seguridad de todos los archivos YAML activos de Netplan:

   ```bash
   sudo tar -C /etc/netplan -czf /opt/linux-essentials/infra/netplan/netplan-before-lab09.tar.gz .
   sudo chown labadmin:labadmin /opt/linux-essentials/infra/netplan/netplan-before-lab09.tar.gz
   ```

4. Inicialice un repositorio Git local para mantener trazabilidad de la infraestructura:

   ```bash
   cd /opt/linux-essentials/infra
   git init
   git config user.name "labadmin"
   git config user.email "labadmin@srv-lab.local"
   ```

5. Cree un archivo `.gitignore` para excluir archivos temporales y evidencias operativas que no deban versionarse:

   ```bash
   cat > /opt/linux-essentials/infra/.gitignore <<'EOF'
   *.log
   *.tar.gz
   .env.local
   EOF
   ```

**Salida esperada:**

- Existe el directorio `/opt/linux-essentials/infra`.
- Se genera el archivo `/var/log/linux-essentials/lab09-red-inicial.txt`.
- Se crea una copia `netplan-before-lab09.tar.gz`.
- El repositorio Git muestra la rama inicial sin commits.

**Verificación:**

```bash
ls -ld /opt/linux-essentials/infra /var/log/linux-essentials
ls -l /opt/linux-essentials/infra/netplan/netplan-before-lab09.tar.gz
cd /opt/linux-essentials/infra && git status
```

---

### Paso 2: Crear y validar la configuración persistente de Netplan

**Objetivo:** reemplazar la configuración manual o previa de red por un archivo YAML persistente que conserve DHCP en NAT y la IP estática Host-Only.

**Instrucciones:**

1. Cree una copia versionable de la configuración Netplan que se aplicará:

   ```bash
   cat > /opt/linux-essentials/infra/netplan/50-linux-essentials.yaml <<'EOF'
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
   ```

2. Revise visualmente el contenido y la indentación:

   ```bash
   cat /opt/linux-essentials/infra/netplan/50-linux-essentials.yaml
   ```

3. Copie el archivo al directorio activo de Netplan con permisos restrictivos:

   ```bash
   sudo find /etc/netplan -maxdepth 1 -type f -name '*.yaml' -delete
   sudo install -o root -g root -m 0600 \
     /opt/linux-essentials/infra/netplan/50-linux-essentials.yaml \
     /etc/netplan/50-linux-essentials.yaml
   ```

4. Genere la configuración sin aplicarla todavía:

   ```bash
   sudo netplan generate
   ```

5. Pruebe el cambio desde la consola local de VirtualBox. Mantenga abierta esta consola hasta confirmar la conectividad:

   ```bash
   sudo netplan try --timeout 120
   ```

6. Cuando Netplan solicite confirmación y la conectividad siga disponible, confirme con `ENTER`.

7. Aplique explícitamente la configuración persistente:

   ```bash
   sudo netplan apply
   ```

8. Registre el resultado final de interfaces y rutas:

   ```bash
   {
     echo "=== Estado posterior a Netplan ==="
     date -u
     ip -br address
     ip route
     getent hosts archive.ubuntu.com || true
   } | tee /var/log/linux-essentials/lab09-red-final.txt
   ```

**Salida esperada:**

- `netplan generate` termina sin errores de sintaxis.
- `netplan try` permite confirmar el cambio.
- `enp0s3` recibe una dirección IPv4 DHCP de la red NAT.
- `enp0s8` conserva `192.168.56.20/24`.
- Existe una ruta predeterminada a través de `enp0s3`.

**Verificación:**

```bash
ip -br address show enp0s3
ip -br address show enp0s8
ip route
ping -c 2 1.1.1.1
getent hosts registry-1.docker.io
sudo netplan get
```

La dirección `192.168.56.20/24` debe aparecer en `enp0s8`. No configure una puerta de enlace en la interfaz Host-Only salvo instrucción explícita del instructor.

---

### Paso 3: Crear los archivos de entorno y la configuración de Nginx

**Objetivo:** definir variables no secretas para los perfiles `dev` y `prod`, y crear un proxy Nginx que enrute `/health` hacia el servicio interno.

**Instrucciones:**

1. Cree el archivo de variables para desarrollo:

   ```bash
   cat > /opt/linux-essentials/infra/.env.dev <<'EOF'
   DEPLOY_ENV=dev
   TELEMETRY_ENV=development
   LOG_LEVEL=debug
   EOF
   ```

2. Cree el archivo de variables para producción:

   ```bash
   cat > /opt/linux-essentials/infra/.env.prod <<'EOF'
   DEPLOY_ENV=prod
   TELEMETRY_ENV=production
   LOG_LEVEL=info
   EOF
   ```

3. Asigne permisos que permitan lectura al usuario administrativo, pero eviten acceso global:

   ```bash
   chmod 0640 /opt/linux-essentials/infra/.env.dev /opt/linux-essentials/infra/.env.prod
   ```

4. Cree la configuración de Nginx. El bloque `/health` se comunica con el nombre DNS interno `telemetry-api`, proporcionado por la red Docker:

   ```bash
   cat > /opt/linux-essentials/infra/nginx/default.conf <<'EOF'
   server {
       listen 80;
       server_name _;

       access_log /var/log/nginx/access.log;
       error_log /var/log/nginx/error.log warn;

       location = /health {
           proxy_pass http://telemetry-api:9080/health;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       location / {
           return 404;
       }
   }
   EOF
   ```

5. Revise los archivos creados:

   ```bash
   sed -n '1,120p' /opt/linux-essentials/infra/.env.dev
   sed -n '1,120p' /opt/linux-essentials/infra/.env.prod
   sed -n '1,160p' /opt/linux-essentials/infra/nginx/default.conf
   ```

**Salida esperada:**

- Se crean `.env.dev` y `.env.prod` sin contraseñas, tokens ni credenciales.
- Nginx solo atiende la ruta `/health`.
- Las demás rutas devuelven HTTP `404`.
- La configuración usa `telemetry-api:9080` como destino interno, sin IP fija del contenedor.

**Verificación:**

```bash
find /opt/linux-essentials/infra -maxdepth 2 -type f -printf '%M %p\n'
grep -n "telemetry-api:9080" /opt/linux-essentials/infra/nginx/default.conf
```

---

### Paso 4: Crear la definición Docker Compose reproducible

**Objetivo:** crear una composición YAML con perfiles, red interna, volumen nombrado, comprobaciones de salud, políticas de reinicio y nombres fijos de contenedor.

**Instrucciones:**

1. Cree el archivo `/opt/linux-essentials/infra/compose.yaml`:

   ```bash
   cat > /opt/linux-essentials/infra/compose.yaml <<'EOF'
   # Sintaxis basada en Compose Specification.
   name: linux-essentials

   services:
     telemetry-api:
       image: linuxessentials/telemetry-api:1.0.0
       container_name: telemetry-api
       profiles:
         - dev
         - prod
       env_file:
         - .env.${DEPLOY_ENV}
       environment:
         TELEMETRY_ENV: ${TELEMETRY_ENV}
         LOG_LEVEL: ${LOG_LEVEL}
         TELEMETRY_DATA_DIR: /var/lib/linux-essentials/telemetry
       expose:
         - "9080"
       volumes:
         - telemetry_data:/var/lib/linux-essentials/telemetry
       networks:
         - telemetry_net
       restart: unless-stopped
       healthcheck:
         test:
           - CMD-SHELL
           - python3 -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:9080/health', timeout=3)"
         interval: 15s
         timeout: 5s
         retries: 5
         start_period: 15s

     nginx:
       image: nginx:1.27.3-alpine
       container_name: nginx-telemetry-proxy
       profiles:
         - dev
         - prod
       depends_on:
         telemetry-api:
           condition: service_healthy
       ports:
         - "192.168.56.20:8080:80"
       volumes:
         - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
       networks:
         - telemetry_net
       restart: unless-stopped
       healthcheck:
         test:
           - CMD-SHELL
           - wget -q -O /dev/null http://127.0.0.1/health
         interval: 15s
         timeout: 5s
         retries: 5
         start_period: 10s

   networks:
     telemetry_net:
       name: telemetry_net
       internal: true
       ipam:
         config:
           - subnet: 172.28.0.0/16

   volumes:
     telemetry_data:
       name: telemetry_data
   EOF
   ```

2. Compruebe que la imagen local de telemetría sigue disponible:

   ```bash
   docker image inspect linuxessentials/telemetry-api:1.0.0 \
     --format 'Imagen={{.RepoTags}}'
   ```

3. Descargue solamente la imagen de Nginx si no está disponible localmente:

   ```bash
   docker image inspect nginx:1.27.3-alpine >/dev/null 2>&1 || \
     docker pull nginx:1.27.3-alpine
   ```

4. Valide la composición para el perfil de desarrollo. El comando resuelve variables, valida la estructura YAML y muestra la configuración final:

   ```bash
   cd /opt/linux-essentials/infra
   docker compose --env-file .env.dev --profile dev config
   ```

5. Ejecute una validación silenciosa, útil para automatización:

   ```bash
   docker compose --env-file .env.dev --profile dev config -q
   ```

**Salida esperada:**

- La validación de Compose finaliza sin errores.
- La salida muestra la red `telemetry_net` como `internal: true`.
- El servicio `telemetry-api` no posee una sección `ports`.
- El servicio `nginx` publica `192.168.56.20:8080:80`.
- Los dos servicios contienen `restart: unless-stopped`.

**Verificación:**

```bash
cd /opt/linux-essentials/infra
docker compose --env-file .env.dev --profile dev config | grep -E 'telemetry_net|telemetry_data|9080|8080|unless-stopped'
grep -nE 'ports:|expose:|internal:|restart:' compose.yaml
```

---

### Paso 5: Desplegar y validar el perfil de desarrollo

**Objetivo:** iniciar el perfil `dev`, comprobar los contenedores, validar los healthchecks y confirmar que la API no queda expuesta directamente al host.

**Instrucciones:**

1. Despliegue la composición usando el archivo de entorno de desarrollo:

   ```bash
   cd /opt/linux-essentials/infra
   docker compose --env-file .env.dev --profile dev up -d
   ```

2. Espere unos segundos para que se completen las comprobaciones de salud:

   ```bash
   sleep 20
   ```

3. Liste el estado de los servicios:

   ```bash
   docker compose --env-file .env.dev --profile dev ps
   ```

4. Consulte el estado de healthcheck de cada contenedor:

   ```bash
   docker inspect -f '{{.Name}}: {{.State.Health.Status}}' telemetry-api
   docker inspect -f '{{.Name}}: {{.State.Health.Status}}' nginx-telemetry-proxy
   ```

5. Compruebe los puertos publicados en el host:

   ```bash
   docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
   sudo ss -lntp | grep -E ':8080|:9080' || true
   ```

6. Realice una solicitud HTTP al proxy desde `srv-lab`:

   ```bash
   curl -i --max-time 10 http://192.168.56.20:8080/health
   ```

7. Confirme que una ruta no definida no publica contenido adicional:

   ```bash
   curl -i --max-time 10 http://192.168.56.20:8080/
   ```

**Salida esperada:**

- Los contenedores `telemetry-api` y `nginx-telemetry-proxy` aparecen en ejecución.
- Ambos healthchecks muestran estado `healthy`.
- La solicitud a `/health` responde HTTP `200` si la API recuperada implementa correctamente ese endpoint.
- La solicitud a `/` responde HTTP `404`.
- El host escucha en `192.168.56.20:8080`, pero no en `0.0.0.0:9080`.

**Verificación:**

```bash
docker inspect -f '{{.State.Health.Status}}' telemetry-api
docker inspect -f '{{.State.Health.Status}}' nginx-telemetry-proxy
curl -fsS http://192.168.56.20:8080/health
sudo ss -lntp | grep ':9080' || echo "Correcto: el puerto 9080 no está publicado en el host"
```

---

### Paso 6: Verificar DNS interno, red, volumen y evidencias operativas

**Objetivo:** comprobar el aislamiento de red, la resolución DNS interna de Docker, la existencia del volumen persistente y registrar procedimientos de recuperación.

**Instrucciones:**

1. Verifique que Nginx puede resolver el nombre interno del servicio:

   ```bash
   docker exec nginx-telemetry-proxy getent hosts telemetry-api
   ```

2. Compruebe que Nginx puede acceder internamente al endpoint de telemetría:

   ```bash
   docker exec nginx-telemetry-proxy \
     wget -q -O - http://telemetry-api:9080/health
   ```

3. Inspeccione la red Docker:

   ```bash
   docker network inspect telemetry_net
   ```

4. Inspeccione el volumen nombrado:

   ```bash
   docker volume inspect telemetry_data
   ```

5. Obtenga un resumen compacto de la topología:

   ```bash
   docker network inspect telemetry_net \
     --format '{{range .Containers}}{{.Name}} -> {{.IPv4Address}}{{println}}{{end}}'
   ```

6. Registre evidencias técnicas del despliegue:

   ```bash
   {
     echo "=== Fecha UTC ==="
     date -u
     echo
     echo "=== Docker Compose PS ==="
     docker compose --env-file .env.dev --profile dev ps
     echo
     echo "=== Red telemetry_net ==="
     docker network inspect telemetry_net
     echo
     echo "=== Volumen telemetry_data ==="
     docker volume inspect telemetry_data
     echo
     echo "=== Puertos publicados ==="
     docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
   } | tee /var/log/linux-essentials/lab09-contenedores-dev.txt
   ```

7. Cree el documento de rollback dentro del repositorio:

   ```bash
   cat > /opt/linux-essentials/infra/rollback.md <<'EOF'
   ### Rollback de Docker Compose

   Detener y eliminar contenedores y red del perfil desplegado:

   ```bash
   cd /opt/linux-essentials/infra
   docker compose --env-file .env.dev --profile dev down
   ```

   El volumen `telemetry_data` se conserva al ejecutar `down`. Eliminarlo solo después de verificar que no contiene datos necesarios:

   ```bash
   docker volume rm telemetry_data
   ```

   ### Rollback de Netplan

   Ejecutar exclusivamente desde consola local de VirtualBox:

   ```bash
   sudo rm -f /etc/netplan/*.yaml
   sudo tar -xzf /opt/linux-essentials/infra/netplan/netplan-before-lab09.tar.gz -C /etc/netplan
   sudo netplan generate
   sudo netplan try --timeout 120
   sudo netplan apply
   ```
   EOF
   ```

8. Versione los archivos de infraestructura:

   ```bash
   cd /opt/linux-essentials/infra
   git add compose.yaml nginx/default.conf netplan/50-linux-essentials.yaml \
     .env.dev .env.prod .gitignore rollback.md
   git commit -m "lab09: automatizar red y telemetria con Compose"
   ```

**Salida esperada:**

- `telemetry-api` se resuelve desde Nginx mediante DNS interno de Docker.
- La red muestra una subred `172.28.0.0/16`.
- El volumen `telemetry_data` existe y está asociado al motor Docker.
- Se crea evidencia en `/var/log/linux-essentials/lab09-contenedores-dev.txt`.
- El repositorio Git contiene un commit del laboratorio.

**Verificación:**

```bash
docker exec nginx-telemetry-proxy getent hosts telemetry-api
docker network inspect telemetry_net --format '{{.Internal}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
docker volume ls | grep telemetry_data
test -s /var/log/linux-essentials/lab09-contenedores-dev.txt && echo "Evidencia registrada"
cd /opt/linux-essentials/infra && git log --oneline -1
```

## Validación y Pruebas

Complete la siguiente lista antes de considerar finalizado el laboratorio:

| Prueba | Comando | Resultado esperado |
|---|---|---|
| Hostname correcto | `hostnamectl --static` | `srv-lab` |
| Interfaz NAT por DHCP | `ip -br address show enp0s3` | Dirección IPv4 asignada por NAT |
| Interfaz Host-Only estática | `ip -br address show enp0s8` | `192.168.56.20/24` |
| Validación Netplan | `sudo netplan generate` | Sin errores |
| Configuración Compose válida | `docker compose --env-file .env.dev --profile dev config -q` | Sin salida y código 0 |
| Contenedor API saludable | `docker inspect -f '{{.State.Health.Status}}' telemetry-api` | `healthy` |
| Contenedor Nginx saludable | `docker inspect -f '{{.State.Health.Status}}' nginx-telemetry-proxy` | `healthy` |
| DNS interno Docker | `docker exec nginx-telemetry-proxy getent hosts telemetry-api` | Dirección IP de `172.28.0.0/16` |
| Endpoint publicado | `curl -fsS http://192.168.56.20:8080/health` | Respuesta HTTP satisfactoria |
| Puerto API no expuesto | `sudo ss -lntp \| grep ':9080'` | Sin listener publicado en el host |
| Red aislada | `docker network inspect telemetry_net --format '{{.Internal}}'` | `true` |
| Evidencia disponible | `ls -l /var/log/linux-essentials/lab09-*.txt` | Archivos de evidencia presentes |
| Cambios versionados | `git -C /opt/linux-essentials/infra status` | Árbol de trabajo limpio |

## Solución de Problemas

### Incidencia 1: `netplan try` pierde conectividad o no puede confirmar la configuración

**Síntomas:**

- Se pierde la conexión SSH después de iniciar `netplan try`.
- `enp0s8` no muestra `192.168.56.20/24`.
- `netplan generate` informa un error de sintaxis o de indentación YAML.

**Causa probable:**

La interfaz fue escrita con un nombre incorrecto, existe una indentación inválida, hay archivos YAML conflictivos en `/etc/netplan`, o se aplicó el cambio desde una única sesión remota.

**Corrección:**

1. Acceda a la consola local de VirtualBox.
2. Revise los archivos activos:

   ```bash
   sudo ls -la /etc/netplan
   sudo cat /etc/netplan/50-linux-essentials.yaml
   ```

3. Restaure la copia de seguridad y pruebe de nuevo:

   ```bash
   sudo rm -f /etc/netplan/*.yaml
   sudo tar -xzf /opt/linux-essentials/infra/netplan/netplan-before-lab09.tar.gz -C /etc/netplan
   sudo netplan generate
   sudo netplan try --timeout 120
   ```

4. Corrija el archivo en `/opt/linux-essentials/infra/netplan/50-linux-essentials.yaml`, cópielo nuevamente a `/etc/netplan/` y repita la validación.

### Incidencia 2: Nginx inicia, pero `/health` responde `502 Bad Gateway` o el contenedor queda `unhealthy`

**Síntomas:**

- `curl http://192.168.56.20:8080/health` devuelve HTTP `502`.
- `docker compose ps` muestra `unhealthy`.
- El nombre `telemetry-api` no se resuelve dentro del contenedor Nginx.

**Causa probable:**

El servicio de telemetría no completó su healthcheck, el endpoint `/health` no está disponible en el puerto interno `9080`, la imagen local no corresponde a la versión esperada o Nginx se inició antes de que la API estuviera lista.

**Corrección:**

1. Revise los logs de ambos servicios:

   ```bash
   cd /opt/linux-essentials/infra
   docker compose --env-file .env.dev --profile dev logs --tail=100 telemetry-api nginx
   ```

2. Verifique el estado detallado de la API:

   ```bash
   docker inspect telemetry-api --format '{{json .State.Health}}'
   docker exec telemetry-api python3 -c \
     "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:9080/health').read().decode())"
   ```

3. Compruebe la resolución DNS y conectividad interna:

   ```bash
   docker exec nginx-telemetry-proxy getent hosts telemetry-api
   docker exec nginx-telemetry-proxy wget -q -O - http://telemetry-api:9080/health
   ```

4. Si la API quedó detenida, reinicie el despliegue sin eliminar el volumen:

   ```bash
   cd /opt/linux-essentials/infra
   docker compose --env-file .env.dev --profile dev down
   docker compose --env-file .env.dev --profile dev up -d
   ```

## Limpieza

La configuración creada será reutilizada en el laboratorio posterior; por tanto, no elimine `/opt/linux-essentials/infra`, el commit Git ni la copia de seguridad de Netplan.

Para detener temporalmente el entorno de desarrollo y conservar el volumen de datos:

```bash
cd /opt/linux-essentials/infra
docker compose --env-file .env.dev --profile dev down
```

Verifique que los contenedores fueron eliminados y que el volumen persiste:

```bash
docker ps -a --filter name=telemetry-api --filter name=nginx-telemetry-proxy
docker volume inspect telemetry_data
```

Solo si el instructor solicita una limpieza completa de datos de telemetría, elimine el volumen explícitamente:

```bash
docker volume rm telemetry_data
```

No elimine el archivo `/etc/netplan/50-linux-essentials.yaml` al finalizar, ya que representa la configuración persistente validada de red de `srv-lab`.

## Resumen

En este laboratorio se creó una configuración reproducible de red e infraestructura basada en YAML. Netplan mantiene `enp0s3` con DHCP para acceso NAT y `enp0s8` con la dirección administrativa estática `192.168.56.20/24`. La infraestructura Docker Compose define una red interna aislada, un volumen nombrado persistente, perfiles `dev` y `prod`, variables no secretas y healthchecks.

El servicio `telemetry-api` permanece accesible únicamente dentro de la red Docker mediante `telemetry-api:9080`. Nginx actúa como único punto de publicación en `192.168.56.20:8080`, aplicando el principio de mínima exposición. Los archivos de infraestructura, las evidencias y los procedimientos de rollback quedan disponibles para su reutilización en el despliegue multi-entorno del siguiente laboratorio.
