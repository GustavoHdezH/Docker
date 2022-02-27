<div align="center"> 
  <img src="src/sentry.png">
</div>
<div> 
  <h1 align="center">Sentry</h1>
  <p>
    Es un software cuyo funcionamiento principal es rastear errores de software y permitira al equipo de desarrollo encontrar y corregir rapidamente errores en produccion de sus aplicaciones.
  </p>
</div>

## Requerimientos

Se requiere de **Redis** y de **PostgreSQL** para almacenar la información correspondiente en cada uno de los servicios.

# Configuración

Se debe contar con una carpeta que contenga la información para el despliegue del servidor como la siguiente.

| Modulo | Descripción |
|--------|-------------|
| sentry/ | Carpeta que contiene el proyecto |
| sentry/docker-compose.yml | Archivo compose que contiene la configuración del servidor. |
| sentry/.env | Archivo que contiene las claves de la base. |

Adicionalmente se debe de crear un archivo .env dentro de la carpeta que contiene el docker-compose.yml, que debe contener la siguiente información. De la cual el parametro de SENTRY_SECRET_KEY, se explicara como optenerla mas adelante.

```bash
SENTRY_SECRET_KEY=***********
SENTRY_POSTGRES_HOST=postgres
SENTRY_POSTGRES_PORT=5432
SENTRY_DB_NAME=sentry
SENTRY_DB_USER=sentry
SENTRY_DB_PASSWORD=**********
SENTRY_REDIS_HOST=redis
SENTRY_REDIS_PORT=6379
```

1. Para generar la secret key se debe de ejecutar la siguiente instrucción.

```bash
docker-compose run --rm sentry config generate-secret-key
```

2. Una vez optenida la SENTRY_SECRET_KEY, se ejecuta la siguiente instrucción para inicializar la base de datos.

```bash
docker-compose run --rm sentry upgrade
```

# Compose

A continuación se describe el contenido del fichero.

```bash
version: '3.7'

services:
  redis:
    container_name: sentry-redis
    network_mode: docker_sentry
    hostname: redis
    image: redis:latest
    restart: always
    ports:
        - "6379:6379"
    volumes:
        - './sentry-redis:/data'

  postgres:
    image: postgres:12
    container_name: sentry-postgres
    network_mode: docker_sentry
    hostname: postgres
    restart: always
    environment:
      POSTGRES_USER: sentry
      POSTGRES_PASSWORD: **********
      POSTGRES_DB: sentry
    ports:
        - "5432:5432"
    volumes:
     - ./sentry-postgres/:/var/lib/postgresql/data

  sentry:
    image: sentry:9.1.2
    container_name: sentry-server
    network_mode: docker_sentry
    hostname: sentry
    restart: always
    env_file:
     - .env
    depends_on:
      - redis
      - postgres
    ports:
     - 9000:9000
    volumes:
      - './sentry-server:/var/lib/sentry/files'

  cron:
    image: sentry:9.1.2
    container_name: sentry-cron
    network_mode: docker_sentry
    hostname: cron
    restart: always
    env_file:
     - .env
    depends_on:
      - redis
      - postgres
    command: "sentry run cron"
    volumes:
      - './sentry-cron:/var/lib/sentry/files'

  worker:
    image: sentry:9.1.2
    container_name: worker-sentry
    network_mode: docker_sentry
    hostname: worker
    restart: always
    env_file:
      - .env
    depends_on:
      - redis
      - postgres
    command: "sentry run worker"
    volumes:
      - './sentry-worker:/var/lib/sentry/files'

volumes:
   data:
```

# Envio de correos

Esta sección describe como realizar la configuración para el envio de correos y se debe de configurar en el contenedor que contiene el servidor, el cron y el worker.

1. Entrar en el contenedor
```bash
docker exec -it -u root container_name /bin/bash
```
2. Actualizar el contenedor
```bash
apt update
```
3. Instalar un editor de texto
```bash
apt intall nano
```
4. Configurar el archivo de credenciales
```
nano /etc/sentry/config.yml
```
5. Reiniciar el stack completo dentro del directorio de sentry
```bash
docker-compose restart
```