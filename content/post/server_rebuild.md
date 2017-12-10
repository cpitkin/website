---
title: "Server_rebuild"
date: 2017-10-21
tags: [docker, ops, automation]
draft: true
---

Building a Nextcloud server using conainters

```bash
docker volume create --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.103,rw \
    --opt device=:/mnt/bigdaddy/user-files \
    user-files

docker run -it -v user-files:/user-files busybox
```

```bash
docker run --name mariadb -v /run/secrets:/run/secrets -v mariadb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mariadb_root_pass --rm mariadb:latest
```

Testing NFS volume mount
```bash
docker run -i -t --volume-driver=nfs -v nfs-server/mount-point:/target busybox
```

## Getting the DB setup
docker run -d --name mariadb -v /run/secrets:/run/secrets -v mariadb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mariadb_root_pass -e MYSQL_PASSWORD_FILE=/run/secrets/nextcloud_pass -e MYSQL_DATABASE=nextcloud -e MYSQL_USER=nextcloud mariadb:latest

_NOTE_ could probably use an exec here to temp open the port to check for db and admin user creation
docker run --name mariadb -v mariadb:/var/lib/mysql -d -p 3306:3306 mariadb:latest

docker exec -it mariadb /bin/bash

Create an admin user for MariaDB
```sql
CREATE USER 'mariadb_admin'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON * . * TO 'mariadb_admin'@'%';
FLUSH PRIVILEGES;
```
Test the connection using the command line or SQL GUI tool

Create a database and user for Nextcloud
```sql
CREATE DATABASE IF NOT EXISTS nextcloud;
CREATE USER IF NOT EXISTS 'nextcloud'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON * . * TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
```

## On to Nextcloud setup

FPM setup (needs a web server)
```bash
docker run -d --link mariadb:mariadb \
  --mount volume-driver=local,src=nextcloud,target=/var/www/html \
  --mount volume-driver=local,src=apps,target=/var/www/html/custom_apps \
  --mount volume-driver=local,src=config,target=/var/www/html/config \
  --mount volume-driver=nfs,src=fontaine-futuristics/mnt/bigdaddy/user-files,target=/var/www/html/data \
  --mount volume-driver=local,src=theme,target=/var/www/html/themes \
  nextcloud
```

Backed in Apache
```bash
docker run -d -v nextcloud:/var/www/html   -v apps:/var/www/html/custom_apps -v config:/var/www/html/config -v user-files:/var/www/html/data -v theme:/var/www/html/themes --name nextcloud --label-file /traefik/nextcloud_labels --network web nextcloud:12-apache
```

## Traefik
```toml
debug = true
checkNewVersion = true
logLevel = "ERROR"
defaultEntryPoints = ["http", "https"]

[entryPoints]
    [entryPoints.http]
    address = ":80"
    compress = true
      [entryPoints.http.redirect]
      entryPoint = "https"
    [entryPoints.https]
      address = ":443"
      [entryPoints.https.tls]
      minVersion = "VersionTLS1"
        [[entryPoints.https.tls.certificates]]
        CertFile = "/ssl/files.ariealview.com/fullchain.pem"
        KeyFile = "/ssl/files.ariealview.com/privkey.pem"

[retry]

[acme]
email = "charlie@protonmail.ch"
storage = "acme.json"
entryPoint = "https"
OnHostRule = true
# caServer = "https://acme-staging.api.letsencrypt.org/directory"

[web]
address = ":9090"
readOnly = true
  [web.auth.basic]
  users = ["traefik:$2y$05$6lTdWraM3oc1l.tUw403JuFBJyhN2L6foAWH.dI5zVnJKAt5NlJYm"]

[docker]
endpoint = "unix:///var/run/docker.sock"
# domain = "ariealview.com"
watch = true
exposedbydefault = false
swarmmode = false
```

```
traefik.frontend.passHostHeader=true
traefik.backend=nextcloud
traefik.docker.network=web
traefik.frontend.rule=Host:files.ariealview.com
traefik.enable=true
traefik.port=8080
```

docker run -d --restart always -v /var/run/docker.sock:/var/run/docker.sock -v /traefik/traefik.toml:/traefik.toml -v /traefik/acme.json:/acme.json -v /traefik/ssl:/ssl -p 80:80 -p 443:443 -p 9090:9090 --network web --name traefik traefik

docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock -v /traefik/traefik.toml:/traefik.toml -v /traefik/acme.json:/acme.json -v /traefik/ssl:/ssl -p 80:80 -p 443:443 -p 9090:9090 --network web --name traefik traefik

```nginx
GNU nano 2.5.3                                  File: files.conf

server {
  listen 443 ssl http2;
  server_name files.ariealview.com;
ssl_certificate /etc/letsencrypt/live/files.ariealview.com/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/files.ariealview.com/privkey.pem; # managed by Certbot
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-$
  ssl_prefer_server_ciphers on;
  ssl_dhparam /etc/wok/dhparams.pem;
  ssl_session_timeout 10m;

  add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";
#    add_header X-Frame-Options DENY;
#    add_header X-Content-Type-Options nosniff;


  location / {
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass http://192.168.1.111;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      #proxy_redirect http://192.168.1.111 https://files.airealview.com/;
  }

}


server {
  listen 80 default;
  server_name files.ariealview.com;
  location / {
      return 301 https://$host$request_uri;
  }

  # Redirect non-https traffic to https
  # if ($scheme != "https") {
  #     return 301 https://$host$request_uri;
  # } # managed by Certbot

}
```
