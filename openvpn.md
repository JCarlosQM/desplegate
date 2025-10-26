
## 🔧 Configuración de VPN OpenVPN en pfSense

### 1. Crear la Autoridad Certificadora (CA)

Ruta: **System → Certificate Manager → CAs**

| Campo                 | Valor                                    |
| --------------------- | ---------------------------------------- |
| **Descriptive name**  | Nombre identificativo (ej. *PAICO_CA*)   |
| **Method**            | Create an internal Certificate Authority |
| **Trust Store**       | ✔️ (habilitado)                          |
| **Key type**          | RSA                                      |
| **Common name**       | Nombre de la CA (ej. *PAICO_CA*)         |
| **State or Province** | Tu región o departamento                 |

---

### 2. Crear el Certificado del Servidor

Ruta: **System → Certificate Manager → Certificates**

| Campo                     | Valor                          |
| ------------------------- | ------------------------------ |
| **Method**                | Create an internal Certificate |
| **Descriptive name**      | Ej. *OpenVPN_Server_Cert*      |
| **Certificate Authority** | Selecciona la CA creada        |
| **Lifetime (days)**       | 365                            |
| **Common name**           | Ej. *openvpn-server*           |
| **Certificate Type**      | Server Certificate             |

---

### 3. Configurar el Servidor OpenVPN

Ruta: **VPN → OpenVPN → Servers**

| Campo                          | Valor                                  |
| ------------------------------ | -------------------------------------- |
| **Description**                | OPENVPN_LOCAL                          |
| **Server mode**                | Remote Access (SSL/TLS + User Auth)    |
| **Backend for authentication** | Local Database                         |
| **Peer Certificate Authority** | Selecciona la CA creada                |
| **Server Certificate**         | Selecciona el certificado del servidor |
| **Protocol**                   | UDP                                    |
| **Interface**                  | WAN                                    |
| **IPv4 Tunnel Network**        | 172.16.1.0/24                          |
| **IPv4 Local Network(s)**      | 192.168.1.0/24                         |
| **Gateway creation**           | IPv4 only                              |

Guarda y aplica los cambios.

---

### 4. Reglas de Firewall

#### WAN

Ruta: **Firewall → Rules → WAN**

| Campo                      | Valor       |
| -------------------------- | ----------- |
| **Action**                 | Pass        |
| **Interface**              | WAN         |
| **Address Family**         | IPv4        |
| **Protocol**               | UDP         |
| **Source**                 | WAN subnet  |
| **Destination**            | WAN address |
| **Destination Port Range** | OpenVPN     |

#### OpenVPN

Ruta: **Firewall → Rules → OpenVPN**

| Campo         | Valor   |
| ------------- | ------- |
| **Action**    | Pass    |
| **Interface** | OpenVPN |
| **Protocol**  | Any     |

Aplica los cambios.

---

### 5. Instalar el Paquete de Exportación

Ruta: **System → Package Manager → Available Packages**

* Instalar: **openvpn-client-export**

---

### 6. Crear Usuarios VPN

Ruta: **System → User Manager → Users → Add**

| Campo                     | Valor                                      |
| ------------------------- | ------------------------------------------ |
| **Username**              | Ej. *usuario1*                             |
| **Password**              | Contraseña segura                          |
| **Full name**             | Nombre completo                            |
| **Certificate**           | Crear un nuevo certificado para el usuario |
| **Descriptive name**      | Ej. *User VPN*                             |
| **Certificate Authority** | Selecciona la CA creada                    |

Guarda el usuario.

---

### 7. Exportar Configuración del Cliente

Ruta: **VPN → OpenVPN → Client Export**

| Campo                    | Valor                                      |
| ------------------------ | ------------------------------------------ |
| **Remote Access Server** | Selecciona el servidor creado              |
| **Export Type**          | “Most Clients” o “Viscosity Inline Config” |

Descarga el archivo `.ovpn` generado y distribúyelo al usuario.

---

### 8. Monitoreo

Ruta: **Status → Dashboard → OpenVPN**

Verifica:

* Usuarios conectados
* Estado del servicio
* Estadísticas de túnel

---

### 🔗 Referencia

* Documentación oficial: [https://openvpn.net/](https://openvpn.net/)
* Video 1: [https://www.youtube.com/watch?v=xNf-vesJwnc](https://www.youtube.com/watch?v=xNf-vesJwnc)
* Video 2: [https://www.youtube.com/watch?v=IhpLaH4085M](https://www.youtube.com/watch?v=IhpLaH4085M)


