# 📘 Manual Exhaustivo: Migración y Despliegue de Nextcloud + OnlyOffice para GRUPO PAICO (Octubre 2025)

## Introducción

Este manual adaptado proporciona una guía **completa, precisa y secuencial** para migrar el sitio web de GRUPO PAICO de cPanel a tu servidor Ubuntu 24.04 LTS en DMZ (IP interna: 10.10.10.2), integrar **Nextcloud** con **OnlyOffice Document Server** usando **Docker**, y configurar **Apache2** como proxy inverso. Incorpora la obtención e instalación de un **certificado SSL Wildcard de Namecheap** para cubrir `*.grupopaico.com.pe` (válido para subdominios como `nube.grupopaico.com.pe` y `docs.grupopaico.com.pe`). 

### Objetivos clave:
- **Migración segura**: Transferir el sitio actual de https://grupopaico.com.pe (de cPanel) al nuevo servidor con 1TB SSD y 16-32 GB RAM, manteniendo downtime mínimo.
- **Accesibilidad vía DNS**: Servicios accesibles externamente a través de:
  - `https://grupopaico.com.pe` → Sitio web principal (migrado de cPanel).
  - `https://nube.grupopaico.com.pe` → Nextcloud para almacenamiento colaborativo (15 usuarios).
  - `https://docs.grupopaico.com.pe` → OnlyOffice para edición de documentos.
- **Integración con pfSense**: Forwarding ya configurado (puertos 80/443 de IP pública 190.117.188.175 a 10.10.10.2). Verificaremos y optimizaremos.
- **Eficiencia para MYPE**: Recursos Docker escalados para 16-32 GB RAM (asignamos ~70% para evitar OOM), healthchecks y redes aisladas.
- **Seguridad empresarial**: Wildcard SSL (DV, 1 año), 2FA, E2EE, backups y hardening OWASP 2025.
- **Buenas prácticas**: Zero-downtime migration, validación DNS para wildcard, monitoreo y automatizaciones.

**Tiempo estimado**: 4-5 horas (incluyendo compra SSL y migración) + 1 hora pruebas.
**Requisitos previos**:
- **Hardware**: Servidor Ubuntu en DMZ: 1TB SSD, 16-32 GB RAM, 4+ vCPU. IP estática 10.10.10.2.
- **Red**: pfSense IP WAN 190.117.188.175 con forwarding 80/443 → 10.10.10.2. DNS A records para subdominios apuntando a 190.117.188.175 (usa Namecheap DNS o proveedor actual).
- **Acceso**: SSH root a 10.10.10.2 (desde pfSense LAN/DMZ). Acceso cPanel actual para backup. Cuenta Namecheap para SSL.
- **Herramientas**: `rsync` para migración, `pwgen` para passwords.
- **Advertencia**: **Backup completo** de cPanel y pfSense antes. Prueba migración en staging. Sustituye placeholders (e.g., email admin).

**Notas generales**:
- Comandos en **servidor Ubuntu** como root (`sudo -i`).
- Ubicaciones: Sitio en `/var/www/grupopaico.com.pe`, Docker en `/opt/docker/nextcloud-onlyoffice`.
- Verificaciones: Incluidas (e.g., `systemctl status`).
- Logs: `journalctl -u apache2 -f` o `docker compose logs -f`.
- Versiones: Nextcloud 32, OnlyOffice 9.0.4.1, MariaDB 11, Redis 8 (2025).

---

## 🛡️ 0. Verificación y Optimización de pfSense (Ya Configurado)

**Por qué**: Asegura forwarding seguro y DNS para wildcard (*.grupopaico.com.pe). Buenas prácticas: Limita sources a IPs de oficina.

Accede pfSense GUI: https://IP_pfSense_interna (e.g., 10.10.10.1).

1. **Verificar Port Forwarding**:
   - **Firewall > NAT > Port Forward**: Confirma reglas para TCP 80/443 (WAN → 10.10.10.2:80/443). Si no: Add como antes (Source: Any o Alias oficina, Destination: WAN Address).
   - **Apply Changes**.

2. **Verificar Firewall Rules**:
   - **Firewall > Rules > WAN**: Pass TCP 80/443 a 10.10.10.2. Limita: Source → Alias (IPs oficina, e.g., 190.117.188.0/24 si estática).
   - **Firewall > Rules > OPT1 (DMZ)**: Pass outbound desde 10.10.10.2 a internet (para Docker pulls/SSL validación).
   - **Apply Changes**.

3. **Configurar DNS para Subdominios**:
   - En Namecheap (o proveedor DNS): Agrega A records:
     - `nube.grupopaico.com.pe` → 190.117.188.175
     - `docs.grupopaico.com.pe` → 190.117.188.175
     - Wildcard: `*.grupopaico.com.pe` → 190.117.188.175 (opcional, pero cubre subs futuros).
   - En pfSense **Services > DNS Resolver**: Custom Options: `host-override="grupopaico.com.pe" 10.10.10.2` (para acceso interno sin hairpin).
   - Prueba: `nslookup nube.grupopaico.com.pe` (desde LAN/DMZ, debe resolver 10.10.10.2 internamente).

4. **Monitoreo**: **Status > System Logs > Firewall** – Busca denegados post-setup.

**Tiempo**: 10 min. Si issue: Verifica NAT Reflection en **System > Advanced > Firewall & NAT**.

---

## 🔒 1. Obtención del Certificado SSL Wildcard en Namecheap

**Por qué**: Wildcard cubre `*.grupopaico.com.pe` (e.g., subs ilimitados). DV rápido (horas), precio ~$89/año (Sectigo PositiveSSL Wildcard, validez 1 año, 256-bit encriptación).

### Paso a Paso: Compra y Activación (Hazlo mañana)
1. **Login/Registro en Namecheap**:
   - Ve a https://www.namecheap.com/.
   - Login con cuenta (o crea). Verifica dominio `grupopaico.com.pe` en "Domain List".

2. **Comprar Certificado**:
   - Navega: **Security > SSL Certificates > PositiveSSL Wildcard** (DV, $88.88/año en 2025; o Sectigo EV para $299 si necesitas validación empresa).
   - Selecciona: 1 dominio + subs (*.grupopaico.com.pe), validez 1 año.
   - Cart > Checkout (pago tarjeta/PayPal). Recibes email confirmación.

3. **Generar CSR (Certificate Signing Request) en Servidor Ubuntu**:
   - SSH a 10.10.10.2: `ssh root@10.10.10.2`.
   - Instala OpenSSL si no: `sudo apt install openssl`.
   - Genera private key y CSR:
     ```bash
     sudo mkdir -p /etc/ssl/private
     sudo openssl req -new -newkey rsa:2048 -nodes -keyout /etc/ssl/private/grupopaico.com.pe.key -out /root/grupopaico.com.pe.csr
     ```
     - **Explicación**: `-nodes` sin passphrase (para auto-restart Apache). Responde prompts: Common Name: `grupopaico.com.pe`, Organization: "GRUPO PAICO", etc. (resto opcional).
   - Verifica: `cat /root/grupopaico.com.pe.csr` (copia el contenido BEGIN...END).

4. **Activar Certificado en Namecheap**:
   - En email o Dashboard > **SSL Certificates > Activate**.
   - Pega CSR en campo.
   - Validación DV: Elige **DNS Validation** (recomendado para wildcard; evita email issues).
     - Namecheap genera TXT record (e.g., _acme-challenge.grupopaico.com.pe → value).
     - Agrega TXT en DNS proveedor (espera 5-15 min propagación).
   - Submit. Procesa en 15-30 min (email "Issued").

5. **Descargar Certificado**:
   - Dashboard > SSL > Download: ZIP con `grupopaico_com_pe.crt` (cert), `grupopaico_com_pe.ca-bundle` (chain), private key (ya tienes).
   - Transfiere a servidor: SCP ZIP a `/root/ssl-namecheap.zip`, unzip: `unzip /root/ssl-namecheap.zip -d /etc/ssl/certs/`.
   - Renombra: `sudo mv /etc/ssl/certs/grupopaico_com_pe.crt /etc/ssl/certs/grupopaico.com.pe.crt`
   - `sudo mv /etc/ssl/certs/grupopaico_com_pe.ca-bundle /etc/ssl/certs/grupopaico.com.pe.ca-bundle`
   - Protege: `sudo chmod 600 /etc/ssl/private/grupopaico.com.pe.key`

**Common Issues**: Propagación DNS lenta – usa `dig TXT _acme-challenge.grupopaico.com.pe`. No passphrase en key para prod.
**Best Practices**: Renueva 30 días antes (manual, ya que no auto como Let's Encrypt). Costo: $89/año. Tiempo: 30-60 min.

---

## 📦 2. Preparar el Servidor Ubuntu y Migrar Sitio de cPanel

SSH: `ssh root@10.10.10.2`.

1. **Actualizar Sistema**:
   ```bash
   sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
   sudo reboot  # Si kernel update
   ```

2. **Instalar Dependencias (Apache, PHP, etc.)**:
   ```bash
   sudo apt install -y apache2 php8.3 libapache2-mod-php8.3 php8.3-mbstring php8.3-xml php8.3-curl php8.3-gd php8.3-zip php8.3-intl php8.3-imagick php8.3-bcmath php8.3-gmp unzip pwgen rsync openssl
   sudo systemctl enable --now apache2
   ```

3. **Migrar Sitio de cPanel**:
   - En cPanel actual: **Backup Wizard > Full Backup** (descarga ZIP a local).
   - O usa rsync (si SSH en cPanel): `rsync -avz -e ssh user@oldserver:/home/user/public_html/ /tmp/cpanel-backup/`
   - En Ubuntu:
     ```bash
     sudo mkdir -p /var/www/grupopaico.com.pe
     sudo rsync -avz /tmp/cpanel-backup/ /var/www/grupopaico.com.pe/  # Ajusta path
     sudo chown -R www-data:www-data /var/www/grupopaico.com.pe
     sudo chmod -R 755 /var/www/grupopaico.com.pe
     ```
   - Migra DB si aplica (e.g., MySQL dump de cPanel → nuevo DB local): `mysqldump -u user -p olddb > /root/olddb.sql`, luego importa.
   - Prueba local: `curl http://10.10.10.2` (debe servir index.html de cPanel).

4. **Instalar Docker**:
   ```bash
   sudo apt install -y ca-certificates curl gnupg lsb-release
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   sudo systemctl enable --now docker
   sudo usermod -aG docker root
   ```
   - Config `/etc/docker/daemon.json`:
     ```json
     {
       "storage-driver": "overlay2",
       "exec-opts": ["native.cgroupdriver=systemd"]
     }
     ```
   - `sudo systemctl restart docker`.

5. **Firewall Local**:
   ```bash
   sudo apt install -y ufw
   sudo ufw allow OpenSSH
   sudo ufw allow from 10.10.10.1 to any port 80,443 proto tcp  # pfSense DMZ
   sudo ufw --force enable
   ```

**Verificación**: `php -v`, `docker version`, `ls /var/www/grupopaico.com.pe`. Tiempo: 30 min.

---

## 📁 3. Configuración Inicial de Apache

1. **Respaldar**:
   ```bash
   sudo cp -a /etc/apache2/sites-available /root/apache-backup-available-$(date +%F)
   sudo cp -a /etc/apache2/sites-enabled /root/apache-backup-enabled-$(date +%F)
   ```

2. **Habilitar Módulos**:
   ```bash
   sudo a2enmod proxy proxy_http proxy_wstunnel headers ssl rewrite proxy_balancer lbmethod_byrequests deflate expires
   sudo systemctl restart apache2
   ```

**Tiempo**: 5 min.

---

## 🔒 4. Instalar Certificado SSL Wildcard de Namecheap en Apache

**Por qué**: Integra el wildcard para HTTPS en todos VirtualHosts.

1. **Instalar Certs (post-descarga)**:
   - Ya hecho en paso 1. Verifica: `ls /etc/ssl/certs/grupopaico.com.pe.*` y `/etc/ssl/private/grupopaico.com.pe.key`.

2. **Configurar Apache para SSL**:
   - En cada VirtualHost (paso 5), usa:
     ```
     SSLEngine on
     SSLCertificateFile /etc/ssl/certs/grupopaico.com.pe.crt
     SSLCertificateKeyFile /etc/ssl/private/grupopaico.com.pe.key
     SSLCertificateChainFile /etc/ssl/certs/grupopaico.com.pe.ca-bundle
     ```

3. **Prueba Instalación**:
   - Después de VirtualHosts: `sudo apache2ctl configtest` (Syntax OK).
   - Reinicia: `sudo systemctl restart apache2`.
   - Verifica: `openssl s_client -connect 10.10.10.2:443 -servername grupopaico.com.pe` (debe mostrar cert wildcard).

**Renewal**: Manual anual; script reminder cron: `0 0 1 1 * echo "Renovar SSL" | mail -s "SSL Renewal" admin@grupopaico.com.pe`. Tiempo: 10 min.

---

## 🌐 5. Configurar VirtualHosts en Apache (Adaptados a GRUPO PAICO)

1. **Crear Directorio Principal** (ya migrado).

### 5.1 `/etc/apache2/sites-available/grupopaico.com.pe.conf`
```bash
sudo nano /etc/apache2/sites-available/grupopaico.com.pe.conf
```
Contenido:
```
<VirtualHost *:80>
    ServerName grupopaico.com.pe
    ServerAlias www.grupopaico.com.pe
    DocumentRoot /var/www/grupopaico.com.pe
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
    ErrorLog ${APACHE_LOG_DIR}/paico_http_error.log
    CustomLog ${APACHE_LOG_DIR}/paico_http_access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName grupopaico.com.pe
    ServerAlias www.grupopaico.com.pe
    DocumentRoot /var/www/grupopaico.com.pe
    <Directory /var/www/grupopaico.com.pe>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/grupopaico.com.pe.crt
    SSLCertificateKeyFile /etc/ssl/private/grupopaico.com.pe.key
    SSLCertificateChainFile /etc/ssl/certs/grupopaico.com.pe.ca-bundle
    # Headers OWASP 2025
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;"
    ErrorLog ${APACHE_LOG_DIR}/paico_https_error.log
    CustomLog ${APACHE_LOG_DIR}/paico_https_access.log combined
</VirtualHost>
```

### 5.2 `/etc/apache2/sites-available/nube.grupopaico.com.pe.conf`
```bash
sudo nano /etc/apache2/sites-available/nube.grupopaico.com.pe.conf
```
Contenido (similar, con ProxyPass a 8081, headers ajustados):
```
<VirtualHost *:80>
    ServerName nube.grupopaico.com.pe
    RewriteEngine On
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName nube.grupopaico.com.pe
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/grupopaico.com.pe.crt
    SSLCertificateKeyFile /etc/ssl/private/grupopaico.com.pe.key
    SSLCertificateChainFile /etc/ssl/certs/grupopaico.com.pe.ca-bundle
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl "on"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    AllowEncodedSlashes NoDecode
    ProxyPass "/" "http://127.0.0.1:8081/" retry=0
    ProxyPassReverse "/" "http://127.0.0.1:8081/"
    # Headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
    Header always set Content-Security-Policy "default-src 'self' https://nube.grupopaico.com.pe https://docs.grupopaico.com.pe; script-src 'self' 'unsafe-inline'; connect-src 'self' ws://127.0.0.1:8081"
    ErrorLog ${APACHE_LOG_DIR}/nube_error.log
    CustomLog ${APACHE_LOG_DIR}/nube_access.log combined
</VirtualHost>
```

### 5.3 `/etc/apache2/sites-available/docs.grupopaico.com.pe.conf`
```bash
sudo nano /etc/apache2/sites-available/docs.grupopaico.com.pe.conf
```
Contenido (ProxyPass a 8082, WebSocket):
```
<VirtualHost *:80>
    ServerName docs.grupopaico.com.pe
    RewriteEngine On
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName docs.grupopaico.com.pe
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/grupopaico.com.pe.crt
    SSLCertificateKeyFile /etc/ssl/private/grupopaico.com.pe.key
    SSLCertificateChainFile /etc/ssl/certs/grupopaico.com.pe.ca-bundle
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl "on"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
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
    # Headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
    Header always set Content-Security-Policy "default-src 'self' https://docs.grupopaico.com.pe; script-src 'self' 'unsafe-inline'; connect-src 'self' wss://docs.grupopaico.com.pe"
    ErrorLog ${APACHE_LOG_DIR}/docs_error.log
    CustomLog ${APACHE_LOG_DIR}/docs_access.log combined
</VirtualHost>
```

4. **Activar**:
   ```bash
   sudo a2ensite grupopaico.com.pe.conf nube.grupopaico.com.pe.conf docs.grupopaico.com.pe.conf
   sudo a2dissite 000-default.conf
   sudo apache2ctl configtest
   sudo systemctl reload apache2
   ```

**Verificación**: `curl -I https://grupopaico.com.pe` (cert wildcard, headers). Tiempo: 20 min.

---

## 🐳 6. Configuración de Docker

1. **Directorio**:
   ```bash
   sudo mkdir -p /opt/docker/nextcloud-onlyoffice
   cd /opt/docker/nextcloud-onlyoffice
   ```

2. **.env**:
   ```bash
   sudo nano .env
   ```
   ```
   MYSQL_ROOT_PASSWORD=$(pwgen -s 32 1)
   MYSQL_DATABASE=nextcloud_paico
   MYSQL_USER=paico_user
   MYSQL_PASSWORD=$(pwgen -s 32 1)
   NEXTCLOUD_ADMIN_USER=admin_paico
   NEXTCLOUD_ADMIN_PASSWORD=$(pwgen -s 32 1)
   ONLYOFFICE_JWT_SECRET=$(pwgen -s 32 1)
   ```
   - Anota passwords.

3. **docker-compose.yml** (escalado para 16-32GB RAM: ~12GB total asignado):
   ```bash
   sudo nano docker-compose.yml
   ```
   Contenido (similar anterior, con dominios paico, recursos: db 2G, redis 1G, nextcloud 8G, onlyoffice 4G):
   ```
   version: "3.8"
   services:
     db:
       image: mariadb:11
       restart: unless-stopped
       healthcheck:
         test: ["CMD", "mariadb-admin", "--protocol=tcp", "ping"]
         interval: 10s
         timeout: 5s
         retries: 3
       environment:
         - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
         - MYSQL_DATABASE=${MYSQL_DATABASE}
         - MYSQL_USER=${MYSQL_USER}
         - MYSQL_PASSWORD=${MYSQL_PASSWORD}
       volumes:
         - nc-db:/var/lib/mysql
       deploy:
         resources:
           limits:
             cpus: '1.0'
             memory: 2G
           reservations:
             cpus: '0.5'
             memory: 1G
       networks:
         - backend

     redis:
       image: redis:8-alpine
       restart: unless-stopped
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 10s
       command: redis-server --appendonly yes
       volumes:
         - redis-data:/data
       deploy:
         resources:
           limits:
             cpus: '0.5'
             memory: 1G
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
       volumes:
         - nc-data:/var/www/html
       extra_hosts:
         - "docs.grupopaico.com.pe:host-gateway"
         - "nube.grupopaico.com.pe:host-gateway"
         - "grupopaico.com.pe:host-gateway"
       deploy:
         resources:
           limits:
             cpus: '4.0'
             memory: 8G
           reservations:
             cpus: '2.0'
             memory: 4G
       networks:
         - backend

     onlyoffice:
       image: onlyoffice/documentserver:9.0.4.1
       restart: unless-stopped
       ports:
         - "127.0.0.1:8082:80"
       volumes:
         - onlyoffice-data:/var/www/onlyoffice/Data
       environment:
         - JWT_ENABLED=true
         - JWT_SECRET=${ONLYOFFICE_JWT_SECRET}
       deploy:
         resources:
           limits:
             cpus: '2.0'
             memory: 4G
           reservations:
             cpus: '1.0'
             memory: 2G
       networks:
         - backend

   volumes:
     nc-db:
     nc-data:
     onlyoffice-data:
     redis-data:

   networks:
     backend:
       driver: bridge
   ```

4. **Iniciar**:
   ```bash
   sudo docker compose pull
   sudo docker compose up -d
   sudo docker compose ps
   sudo docker stats  # ~14GB total uso máx
   ```

**Verificación**: Logs sin errors. Tiempo: 15 min.

---

## ⚙️ 7. Configuración de Nextcloud

1. **Conectividad**:
   ```bash
   curl -I https://docs.grupopaico.com.pe/ --insecure
   curl -I https://nube.grupopaico.com.pe/ --insecure
   sudo docker compose exec nextcloud bash -c "getent hosts docs.grupopaico.com.pe && curl -I https://docs.grupopaico.com.pe/ --insecure"
   ```

2. **OCC Config**:
   ```bash
   cd /opt/docker/nextcloud-onlyoffice
   sudo docker compose exec -u www-data nextcloud php occ config:system:set trusted_domains 1 --value="nube.grupopaico.com.pe"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set trusted_domains 2 --value="grupopaico.com.pe"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set overwrite.cli.url --value="https://nube.grupopaico.com.pe/"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set overwriteprotocol --value="https"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set trusted_proxies 0 --value="127.0.0.1"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set trusted_proxies 1 --value="172.17.0.0/16"
   sudo docker compose exec -u www-data nextcloud php occ config:system:set trusted_proxies 2 --value="10.10.10.1"  # pfSense
   ```

3. **App OnlyOffice**:
   ```bash
   sudo docker compose exec -u www-data nextcloud php occ app:install onlyoffice
   sudo docker compose exec -u www-data nextcloud php occ app:enable onlyoffice
   ```

4. **UI Config**:
   - Accede https://nube.grupopaico.com.pe (admin_paico / pass).
   - Settings > OnlyOffice: Address `https://docs.grupopaico.com.pe/`, JWT secret de .env, Verify Off (temporal), Default open in OnlyOffice.

**Prueba**: Edición .docx colaborativa. Tiempo: 15 min.

---

## 🔄 8. Mantenimiento y Automatizaciones

1. **Cron**:
   ```bash
   sudo crontab -e
   ```
   ```
   */5 * * * * cd /opt/docker/nextcloud-onlyoffice && docker compose exec --user www-data nextcloud php /var/www/html/cron.php
   0 2 * * * cd /opt/docker/nextcloud-onlyoffice && docker compose exec db mysqldump -u ${MYSQL_USER} -p${MYSQL_PASSWORD} ${MYSQL_DATABASE} > /root/backup-nc-$(date +%F).sql && tar czf /root/backup-volumes-$(date +%F).tar.gz nc-db nc-data onlyoffice-data redis-data
   0 0 1 12 * echo "Renovar SSL Namecheap" | mail -s "SSL Alert" admin@grupopaico.com.pe
   ```

2. **Update Script `/root/update.sh`**:
   ```bash
   #!/bin/bash
   cd /opt/docker/nextcloud-onlyoffice
   docker compose pull && docker compose up -d
   docker compose exec -u www-data nextcloud php occ upgrade
   docker compose exec -u www-data nextcloud php occ maintenance:mode --off
   chmod +x /root/update.sh
   ```
   Cron: `0 3 * * 0 /root/update.sh`.

**Tiempo**: 10 min.

---

## 🛡️ 9. Mejoras de Seguridad

1. **Fail2Ban**:
   ```bash
   sudo apt install -y fail2ban
   sudo nano /etc/fail2ban/jail.local
   ```
   ```
   [apache-auth]
   enabled = true
   [apache-badbots]
   enabled = true
   ```
   `sudo systemctl restart fail2ban`.

2. **Nextcloud**:
   - UI: Install Two-Factor TOTP, Enable E2EE, Strong passwords.

3. **Apache Hardening**:
   ```bash
   sudo nano /etc/apache2/conf-available/security.conf
   ```
   ```
   ServerTokens Prod
   ServerSignature Off
   TraceEnable Off
   ```
   `sudo a2enconf security && sudo systemctl restart apache2`.

4. **Backups Externos**: `sudo apt install rclone`, config a Drive, cron sync /root/backup.

**Verificación**: `fail2ban-client status`, securityheaders.com (A+ score). Tiempo: 15 min.

---

## ✅ Checklist Final para GRUPO PAICO

- [ ] SSL Wildcard comprado/instalado (cert chain OK).
- [ ] pfSense forwarding/DNS verificado.
- [ ] Sitio migrado, accesible https://grupopaico.com.pe.
- [ ] Docker up, red backend, recursos escalados.
- [ ] Nextcloud/OnlyOffice integrados, sin warnings.
- [ ] Pruebas: Acceso externo, colaboración, backups.
- [ ] Crons/seguridad activos.

🎉 **¡Migración completa!** Monitorea con `docker stats`. Contacta si logs errors. ¡GRUPO PAICO en la nube self-hosted! 🚀
