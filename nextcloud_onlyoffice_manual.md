# 📘 Manual de instalación: Nextcloud + OnlyOffice con Docker Compose

Este manual documenta los pasos para instalar y desplegar **Nextcloud** y **OnlyOffice DocumentServer** en el mismo servidor, **sin interferir con Apache2**, usando **puertos alternativos HTTP**.

---

## 1️⃣ Instalar Docker y Docker Compose

```bash
# Actualizar repos
sudo apt update && sudo apt upgrade -y

# Instalar paquetes básicos
sudo apt install -y ca-certificates curl gnupg lsb-release

# Agregar clave de Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Agregar repo oficial
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg]   https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"   | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker + plugins + Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Habilitar y arrancar servicio
sudo systemctl enable --now docker

# Verificar instalación
docker --version
docker compose version
```

---

## 2️⃣ Preparar estructura de directorios

```bash
# Carpeta base para proyectos
mkdir -p ~/docker/nextcloud-onlyoffice
cd ~/docker/nextcloud-onlyoffice
```

---

## 3️⃣ Crear archivo `docker-compose.yml`

Guarda este contenido en:  
```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

volumes:
  nextcloud:
  db:
  postgresql_data:

services:
  # ======================
  # Nextcloud + MariaDB
  # ======================
  db:
    image: mariadb:10.6
    restart: always
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=passr   # 🔐 cambia esta contraseña
      - MYSQL_PASSWORD=passm        # 🔐 cambia esta contraseña
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud
    restart: always
    ports:
      - "8081:80"   # 🚪 Cambia el puerto externo si quieres (por defecto LAN 8081)
    depends_on:
      - db
    volumes:
      - nextcloud:/var/www/html
    environment:
      - MYSQL_PASSWORD=passm
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db

  # ======================
  # OnlyOffice DocumentServer
  # ======================
  onlyoffice-documentserver:
    image: onlyoffice/documentserver
    container_name: onlyoffice-documentserver
    depends_on:
      - onlyoffice-postgresql
      - onlyoffice-rabbitmq
    environment:
      - DB_TYPE=postgres
      - DB_HOST=onlyoffice-postgresql
      - DB_PORT=5432
      - DB_NAME=onlyoffice
      - DB_USER=onlyoffice
      - AMQP_URI=amqp://guest:guest@onlyoffice-rabbitmq
      - JWT_ENABLED=true
      - JWT_SECRET=secret   # 🔐 cambia este secreto
      - JWT_HEADER=Authorization
      - JWT_IN_BODY=true
    ports:
      - "8082:80"   # 🚪 Cambia el puerto externo si quieres (por defecto LAN 8082)
    restart: always
    volumes:
      - /var/www/onlyoffice/Data
      - /var/log/onlyoffice
      - /var/lib/onlyoffice/documentserver/App_Data/cache/files
      - /var/www/onlyoffice/documentserver-example/public/files
      - /usr/share/fonts

  onlyoffice-rabbitmq:
    container_name: onlyoffice-rabbitmq
    image: rabbitmq:3
    restart: always
    expose:
      - '5672'

  onlyoffice-postgresql:
    container_name: onlyoffice-postgresql
    image: postgres:15
    environment:
      - POSTGRES_DB=onlyoffice
      - POSTGRES_USER=onlyoffice
      - POSTGRES_HOST_AUTH_METHOD=trust
    restart: always
    expose:
      - '5432'
    volumes:
      - postgresql_data:/var/lib/postgresql
```

---

## 4️⃣ Levantar los servicios

```bash
# Descargar imágenes y levantar contenedores
docker compose up -d

# Verificar estado
docker ps
```

Salida esperada (ejemplo):
```
CONTAINER ID   NAME                      PORTS
abc123...      app                       0.0.0.0:8081->80/tcp
def456...      onlyoffice-documentserver 0.0.0.0:8082->80/tcp
```

---

## 5️⃣ Probar acceso en LAN

- Nextcloud → [http://10.10.10.2:8081](http://10.10.10.2:8081)  
- OnlyOffice → [http://10.10.10.2:8082](http://10.10.10.2:8082)  

---

## 6️⃣ Integrar OnlyOffice en Nextcloud

En **Nextcloud**:  
1. Ve a **Apps → Office & text**  
2. Instala el plugin **ONLYOFFICE**  
3. Configura:  
   - Dirección del servicio: `http://10.10.10.2:8082`  
   - Secret: `secret` (o el valor que hayas definido en `JWT_SECRET`)  

---

## 7️⃣ Acceso desde WAN (opcional)

En tu **pfSense**, crea reglas de **port forwarding**:  
- WAN:8081 → DMZ:10.10.10.2:8081 (Nextcloud)  
- WAN:8082 → DMZ:10.10.10.2:8082 (OnlyOffice)  

Acceso desde fuera:  
- `http://TU_IP_PUBLICA:8081`  
- `http://TU_IP_PUBLICA:8082`  

---

## 🔐 Notas de seguridad
- Usa **contraseñas seguras** en MySQL y JWT.  
- En producción, lo ideal es **cerrar puertos 8081/8082** y usar **reverse proxy con Apache2 + SSL (443)**.  
- No dejes `trust` como método de autenticación en PostgreSQL para producción.  
