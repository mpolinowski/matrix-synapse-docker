


# Matrix Secure Messaging

- [matrix.org](https://matrix.org)
- [@matrix.org Github](https://github.com/matrix-org/synapse/tree/master/docker)
- [element.io](https://element.io/get-started)
- [@Element Github](https://github.com/vector-im)

## Setup

Create a work directory and set the necessary permissions:

```bash
mkdir -p /opt/matrix/synapse/data
sudo chown -R 1000:1000 /opt/matrix
```

Then we need an internal docker network for the services to communicate on:

```bash
docker network create matrix
```


## Set up the Reverse Proxy

### NGINX Proxy Service


```bash
nano /opt/matrix/docker-compose.yml
```

```yml
version: "3"

services:
    proxy:
        image: "jwilder/nginx-proxy"
        container_name: "proxy"
        volumes:
            - "certs:/etc/nginx/certs"
            - "vhost:/etc/nginx/vhost.d"
            - "html:/usr/share/nginx/html"
            - "/run/docker.sock:/tmp/docker.sock:ro"
        networks: ["matrix"]
        restart: "always"
        ports:
            - "80:80"
            - "443:443"

networks:
    matrix:
        external: true

volumes:
    certs:
    vhost:
    html:
```

### Letsencrypt Proxy Companion Service


```bash
nano /opt/matrix/docker-compose.yml
```

```yml
version: "3"

services:
    proxy:

        ...

    letsencrypt:
        image: "jrcs/letsencrypt-nginx-proxy-companion"
        container_name: "letsencrypt"
        volumes:
            - "certs:/etc/nginx/certs"
            - "vhost:/etc/nginx/vhost.d"
            - "html:/usr/share/nginx/html"
            - "/run/docker.sock:/var/run/docker.sock:ro"
        environment:
            NGINX_PROXY_CONTAINER: "proxy"
        networks: ["matrix"]
        restart: "always"
        depends_on: ["proxy"]

        ...
```

Run the compose file with:

```bash
docker-compose up -d
```

```bash
curl localhost
```

```html
<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

## Set up Synapse

### Configuration


```bash
nano /opt/matrix/synapse/docker-compose.yml
```

```yml
version: "3"

services:
    synapse:
        image: "matrixdotorg/synapse:latest"
        container_name: "synapse"
        volumes:
            - "./data:/data"
        environment:
            # Replace this with your domain
            VIRTUAL_HOST: "sub.domain.com"
            VIRTUAL_PORT: 8008
            # Replace this with your domain
            LETSENCRYPT_HOST: "sub.domain.com"
            # Replace this with your domain
            SYNAPSE_SERVER_NAME: "sub.domain.com"
            SYNAPSE_REPORT_STATS: "yes"
        networks: ["matrix"]


networks:
    matrix:
        external: true
```

Generate the configuration file:

```bash
docker-compose run --rm synapse generate
```

```bash
nano /opt/matrix/synapse/data/homeserver.yaml
```

```yml
## Server ##
server_name: "sub.domain.com"
## TLS-enabled listener: for when matrix traffic is sent directly to synapse. ##
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true

    resources:
      - names: [client, federation]
        compress: false
## Registration ##
enable_registration: true 
```

Now that everything is in place, you can start synapse using a command as simple as


```bash
docker-compose up -d
```

## Adding PostgreSQL


```bash
nano /opt/matrix/synapse/docker-compose.yml
```


```yml
version: "3"

services:
    synapse:

        ...

    postgresql:
        image: postgres:latest
        restart: always
        environment:
            POSTGRES_PASSWORD: somepassword
            POSTGRES_USER: synapse
            POSTGRES_DB: synapse
            POSTGRES_INITDB_ARGS: "--encoding='UTF8' --lc-collate='C' --lc-ctype='C'"
        volumes:
            - "postgresdata:/var/lib/postgresql/"
        networks: ["matrix"]

        ...

networks:
    matrix:
        external: true

volumes:
    postgresdata:
```

### Configure Synapse

```bash
nano /opt/matrix/synapse/data/homeserver.yaml
```

Comment out the SQLite block:

```yml
## Database ##
database:
  name: sqlite3
  args:
    database: /data/homeserver.db
```

And replace it with:

```yml
## Database ##
database:
    name: psycopg2
    args:
        user: synapse
        password: somepassword
        host: postgresql
        database: synapse
        cp_min: 5
        cp_max: 10
```


```bash
docker-compose down
docker-compose up -d
```

## Connect a Client

[List of compatible Clients](https://matrix.org/clients-matrix/)