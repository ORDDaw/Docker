# Tablón de notas: Flask + Redis + Docker

Aplicación de tablón de notas con Flask y Redis, ejecutada con Gunicorn dentro de Docker y publicada a través de Nginx.

El stack tiene tres servicios:

- **`web`**: Flask + Gunicorn, escucha en el puerto `8000` dentro de la red de Compose.
- **`proxy`**: Nginx 1.27, publica el puerto `8080` y reenvía las peticiones a `web:8000`.
- **`cache`**: Redis 7.4 con persistencia AOF y un volumen nombrado.

La aplicación no tiene autenticación: cualquier persona que pueda acceder a los puertos publicados puede leer y añadir notas. Nginx solo hace de reverse proxy; no añade autenticación ni TLS.

## Índice

- [Requisitos](#requisitos)
- [Variables de entorno](#variables-de-entorno)
- [Puesta en marcha](#puesta-en-marcha)
- [Versión multietapa opcional](#versión-multietapa-opcional)
- [Pruebas de comprobación](#pruebas-de-comprobación)
- [Dónde quedan los datos](#dónde-quedan-los-datos)
- [Cómo ver los registros](#cómo-ver-los-registros)
- [Cómo parar y limpiar](#cómo-parar-y-limpiar)
- [Solución de problemas](#solución-de-problemas)
- [Nota sobre la imagen antigua](#nota-sobre-la-imagen-antigua)
- [Registro de prompts](REGISTRO_PROMPTS.md)

## Requisitos

- **Docker Engine 24 o superior** y el plugin **Docker Compose**.
  - Probado con Docker `29.8.0` y Compose `v5.5.1`.
  - Comprueba las versiones con:

    ```bash
    docker --version
    docker compose version
    ```

- Usuario con permiso para usar Docker. Si aparece `permission denied` al conectar con `/var/run/docker.sock`, añade el usuario al grupo `docker` y vuelve a iniciar sesión:

  ```bash
  sudo usermod -aG docker "$USER"
  ```

- Puertos **8000** y **8080** libres en el host. Puedes comprobarlo con:

  ```bash
  ss -ltn | grep -E ':(8000|8080)' || true
  ```

- Espacio libre suficiente para las imágenes y el volumen. La imagen web ocupa unos **204 MB**, Nginx unos **75 MB** y Redis unos **58 MB**; se recomiendan al menos **2 GB** libres para trabajar con margen.

- Conexión a Internet la primera vez, para descargar las imágenes base y las dependencias.

- Opcional: `curl` para ejecutar las pruebas desde terminal y un navegador para usar la interfaz.

No hace falta instalar Python ni Redis en el host: ambos se ejecutan dentro de contenedores. Las dependencias Python están fijadas en `requirements.txt` (`Flask 3.0.3`, `redis 5.0.4` y `gunicorn 22.0.0`) y se instalan al construir la imagen.

Los nombres de contenedor que aparecen en los ejemplos (`notas-web-1`, `notas-proxy-1`, `notas-cache-1`) corresponden al proyecto Compose llamado `notas`. Si usas otro nombre de proyecto, consulta los nombres reales con `docker compose ps`.

## Variables de entorno

Todas las variables de la aplicación se definen en `docker-compose.yml`, dentro de `services.web.environment`. **No se declaran en el Dockerfile ni se hornean en la imagen.**

| Variable | Valor actual | Uso |
|---|---|---|
| `REDIS_URL` | `redis://cache:6379/0` | Conexión de Flask con Redis. `cache` es el nombre del servicio Compose. Sin esta variable, la app usa `redis://localhost:6379` por defecto, que no sirve dentro del contenedor. |
| `TITULO_APP` | `Tablón de notas` | Título mostrado en la interfaz. Si no se define, la app usa el mismo valor por defecto. |
| `PYTHONDONTWRITEBYTECODE` | `1` | Evita que Python escriba archivos `.pyc` en el contenedor. |
| `PYTHONUNBUFFERED` | `1` | Hace que la salida de Python no se almacene en búfer, para ver los logs con menos retraso. |

No hay ningún archivo `.env` en este proyecto. Si necesitas cambiar un valor, edita la sección `environment` de `docker-compose.yml` y recrea el servicio:

```yaml
services:
  web:
    environment:
      REDIS_URL: "redis://cache:6379/0"
      TITULO_APP: "Mi tablón"
      PYTHONDONTWRITEBYTECODE: "1"
      PYTHONUNBUFFERED: "1"
```

```bash
docker compose up -d
```

No pongas contraseñas ni claves en el Dockerfile. Si en el futuro se necesita un secreto real, utiliza Docker secrets o un gestor de secretos, no una instrucción `ENV`.

> La imagen antigua `notas-malo` contiene `CLAVE_ADMIN=admin1234`, pero la aplicación **no lee esa variable** y no forma parte de este stack. Si conservas esa imagen, elimínala o rota el valor; nunca la publiques en un registro.

## Puesta en marcha

Desde la carpeta `notas` (ajusta la ruta si has clonado el proyecto en otro sitio):

```bash
cd ~/practicaDocker/notas
```

Valida la configuración de Compose:

```bash
docker compose config --quiet
```

Construye la imagen y arranca los servicios en segundo plano:

```bash
docker compose up -d --build
```

Comprueba el estado:

```bash
docker compose ps
```

Espera a que los tres servicios aparezcan como `healthy`. El servicio `cache` tiene un healthcheck con `redis-cli ping`; `web` espera a que Redis esté sano y `proxy` espera a que `web` esté sano, gracias a `depends_on: condition: service_healthy`.

Prueba rápida a través del proxy:

```bash
curl -fsS http://localhost:8080/salud
```

Respuesta esperada (el número de notas es `0` en una instalación nueva; variará según los datos que ya existan):

```json
{"estado":"ok","notas":0}
```

Abre la interfaz en:

```text
http://localhost:8080
```

El puerto `8000` sigue publicado para comparar directamente con Gunicorn. Nginx escucha en `8080` y reenvía a `web:8000`; la configuración está en `nginx/default.conf`.

### Aviso de exposición

El Compose actual publica los dos puertos:

```yaml
web:
  ports:
    - "8000:8000"
proxy:
  ports:
    - "8080:80"
```

Eso los expone en **todas las interfaces** del host. Como la aplicación no tiene autenticación, para uso local es preferible limitar ambos a loopback:

```yaml
web:
  ports:
    - "127.0.0.1:8000:8000"
proxy:
  ports:
    - "127.0.0.1:8080:80"
```

Si no necesitas el acceso directo a Gunicorn, puedes eliminar el puerto de `web` y dejar solo el proxy. Nginx tampoco añade autenticación, TLS ni limitación de peticiones: para acceso remoto, añade esas capas y un firewall; no publiques este tablón directamente en Internet.

## Versión multietapa opcional

El stack usa por defecto `Dockerfile` (una sola etapa). También está `Dockerfile.multistage`, que instala las dependencias en una etapa `builder` y copia a la imagen final solo el árbol instalado y el código de la aplicación:

```bash
docker build -t notas-web:single .
docker build -f Dockerfile.multistage -t notas-web:multistage .
docker image ls | grep notas-web
```

| Imagen | DISK USAGE | CONTENT SIZE | Tamaño exacto |
|---|---:|---:|---:|
| `notas-web:single` | 204 MB | 49,5 MB | 203.576.414 bytes |
| `notas-web:multistage` | 191 MB | 46,1 MB | 190.555.335 bytes |

La variante multietapa ahorra unos **13 MB** (6,4 %) porque no copia la capa de instalación de `pip` ni `requirements.txt` a la imagen final. El ahorro no es mayor porque las dependencias son wheels y no hay un toolchain de compilación que eliminar.

Para que Compose use esta variante, cambia el `dockerfile` del servicio `web`:

```yaml
web:
  build:
    context: .
    dockerfile: Dockerfile.multistage
```

Después, reconstruye:

```bash
docker compose up -d --build web proxy
```

## Pruebas de comprobación

| Prueba | Comando | Resultado esperado |
|---|---|---|
| Sintaxis de Compose | `docker compose config --quiet` | Termina sin errores |
| Servicios levantados | `docker compose ps` | `web`, `proxy` y `cache` en estado `healthy` |
| Salud directa a Gunicorn | `curl -fsS http://localhost:8000/salud` | JSON con `"estado":"ok"` |
| Salud a través del proxy | `curl -fsS http://localhost:8080/salud` | JSON con `"estado":"ok"` |
| Portada a través del proxy | `curl -fsS -o /dev/null -w '%{http_code}\n' http://localhost:8080/` | `200` |
| Servidor directo | `curl -sI http://localhost:8000/salud \| grep -i '^Server:'` | `Server: gunicorn` |
| Servidor del proxy | `curl -sI http://localhost:8080/salud \| grep -i '^Server:'` | `Server: nginx` |
| Upstream del proxy | `curl -sI http://localhost:8080/salud \| grep -i '^X-Upstream-Server:'` | `X-Upstream-Server: gunicorn` |
| Configuración de Nginx | `docker compose exec -T proxy nginx -t` | Sintaxis correcta |
| Usuario no root de web | `docker compose exec -T web id` | `uid=10001(appuser) gid=10001(appuser)` |
| Redis sin puerto publicado | `docker port notas-cache-1` | Sin salida |
| Redis responde | `docker compose exec -T cache redis-cli ping` | `PONG` |
| Número de notas | `docker compose exec -T cache redis-cli LLEN notas` | Un número entero |
| Healthcheck web | `docker inspect --format '{{.State.Health.Status}}' notas-web-1` | `healthy` |
| Healthcheck proxy | `docker inspect --format '{{.State.Health.Status}}' notas-proxy-1` | `healthy` |
| Healthcheck Redis | `docker inspect --format '{{.State.Health.Status}}' notas-cache-1` | `healthy` |
| Logs recientes de web | `docker compose logs --tail=20 web` | Peticiones y errores de Gunicorn |
| Logs recientes de proxy | `docker compose logs --tail=20 proxy` | Peticiones y errores de Nginx |

Las cabeceras que responden en cada caso se pueden ver directamente con:

```bash
curl -I http://localhost:8000/salud
curl -I http://localhost:8080/salud
```

El primer caso devuelve `Server: gunicorn`. El segundo devuelve `Server: nginx` y añade `X-Upstream-Server: gunicorn`, que permite comprobar que Nginx está reenviando a Gunicorn.

El primer healthcheck de `web` puede tardar hasta unos 30 segundos porque su intervalo es de `30s`. `proxy` espera a que `web` esté sano antes de arrancar; su propio healthcheck se ejecuta cada `10s`.

### Prueba de escritura opcional

Esta prueba añade una nota temporal a través del proxy y después la elimina:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
  -X POST \
  --data-urlencode 'texto=Prueba README' \
  http://localhost:8080/notas
```

Debe devolver `302` (la app redirige a `/`). Comprueba que se ha guardado:

```bash
curl -fsS http://localhost:8080/salud
```

Y elimina la nota de prueba:

```bash
docker compose exec -T cache redis-cli LREM notas 0 'Prueba README'
```

`LREM` devuelve `1` si encontró y eliminó la nota. No uses esta prueba si no quieres modificar los datos, aunque es reversible.

## Dónde quedan los datos

Los datos de Redis están en un **volumen nombrado de Docker**, no dentro de la imagen y no en un bind mount del proyecto.

- Nombre lógico en Compose: `redis-data`
- Nombre real habitual: `notas_redis-data`
- Punto de montaje dentro del contenedor: `/data`

Comprueba el volumen:

```bash
docker volume ls | grep notas
docker volume inspect notas_redis-data
```

Para ver la ruta real en el host:

```bash
docker volume inspect notas_redis-data --format '{{.Mountpoint}}'
```

La ruta exacta depende de la instalación de Docker (por ejemplo, en Docker instalado como snap puede estar bajo `/var/snap/docker/...`). No edites archivos del volumen con Redis en marcha.

Contenido típico:

```bash
docker compose exec -T cache ls -la /data
```

Verás el archivo `dump.rdb` y/o el directorio `appendonlydir`, porque Redis se arranca con:

```yaml
command: ["redis-server", "--appendonly", "yes"]
```

Consecuencias:

- `docker compose stop` y `docker compose down` **conservan** el volumen y las notas.
- `docker compose down -v` **elimina el volumen** y, por tanto, las notas.
- Borrar o recrear la imagen de la aplicación no borra los datos.
- El nombre real del volumen depende del nombre del proyecto Compose; si cambias la carpeta o el nombre del proyecto, puedes acabar creando otro volumen.

No hay copia de seguridad automática. Para una copia puntual puedes forzar un guardado RDB síncrono y copiarlo fuera del contenedor:

```bash
docker compose exec -T cache redis-cli SAVE
docker cp notas-cache-1:/data/dump.rdb ./dump-$(date +%F).rdb
```

Para una copia completa y consistente, detén Redis antes de copiar todo `/data`. Guarda la copia en otro equipo o ubicación y prueba la restauración; un volumen persistente no es un backup.

## Cómo ver los registros

Ver los logs de la aplicación en tiempo real:

```bash
docker compose logs -f web
```

Ver los logs de Redis:

```bash
docker compose logs -f cache
```

Ver los logs de Nginx:

```bash
docker compose logs -f proxy
```

Ver los últimos 100 líneas de todos los servicios:

```bash
docker compose logs --tail=100
```

Ver solo las últimas líneas de web o del proxy:

```bash
docker compose logs --tail=50 web
docker compose logs --tail=50 proxy
```

También puedes usar Docker directamente:

```bash
docker logs notas-web-1
docker logs -f notas-web-1
docker logs --tail=100 notas-proxy-1
docker logs --tail=100 notas-cache-1
```

Gunicorn escribe los logs de acceso y de error en la salida estándar, por lo que aparecen en `docker compose logs`. Verás una petición a `/salud` aproximadamente cada 30 segundos: es el healthcheck de `web`. Nginx registra sus propias peticiones y healthchecks en `docker compose logs proxy`.

Para ver el resultado detallado de los healthchecks:

```bash
docker inspect --format '{{json .State.Health}}' notas-web-1
docker inspect --format '{{json .State.Health}}' notas-proxy-1
docker inspect --format '{{json .State.Health}}' notas-cache-1
```

Guardar los logs antes de eliminar los contenedores:

```bash
docker compose logs --no-color > logs.txt
```

Los logs se guardan en el contenedor y se pierden al hacer `docker compose down`, porque los contenedores se eliminan. Si necesitas conservarlos, guárdalos antes.

El driver actual es `json-file` sin límite de tamaño. En una ejecución prolongada pueden crecer y ocupar disco; si es un entorno permanente, añade rotación en Compose:

```yaml
services:
  web:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## Cómo parar y limpiar

| Objetivo | Comando | Efecto |
|---|---|---|
| Detener sin borrar nada | `docker compose stop` | Para los contenedores; conserva contenedores, volumen y datos |
| Volver a arrancar | `docker compose start` | Arranca los contenedores detenidos |
| Reiniciar un servicio | `docker compose restart web` | Reinicia solo `web`; también funciona con `proxy` o `cache` |
| Eliminar contenedores y red | `docker compose down` | Elimina contenedores y red; **conserva** el volumen y las notas |
| Eliminar también los datos | `docker compose down -v` | Elimina el volumen `notas_redis-data` y las notas. **Irreversible** |
| Eliminar la imagen web | `docker image rm notas-web:local` | Borra la imagen construida; el volumen no se toca |
| Eliminar la imagen de Nginx | `docker image rm nginx:1.27-alpine` | Borra la imagen del proxy; no afecta a las notas |
| Borrar las notas sin borrar el volumen | `docker compose exec -T cache redis-cli DEL notas` | Vacía la lista de notas; el volumen sigue existiendo. **Irreversible** |
| Ver volúmenes restantes | `docker volume ls \| grep notas` | Muestra el volumen del proyecto |

Secuencia de limpieza completa del stack actual:

```bash
# 1. Guarda los logs si los necesitas
docker compose logs --no-color > logs.txt

# 2. Elimina contenedores, red y volumen de datos
docker compose down -v

# 3. Elimina la imagen construida y la de Nginx si no la usas en otro proyecto
docker image rm notas-web:local
docker image rm nginx:1.27-alpine
```

Si también construiste la imagen antigua de auditoría:

```bash
docker image rm notas-malo
```

Para comprobar que ya no queda el volumen:

```bash
docker volume ls | grep notas || true
```

Si solo quieres parar el servicio y conservar las notas, usa `docker compose stop` o `docker compose down` **sin** `-v`. No uses `docker compose down -v` si no tienes una copia de seguridad.

Evita `docker system prune` como comando rutinario: puede borrar imágenes, redes y caché de otros proyectos. Si necesitas limpiar solo este stack, usa los comandos anteriores.

## Solución de problemas

### `permission denied` al conectar con Docker

El usuario no tiene acceso al socket. Añádelo al grupo `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Después cierra la sesión y vuelve a entrar, o reinicia la sesión gráfica. Comprueba con:

```bash
docker info
```

### El puerto 8000 o 8080 ya está ocupado

Comprueba qué los usa:

```bash
ss -ltnp | grep -E ':(8000|8080)' || true
```

Puedes parar el otro servicio o cambiar el puerto del host en `docker-compose.yml`, por ejemplo:

```yaml
web:
  ports:
    - "8001:8000"
proxy:
  ports:
    - "8081:80"
```

En ese caso la aplicación directa queda en `http://localhost:8001` y el proxy en `http://localhost:8081`; los puertos internos siguen siendo `8000` y `80`.

### `proxy` devuelve `502 Bad Gateway`

Comprueba que `web` está sano y que Nginx puede resolver el nombre `web`:

```bash
docker compose ps
docker compose logs --tail=50 proxy
docker compose logs --tail=50 web
curl -i http://localhost:8000/salud
docker compose exec -T proxy nginx -t
```

Si `web` no está `healthy`, revisa Redis y la aplicación antes de tocar Nginx. Nginx solo reenvía; no arregla un fallo de Gunicorn o de Redis.

### `web` aparece como `unhealthy`

Revisa primero Redis y los logs:

```bash
docker compose ps
docker compose logs --tail=50 cache
docker compose logs --tail=50 web
curl -i http://localhost:8000/salud
```

Si `/salud` devuelve `503` con `{"estado":"sin cache"}`, Redis no está disponible. Comprueba su estado y vuelve a arrancarlo:

```bash
docker compose start cache
docker compose exec -T cache redis-cli ping
```

### La aplicación no ve las notas después de mover el proyecto

El volumen está asociado al nombre del proyecto Compose. Si cambias la carpeta del proyecto, Compose puede crear un volumen distinto. Comprueba los volúmenes existentes:

```bash
docker volume ls | grep notas
```

No copies datos entre volúmenes sin parar Redis y sin una copia previa.

## Nota sobre la imagen antigua

La carpeta `sin-revisar/` contiene material de auditoría y **no forma parte de este stack**:

- Su Dockerfile arranca Flask con `python app.py`, en modo debug y como root.
- Su Compose usa un contexto incorrecto y no construye tal cual.
- Su imagen contiene `CLAVE_ADMIN=admin1234`, aunque la aplicación no usa esa variable.
- Su Redis publica el puerto `6379` y no tiene autenticación.

No uses esa configuración para ejecutar la aplicación. Si solo quieres el stack actual, trabaja siempre desde `notas/docker-compose.yml` y elimina la imagen antigua cuando ya no la necesites.
