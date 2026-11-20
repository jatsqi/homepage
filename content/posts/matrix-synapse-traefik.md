+++
date = '2026-11-20T19:18:46Z'
draft = true
title = 'Matrix Synapse Traefik'
author = 'Johannes Quast'
+++

There are a lot of guides about self-hosting your own Matrix Homeserver of your choice with Docker, such as Synapse, Tuwunel and so on.
Unfortunately, for people like me with an existing HomeLab, there seems to be no guide which explains _end-to-end_ how to get to a set up which also supports voice channels and how to expose everything to the internet so me and my friends can have fun. I mean, dont't get me wrong, all of this information is available _somewhere_, but I found this a rather tedious process to piece together all the components I need, so let's .

At the end of this guide, we will have a Docker setup

* ... that includes Synapse as our Homeserver. I mainly chose Synapse as its the current "reference implementation" and I very much prefer stability over feature richness.
* ... that supports voice channels via a separate LiveKit server.
* **(optional)** ... that uses Traefik as a reverse proxy to expose everything to the Internet, so my friends don't need a VPN to interact with the services. For the RTC part in particular that's something I've not found for Traefik in the official documentation. However, there are guides available for [Nginx, Caddy and HAProxy](https://github.com/element-hq/element-call/blob/main/docs/self_hosting.md) if you want to use a different reverse proxy.
* **(optional)** ... that has the correct port-forwarding and NAT configuration set in OpnSense so all traffic is routed correctly.

## Prerequisites

We need:

* A server with [Docker](https://docs.docker.com/engine/install/) installed.
* A domain and a public IP address.
* A bit of patience...

## Setting upo the Homeserver itself

Let's start with the easy part, as there is a lot of documentation available on how to do that.
First, we have to create a few files:

* Files needed for Synapse to function properly.
* A `docker-compose.yml`, which contains the definitions for all of our services.
* A `.env`, for passwords and such.

Do not start the containers yet!

```yaml
services:

  synapse:
    image: docker.io/matrixdotorg/synapse:latest
    restart: unless-stopped
    environment:
      - SYNAPSE_CONFIG_PATH=/data/homeserver.yaml
    volumes:
      - ./synapse-data:/data
    depends_on:
      - db
    ports:
      - 8008:8008/tcp

  db:
    container_name: synapse_db
    hostname: synapse_db
    image: docker.io/postgres:14-alpine
    environment:
      - POSTGRES_USER=synapse
      # Change the password in the .env file!
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
    volumes:
      - ./synapse-db/schemas:/var/lib/postgresql/data
```

The corresponding `.env` file, which you can also just inline, should look like this:

```
POSTGRES_PASSWORD=<my super cool password>
```

There are few things going on here, so let's discuss them:

* The compose file consists of two services at the moment: the Synapse server and its database.
* The data will be stored in two local folders, namely `synapse-db` and `synapse-data` for the database and configuration files respectively. You can also use bind mounts here, but let's keep things simple and just use local folders.
* The `.env` file contains our passwords.

Next, we can continue with the configuration for Synapse. First, we need to generate a signing key and a default configuration file for Synapse.

```shell
docker run -it --rm --volume ./synapse-data:/data -e SYNAPSE_SERVER_NAME=<yourdomain.tld> -e SYNAPSE_REPORT_STATS=yes matrixdotorg/synapse:latest generate
```

This will spin up a Synapse docker container and run the `generate` command, which generates a local `synapse-data` folder containing three files. Afterwards, the file structure should look like this:

```
synapse-data/
├─ homeserver.yaml
├─ <domain.tld>.signing.key
├─ <domain.tld>.log.config
```

We are only interested in the `homeserver.yaml`, for now, so let's open it and see what it's inside!

```yaml
server_name: "<yourdomain.tld>"
pid_file: /data/homeserver.pid

listeners:
  - port: 8008
    resources:
    - compress: false
      names:
      - client
      - federation
    tls: false
    type: http
    x_forwarded: true

database:
  name: sqlite3
  args:
    database: /data/homeserver.db

log_config: "/data/example.de.log.config"
media_store_path: /data/media_store
registration_shared_secret: "L89Cjuxe:-wPKJmK+qmev6w60-G~CkL@8K9+9xWCdmKPHwY:je"
report_stats: true
macaroon_secret_key: "y=7SF8rHk#VTA#~V#A#=DvRC=TrBxZoUfh5SP_PvN-qF_QvKDW"
form_secret: "0ZrD7cySVNIjZLDiiQ1hO#rusJph.+vjjis;sN;K9J5sXle9UJ"
signing_key_path: "/data/example.de.signing.key"
trusted_key_servers:
  - server_name: "matrix.org"
```

That's a fairly small configuration file for such a service. Indeed, there are a [lot more options](https://element-hq.github.io/synapse/latest/usage/configuration/config_documentation.html) you can configure, but let's keep things simple for now.
One thing we have to edit is the `database` section, as we are **not** using SQLLite, but a shiny PostgreSQL database we added to the compose file earlier.

```yaml
database:
  name: psycopg2
  txn_limit: 10000
  args:
    user: synapse
    # Copy this password from .env file
    password: <your password from the .env file>
    database: synapse
    host: synapse_db
    port: 5432
    cp_min: 5
    cp_max: 10
```


