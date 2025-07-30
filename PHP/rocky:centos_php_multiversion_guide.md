# Guía: Instalación de Múltiples Versiones de PHP en CentOS 7 / Rocky Linux

## Objetivo
Instalar PHP 8.3 junto a PHP 8.1 existente, manteniendo 8.1 como versión por defecto y configurando Apache para usar PHP 8.3 en módulos específicos.

## Prerrequisitos
- CentOS 7 o Rocky Linux
- Apache (httpd) instalado y funcionando
- PHP 8.1 ya instalado y configurado
- Permisos de administrador (sudo/root)

## Paso 1: Habilitar repositorios necesarios

### Para CentOS 7:
```bash
# Instalar EPEL
sudo yum install -y epel-release

# Instalar repositorio Remi
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm

# Habilitar repositorio Remi para PHP 8.3
sudo yum-config-manager --enable remi-php83
```

### Para Rocky Linux:
```bash
# Instalar EPEL
sudo dnf install -y epel-release

# Instalar repositorio Remi
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm

# Habilitar repositorio Remi para PHP 8.3
sudo dnf config-manager --set-enabled remi
sudo dnf module reset php
sudo dnf module enable php:remi-8.3
```

## Paso 2: Instalar PHP 8.3 con módulos completos para Laravel

### Para CentOS 7:
```bash
# Instalar PHP 8.3 y módulos esenciales para Laravel
sudo yum install -y php83-php php83-php-fpm php83-php-cli php83-php-common \
php83-php-mysqlnd php83-php-pgsql php83-php-sqlite3 php83-php-xml \
php83-php-curl php83-php-gd php83-php-imagick php83-php-opcache \
php83-php-soap php83-php-zip php83-php-intl php83-php-bcmath \
php83-php-json php83-php-mbstring php83-php-pdo php83-php-fileinfo \
php83-php-tokenizer php83-php-xmlwriter php83-php-simplexml \
php83-php-dom php83-php-session php83-php-ctype php83-php-filter \
php83-php-hash php83-php-openssl php83-php-pcre php83-php-redis \
php83-php-memcached php83-php-iconv

# Crear enlaces simbólicos para facilitar el uso
sudo ln -sf /opt/remi/php83/root/usr/bin/php /usr/bin/php83
sudo ln -sf /opt/remi/php83/root/usr/bin/php-fpm /usr/bin/php83-fpm
```

### Para Rocky Linux:
```bash
# Instalar PHP 8.3 con módulos completos para Laravel
sudo dnf install -y php83-php php83-php-fpm php83-php-cli php83-php-common \
php83-php-mysqlnd php83-php-pgsql php83-php-sqlite3 php83-php-xml \
php83-php-curl php83-php-gd php83-php-imagick php83-php-opcache \
php83-php-soap php83-php-zip php83-php-intl php83-php-bcmath \
php83-php-json php83-php-mbstring php83-php-pdo php83-php-fileinfo \
php83-php-tokenizer php83-php-xmlwriter php83-php-simplexml \
php83-php-dom php83-php-session php83-php-ctype php83-php-filter \
php83-php-hash php83-php-openssl php83-php-pcre php83-php-redis \
php83-php-memcached php83-php-iconv

# Crear enlaces simbólicos
sudo ln -sf /opt/remi/php83/root/usr/bin/php /usr/bin/php83
sudo ln -sf /opt/remi/php83/root/usr/bin/php-fpm /usr/bin/php83-fpm
```

## Paso 3: Buscar e instalar módulos adicionales de PHP

### Buscar módulos disponibles

```bash
# CentOS 7 - Buscar módulos PHP 8.3 disponibles
yum search php83-php-

# Rocky Linux - Buscar módulos PHP 8.3 disponibles
dnf search php83-php-

# Ver todos los módulos instalados para PHP 8.3
php83 -m

# Ver módulos instalados con detalles
php83 -m | sort
```

### Instalar módulos específicos

#### ImageMagick (Imagick)
```bash
# Para ambos sistemas
sudo yum install -y php83-php-imagick ImageMagick ImageMagick-devel (CentOS 7)
sudo dnf install -y php83-php-imagick ImageMagick ImageMagick-devel (Rocky Linux)

# Verificar instalación
php83 -m | grep imagick
```

#### Otros módulos útiles para desarrollo

```bash
# Módulos para desarrollo y depuración
sudo yum install -y php83-php-xdebug php83-php-devel (CentOS 7)
sudo dnf install -y php83-php-xdebug php83-php-devel (Rocky Linux)

# Módulos para diferentes bases de datos
sudo yum install -y php83-php-odbc php83-php-mongodb (CentOS 7)
sudo dnf install -y php83-php-odbc php83-php-mongodb (Rocky Linux)

# Módulos para procesamiento de imágenes adicionales
sudo yum install -y php83-php-gmp (CentOS 7)
sudo dnf install -y php83-php-gmp (Rocky Linux)

# Módulos para caching
sudo yum install -y php83-php-apcu (CentOS 7)
sudo dnf install -y php83-php-apcu (Rocky Linux)
```

### Módulos esenciales para Laravel (verificación)

Laravel requiere las siguientes extensiones PHP. Verifica que estén instaladas:

```bash
# Verificar módulos requeridos por Laravel
php83 -m | grep -E "(openssl|pdo|mbstring|tokenizer|xml|ctype|json|bcmath|fileinfo)"

# Si falta algún módulo, instálalo:
# sudo yum install -y php83-php-[nombre-modulo] (CentOS 7)
# sudo dnf install -y php83-php-[nombre-modulo] (Rocky Linux)
```

#### Lista completa de módulos recomendados para Laravel:

| Módulo | Paquete | Propósito |
|--------|---------|-----------|
| OpenSSL | php83-php-openssl | Encriptación y seguridad |
| PDO | php83-php-pdo | Acceso a base de datos |
| Mbstring | php83-php-mbstring | Manejo de strings multibyte |
| Tokenizer | php83-php-tokenizer | Análisis de tokens PHP |
| XML | php83-php-xml | Procesamiento XML |
| Ctype | php83-php-ctype | Validación de caracteres |
| JSON | php83-php-json | Manejo de JSON |
| BCMath | php83-php-bcmath | Matemáticas de precisión arbitraria |
| Fileinfo | php83-php-fileinfo | Información de archivos |
| GD | php83-php-gd | Manipulación de imágenes |
| Curl | php83-php-curl | Cliente HTTP |
| Zip | php83-php-zip | Compresión de archivos |
| Intl | php83-php-intl | Internacionalización |
| Redis | php83-php-redis | Cache y sesiones |
| Imagick | php83-php-imagick | Procesamiento avanzado de imágenes |

### Instalar todos los módulos recomendados para Laravel de una vez:

```bash
# CentOS 7
sudo yum install -y php83-php-openssl php83-php-pdo php83-php-mbstring \
php83-php-tokenizer php83-php-xml php83-php-ctype php83-php-json \
php83-php-bcmath php83-php-fileinfo php83-php-gd php83-php-curl \
php83-php-zip php83-php-intl php83-php-redis php83-php-imagick \
php83-php-dom php83-php-xmlwriter php83-php-simplexml php83-php-session \
php83-php-filter php83-php-hash php83-php-pcre php83-php-iconv

# Rocky Linux
sudo dnf install -y php83-php-openssl php83-php-pdo php83-php-mbstring \
php83-php-tokenizer php83-php-xml php83-php-ctype php83-php-json \
php83-php-bcmath php83-php-fileinfo php83-php-gd php83-php-curl \
php83-php-zip php83-php-intl php83-php-redis php83-php-imagick \
php83-php-dom php83-php-xmlwriter php83-php-simplexml php83-php-session \
php83-php-filter php83-php-hash php83-php-pcre php83-php-iconv
```

## Paso 4: Verificar instalaciones y módulos

```bash
# Verificar PHP 8.1 (versión actual)
php --version

# Verificar PHP 8.3 (nueva instalación)
php83 --version

# Verificar ubicaciones de PHP-FPM
which php-fpm     # PHP 8.1
which php83-fpm   # PHP 8.3
```

## Paso 5: Configurar PHP-FPM para ambas versiones

### Configurar PHP 8.3 FPM

```bash
# Copiar configuración de PHP-FPM para PHP 8.3
sudo cp /etc/php-fpm.d/www.conf /opt/remi/php83/root/etc/php-fpm.d/www.conf

# Editar configuración de PHP 8.3 FPM
sudo nano /opt/remi/php83/root/etc/php-fpm.d/www.conf
```

Modificar las siguientes líneas en el archivo de configuración:

```ini
[www83]
user = apache
group = apache
listen = /var/run/php83-fpm.sock
listen.owner = apache
listen.group = apache
listen.mode = 0660
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

### Crear archivo de servicio para PHP 8.3 FPM

```bash
# Crear archivo de servicio systemd para PHP 8.3
sudo nano /etc/systemd/system/php83-fpm.service
```

Contenido del archivo:

```ini
[Unit]
Description=The PHP 8.3 FastCGI Process Manager
After=network.target

[Service]
Type=notify
PIDFile=/opt/remi/php83/root/var/run/php-fpm/php-fpm.pid
ExecStart=/opt/remi/php83/root/usr/sbin/php-fpm --nodaemonize
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=true
RuntimeDirectory=php83-fpm
RuntimeDirectoryMode=755

[Install]
WantedBy=multi-user.target
```

## Paso 6: Iniciar y habilitar servicios PHP-FPM

```bash
# Recargar systemd
sudo systemctl daemon-reload

# Iniciar y habilitar PHP 8.1 FPM (si no estaba activo)
sudo systemctl start php-fpm
sudo systemctl enable php-fpm

# Iniciar y habilitar PHP 8.3 FPM
sudo systemctl start php83-fpm
sudo systemctl enable php83-fpm

# Verificar estado
sudo systemctl status php-fpm
sudo systemctl status php83-fpm
```

## Paso 7: Configurar Apache para múltiples versiones

### Habilitar módulo proxy_fcgi en Apache

```bash
# Para CentOS 7
sudo yum install -y httpd-devel
echo "LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so" | sudo tee -a /etc/httpd/conf.modules.d/00-proxy.conf

# Para Rocky Linux
sudo dnf install -y httpd-devel
echo "LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so" | sudo tee -a /etc/httpd/conf.modules.d/00-proxy.conf
```

### Configuración global de PHP en Apache

Editar configuración principal de Apache:

```bash
sudo nano /etc/httpd/conf.d/php.conf
```

Agregar o modificar:

```apache
# Configuración PHP 8.1 por defecto
<FilesMatch \.php$>
    SetHandler "proxy:unix:/var/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>

# Directorio de prueba para PHP 8.3
<Directory "/var/www/html/php83-modules">
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
    </FilesMatch>
</Directory>
```

## Paso 8: Configuración por Virtual Host

### Opción A: Sitio completo con PHP 8.3

```bash
sudo nano /etc/httpd/conf.d/sitio-php83.conf
```

Contenido:

```apache
<VirtualHost *:80>
    ServerName sitio-php83.local
    DocumentRoot /var/www/sitio-php83
    
    # Usar PHP 8.3 para todo el sitio
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
    </FilesMatch>
    
    <Directory "/var/www/sitio-php83">
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog /var/log/httpd/sitio-php83_error.log
    CustomLog /var/log/httpd/sitio-php83_access.log combined
</VirtualHost>
```

### Opción B: Directorios específicos con PHP 8.3

Editar el virtual host principal:

```bash
sudo nano /etc/httpd/conf.d/000-default.conf
```

O crear configuración específica:

```apache
# En el VirtualHost principal
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html
    
    # PHP 8.1 por defecto (configuración global)
    
    # Módulos específicos con PHP 8.3
    <Directory "/var/www/html/modulo-especial">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
        </FilesMatch>
    </Directory>
    
    <Directory "/var/www/html/app-nueva">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
        </FilesMatch>
    </Directory>
</VirtualHost>
```

## Paso 9: Ajustar configuraciones de firewall y SELinux

### Configurar SELinux (si está habilitado)

```bash
# Verificar estado de SELinux
sestatus

# Si SELinux está activo, permitir conexiones de red para httpd
sudo setsebool -P httpd_can_network_connect 1

# Permitir que httpd se conecte a los sockets de PHP-FPM
sudo setsebool -P httpd_execmem 1
```

### Configurar firewall (si es necesario)

```bash
# Para CentOS 7 (firewalld)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Para iptables (si se usa en lugar de firewalld)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

## Paso 10: Crear archivos de prueba

```bash
# Crear directorio para pruebas
sudo mkdir -p /var/www/html/php83-test
sudo mkdir -p /var/www/html/modulo-especial

# Archivo de prueba PHP 8.1
echo "<?php echo 'PHP Version: ' . phpversion(); phpinfo(); ?>" | sudo tee /var/www/html/info81.php

# Archivo de prueba PHP 8.3
echo "<?php echo 'PHP Version: ' . phpversion(); phpinfo(); ?>" | sudo tee /var/www/html/php83-test/info83.php

# Asignar permisos correctos
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/
```

## Paso 11: Reiniciar servicios

```bash
# Reiniciar todos los servicios
sudo systemctl restart php-fpm
sudo systemctl restart php83-fpm
sudo systemctl restart httpd

# Verificar que todo esté funcionando
sudo systemctl status php-fpm php83-fpm httpd
```

## Comandos de gestión específicos para CentOS/Rocky

### Gestión de paquetes y módulos

```bash
# CentOS 7 - Ver paquetes PHP instalados
yum list installed | grep php

# Rocky Linux - Ver paquetes PHP instalados
dnf list installed | grep php

# Ver información detallada de un módulo específico
yum info php83-php-imagick (CentOS 7)
dnf info php83-php-imagick (Rocky Linux)

# Buscar módulos específicos
yum search imagick | grep php83 (CentOS 7)
dnf search imagick | grep php83 (Rocky Linux)

# Actualizar PHP 8.1
sudo yum update php\* (CentOS 7)
sudo dnf update php\* (Rocky Linux)

# Actualizar PHP 8.3
sudo yum update php83\* (CentOS 7)
sudo dnf update php83\* (Rocky Linux)
```

### Script para verificar módulos de Laravel

Crear un script para verificar que todos los módulos requeridos estén instalados:

```bash
# Crear script de verificación
sudo nano /usr/local/bin/check-laravel-modules.sh
```

Contenido del script:

```bash
#!/bin/bash

echo "=== Verificación de módulos PHP para Laravel ==="
echo "Verificando PHP 8.1:"
echo "================================"

required_modules=("openssl" "pdo" "mbstring" "tokenizer" "xml" "ctype" "json" "bcmath" "fileinfo" "curl" "zip" "intl" "gd")
optional_modules=("redis" "imagick" "memcached" "opcache")

echo "Módulos requeridos:"
for module in "${required_modules[@]}"; do
    if php -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "○ $module - No instalado"
    fi
done

echo ""
echo "Verificando PHP 8.3:"
echo "================================"

echo "Módulos requeridos:"
for module in "${required_modules[@]}"; do
    if php83 -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php83 -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "○ $module - No instalado"
    fi
done

echo ""
echo "=== Versiones de PHP ==="
echo "PHP 8.1: $(php --version | head -1)"
echo "PHP 8.3: $(php83 --version | head -1)"
```

Hacer ejecutable el script:

```bash
sudo chmod +x /usr/local/bin/check-laravel-modules.sh

# Ejecutar verificación
sudo /usr/local/bin/check-laravel-modules.sh
```

### Instalación manual de módulos via PECL (si no están disponibles en repositorios)

```bash
# Instalar PECL para PHP 8.3
sudo yum install -y php83-php-pear php83-php-devel gcc (CentOS 7)
sudo dnf install -y php83-php-pear php83-php-devel gcc (Rocky Linux)

# Usar PECL de PHP 8.3 para instalar módulos
/opt/remi/php83/root/usr/bin/pecl install redis
/opt/remi/php83/root/usr/bin/pecl install imagick

# Habilitar módulos en php.ini
echo "extension=redis.so" | sudo tee -a /opt/remi/php83/root/etc/php.ini
echo "extension=imagick.so" | sudo tee -a /opt/remi/php83/root/etc/php.ini

# Reiniciar PHP-FPM después de cambios
sudo systemctl restart php83-fpm
```

### Ubicaciones importantes

```bash
# Configuraciones PHP
/etc/php.ini                                    # PHP 8.1
/opt/remi/php83/root/etc/php.ini               # PHP 8.3

# Configuraciones PHP-FPM
/etc/php-fpm.d/                                 # PHP 8.1
/opt/remi/php83/root/etc/php-fpm.d/            # PHP 8.3

# Logs
/var/log/php-fpm/                               # PHP 8.1 FPM
/opt/remi/php83/root/var/log/php-fpm/          # PHP 8.3 FPM
/var/log/httpd/                                 # Apache
```

### Troubleshooting específico

```bash
# Ver logs en tiempo real
sudo tail -f /var/log/httpd/error_log
sudo tail -f /var/log/php-fpm/error.log
sudo tail -f /opt/remi/php83/root/var/log/php-fpm/error.log

# Verificar sockets
ls -la /var/run/php-fpm/
ls -la /var/run/php83-fpm.sock

# Probar configuración de Apache
sudo httpd -t

# Ver módulos cargados en Apache
httpd -M | grep proxy
```

## Diferencias entre CentOS 7 y Rocky Linux

### CentOS 7:
- Usa `yum` como gestor de paquetes
- Versiones de paquetes más antiguas
- Soporte hasta junio 2024

### Rocky Linux:
- Usa `dnf` como gestor de paquetes
- Sistema de módulos más avanzado
- Soporte a largo plazo activo

## Notas importantes

1. **Repositorio Remi**: Es la fuente más confiable para múltiples versiones de PHP en RHEL/CentOS
2. **Rutas específicas**: PHP 8.3 se instala en `/opt/remi/php83/` para evitar conflictos
3. **Servicios separados**: Cada versión de PHP-FPM ejecuta como servicio independiente
4. **SELinux**: Puede requerir configuración adicional según las políticas de seguridad
5. **Actualizaciones**: Mantén ambas versiones actualizadas regularmente

Esta configuración te permite usar PHP 8.1 como predeterminado mientras ejecutas PHP 8.3 en módulos o sitios específicos de manera segura y eficiente.