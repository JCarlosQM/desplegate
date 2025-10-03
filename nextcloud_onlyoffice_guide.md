# 🚀 Guía Completa: Nextcloud + OnlyOffice con Apache, Docker y Certbot

Este documento explica paso a paso cómo desplegar **Nextcloud + OnlyOffice** en un servidor Ubuntu, usando **Docker** para los servicios y **Apache2 + Certbot** como proxy inverso con HTTPS.  
Con esta guía podrás tener tu propia nube privada accesible en:

- `https://jcarlosuser.ddns.net` → sitio principal
- `https://nube.jcarlosuser.ddns.net` → Nextcloud
- `https://docs.jcarlosuser.ddns.net` → OnlyOffice Document Server

---

## 📦 1. Preparar el servidor

Actualizar paquetes e instalar **Apache2 + PHP mínimo**:

```bash
sudo apt update
sudo apt install -y apache2 php libapache2-mod-php php-mbstring php-xml php-curl unzip
```

Instalar **Certbot con plugin Apache**:

```bash
sudo apt install -y certbot python3-certbot-apache
```

Instalar **Docker + plugin Compose oficial**:

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Habilitar Docker:

```bash
sudo systemctl enable --now docker
sudo docker version
```

---

## 📁 2. Configuración inicial de Apache

Respaldar configuraciones actuales:

```bash
sudo cp -a /etc/apache2/sites-available /root/apache-sites-available.bak.$(date +%F)
sudo cp -a /etc/apache2/sites-enabled  /root/apache-sites-enabled.bak.$(date +%F)
```

Habilitar módulos necesarios:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel headers ssl rewrite
sudo systemctl restart apache2
```

---

## 🔒 3. Emitir certificados SSL con Certbot

Detener Apache para liberar el puerto 80:

```bash
sudo systemctl stop apache2
```

Emitir certificados para los 3 dominios:

```bash
sudo certbot certonly --standalone   --agree-tos --no-eff-email   -m tu-email@ejemplo.com   -d jcarlosuser.ddns.net -d nube.jcarlosuser.ddns.net -d docs.jcarlosuser.ddns.net
```

Reiniciar Apache:

```bash
sudo systemctl start apache2
```

---

## 🌐 4. Configurar VirtualHosts en Apache

### 📌 `/etc/apache2/sites-available/jcarlosuser.ddns.net.conf`

```apache
<VirtualHost *:80>
    ServerName jcarlosuser.ddns.net
    ServerAlias www.jcarlosuser.ddns.net
    DocumentRoot /var/www/jcarlosuser.ddns.net
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
    ErrorLog ${APACHE_LOG_DIR}/jcarlos_http_error.log
    CustomLog ${APACHE_LOG_DIR}/jcarlos_http_access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName jcarlosuser.ddns.net
    ServerAlias www.jcarlosuser.ddns.net

    DocumentRoot /var/www/jcarlosuser.ddns.net
    <Directory /var/www/jcarlosuser.ddns.net>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/jcarlosuser.ddns.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/jcarlosuser.ddns.net/privkey.pem

    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "no-referrer"

    ErrorLog ${APACHE_LOG_DIR}/jcarlos_https_error.log
    CustomLog ${APACHE_LOG_DIR}/jcarlos_https_access.log combined
</VirtualHost>
```

### 📌 `/etc/apache2/sites-available/nube.jcarlosuser.ddns.net.conf`

```apache
<VirtualHost *:80>
    ServerName nube.jcarlosuser.ddns.net
    RewriteEngine On
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName nube.jcarlosuser.ddns.net

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/jcarlosuser.ddns.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/jcarlosuser.ddns.net/privkey.pem

    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl "on"
    AllowEncodedSlashes NoDecode

    ProxyPass "/" "http://127.0.0.1:8081/" retry=0
    ProxyPassReverse "/" "http://127.0.0.1:8081/"

    ErrorLog ${APACHE_LOG_DIR}/nube_error.log
    CustomLog ${APACHE_LOG_DIR}/nube_access.log combined
</VirtualHost>
```

### 📌 `/etc/apache2/sites-available/docs.jcarlosuser.ddns.net.conf`

```apache
<VirtualHost *:80>
    ServerName docs.jcarlosuser.ddns.net
    RewriteEngine On
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName docs.jcarlosuser.ddns.net

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/jcarlosuser.ddns.net/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/jcarlosuser.ddns.net/privkey.pem

    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl "on"
    LimitRequestBody 0
    AllowEncodedSlashes NoDecode

    ProxyPass "/" "http://127.0.0.1:8082/" retry=0
    ProxyPassReverse "/" "http://127.0.0.1:8082/"

    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*) ws://127.0.0.1:8082/$1 [P,L]

    <Proxy *>
        Require all granted
    </Proxy>

    ErrorLog ${APACHE_LOG_DIR}/docs_error.log
    CustomLog ${APACHE_LOG_DIR}/docs_access.log combined
</VirtualHost>
```

Activar los sitios y reiniciar Apache:

```bash
sudo a2ensite jcarlosuser.ddns.net.conf
sudo a2ensite nube.jcarlosuser.ddns.net.conf
sudo a2ensite docs.jcarlosuser.ddns.net.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

## 🐳 5. Configuración de Docker

Crear carpeta de trabajo:

```bash
sudo mkdir -p /opt/docker/nextcloud-onlyoffice
cd /opt/docker/nextcloud-onlyoffice
```

Archivo `.env`:

```bash
# DB
MYSQL_ROOT_PASSWORD=StrongRootPassHere
MYSQL_DATABASE=nextcloud
MYSQL_USER=ncuser
MYSQL_PASSWORD=StrongNcDBPass

# Nextcloud admin
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=StrongAdminPass

# JWT for OnlyOffice
ONLYOFFICE_JWT_SECRET=MiSecretoJWTMuyFuerte
```

Archivo `docker-compose.yml`:

```yaml
version: "3.8"
services:
  db:
    image: mariadb:10.5
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - nc-db:/var/lib/mysql

  redis:
    image: redis:alpine
    restart: unless-stopped

  nextcloud:
    image: nextcloud:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:80"
    depends_on:
      - db
      - redis
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - NEXTCLOUD_ADMIN_USER=${NEXTCLOUD_ADMIN_USER}
      - NEXTCLOUD_ADMIN_PASSWORD=${NEXTCLOUD_ADMIN_PASSWORD}
    volumes:
      - nc-data:/var/www/html
    extra_hosts:
      - "docs.jcarlosuser.ddns.net:host-gateway"
      - "nube.jcarlosuser.ddns.net:host-gateway"
      - "jcarlosuser.ddns.net:host-gateway"

  onlyoffice:
    image: onlyoffice/documentserver:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:8082:80"
    volumes:
      - onlyoffice-data:/var/www/onlyoffice/Data
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=${ONLYOFFICE_JWT_SECRET}

volumes:
  nc-db:
  nc-data:
  onlyoffice-data:
```

Iniciar servicios:

```bash
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

---

## ⚙️ 6. Configuración de Nextcloud

Probar conectividad desde el host:

```bash
curl -I https://docs.jcarlosuser.ddns.net/ --insecure
curl -I https://nube.jcarlosuser.ddns.net/ --insecure
```

Probar desde dentro del contenedor:

```bash
sudo docker compose exec nextcloud bash -c "getent hosts docs.jcarlosuser.ddns.net; curl -I https://docs.jcarlosuser.ddns.net/"
```

Configurar dominios confiables y proxies:

```bash
sudo docker compose exec nextcloud php occ config:system:set trusted_domains 1 --value="nube.jcarlosuser.ddns.net"
sudo docker compose exec nextcloud php occ config:system:set trusted_domains 2 --value="jcarlosuser.ddns.net"
sudo docker compose exec nextcloud php occ config:system:set overwrite.cli.url --value="https://nube.jcarlosuser.ddns.net"
sudo docker compose exec nextcloud php occ config:system:set overwriteprotocol --value="https"
sudo docker compose exec nextcloud php occ config:system:set trusted_proxies 0 --value="127.0.0.1"
sudo docker compose exec nextcloud php occ config:system:set trusted_proxies 1 --value="172.17.0.0/16"
```

Instalar la app OnlyOffice en Nextcloud:

```bash
sudo docker compose exec nextcloud php occ app:install onlyoffice
```

Configurar en la interfaz de Nextcloud:

- Servidor OnlyOffice: `https://docs.jcarlosuser.ddns.net/`
- Desactivar verificación de certificado
- JWT: `MiSecretoJWTMuyFuerte`

---

## 🔄 7. Mantenimiento

### Renovación automática de certificados

Verificar que Certbot tiene cron configurado:

```bash
systemctl list-timers | grep certbot
```

### Cron de Nextcloud

Verificar la disponibilidad los paquetes docker

```bash
sudo docker ps
```

Añadir la línea:

```cron
*/5 * * * * docker exec --user www-data nextcloud-onlyoffice-nextcloud-1 php /var/www/html/cron.php

```

---

## ✅ Checklist final

- [x] `https://jcarlosuser.ddns.net` abre tu web principal.  
- [x] `https://nube.jcarlosuser.ddns.net` carga Nextcloud sin advertencias.  
- [x] `https://docs.jcarlosuser.ddns.net` responde el Document Server.  
- [x] Archivos se abren en OnlyOffice sin descargas forzadas.  
- [x] Certificados válidos y renovación automática activa.  

🎉 ¡Listo! Tienes tu nube privada Nextcloud + OnlyOffice 100% operativa.
