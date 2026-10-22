# Registro de prompts de la práctica

Registro de las solicitudes del usuario y de las correcciones realizadas durante la práctica. No incluye instrucciones internas del sistema ni razonamiento privado.

- **Proyecto:** tablón de notas Flask + Redis + Docker
- **Ruta:** `~/practicaDocker/notas`
- **Fecha:** 2026-10-22

> Las celdas con `—` indican que no hubo una corrección técnica en ese paso. No se han inventado errores para rellenar la tabla.

| Nº | Prompt (resumido) | Qué hizo el agente | Qué corregí yo |
|---:|---|---|---|
| 1 | «Dime qué hace la aplicación, qué necesita para arrancar (puertos, variables, dependencias) y qué estrategia de contenerización propones.» | Revisó `app.py`, `requirements.txt` y `templates/`; explicó el tablón Flask + Redis, sus rutas, variables y propuso `python:3.12-slim` + Gunicorn + Redis en Compose. | — La explicación fue correcta; la usé para pedir el Dockerfile. |
| 2 | «Dame el Dockerfile con `python:3.12-slim`, dependencias antes del código, usuario sin privilegios, `EXPOSE 8000`, `HEALTHCHECK` contra `/salud`, Gunicorn y `CMD` JSON; haz también `.dockerignore`.» | Entregó el contenido en el chat, pero no creó los archivos porque seguía en modo Plan. | Le pedí que lo hiciera de verdad; no podía quedarse en el texto. |
| 3 | «Hazme otra vez lo que te he pedido, que estaba en Plan y no podías hacer nada.» | Creó `notas/Dockerfile` y `notas/.dockerignore`; intentó construir, pero el socket de Docker devolvió permiso denegado. | Le insistí en que ejecutara y creara los archivos, no solo que los explicara. |
| 4 | «Vale, pero dime qué has puesto en cada uno.» | Explicó el Dockerfile y el `.dockerignore` instrucción por instrucción; aclaró que todavía no había creado Compose. | Le pedí detalle del contenido; no bastaba con decir que los archivos estaban listos. |
| 5 | «Revisa el resultado línea por línea y corrige lo que no se cumpla.» | Pasó Hadolint, revisó los requisitos y corrigió `USER appuser` → `USER 10001:10001`, pasó el `HEALTHCHECK CMD` a formato JSON y añadió variantes de Compose al `.dockerignore`. | Le exigí una revisión real contra los requisitos; detectó y corrigió esos puntos. |
| 6 | «Dame el `docker-compose.yml` con `web` y `cache`; Redis sin puertos publicados, volumen nombrado y healthcheck; `web` debe esperar a `service_healthy`; variables en Compose, nunca en la imagen.» | Creó/propuso el Compose con `web` y `cache`, Redis con AOF y volumen, healthchecks y variables; quitó del Dockerfile las variables de entorno de la aplicación. | Le recordé que las variables no debían quedar dentro de la imagen y que Redis no debía publicar el puerto. |
| 7 | «Crea el archivo de docker-compose, que no lo has creado.» | Comprobó que existía en la raíz, pero no en `notas/`; lo dejó en `notas/docker-compose.yml`, que es la ubicación usada por el stack. | Le dije que el archivo no aparecía donde trabajaba; lo comprobé con `ls`. |
| 8 | «[Captura] Me da error de permisos, ¿cómo lo arreglo?» | Diagnosticó el acceso a `/var/run/docker.sock` y la pertenencia al grupo `docker`; propuso `usermod`, reiniciar sesión/`newgrp` y revisar el servicio. | Le mostré el error real con una captura; no era un fallo del Dockerfile. |
| 9 | «Me sigue sin funcionar; no sé si es porque instalé Docker con snap.» | Revisó grupos y socket y tuvo en cuenta el contexto de Snap; en ese momento seguía sin poder ejecutar Docker desde la sesión y buscó alternativas hasta resolver el acceso. | Le aclaré que Docker venía de Snap; con ese dato se localizó mejor el problema de permisos. |
| 10 | «Dame un informe de problemas del Dockerfile y del DockerCompose.» | Leyó los archivos, ejecutó Hadolint y `docker compose config`, y entregó un informe con problemas y recomendaciones. | No acepté el informe a ciegas; después le pedí verificarlo con comandos. |
| 11 | «Verifica tú cada punto con comandos y rellena una tabla: problema, ¿es cierto? (evidencia), gravedad y corrección. Incluye tamaño de las dos imágenes, usuario de cada contenedor, ubicación de `CLAVE_ADMIN`, servidor web de cada versión y al menos un punto falso, irrelevante o exagerado.» | Construyó `notas-malo`, midió las imágenes, inspeccionó usuarios, `CLAVE_ADMIN`, servidores web y el Compose antiguo; entregó una tabla con evidencia y señaló puntos exagerados o irrelevantes del informe. | Le obligué a demostrar cada afirmación y a incluir un punto no fiable; comprobé los comandos y resultados. |
| 12 | «Escribe `README.md` con requisitos, variables de entorno, puesta en marcha, pruebas de comprobación, dónde quedan los datos, cómo ver los registros y cómo parar y limpiar.» | Creó `notas/README.md` con todas esas secciones, verificó la puesta en marcha y una escritura temporal; después eliminó la nota de prueba. | Le pedí la documentación completa y revisé que las pruebas funcionaran. |
| 13 | «Pide al agente una versión multietapa del Dockerfile que instale las dependencias en una etapa y copie solo lo necesario a la final, y compara tamaños.» | Creó `Dockerfile.multistage` con etapas `builder` y `runtime`, construyó las dos imágenes y comparó **204 MB** frente a **191 MB**; validó `/salud`, usuario no root y healthcheck. | Le pedí una comparación real de tamaños y la validación de la imagen, no una estimación. |
| 14 | «Añade después un servicio proxy con Nginx delante de Gunicorn, comprueba con `curl -I` qué servidor responde en cada caso y añade el registro de prompts.» | Añadió `proxy` a Compose, creó `nginx/default.conf`, configuró healthcheck y `X-Upstream-Server`; comprobó que `8000` responde Gunicorn y `8080` responde Nginx con upstream Gunicorn; creó un primer `REGISTRO_PROMPTS.md`. | Le pedí comprobar las cabeceras en ambos accesos y dejar constancia; mantuve `8000` directo para comparar. |
| 15 | «Hazme el registro de prompts de toda la práctica; te paso una imagen de cómo tiene que ser la tabla.» | Reconstruyó el historial completo y lo adaptó a las columnas `Nº`, `Prompt (resumido)`, `Qué hizo el agente` y `Qué corregí yo`. | Le indiqué el formato exacto de la tabla y le pedí que incluyera toda la práctica. |

## Evidencia final del proxy

```bash
curl -I http://localhost:8000/salud
# HTTP/1.1 200 OK
# Server: gunicorn

curl -I http://localhost:8080/salud
# HTTP/1.1 200 OK
# Server: nginx
# X-Upstream-Server: gunicorn
```

## Estado final

- `notas-web-1`: `healthy`.
- `notas-proxy-1`: `healthy`.
- `notas-cache-1`: `healthy`.
- Imagen del stack actual: `notas-web:local`, 204 MB.
- Imagen multietapa de comparación: `notas-web:multistage`, 191 MB.
