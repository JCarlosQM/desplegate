# 📖 Manual de Despliegue Web Apache2 + PHP para grupopaico.com.pe

Este manual describe el proceso paso a paso para instalar, configurar y desplegar
el sitio web oficial **grupopaico.com.pe** en un servidor **Ubuntu + Apache2**, 
exponiéndolo al puerto **80** a través de **pfSense**.

---

## 🚀 0. Instalación de Apache2 + PHP

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mbstring php-xml php-curl unzip

# Habilitar módulos necesarios
sudo a2enmod rewrite headers

# Activar Apache2 y asegurarse de que se inicie con el sistema
sudo systemctl enable --now apache2

# Verificar estado del servicio
sudo systemctl status apache2 --no-pager
```

---

## 📂 1. Subir los archivos del proyecto (PAICO)

Desde tu **PC local** (donde está la carpeta `PAICO`):

```bash
scp -r ./PAICO usuario@10.10.10.2:/tmp/
```

En el **servidor**:

```bash
sudo rsync -av /tmp/PAICO/ /var/www/grupopaico.com.pe/PAICO/
```

---

## 📂 2. (Alternativa) Crear directorio y copiar manualmente

En caso de no usar `scp`:

```bash
sudo mkdir -p /var/www/grupopaico.com.pe/PAICO
sudo rsync -av /ruta/donde/están/PAICO/ /var/www/grupopaico.com.pe/PAICO/
```

---

## 🔑 3. Permisos de archivos y carpetas

```bash
# Asignar usuario y grupo de Apache
sudo chown -R www-data:www-data /var/www/grupopaico.com.pe

# Permisos adecuados
sudo find /var/www/grupopaico.com.pe -type d -exec chmod 755 {} \;
sudo find /var/www/grupopaico.com.pe -type f -exec chmod 644 {} \;
```

---

## 🧪 4. Archivo de prueba PHP

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/grupopaico.com.pe/PAICO/index.php
```

Esto sirve para comprobar que PHP funciona correctamente.

---

## ⚙️ 5. Copia de seguridad de configuración de Apache

```bash
sudo cp /etc/apache2/ports.conf /etc/apache2/ports.conf.bak
```

---

## ⚙️ 6. Configuración de Apache para escuchar en la IP interna

Editar el archivo:

```bash
sudo nano /etc/apache2/ports.conf
```

Y asegurarse de que contenga:

```apache
Listen 10.10.10.2:80
```

---

## 🌐 7. VirtualHost para grupopaico.com.pe

Crear el archivo de configuración:

```bash
sudo nano /etc/apache2/sites-available/grupopaico.com.pe.conf
```

Contenido sugerido:

```apache
<VirtualHost 10.10.10.2:80>
    ServerAdmin admin@grupopaico.com.pe
    ServerName grupopaico.com.pe
    ServerAlias www.grupopaico.com.pe

    DocumentRoot /var/www/grupopaico.com.pe/PAICO

    <Directory /var/www/grupopaico.com.pe/PAICO>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/grupopaico_error.log
    CustomLog ${APACHE_LOG_DIR}/grupopaico_access.log combined
</VirtualHost>
```

---

## ⚡ 8. Habilitar sitio y aplicar cambios

```bash
# Deshabilitar el sitio por defecto (opcional)
sudo a2dissite 000-default.conf || true

# Habilitar el nuevo VirtualHost
sudo a2ensite grupopaico.com.pe.conf

# Verificar sintaxis de configuración
sudo apache2ctl configtest

# Recargar Apache
sudo systemctl reload apache2
```

---

## 🔒 9. Configuración de firewall (UFW)

```bash
sudo ufw allow 80/tcp
sudo ufw reload
sudo ufw status verbose

# Verificar que Apache escucha en el puerto 80
sudo apt-get install iproute2
sudo ss -tlnp | grep ':80' || true
```

Salida esperada (ejemplo):

```
LISTEN 0 128 10.10.10.2:80 0.0.0.0:* users:(("apache2",pid=xxxx,fd=...))
```

---

## 🔄 10. Reiniciar Apache

```bash
sudo systemctl restart apache2
```

---

## ✅ 11. Comprobaciones de funcionamiento

Desde el servidor:

```bash
# Acceso directo por IP
curl -v http://10.10.10.2/

# Simular acceso por dominio (Host header)
curl -v -H "Host: www.grupopaico.com.pe" http://10.10.10.2/
```

Desde un navegador externo:

- http://190.117.188.172 → debe mostrar tu web
- http://www.grupopaico.com.pe (cuando apuntes DNS al servidor público)

---

## 🎯 Resumen

- Archivos en: `/var/www/grupopaico.com.pe/PAICO`
- VirtualHost: `/etc/apache2/sites-available/grupopaico.com.pe.conf`
- Logs en: `/var/log/apache2/grupopaico_error.log` y `grupopaico_access.log`
- IP WAN: `190.117.188.172:80` redirige a `10.10.10.2:80` (pfSense)

Si todo está bien, tu web **ya debería estar accesible públicamente** 🚀

