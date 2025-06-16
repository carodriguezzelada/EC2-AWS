# Guía Completa: Crear Instancias EC2 en la Consola AWS

## 🚀 Paso 1: Acceder a la Consola EC2

1. **Iniciar sesión** en [AWS Console](https://console.aws.amazon.com)
2. **Buscar "EC2"** en la barra de búsqueda superior
3. **Seleccionar** el servicio EC2
4. **Verificar región** en la esquina superior derecha (ej: us-east-1)

---

## 🖥️ Paso 2: Crear Primera Instancia (Airviro)

### 2.1 Iniciar el Proceso
1. Click en **"Launch Instance"** (botón naranja)
2. **Nombre**: `Airviro-Production-Server`

### 2.2 Seleccionar AMI (Sistema Operativo)
1. En **"Application and OS Images"**:
   - Click en **"Browse more AMIs"**
   - En la pestaña **"Community AMIs"**
   - Buscar: `Rocky Linux 9`
   - **Seleccionar**: Rocky Linux 9 (arquitectura x86_64)
   - ✅ Verificar que sea **"Free tier eligible"** si aplica

### 2.3 Tipo de Instancia
1. En **"Instance type"**:
   - **Seleccionar**: `c5.xlarge`
   - 📊 **Especificaciones**: 4 vCPUs, 8 GiB RAM
   - 💰 **Costo estimado**: ~$0.17/hora

### 2.4 Key Pair (Acceso SSH)
1. En **"Key pair (login)"**:
   - Si tienes key pair: **Seleccionar existente**
   - Si no tienes: 
     - Click **"Create new key pair"**
     - **Name**: `airviro-keypair`
     - **Key pair type**: RSA
     - **Private key format**: .pem
     - Click **"Create key pair"**
     - 💾 **¡IMPORTANTE!** Descargar y guardar el archivo .pem

### 2.5 Configuración de Red
1. En **"Network settings"** click **"Edit"**:
   - **VPC**: Seleccionar tu VPC (o default)
   - **Subnet**: Seleccionar subnet pública
   - **Auto-assign public IP**: **Enable**

2. **Security Group** (Firewall):
   - **Create security group**: ✅ Activado
   - **Security group name**: `airviro-sg`
   - **Description**: `Security group para servidor Airviro`
   
3. **Reglas de seguridad**:
   ```
   Regla 1 (SSH):
   - Type: SSH
   - Protocol: TCP
   - Port: 22
   - Source: 0.0.0.0/0 (⚠️ Cambiar por tu IP para mayor seguridad)
   
   Regla 2 (Aplicación Airviro):
   - Type: Custom TCP
   - Protocol: TCP
   - Port: 8080
   - Source: 0.0.0.0/0
   
   Regla 3 (HTTPS - Opcional):
   - Type: HTTPS
   - Protocol: TCP
   - Port: 443
   - Source: 0.0.0.0/0
   ```

### 2.6 Configurar Almacenamiento
1. En **"Configure storage"**:
   - **Root volume**:
     - **Size**: `50 GiB`
     - **Volume type**: `gp3` (mejor rendimiento)
     - **IOPS**: 3000 (default)
     - **Throughput**: 125 MiB/s
     - ✅ **Encrypt**: Activado
     - ✅ **Delete on termination**: Activado

### 2.7 Configuración Avanzada
1. En **"Advanced details"** (expandir):
   - **Monitoring**: ✅ **Enable detailed monitoring**
   - **Termination protection**: ✅ **Enable** (para producción)
   
2. **User data** (script de inicialización):
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y htop wget curl git vim
   hostnamectl set-hostname airviro-server
   
   # Configurar timezone
   timedatectl set-timezone America/Santiago
   
   # Configurar firewall básico
   systemctl enable firewalld
   systemctl start firewalld
   firewall-cmd --permanent --add-service=ssh
   firewall-cmd --permanent --add-port=8080/tcp
   firewall-cmd --reload
   
   # Log de configuración
   echo "$(date): Servidor Airviro configurado exitosamente" >> /var/log/setup.log
   ```

### 2.8 Tags (Etiquetas)
1. En **"Tags"** click **"Add tag"**:
   ```
   Key: Name          Value: Airviro-Production
   Key: Environment   Value: Production
   Key: Application   Value: Airviro
   Key: Owner         Value: [Tu nombre/equipo]
   Key: CostCenter    Value: IT-AIRVIRO
   ```

### 2.9 Revisar y Lanzar
1. **Revisar** toda la configuración en el resumen
2. Click **"Launch instance"**
3. ✅ **Confirmación**: "Successfully initiated launch of instance"

---

## 🌐 Paso 3: Crear Segunda Instancia (Laravel)

### 3.1 Iniciar Proceso
1. Click **"Launch instance"** nuevamente
2. **Name**: `Laravel-Production-Server`

### 3.2 Configuración Laravel
**Repetir pasos 2.2 (misma AMI Rocky Linux 9)**

**Tipo de instancia**:
- **Seleccionar**: `t3.large`
- 📊 **Especificaciones**: 2 vCPUs, 8 GiB RAM, Burst capability
- 💰 **Costo**: ~$0.083/hora

### 3.3 Key Pair
- **Usar el mismo** key pair creado anteriormente o crear uno nuevo

### 3.4 Network Settings - Laravel
**Security Group específico**:
- **Security group name**: `laravel-sg`
- **Description**: `Security group para servidor Laravel`

**Reglas**:
```
Regla 1 (SSH):
- Type: SSH, Port: 22, Source: 0.0.0.0/0

Regla 2 (HTTP):
- Type: HTTP, Port: 80, Source: 0.0.0.0/0

Regla 3 (HTTPS):
- Type: HTTPS, Port: 443, Source: 0.0.0.0/0

Regla 4 (Laravel Dev - Opcional):
- Type: Custom TCP, Port: 8000, Source: 0.0.0.0/0
```

### 3.5 Almacenamiento Laravel
- **Size**: `30 GiB` (suficiente para Laravel)
- **Volume type**: `gp3`
- ✅ **Encrypt**: Activado

### 3.6 User Data para Laravel
```bash
#!/bin/bash
yum update -y
yum install -y epel-release

# Instalar stack LAMP para Laravel
yum install -y httpd mariadb-server
yum install -y php php-cli php-fpm php-mysql php-json php-opcache php-mbstring php-xml php-gd php-curl php-zip

# Instalar Node.js (para assets)
curl -sL https://rpm.nodesource.com/setup_18.x | bash -
yum install -y nodejs

# Instalar Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
chmod +x /usr/local/bin/composer

# Configurar servicios
systemctl enable httpd mariadb
systemctl start httpd mariadb

# Configurar hostname
hostnamectl set-hostname laravel-server

# Configurar timezone
timedatectl set-timezone America/Santiago

# Configurar firewall
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Crear directorio para Laravel
mkdir -p /var/www/html/laravel
chown apache:apache /var/www/html/laravel

echo "$(date): Servidor Laravel configurado exitosamente" >> /var/log/setup.log
```

### 3.7 Tags Laravel
```
Key: Name          Value: Laravel-Production
Key: Environment   Value: Production
Key: Application   Value: Laravel
Key: Owner         Value: [Tu nombre/equipo]
Key: CostCenter    Value: IT-LARAVEL
```

---

## 🔍 Paso 4: Verificar Instancias

### 4.1 Monitorear Estado
1. En **EC2 Dashboard** → **"Instances"**
2. **Verificar estado**:
   - ✅ **Instance State**: Running
   - ✅ **Status Check**: 2/2 checks passed

### 4.2 Obtener Información de Conexión
**Para cada instancia, anotar**:
- **Instance ID**: i-0123456789abcdef0
- **Public IPv4 address**: 54.123.45.67
- **Private IPv4 address**: 10.0.1.100

---

## 🌐 Paso 5: Configurar IPs Elásticas (Recomendado)

### 5.1 Crear Elastic IPs
1. En el menú EC2 → **"Elastic IPs"**
2. Click **"Allocate Elastic IP address"**
3. **Network Border Group**: Mantener default
4. **Tags**: 
   - Name: `Airviro-EIP`
5. Click **"Allocate"**
6. **Repetir** para Laravel con tag `Laravel-EIP`

### 5.2 Asociar IPs Elásticas
1. **Seleccionar** la IP elástica
2. Click **"Actions"** → **"Associate Elastic IP address"**
3. **Resource type**: Instance
4. **Instance**: Seleccionar instancia correspondiente
5. Click **"Associate"**

---

## 🔐 Paso 6: Conectarse a las Instancias

### 6.1 Usando SSH (Linux/Mac)
```bash
# Cambiar permisos del archivo key
chmod 400 ~/Downloads/airviro-keypair.pem

# Conectar a Airviro
ssh -i ~/Downloads/airviro-keypair.pem ec2-user@[IP_PUBLICA_AIRVIRO]

# Conectar a Laravel
ssh -i ~/Downloads/airviro-keypair.pem ec2-user@[IP_PUBLICA_LARAVEL]
```

### 6.2 Usando PuTTY (Windows)
1. **Convertir .pem a .ppk** usando PuTTYgen
2. **Configurar PuTTY**:
   - Host: IP pública de la instancia
   - Port: 22
   - Connection → SSH → Auth → Browse → Seleccionar archivo .ppk

### 6.3 Usar EC2 Instance Connect (Desde consola)
1. **Seleccionar instancia** en la consola
2. Click **"Connect"**
3. Tab **"EC2 Instance Connect"**
4. **User name**: `ec2-user`
5. Click **"Connect"**

---

## ✅ Paso 7: Verificación Post-Instalación

### 7.1 Comandos de Verificación
```bash
# Verificar sistema
cat /etc/rocky-release
uname -a
df -h
free -h

# Verificar servicios (Laravel)
systemctl status httpd
systemctl status mariadb

# Verificar conectividad
curl -I http://localhost
```

### 7.2 Configuración Inicial de Seguridad
```bash
# Actualizar sistema
sudo yum update -y

# Configurar usuario adicional (opcional)
sudo adduser [tu-usuario]
sudo usermod -aG wheel [tu-usuario]

# Configurar fail2ban
sudo yum install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## 🚨 Checklist Final

- [ ] ✅ Instancia Airviro (c5.xlarge) creada y funcionando
- [ ] ✅ Instancia Laravel (t3.large) creada y funcionando
- [ ] ✅ Security groups configurados correctamente
- [ ] ✅ IPs elásticas asignadas
- [ ] ✅ Conexión SSH funcional a ambas instancias
- [ ] ✅ User data ejecutado correctamente
- [ ] ✅ Tags aplicados para organización
- [ ] ✅ Monitoreo habilitado
- [ ] ✅ Encriptación de volúmenes activada

---

## 💡 Próximos Pasos

1. **Configurar aplicaciones específicas** en cada servidor
2. **Implementar backup automatizado**
3. **Configurar monitoreo con CloudWatch**
4. **Establecer procedimientos de mantenimiento**
5. **Documentar arquitectura y procedimientos**

---

## 🆘 Troubleshooting Común

### Problema: No puedo conectar por SSH
**Solución**:
- Verificar security group permite puerto 22
- Verificar IP pública asignada
- Verificar permisos del archivo .pem (chmod 400)

### Problema: User data no se ejecutó
**Solución**:
- Revisar logs: `sudo cat /var/log/cloud-init-output.log`
- Verificar sintaxis del script

### Problema: Alto costo
**Solución**:
- Parar instancias cuando no se usen
- Considerar Reserved Instances para uso continuo
- Monitorear uso con AWS Cost Explorer