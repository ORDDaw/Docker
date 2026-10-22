# Imagen base exigida: Python 3.12 slim.
FROM python:3.12-slim

# La configuración de entorno se define en docker-compose.yml, no en la imagen.
WORKDIR /app

# 1. Dependencias ANTES de copiar el código.
#    Si solo cambia app.py o templates/, esta capa se reutiliza desde la caché.
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# 2. Usuario sin privilegios para la ejecución.
RUN groupadd --gid 10001 appuser \
    && useradd --uid 10001 --gid appuser --create-home --shell /usr/sbin/nologin appuser

# 3. Código de la aplicación, ya con el propietario correcto.
COPY --chown=appuser:appuser app.py ./
COPY --chown=appuser:appuser templates ./templates

# 4. A partir de aquí, el proceso NO corre como root.
#    UID/GID numéricos para que el usuario sea resoluble también por el host.
USER 10001:10001

# 5. Puerto que expone el contenedor (documental; se publica con -p o Compose).
EXPOSE 8000

# 6. Healthcheck contra /salud.
#    Si Redis no responde, /salud devuelve 503 y el healthcheck falla.
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/salud', timeout=2)"]

# 7. Arranque con Gunicorn, NO con el servidor de desarrollo de Flask.
# 8. CMD en formato JSON (exec form).
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "30", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
