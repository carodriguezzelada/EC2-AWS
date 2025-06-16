# Guía Completa para Configurar OpenVPN en EC2 para Acceso SSH

Esta guía detalla cómo configurar un servidor OpenVPN en AWS EC2 para permitir que usuarios con IPs dinámicas accedan de forma segura por SSH a una instancia EC2 principal.

## Tabla de Contenidos
1. [Preparación de la Infraestructura](#preparación-de-la-infraestructura)
2. [Instalación y Configuración de OpenVPN](#instalación-y-configuración-de-openvpn)
3. [Configurar el Servidor OpenVPN](#configurar-el-servidor-openvpn)
4. [Crear Configuraciones de Cliente](#crear-configuraciones-de-cliente)
5. [Configurar Autenticación de Dos Factores (2FA)](#configurar-autenticación-de-dos-factores-2fa)
5. [Configurar Autenticación de Dos Factores (2FA)](#configurar-autenticación-de-dos-factores-2fa)
6. [Configurar el Acceso SSH a tu EC2 Principal](#configurar-el-acceso-ssh-a-tu-ec2-principal)
7. [Guía para los Usuarios](#guía-para-los-usuarios)
8. [Mantenimiento y Seguridad](#mantenimiento-y-seguridad)

---

## Preparación de la Infraestructura

### 1. Crear la Instancia EC2 para OpenVPN

1. **Inicia sesión en la consola AWS**
2. **Lanza una nueva instancia EC2**:
   - **AMI**: Ubuntu Server 22.04 LTS
   - **Tipo**: t3.nano (suficiente para OpenVPN con pocos usuarios)
   - **Configuración de red**: VPC predeterminada, habilitar asignación de IP pública
   - **Almacenamiento**: 8GB GP3 (suficiente)
   - **Grupo de seguridad**: Crear nuevo con las siguientes reglas:
     - SSH (TCP 22): Tu IP actual (para configuración)
     - OpenVPN (UDP 1194): 0.0.0.0/0
     - ICMP: 0.0.0.0/0 (opcional, para ping)
   - **Nombre**: "openvpn-server"
   - **Par de claves**: Crea o selecciona uno existente

### 2. Asignar una IP Elástica

1. **En EC2 Dashboard, navega a "IPs Elásticas"**
2. **Haz clic en "Asignar dirección IP elástica"**
3. **Selecciona "Grupo de direcciones IPv4 de Amazon"** y haz clic en "Asignar"
4. **Selecciona la IP recién creada** y elige "Acciones" → "Asociar dirección IP elástica"
5. **Selecciona tu instancia OpenVPN** y haz clic en "Asociar"
6. **Anota la dirección IP elástica** (la necesitarás más adelante)

### 3. Configurar Grupo de Seguridad para EC2 Principal

1. **Localiza tu instancia EC2 principal** donde quieres permitir SSH
2. **Modifica su grupo de seguridad** para permitir conexiones SSH (puerto 22) solo desde la subred que usará OpenVPN (lo configuraremos más adelante)

---

## Instalación y Configuración de OpenVPN

### 1. Conectarse a la Instancia OpenVPN

```bash
ssh -i tu-clave.pem ubuntu@tu-ip-elastica
```

### 2. Actualizar el Sistema e Instalar OpenVPN

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openvpn easy-rsa
```

### 3. Configurar la Autoridad Certificadora (CA)

```bash
# Crear estructura de directorios para Easy-RSA
mkdir -p ~/easy-rsa
ln -s /usr/share/easy-rsa/* ~/easy-rsa/
cd ~/easy-rsa
```

```bash
# Inicializar el directorio PKI
./easyrsa init-pki

# Crear la autoridad certificadora
./easyrsa build-ca nopass
```

> **Nota**: Cuando te pida "Common Name", puedes ingresar algo como "OpenVPN-CA"

### 4. Generar Certificado y Clave del Servidor

```bash
./easyrsa build-server-full server nopass
./easyrsa gen-dh
openvpn --genkey secret ~/easy-rsa/pki/ta.key
```

### 5. Generar Certificados para Clientes

Repite estos pasos para cada usuario (user1, user2, user3):

```bash
./easyrsa build-client-full user1 nopass
./easyrsa build-client-full user2 nopass
./easyrsa build-client-full user3 nopass
```

### 6. Crear Estructura de Carpetas para OpenVPN

```bash
sudo mkdir -p /etc/openvpn/server/
sudo mkdir -p /etc/openvpn/certs/
sudo mkdir -p /etc/openvpn/clients/
```

### 7. Copiar los Certificados al Directorio de OpenVPN

```bash
# Certificados del servidor
sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/certs/
sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/certs/
sudo cp ~/easy-rsa/pki/ta.key /etc/openvpn/certs/
sudo cp ~/easy-rsa/pki/issued/server.crt /etc/openvpn/certs/
sudo cp ~/easy-rsa/pki/private/server.key /etc/openvpn/certs/

# Certificados de clientes
sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/clients/
sudo cp ~/easy-rsa/pki/ta.key /etc/openvpn/clients/
sudo cp ~/easy-rsa/pki/issued/user*.crt /etc/openvpn/clients/
sudo cp ~/easy-rsa/pki/private/user*.key /etc/openvpn/clients/
```

---

## Configurar el Servidor OpenVPN

### 1. Crear Archivo de Configuración del Servidor

```bash
sudo nano /etc/openvpn/server/server.conf
```

Pega el siguiente contenido:

```
port 1194
proto udp
dev tun
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key
dh /etc/openvpn/certs/dh.pem
tls-auth /etc/openvpn/certs/ta.key 0
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
explicit-exit-notify 1
```

### 2. Crear Directorio para Logs

```bash
sudo mkdir -p /var/log/openvpn/
```

### 3. Configurar el Reenvío de IP

```bash
# Habilitar el reenvío de IP temporalmente
sudo sysctl -w net.ipv4.ip_forward=1

# Habilitar el reenvío de IP permanentemente
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
```

### 4. Configurar NAT

```bash
# Obtén el nombre de la interfaz de red principal
INTERFACE=$(ip route | grep default | awk '{print $5}')

# Configura NAT
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o $INTERFACE -j MASQUERADE

# Guarda las reglas de iptables para que persistan
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

### 5. Iniciar y Habilitar OpenVPN

```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
sudo systemctl status openvpn@server  # Verifica que esté funcionando
```

---

## Crear Configuraciones de Cliente

### 1. Crear Script para Generar Configuraciones de Cliente

```bash
sudo nano /etc/openvpn/clients/make_config.sh
```

Pega el siguiente contenido:

```bash
#!/bin/bash

# Primer argumento: nombre del cliente

KEY_DIR=/etc/openvpn/clients
OUTPUT_DIR=/etc/openvpn/clients
BASE_CONFIG=/etc/openvpn/clients/base.conf

if [ ! -f "$BASE_CONFIG" ]; then
    echo "># Configuración base OpenVPN cliente
    client
    dev tun
    proto udp
    remote TU-IP-ELASTICA 1194
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    mute-replay-warnings
    remote-cert-tls server
    cipher AES-256-CBC
    verb 3
    key-direction 1" > "$BASE_CONFIG"
    
    # Reemplazar TU-IP-ELASTICA con la IP elástica real
    sed -i "s/TU-IP-ELASTICA/$(curl -s http://checkip.amazonaws.com || echo 'your-server-ip')/g" "$BASE_CONFIG"
fi

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
    
echo "Archivo de configuración creado en: ${OUTPUT_DIR}/${1}.ovpn"
```

### 2. Hacer Ejecutable el Script y Crear Configuraciones

```bash
sudo chmod +x /etc/openvpn/clients/make_config.sh
sudo /etc/openvpn/clients/make_config.sh user1
sudo /etc/openvpn/clients/make_config.sh user2
sudo /etc/openvpn/clients/make_config.sh user3
```

### 3. Descargar los Archivos de Configuración

Desde tu máquina local, descarga los archivos .ovpn:

```bash
scp -i tu-clave.pem ubuntu@tu-ip-elastica:/etc/openvpn/clients/user1.ovpn ./
scp -i tu-clave.pem ubuntu@tu-ip-elastica:/etc/openvpn/clients/user2.ovpn ./
scp -i tu-clave.pem ubuntu@tu-ip-elastica:/etc/openvpn/clients/user3.ovpn ./
```

---

## Configurar el Acceso SSH a tu EC2 Principal

### 1. Modificar el Grupo de Seguridad de tu EC2 Principal

1. En la consola AWS, navega a **EC2** → **Grupos de seguridad**
2. Selecciona el grupo de seguridad asociado a tu instancia EC2 principal
3. Edita las reglas de entrada
4. Agrega una regla:
   - **Tipo**: SSH (TCP 22)
   - **Origen**: 10.8.0.0/24 (la subred VPN)
   - **Descripción**: "SSH access from OpenVPN clients"
5. Guarda las reglas

---

## Guía para los Usuarios

### 1. Instalación de Clientes OpenVPN

Instrucciones por sistema operativo:

- **Windows**: [OpenVPN GUI](https://openvpn.net/community-downloads/)
- **macOS**: [Tunnelblick](https://tunnelblick.net/) o OpenVPN Connect
- **Linux**: 
  ```bash
  # Ubuntu/Debian
  sudo apt install openvpn
  
  # CentOS/RHEL/Fedora
  sudo dnf install openvpn
  ```
- **Android/iOS**: OpenVPN Connect app desde la tienda de aplicaciones

### 2. Instrucciones de Conexión

```
1. Instala el cliente OpenVPN para tu sistema operativo
2. Importa el archivo .ovpn proporcionado
3. Conéctate a la VPN
4. Una vez conectado, accede a la instancia EC2 principal:
   ssh usuario@ip-privada-de-tu-ec2
```

### 3. Ejemplo de Uso en Linux

```bash
# Conectar a la VPN
sudo openvpn --config user1.ovpn

# En otra terminal, conectar por SSH
ssh ec2-user@10.0.1.100  # Reemplaza con la IP privada de tu EC2
```

---

## Mantenimiento y Seguridad

### Monitoreo y Diagnóstico

```bash
# Verificar estado del servicio
sudo systemctl status openvpn@server

# Ver logs en tiempo real
sudo tail -f /var/log/openvpn/openvpn-status.log

# Ver conexiones activas
sudo cat /var/log/openvpn/openvpn-status.log

# Reiniciar OpenVPN
sudo systemctl restart openvpn@server
```

### Revocar Certificados

Si necesitas revocar acceso a un usuario:

```bash
cd ~/easy-rsa
./easyrsa revoke user1
./easyrsa gen-crl
sudo cp ~/easy-rsa/pki/crl.pem /etc/openvpn/server/
```

Luego agrega esta línea a `/etc/openvpn/server/server.conf`:
```
crl-verify /etc/openvpn/server/crl.pem
```

Y reinicia OpenVPN:
```bash
sudo systemctl restart openvpn@server
```

### Agregar Nuevos Usuarios

Para agregar un nuevo usuario:

```bash
cd ~/easy-rsa
./easyrsa build-client-full nuevouser nopass
sudo cp ~/easy-rsa/pki/issued/nuevouser.crt /etc/openvpn/clients/
sudo cp ~/easy-rsa/pki/private/nuevouser.key /etc/openvpn/clients/
sudo /etc/openvpn/clients/make_config.sh nuevouser
```

### Configurar Autenticación de Dos Factores (2FA)

#### 1. Instalar Google Authenticator PAM

```bash
sudo apt install -y libpam-google-authenticator
```

#### 2. Configurar Google Authenticator para cada usuario

Primero, crea usuarios del sistema para cada usuario VPN:

```bash
# Crear usuarios (sin shell de login)
sudo useradd -M -s /bin/false user1
sudo useradd -M -s /bin/false user2
sudo useradd -M -s /bin/false user3

# Configurar Google Authenticator para cada usuario
sudo -u user1 google-authenticator
sudo -u user2 google-authenticator
sudo -u user3 google-authenticator
```

Para cada usuario, responde:
- `¿Quieres que los tokens de autenticación sean basados en tiempo?` → **Y**
- `¿Quieres actualizar tu archivo ~/.google_authenticator?` → **Y**
- `¿Quieres deshabilitar múltiples usos del mismo token?` → **Y**
- `Por defecto, los tokens son válidos por 30 segundos...` → **N**
- `¿Quieres habilitar la limitación de velocidad?` → **Y**

> **Importante**: Guarda el código QR y las claves de emergencia para cada usuario

#### 3. Configurar PAM para OpenVPN

```bash
sudo nano /etc/pam.d/openvpn
```

Agrega el siguiente contenido:

```
auth required pam_google_authenticator.so
```

#### 4. Crear Script de Autenticación

```bash
sudo nano /etc/openvpn/scripts/auth-script.sh
```

Agrega el siguiente contenido:

```bash
#!/bin/bash
# Script de autenticación OpenVPN con Google Authenticator

# Leer usuario y contraseña desde OpenVPN
username=$(head -n 1 $1)
password=$(tail -n 1 $1)

# Verificar que el usuario existe
if ! id "$username" &>/dev/null; then
    logger "OpenVPN: Usuario $username no existe"
    exit 1
fi

# Verificar el token de Google Authenticator
if su -s /bin/bash -c "echo '$password' | google-authenticator -t -f ~/.google_authenticator" "$username" 2>/dev/null; then
    logger "OpenVPN: Autenticación 2FA exitosa para $username"
    exit 0
else
    logger "OpenVPN: Autenticación 2FA fallida para $username"
    exit 1
fi
```

```bash
sudo chmod +x /etc/openvpn/scripts/auth-script.sh
sudo mkdir -p /etc/openvpn/scripts/
```

#### 5. Modificar Configuración del Servidor OpenVPN

Edita la configuración del servidor:

```bash
sudo nano /etc/openvpn/server/server.conf
```

Agrega estas líneas al final del archivo:

```
# Configuración de autenticación 2FA
auth-user-pass-verify /etc/openvpn/scripts/auth-script.sh via-env
client-cert-not-required
username-as-common-name
script-security 3
```

#### 6. Reiniciar OpenVPN

```bash
sudo systemctl restart openvpn@server
```

#### 7. Actualizar Configuraciones de Cliente

Modifica el script de generación de configuraciones:

```bash
sudo nano /etc/openvpn/clients/make_config.sh
```

Cambia la configuración base para incluir autenticación por usuario:

```bash
#!/bin/bash

# Primer argumento: nombre del cliente

KEY_DIR=/etc/openvpn/clients
OUTPUT_DIR=/etc/openvpn/clients
BASE_CONFIG=/etc/openvpn/clients/base.conf

if [ ! -f "$BASE_CONFIG" ]; then
    echo "># Configuración base OpenVPN cliente con 2FA
    client
    dev tun
    proto udp
    remote TU-IP-ELASTICA 1194
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    mute-replay-warnings
    remote-cert-tls server
    cipher AES-256-CBC
    verb 3
    key-direction 1
    auth-user-pass" > "$BASE_CONFIG"
    
    # Reemplazar TU-IP-ELASTICA con la IP elástica real
    sed -i "s/TU-IP-ELASTICA/$(curl -s http://checkip.amazonaws.com || echo 'your-server-ip')/g" "$BASE_CONFIG"
fi

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
    
echo "Archivo de configuración creado en: ${OUTPUT_DIR}/${1}.ovpn"
```

#### 8. Regenerar Configuraciones de Cliente

```bash
sudo /etc/openvpn/clients/make_config.sh user1
sudo /etc/openvpn/clients/make_config.sh user2
sudo /etc/openvpn/clients/make_config.sh user3
```

#### 9. Instrucciones para Usuarios con 2FA

Proporciona a cada usuario:

1. **Su archivo .ovpn actualizado**
2. **Su código QR de Google Authenticator** (de cuando ejecutaste `google-authenticator`)
3. **Instrucciones de conexión**:

```
1. Instala Google Authenticator en tu teléfono
2. Escanea el código QR proporcionado
3. Al conectarte a la VPN:
   - Usuario: tu_nombre_usuario (user1, user2, etc.)
   - Contraseña: El código de 6 dígitos de Google Authenticator
```

#### 10. Crear Archivo de Credenciales (Opcional)

Para evitar que los usuarios tengan que escribir credenciales cada vez:

```bash
# El usuario crea un archivo con sus credenciales
nano auth.txt
```

Contenido del archivo:
```
user1
123456
```

Luego modifica el archivo .ovpn del cliente:
```
auth-user-pass auth.txt
```

> **Advertencia**: Esto es menos seguro ya que almacena las credenciales en texto plano

### Seguridad Adicional (Actualizada)

- **Autenticación de dos factores**: ✅ Implementado con Google Authenticator
- **Rotación de certificados**: Rota los certificados cada 6-12 meses
- **Auditoría**: Revisa regularmente los logs de conexión
- **Firewall**: Configura fail2ban para proteger contra ataques de fuerza bruta

### Backup de Configuración

Respalda regularmente estos archivos importantes:

```bash
# Crear backup
sudo tar -czf openvpn-backup-$(date +%Y%m%d).tar.gz \
  /etc/openvpn/ \
  ~/easy-rsa/pki/

# Descargar backup
scp -i tu-clave.pem ubuntu@tu-ip-elastica:openvpn-backup-*.tar.gz ./
```

---

## Costos Estimados

- **Instancia t3.nano**: ~$3.50/mes
- **IP Elástica**: Gratis mientras esté asociada
- **Transferencia de datos**: $0.09/GB saliente
- **Total aproximado**: $5-10/mes (dependiendo del uso)

---

## Solución de Problemas Comunes

### El servidor OpenVPN no inicia

```bash
# Verificar configuración
sudo openvpn --config /etc/openvpn/server/server.conf --verb 6

# Verificar permisos
sudo chown -R root:root /etc/openvpn/certs/
sudo chmod 600 /etc/openvpn/certs/server.key
```

### Los clientes no pueden conectarse

1. Verificar que el grupo de seguridad permita UDP 1194
2. Verificar que la IP elástica esté correcta en el archivo de configuración del cliente
3. Verificar que el reenvío de IP esté habilitado en el servidor

### No hay acceso a internet desde la VPN

```bash
# Verificar NAT
sudo iptables -t nat -L POSTROUTING -v

# Reconfigurar NAT si es necesario
INTERFACE=$(ip route | grep default | awk '{print $5}')
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o $INTERFACE -j MASQUERADE
sudo netfilter-persistent save
```

### Problemas con 2FA

```bash
# Verificar que PAM esté configurado correctamente
sudo cat /etc/pam.d/openvpn

# Verificar logs de autenticación
sudo tail -f /var/log/auth.log

# Probar autenticación manualmente
sudo -u user1 google-authenticator -t -f ~/.google_authenticator
# Introduce el código de 6 dígitos cuando se solicite

# Verificar permisos del script de autenticación
ls -la /etc/openvpn/scripts/auth-script.sh
```

### Regenerar códigos 2FA

Si un usuario pierde acceso a su teléfono:

```bash
# Regenerar configuración de Google Authenticator
sudo -u user1 google-authenticator

# Proporcionar el nuevo código QR al usuario
```

---

## Conclusión

Esta configuración proporciona una solución segura y económica para permitir acceso SSH a instancias EC2 desde ubicaciones con IPs dinámicas. El costo total es significativamente menor que las soluciones VPN administradas de AWS, manteniendo un alto nivel de seguridad y control.

Para soporte adicional o mejoras, considera documentar cualquier modificación específica para tu entorno.