Aquí tienes la **guía completa y lista para ejecutar**. Sigue cada paso en orden en tu servidor Ubuntu 22.04/24.04. Ejecuta como root o anteponiendo `sudo` donde haga falta. No omitas crear directorios ni ajustar permisos.

---

# 0 Resumen rápido (decisiones)

* Dominios: `nube.grupopaico.com.pe` (Nextcloud), `documentos.grupopaico.com.pe` (OnlyOffice), `mail.grupopaico.com.pe` (Poste.io).
* Certificados TLS en `/etc/apache2/ssl/grupopaico/` (crt, key, ca-bundle).
* Apache hará TLS termination y proxy a contenedores en `127.0.0.1`.
* Memoria asignada: Nextcloud 5 GB, OnlyOffice 3 GB (tal como pediste).
* Workspace persistente: `/srv/docker`.

---

# 1 Actualizar servidor e instalar utilidades

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget htop ufw apt-transport-https ca-certificates gnupg lsb-release software-properties-common
sudo reboot   # opcional pero recomendado si kernel se actualizó
```

---

# 2 Instalar Docker y Docker Compose plugin

```bash
# agregar repo Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker

# permitir docker sin sudo (opcional)
sudo usermod -aG docker $USER
# cierra sesión y entra de nuevo si quieres usar docker sin sudo
```

Verifica:

```bash
docker --version
docker run --rm hello-world
```

---

# 3 Crear workspace persistente y directorios

```bash
sudo mkdir -p /srv/docker/nextcloud/{db,html,data,redis}
sudo mkdir -p /srv/docker/onlyoffice/{data,logs,lib,fonts,cache}
sudo mkdir -p /srv/docker/mail/data /srv/docker/mail/certs
sudo chown -R $USER:$USER /srv/docker
sudo chmod -R 750 /srv/docker
```

Si necesitas ajustar UIDs después de probar, instrucciones abajo.

---

# 4 Crear archivo `.env`

Crea `/srv/docker/.env` y protege permisos:

```bash
cat > /srv/docker/.env <<'EOF'
MYSQL_ROOT_PASSWORD=TuRootPassFuerteAquiCámbialo
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
MYSQL_PASSWORD=TuDBPassFuerteAquiCámbialo
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=TuAdminPassFuerteAquiCámbialo
ONLYOFFICE_JWT_SECRET=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | cut -c1-32)
TZ=America/Lima
EOF
sudo chmod 600 /srv/docker/.env
# muestra la clave JWT (si quieres guardarla)
grep ONLYOFFICE_JWT_SECRET /srv/docker/.env
```

Anota `ONLYOFFICE_JWT_SECRET` en lugar seguro.

---

# 5 Contenido: `docker-compose.nextcloud.yml` (Nextcloud + MariaDB + Redis + OnlyOffice)

Crea `/srv/docker/docker-compose.nextcloud.yml` con este contenido:

```yaml
version: "3.8"
services:
  db:
    image: mariadb:11
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mariadb-admin", "--protocol=tcp", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - /srv/docker/nextcloud/db:/var/lib/mysql
    mem_limit: 1500m
    cpus: 1.0
    networks:
      - backend

  redis:
    image: redis:8-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    volumes:
      - /srv/docker/nextcloud/redis:/data
    mem_limit: 512m
    cpus: 0.3
    networks:
      - backend

  nextcloud:
    image: nextcloud:32-apache
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:80"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER}
      - NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD}
      - OVERWRITEPROTOCOL=https
      - OVERWRITEHOST=nube.grupopaico.com.pe
      - OVERWRITECLIURL=https://nube.grupopaico.com.pe
      - TRUSTED_PROXIES=127.0.0.1/32
    volumes:
      - /srv/docker/nextcloud/html:/var/www/html
      - /srv/docker/nextcloud/data:/var/www/html/data
    mem_limit: 5g
    cpus: 3.0
    extra_hosts:
      - "documentos.grupopaico.com.pe:host-gateway"
      - "nube.grupopaico.com.pe:host-gateway"
      - "grupopaico.com.pe:host-gateway"
    networks:
      - backend

  onlyoffice:
    image: onlyoffice/documentserver:9.1.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:8082:80"
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=${ONLYOFFICE_JWT_SECRET}
      - TZ=${TZ}
    volumes:
      - /srv/docker/onlyoffice/data:/var/www/onlyoffice/Data
      - /srv/docker/onlyoffice/logs:/var/log/onlyoffice
      - /srv/docker/onlyoffice/lib:/var/lib/onlyoffice
      - /srv/docker/onlyoffice/fonts:/usr/share/fonts/truetype/custom
      - /srv/docker/onlyoffice/cache:/var/lib/onlyoffice/documentserver/App_Data/cache/files
    mem_limit: 3g
    cpus: 1.5
    networks:
      - backend

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

Guarda el archivo.

---

# 6 Contenido: `docker-compose.mail.yml` (Poste.io)

Crea `/srv/docker/docker-compose.mail.yml`:

```yaml
version: "3.8"
services:
  mailserver:
    image: analogic/poste.io:latest
    container_name: mailserver
    restart: unless-stopped
    hostname: mail
    domainname: grupopaico.com.pe
    environment:
      - TZ=America/Lima
      - HTTPS=OFF
      - HTTP_PORT=80
      - DISABLE_CLAMAV=0
      - DISABLE_SPAMASSASSIN=0
      - POSTMASTER_ADDRESS=postmaster@mail.grupopaico.com.pe
    ports:
      - "25:25"
      - "587:587"
      - "465:465"
      - "143:143"
      - "993:993"
      - "110:110"
      - "995:995"
      - "4190:4190"
      - "127.0.0.1:8085:80"
    volumes:
      - /srv/docker/mail/data:/data
      - /srv/docker/mail/certs:/certs:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

Nota: montamos `/srv/docker/mail/certs` para tus certificados si los necesitas dentro del contenedor. Pero tu Apache actuará como TLS terminator. Si no quieres copiar certificados a esa carpeta, no importa: Poste.io sirve HTTP local y Apache hará HTTPS.

---

# 7 Preparar certificados y enlaces (si quieres montar en Poste.io)

Si quieres que Poste.io vea los certificados (opcional):

```bash
sudo ln -s /etc/apache2/ssl/grupopaico /srv/docker/mail/certs
sudo chown -R $USER:$USER /srv/docker/mail/certs
```

No obligatorio. Apache manejará HTTPS.

---

# 8 Configurar Apache (módulos + vhosts)

Instalar y habilitar módulos:

```bash
sudo apt install -y apache2
sudo a2enmod proxy proxy_http ssl headers proxy_wstunnel rewrite
sudo systemctl enable --now apache2
```

Crea `/etc/apache2/sites-available/nube.grupopaico.com.pe.conf`:

```apache
<VirtualHost *:80>
  ServerName nube.grupopaico.com.pe
  Redirect permanent / https://nube.grupopaico.com.pe/
</VirtualHost>

<VirtualHost *:443>
  ServerName nube.grupopaico.com.pe

  SSLEngine on
  SSLCertificateFile /etc/apache2/ssl/grupopaico/grupopaico.crt
  SSLCertificateKeyFile /etc/apache2/ssl/grupopaico/grupopaico.com.pe.key
  SSLCertificateChainFile /etc/apache2/ssl/grupopaico/grupopaico.ca-bundle

  ProxyPreserveHost On
  ProxyRequests Off
  ProxyPass / http://127.0.0.1:8081/
  ProxyPassReverse / http://127.0.0.1:8081/

  RewriteEngine On
  RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
  RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]

  RequestHeader set X-Forwarded-Proto "https"
  RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
  ErrorLog ${APACHE_LOG_DIR}/nube_error.log
  CustomLog ${APACHE_LOG_DIR}/nube_access.log combined
</VirtualHost>
```

Crea `/etc/apache2/sites-available/documentos.grupopaico.com.pe.conf`:

```apache
<VirtualHost *:80>
  ServerName documentos.grupopaico.com.pe
  Redirect permanent / https://documentos.grupopaico.com.pe/
</VirtualHost>

<VirtualHost *:443>
  ServerName documentos.grupopaico.com.pe

  SSLEngine on
  SSLCertificateFile /etc/apache2/ssl/grupopaico/grupopaico.crt
  SSLCertificateKeyFile /etc/apache2/ssl/grupopaico/grupopaico.com.pe.key
  SSLCertificateChainFile /etc/apache2/ssl/grupopaico/grupopaico.ca-bundle

  ProxyPreserveHost On
  ProxyRequests Off

  ProxyPass /socket.io ws://127.0.0.1:8082/socket.io timeout=300
  ProxyPassReverse /socket.io ws://127.0.0.1:8082/socket.io

  ProxyPass / http://127.0.0.1:8082/ timeout=300
  ProxyPassReverse / http://127.0.0.1:8082/

  RequestHeader set X-Forwarded-Proto "https"
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
  ErrorLog ${APACHE_LOG_DIR}/doc_error.log
  CustomLog ${APACHE_LOG_DIR}/doc_access.log combined
</VirtualHost>
```

Crea `/etc/apache2/sites-available/mail.grupopaico.com.pe.conf`:

```apache
<VirtualHost *:80>
    ServerName mail.grupopaico.com.pe
    Redirect permanent / https://mail.grupopaico.com.pe/
</VirtualHost>

<VirtualHost *:443>
    ServerName mail.grupopaico.com.pe

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/grupopaico/grupopaico.crt
    SSLCertificateKeyFile /etc/apache2/ssl/grupopaico/grupopaico.com.pe.key
    SSLCertificateChainFile /etc/apache2/ssl/grupopaico/grupopaico.ca-bundle

    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass / http://127.0.0.1:8085/ timeout=300
    ProxyPassReverse / http://127.0.0.1:8085/

    RequestHeader set X-Forwarded-Proto "https"
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    ErrorLog ${APACHE_LOG_DIR}/email_error.log
    CustomLog ${APACHE_LOG_DIR}/email_access.log combined
</VirtualHost>
```

Habilita los sitios y recarga Apache:

```bash
sudo a2ensite nube.grupopaico.com.pe.conf documentos.grupopaico.com.pe.conf mail.grupopaico.com.pe.conf
sudo a2dissite 000-default.conf || true
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

# 9 Abrir puertos en firewall y comprobar DNS

Asegura DNS A records para `nube`, `documentos`, `mail` apuntando a tu IP pública y MX: `mail.grupopaico.com.pe`. Verifica con `dig`.

UFW:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 25/tcp
sudo ufw allow 587/tcp
sudo ufw allow 465/tcp
sudo ufw allow 143/tcp
sudo ufw allow 993/tcp
sudo ufw allow 110/tcp
sudo ufw allow 995/tcp
sudo ufw allow 4190/tcp
sudo ufw --force enable
sudo ufw status numbered
```

---

# 10 Ajustes de permisos finales (antes de levantar)

Asegura que directorios existen y permisos básicos:

```bash
sudo mkdir -p /srv/docker/nextcloud/{db,html,data,redis}
sudo mkdir -p /srv/docker/onlyoffice/{data,logs,lib,fonts,cache}
sudo mkdir -p /srv/docker/mail/data /srv/docker/mail/certs
# Propietarios recomendados iniciales (ajusta si falla)
sudo chown -R 33:33 /srv/docker/nextcloud/html /srv/docker/nextcloud/data || true
sudo chown -R 999:999 /srv/docker/nextcloud/db || true
sudo chown -R 1000:1000 /srv/docker/mail/data || true
sudo chown -R $USER:$USER /srv/docker/onlyoffice || true
```

Si estos chown fallan por no existir UID en host, lo corregiremos según logs.

---

# 11 Levantar containers

```bash
cd /srv/docker
# levantar nextcloud + onlyoffice
docker compose -f docker-compose.nextcloud.yml --env-file .env up -d

# espera healthchecks
echo "Esperando 60s para servicios iniciales..."
sleep 60

# levantar poste.io
docker compose -f docker-compose.mail.yml up -d
sleep 20

# revisar estado
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

---

# 12 Comprobaciones iniciales

```bash
# endpoints locales
curl -I http://127.0.0.1:8081 | head -n 5
curl -I http://127.0.0.1:8082 | head -n 5
curl -I http://127.0.0.1:8085 | head -n 5

# logs
docker compose -f docker-compose.nextcloud.yml logs -f nextcloud --tail 50
docker compose -f docker-compose.nextcloud.yml logs -f onlyoffice --tail 50
docker logs -f mailserver --tail 50
```

---

# 13 Post-instalación Nextcloud (wizard y OCC)

Abre en navegador `https://nube.grupopaico.com.pe`. Si instalación nueva, utiliza el wizard con usuario y DB que definiste en `.env`. Si Nextcloud requiere que instales manualmente DB fields usa la siguiente configuración en el wizard:

* Database user: `${MYSQL_USER}`
* Database password: `${MYSQL_PASSWORD}`
* Database name: `${MYSQL_DATABASE}`
* Host: `db`
* Data folder: dejar la predeterminada si montaste `/var/www/html/data`.

Después ejecutar (o si quieres ahora por CLI):

```bash
# establecer trusted domains y proxies
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ config:system:set trusted_domains 1 --value="nube.grupopaico.com.pe"
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ config:system:set trusted_domains 2 --value="documentos.grupopaico.com.pe"
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ config:system:set trusted_proxies 0 --value="127.0.0.1"
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ config:system:set trusted_proxies 1 --value="::1"
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ config:system:set overwrite.cli.url --value="https://nube.grupopaico.com.pe"
docker compose -f /srv/docker/docker-compose.nextcloud.yml restart nextcloud
```

---

# 14 Configurar OnlyOffice en Nextcloud (JWT y URL)

1. En Nextcloud: Apps → Buscar e instalar “ONLYOFFICE” (app oficial).
2. Settings → ONLYOFFICE:

   * Document Editing Service address: `https://documentos.grupopaico.com.pe`
   * Secret key: copia EXACTA de `ONLYOFFICE_JWT_SECRET` en `/srv/docker/.env`
   * Marca “Use JSON Web Tokens (JWT)” y pega la clave.
   * Si aparece opción “Disable certificate verification” marca solo si tus certificados no son válidos. No marques si tus certs son de CA pública.
3. Guardar y probar abriendo un documento.

Comando de comprobación OCC:

```bash
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec nextcloud php occ onlyoffice:documentserver --check
```

Debe devolver conexión OK.

Si hay errores `Invalid token` o `404 converter`:

* Verifica SECRET idéntico en `.env`.
* Revisa logs OnlyOffice:

```bash
docker compose -f /srv/docker/docker-compose.nextcloud.yml exec onlyoffice bash -lc "tail -n 200 /var/log/onlyoffice/documentserver/converter/out.log || true"
```

* Revisa logs Nextcloud relacionados con onlyoffice:

```bash
docker compose -f /srv/docker/docker-compose.nextcloud.yml logs nextcloud | grep -i onlyoffice -i || true
```

---

# 15 Configurar Poste.io (webmail, DKIM, SPF, MX)

1. Abre `https://mail.grupopaico.com.pe` y completa wizard inicial. Crea `admin@grupopaico.com.pe`.
2. En el panel de Poste.io → Domains → agrega `grupopaico.com.pe`.
3. DKIM: Dentro del panel genera la clave DKIM. Copia el registro TXT que te indica (algo así `default._domainkey`) y publícalo en DNS como TXT.
4. SPF en DNS:

```
Tipo: TXT
Nombre: @
Valor: v=spf1 mx ip4:TU.IP.PUBLICA ~all
```

5. DMARC (opcional pero recomendado):

```
Tipo: TXT
Nombre: _dmarc
Valor: v=DMARC1; p=quarantine; rua=mailto:postmaster@grupopaico.com.pe
```

6. PTR: solicita al proveedor de IP que apunte el reverse DNS a `mail.grupopaico.com.pe`.
7. Verifica envío/recepción: envía a Gmail y revisa encabezados `Received-SPF`, `DKIM-Signature`, `DMARC`.

---

# 16 Configurar SMTP en Nextcloud

En Nextcloud Admin → Configuración → Correo saliente:

* Mode: SMTP
* From address: `no-reply@grupopaico.com.pe` (o admin@)
* Server address: `mail.grupopaico.com.pe`
* Port: `587`
* Encryption: STARTTLS
* Authentication: Login
* Username: `no-reply@grupopaico.com.pe`
* Password: contraseña creada en Poste.io
  Guardar y enviar correo de prueba.

---

# 17 Backups mínimos y mantenimiento

Backup DB y datos (script ejemplo `/root/backup_nc.sh`):

```bash
#!/bin/bash
DATE=$(date +%F)
cd /srv/docker
docker compose -f docker-compose.nextcloud.yml exec db sh -c 'exec mysqldump -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' > /root/nextcloud_db_$DATE.sql
tar czf /root/nextcloud_data_$DATE.tar.gz /srv/docker/nextcloud
tar czf /root/onlyoffice_data_$DATE.tar.gz /srv/docker/onlyoffice
tar czf /root/mail_data_$DATE.tar.gz /srv/docker/mail
cp /srv/docker/.env /root/env_backup_$DATE
```

Añade al crontab si quieres:

---

# 18 Comandos de diagnóstico rápidos

```bash
# contenedores
docker ps

# salud servicios
docker compose -f docker-compose.nextcloud.yml ps

# logs
docker compose -f docker-compose.nextcloud.yml logs -f nextcloud --tail 100
docker compose -f docker-compose.nextcloud.yml logs -f onlyoffice --tail 100
docker logs -f mailserver --tail 100

# comprobar páginas públicas
curl -I https://nube.grupopaico.com.pe
curl -I https://documentos.grupopaico.com.pe
curl -I https://mail.grupopaico.com.pe
```

---

# 19 Problemas comunes y correcciones rápidas

* **Nextcloud 500 / permiso**: `sudo chown -R 33:33 /srv/docker/nextcloud/html /srv/docker/nextcloud/data` y reinicia nextcloud.
* **MariaDB no arranca por permisos**: `sudo chown -R 999:999 /srv/docker/nextcloud/db` y reinicia db.
* **OnlyOffice invalid token**: revisa `ONLYOFFICE_JWT_SECRET` idéntico en `.env` y en app Nextcloud. Reinicia servicios.
* **OnlyOffice WebSocket**: si falla realtime, asegúrate de las líneas `/socket.io` en vhost y que `proxy_wstunnel` está habilitado.
* **Correos a SPAM**: PTR, SPF, DKIM, DMARC deben estar presentes y concordar. Revisa encabezados de correos enviados a Gmail.

---

# 20 Lista de verificación final (antes de pasar a producción)

* [ ] DNS A para `nube`, `documentos`, `mail` apuntan a IP pública.
* [ ] MX apunta a `mail.grupopaico.com.pe`.
* [ ] PTR de IP pública apunta a `mail.grupopaico.com.pe`.
* [ ] Certificados en `/etc/apache2/ssl/grupopaico/` con los nombres exactos usados.
* [ ] `docker ps` muestra `db, redis, nextcloud, onlyoffice, mailserver` en estado UP.
* [ ] Nextcloud login funcionando.
* [ ] Abrir y editar documento en OnlyOffice desde Nextcloud.
* [ ] Envío de correos desde Nextcloud funcionando (prueba con Gmail).
* [ ] DKIM publicado y verificado.

---

Si quieres, ahora te preparo **un script único** que realiza los pasos 2→11 automáticamente (crea dirs, escribe archivos compose/.env, habilita Apache, y levanta contenedores). Te lo doy listo para revisar y ejecutar. ¿Lo quieres?
