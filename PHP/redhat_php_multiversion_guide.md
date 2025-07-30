# Guía: Instalación de Múltiples Versiones de PHP en Red Hat Enterprise Linux 7.9

## Objetivo
Instalar PHP 8.3 junto a PHP 8.1 existente, manteniendo 8.1 como versión por defecto y configurando Apache para usar PHP 8.3 en módulos específicos.

## Prerrequisitos
- Red Hat Enterprise Linux 7.9
- Suscripción activa de Red Hat o acceso a repositorios
- Apache (httpd) instalado y funcionando
- PHP 8.1 ya instalado y configurado
- Permisos de administrador (sudo/root)

## Paso 1: Habilitar repositorios necesarios

### Habilitar repositorios de Red Hat

```bash
# Verificar suscripción activa
sudo subscription-manager status

# Habilitar repositorios necesarios de Red Hat
sudo subscription-manager repos --enable=rhel-7-server-rpms
sudo subscription-manager repos --enable=rhel-7-server-extras-rpms
sudo subscription-manager repos --enable=rhel-7-server-optional-rpms

# Instalar EPEL (Extra Packages for Enterprise Linux)
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# Verificar que EPEL esté habilitado
sudo yum repolist | grep epel
```

### Instalar repositorio Remi para múltiples versiones de PHP

```bash
# Descargar e instalar el repositorio Remi
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm

# Verificar que Remi esté instalado
sudo yum repolist all | grep remi

# Habilitar repositorio Remi para PHP 8.3
sudo yum-config-manager --enable remi-php83

# Verificar repositorios habilitados
sudo yum repolist enabled | grep remi
```

## Paso 2: Instalar PHP 8.3 con módulos completos para Laravel

```bash
# Instalar PHP 8.3 y módulos esenciales para Laravel
sudo yum install -y php83-php php83-php-fpm php83-php-cli php83-php-common \
php83-php-mysqlnd php83-php-pgsql php83-php-xml php83-php-curl \
php83-php-gd php83-php-imagick php83-php-opcache php83-php-soap \
php83-php-zip php83-php-intl php83-php-bcmath php83-php-json \
php83-php-mbstring php83-php-pdo php83-php-tokenizer php83-php-fileinfo \
php83-php-dom php83-php-xmlwriter php83-php-simplexml php83-php-session \
php83-php-ctype php83-php-filter php83-php-hash php83-php-openssl \
php83-php-pcre php83-php-redis php83-php-memcached php83-php-iconv \
php83-php-process

# Crear enlaces simbólicos para facilitar el uso
sudo ln -sf /opt/remi/php83/root/usr/bin/php /usr/bin/php83
sudo ln -sf /opt/remi/php83/root/usr/bin/php-fpm /usr/bin/php83-fpm
sudo ln -sf /opt/remi/php83/root/usr/bin/php-config /usr/bin/php83-config
sudo ln -sf /opt/remi/php83/root/usr/bin/phpize /usr/bin/phpize83
```

## Paso 3: Buscar e instalar módulos adicionales de PHP

### Buscar módulos disponibles

```bash
# Buscar módulos PHP 8.3 disponibles en Remi
yum search php83-php- | grep -v "Loaded plugins"

# Ver todos los módulos instalados para PHP 8.3
php83 -m

# Ver módulos instalados con detalles
php83 -m | sort

# Comparar con PHP 8.1
echo "=== PHP 8.1 módulos ===" && php -m | sort
echo "=== PHP 8.3 módulos ===" && php83 -m | sort
```

### Instalar módulos específicos

#### ImageMagick (Imagick)
```bash
# Instalar ImageMagick y el módulo PHP
sudo yum install -y php83-php-imagick ImageMagick ImageMagick-devel

# Si no está disponible en repositorio, instalarlo desde EPEL
sudo yum install -y ImageMagick ImageMagick-devel ImageMagick-perl

# Verificar instalación
php83 -m | grep imagick
convert --version
```

#### Otros módulos útiles para desarrollo

```bash
# Módulos para desarrollo y depuración
sudo yum install -y php83-php-xdebug php83-php-devel

# Módulos para diferentes bases de datos
sudo yum install -y php83-php-odbc php83-php-pdo-dblib php83-php-interbase

# Módulos para procesamiento adicional
sudo yum install -y php83-php-gmp php83-php-ldap php83-php-snmp

# Módulos para caching y rendimiento
sudo yum install -y php83-php-apcu php83-php-memcache

# Módulos para procesamiento de texto y datos
sudo yum install -y php83-php-tidy php83-php-enchant
```

### Módulos esenciales para Laravel (verificación)

Laravel requiere las siguientes extensiones PHP. Verifica que estén instaladas:

```bash
# Verificar módulos requeridos por Laravel
php83 -m | grep -E "(openssl|pdo|mbstring|tokenizer|xml|ctype|json|bcmath|fileinfo)"

# Si falta algún módulo, instálalo:
# sudo yum install -y php83-php-[nombre-modulo]
```

#### Lista completa de módulos recomendados para Laravel:

| Módulo | Paquete RHEL | Propósito |
|--------|-------------|-----------|
| OpenSSL | php83-php-openssl | Encriptación y seguridad |
| PDO | php83-php-pdo | Capa de abstracción de base de datos |
| MySQL | php83-php-mysqlnd | Conectividad MySQL mejorada |
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
| Session | php83-php-session | Gestión de sesiones |
| Filter | php83-php-filter | Filtrado de datos |
| Hash | php83-php-hash | Funciones de hash |

### Instalar todos los módulos recomendados para Laravel de una vez:

```bash
sudo yum install -y php83-php-openssl php83-php-pdo php83-php-mysqlnd \
php83-php-mbstring php83-php-tokenizer php83-php-xml php83-php-ctype \
php83-php-json php83-php-bcmath php83-php-fileinfo php83-php-gd \
php83-php-curl php83-php-zip php83-php-intl php83-php-redis \
php83-php-imagick php83-php-dom php83-php-xmlwriter php83-php-simplexml \
php83-php-session php83-php-filter php83-php-hash php83-php-pcre \
php83-php-iconv php83-php-process
```

## Paso 4: Verificar instalaciones y módulos

```bash
# Verificar PHP 8.1 (versión actual)
php --version

# Verificar PHP 8.3 (nueva instalación)
php83 --version

# Ver módulos cargados en ambas versiones
echo "=== Módulos PHP 8.1 ==="
php -m | sort

echo "=== Módulos PHP 8.3 ==="
php83 -m | sort

# Verificar ubicaciones de PHP-FPM
which php-fpm     # PHP 8.1
which php83-fpm   # PHP 8.3

# Ver información detallada de la instalación
php83 --ini
```

## Paso 5: Configurar PHP-FPM para ambas versiones

### Configurar PHP 8.3 FPM

```bash
# Verificar directorio de configuración de PHP 8.3
ls -la /opt/remi/php83/root/etc/php-fpm.d/

# Crear configuración personalizada para PHP 8.3 FPM
sudo cp /opt/remi/php83/root/etc/php-fpm.d/www.conf /opt/remi/php83/root/etc/php-fpm.d/www83.conf

# Editar configuración de PHP 8.3 FPM
sudo nano /opt/remi/php83/root/etc/php-fpm.d/www83.conf
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
pm.max_requests = 500

; Configuraciones adicionales para Red Hat
php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f www@my.domain.com
php_flag[display_errors] = off
php_admin_value[error_log] = /var/log/php83-fpm-www.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 256M
```

### Deshabilitar pool por defecto de PHP 8.3

```bash
# Renombrar el archivo de configuración por defecto para deshabilitarlo
sudo mv /opt/remi/php83/root/etc/php-fpm.d/www.conf /opt/remi/php83/root/etc/php-fpm.d/www.conf.disabled
```

### Crear archivo de servicio systemd para PHP 8.3 FPM

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
User=root
Group=root

# Configuraciones específicas para Red Hat
RestartSec=10
Restart=always

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

# Verificar estado de ambos servicios
sudo systemctl status php-fpm
sudo systemctl status php83-fpm

# Verificar que los sockets se hayan creado
ls -la /var/run/php*fpm*
```

## Paso 7: Configurar Apache para múltiples versiones

### Habilitar módulo proxy_fcgi en Apache

```bash
# Verificar si mod_proxy_fcgi está disponible
sudo httpd -M | grep proxy

# Si no está cargado, habilitarlo
echo "LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so" | sudo tee -a /etc/httpd/conf.modules.d/00-proxy.conf

# Verificar módulos proxy necesarios
sudo httpd -M | grep proxy
```

### Configuración global de PHP en Apache

Crear archivo de configuración específico para PHP:

```bash
sudo nano /etc/httpd/conf.d/php-multiversion.conf
```

Contenido del archivo:

```apache
# Configuración PHP 8.1 por defecto
<FilesMatch \.php$>
    SetHandler "proxy:unix:/var/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>

# Configuración específica para directorios con PHP 8.3
<Directory "/var/www/html/php83-modules">
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
    </FilesMatch>
</Directory>

# Configuración de seguridad
<Files ".user.ini">
    Require all denied
</Files>

# Prevenir acceso a archivos de configuración PHP
<Files "php.ini">
    Require all denied
</Files>
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
        Options -Indexes +FollowSymLinks
        
        # Configuraciones específicas para Laravel
        <IfModule mod_rewrite.c>
            RewriteEngine On
            RewriteRule ^(.*)$ public/$1 [L]
        </IfModule>
    </Directory>
    
    # Configuración de logs específica para Red Hat
    ErrorLog /var/log/httpd/sitio-php83_error.log
    CustomLog /var/log/httpd/sitio-php83_access.log combined
    LogLevel info
    
    # Configuraciones de seguridad adicionales
    ServerTokens Prod
    ServerSignature Off
</VirtualHost>
```

### Opción B: Directorios específicos con PHP 8.3

Editar el virtual host principal o crear configuración específica:

```bash
sudo nano /etc/httpd/conf.d/000-default.conf
```

Contenido:

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html
    
    # PHP 8.1 por defecto (configuración global)
    
    # Módulos específicos con PHP 8.3
    <Directory "/var/www/html/modulo-especial">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
        </FilesMatch>
        AllowOverride All
        Require all granted
    </Directory>
    
    <Directory "/var/www/html/app-laravel">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php83-fpm.sock|fcgi://localhost"
        </FilesMatch>
        AllowOverride All
        Require all granted
        
        # Configuración específica para Laravel
        <IfModule mod_rewrite.c>
            RewriteEngine On
            RewriteCond %{REQUEST_FILENAME} !-d
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteRule ^(.*)$ index.php [QSA,L]
        </IfModule>
    </Directory>
    
    ErrorLog /var/log/httpd/default_error.log
    CustomLog /var/log/httpd/default_access.log combined
</VirtualHost>
```

## Paso 9: Ajustar configuraciones de firewall y SELinux

### Configurar SELinux para PHP-FPM

```bash
# Verificar estado de SELinux
getenforce

# Si SELinux está activo, configurar políticas necesarias
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_execmem 1

# Permitir que httpd se conecte a los sockets de PHP-FPM
sudo semanage fcontext -a -t httpd_exec_t "/opt/remi/php83/root/usr/sbin/php-fpm"
sudo restorecon -Rv /opt/remi/php83/

# Configurar contexto SELinux para sockets
sudo semanage fcontext -a -t httpd_var_run_t "/var/run/php83-fpm.sock"
sudo restorecon -v /var/run/php83-fpm.sock

# Si hay problemas con SELinux, revisar logs
sudo ausearch -m avc -ts recent | grep php
```

### Configurar firewall

```bash
# Para firewalld (Red Hat 7 por defecto)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Verificar configuración del firewall
sudo firewall-cmd --list-all

# Para iptables (si se usa en lugar de firewalld)
# sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# sudo service iptables save
```

## Paso 10: Crear archivos de prueba

```bash
# Crear directorios para pruebas
sudo mkdir -p /var/www/html/php83-test
sudo mkdir -p /var/www/html/modulo-especial
sudo mkdir -p /var/www/sitio-php83

# Archivo de prueba PHP 8.1
echo "<?php echo '<h1>PHP Version: ' . phpversion() . '</h1>'; phpinfo(); ?>" | sudo tee /var/www/html/info81.php

# Archivo de prueba PHP 8.3
echo "<?php echo '<h1>PHP Version: ' . phpversion() . '</h1>'; phpinfo(); ?>" | sudo tee /var/www/html/php83-test/info83.php

# Archivo de prueba para módulos de Laravel
cat << 'EOF' | sudo tee /var/www/html/php83-test/laravel-check.php
<?php
echo "<h1>Verificación de módulos Laravel para PHP " . phpversion() . "</h1>";

$required_modules = [
    'openssl', 'pdo', 'mbstring', 'tokenizer', 'xml', 'ctype', 
    'json', 'bcmath', 'fileinfo', 'curl', 'zip', 'intl', 'gd'
];

$optional_modules = ['redis', 'imagick', 'memcached', 'opcache'];

echo "<h2>Módulos Requeridos:</h2><ul>";
foreach ($required_modules as $module) {
    $loaded = extension_loaded($module);
    $status = $loaded ? '✓' : '✗';
    $color = $loaded ? 'green' : 'red';
    echo "<li style='color: $color'>$status $module</li>";
}
echo "</ul>";

echo "<h2>Módulos Opcionales:</h2><ul>";
foreach ($optional_modules as $module) {
    $loaded = extension_loaded($module);
    $status = $loaded ? '✓' : '○';
    $color = $loaded ? 'green' : 'orange';
    echo "<li style='color: $color'>$status $module</li>";
}
echo "</ul>";
?>
EOF

# Asignar permisos correctos
sudo chown -R apache:apache /var/www/html/
sudo chown -R apache:apache /var/www/sitio-php83/
sudo chmod -R 755 /var/www/html/
sudo chmod -R 755 /var/www/sitio-php83/

# Configurar contexto SELinux para archivos web
sudo restorecon -Rv /var/www/html/
sudo restorecon -Rv /var/www/sitio-php83/
```

## Paso 11: Reiniciar servicios

```bash
# Reiniciar todos los servicios
sudo systemctl restart php-fpm
sudo systemctl restart php83-fpm
sudo systemctl restart httpd

# Verificar que todo esté funcionando
sudo systemctl status php-fpm php83-fpm httpd

# Verificar puertos en escucha
sudo netstat -tulpn | grep httpd
sudo ss -tulpn | grep php-fpm
```

## Comandos de gestión específicos para Red Hat 7.9

### Gestión de paquetes y módulos

```bash
# Ver paquetes PHP instalados
yum list installed | grep php

# Ver información detallada de un módulo específico
yum info php83-php-imagick

# Buscar módulos específicos
yum search imagick | grep php83

# Ver historial de transacciones de yum
yum history

# Actualizar PHP 8.1
sudo yum update php\*

# Actualizar PHP 8.3
sudo yum update php83\*

# Ver dependencias de un paquete
yum deplist php83-php-imagick

# Limpiar caché de yum
sudo yum clean all
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

echo "=== Verificación de módulos PHP para Laravel en Red Hat 7.9 ==="
echo "Verificando PHP 8.1:"
echo "================================"

required_modules=("openssl" "pdo" "mbstring" "tokenizer" "xml" "ctype" "json" "bcmath" "fileinfo" "curl" "zip" "intl" "gd")
optional_modules=("redis" "imagick" "memcached" "opcache")

echo "Módulos requeridos:"
for module in "${required_modules[@]}"; do
    if php -m 2>/dev/null | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php -m 2>/dev/null | grep -q "^$module$"; then
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
    if php83 -m 2>/dev/null | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php83 -m 2>/dev/null | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "○ $module - No instalado"
    fi
done

echo ""
echo "=== Información del sistema ==="
echo "Versión Red Hat: $(cat /etc/redhat-release)"
echo "PHP 8.1: $(php --version 2>/dev/null | head -1 || echo 'No instalado')"
echo "PHP 8.3: $(php83 --version 2>/dev/null | head -1 || echo 'No instalado')"
echo "SELinux: $(getenforce)"
echo "Apache: $(httpd -v | head -1)"

echo ""
echo "=== Estado de servicios ==="
systemctl is-active php-fpm && echo "✓ PHP 8.1 FPM - Activo" || echo "✗ PHP 8.1 FPM - Inactivo"
systemctl is-active php83-fpm && echo "✓ PHP 8.3 FPM - Activo" || echo "✗ PHP 8.3 FPM - Inactivo"
systemctl is-active httpd && echo "✓ Apache - Activo" || echo "✗ Apache - Inactivo"
```

Hacer ejecutable el script:

```bash
sudo chmod +x /usr/local/bin/check-laravel-modules.sh

# Ejecutar verificación
sudo /usr/local/bin/check-laravel-modules.sh
```

### Instalación manual de módulos via PECL (si no están disponibles en repositorios)

```bash
# Instalar herramientas de desarrollo para PHP 8.3
sudo yum install -y php83-php-pear php83-php-devel gcc gcc-c++ make

# Usar PECL de PHP 8.3 para instalar módulos
/opt/remi/php83/root/usr/bin/pecl install redis
/opt/remi/php83/root/usr/bin/pecl install imagick

# Habilitar módulos en php.ini
echo "extension=redis.so" | sudo tee -a /opt/remi/php83/root/etc/php.ini
echo "extension=imagick.so" | sudo tee -a /opt/remi/php83/root/etc/php.ini

# Reiniciar PHP-FPM después de cambios
sudo systemctl restart php83-fpm

# Verificar que los módulos se cargaron correctamente
php83 -m | grep -E "(redis|imagick)"
```

### Configuración específica de Imagick para Red Hat

```bash
# Verificar que ImageMagick esté instalado en el sistema
convert --version

# Si no está instalado, instalarlo desde EPEL
sudo yum install -y ImageMagick ImageMagick-devel ImageMagick-perl

# Verificar configuración de Imagick en PHP
php83 -m | grep imagick
php83 --ri imagick

# Configurar políticas de ImageMagick si es necesario
sudo nano /etc/ImageMagick/policy.xml

# Verificar permisos SELinux para ImageMagick
ls -laZ /usr/bin/convert
```

### Ubicaciones importantes en Red Hat 7.9

```bash
# Configuraciones PHP
/etc/php.ini                                    # PHP 8.1
/opt/remi/php83/root/etc/php.ini               # PHP 8.3

# Configuraciones PHP-FPM
/etc/php-fpm.d/                                 # PHP 8.1
/opt/remi/php83/root/etc/php-fpm.d/            # PHP 8.3

# Configuraciones Apache
/etc/httpd/conf/httpd.conf                      # Configuración principal
/etc/httpd/conf.d/                              # Configuraciones adicionales
/etc/httpd/conf.modules.d/                      # Módulos

# Logs
/var/log/php-fpm/                               # PHP 8.1 FPM
/opt/remi/php83/root/var/log/php-fpm/          # PHP 8.3 FPM
/var/log/httpd/                                 # Apache
/var/log/messages                               # Sistema general
/var/log/secure                                 # Seguridad y SELinux

# Archivos de servicio
/etc/systemd/system/php83-fpm.service          # Servicio PHP 8.3
/usr/lib/systemd/system/php-fpm.service        # Servicio PHP 8.1
/usr/lib/systemd/system/httpd.service          # Servicio Apache
```

### Troubleshooting específico para Red Hat 7.9

```bash
# Ver logs en tiempo real
sudo tail -f /var/log/httpd/error_log
sudo tail -f /var/log/php-fpm/error.log
sudo tail -f /opt/remi/php83/root/var/log/php-fpm/error.log
sudo tail -f /var/log/messages

# Verificar sockets y conexiones
ls -la /var/run/php*fpm*
sudo netstat -tulpn | grep php-fpm
sudo ss -tulpn | grep php-fpm

# Probar configuración de Apache
sudo httpd -t
sudo httpd -S

# Ver módulos cargados en Apache
httpd -M | grep proxy
httpd -M | grep rewrite

# Verificar configuración PHP-FPM
sudo php-fpm -t
sudo /opt/remi/php83/root/usr/sbin/php-fpm -t

# Diagnosticar problemas de SELinux
sudo ausearch -m avc -ts recent
sudo sealert -a /var/log/audit/audit.log

# Ver configuración actual de PHP
php --ini
php83 --ini

# Verificar permisos de archivos
ls -laZ /var/www/html/
ls -laZ /var/run/php*fpm*
```

## Solución de problemas comunes en Red Hat 7.9

### Error: "502 Bad Gateway"
```bash
# Verificar que PHP-FPM esté ejecutándose
sudo systemctl status php83-fpm

# Verificar permisos del socket
ls -la /var/run/php83-fpm.sock

# Verificar configuración SELinux
sudo getsebool httpd_can_network_connect
sudo setsebool -P httpd_can_network_connect 1
```

### Error: "Permission denied" en logs de Apache
```bash
# Verificar contexto SELinux de archivos web
ls -laZ /var/www/html/

# Restaurar contexto SELinux correcto
sudo restorecon -Rv /var/www/html/
sudo restorecon -Rv /var/run/php83-fpm.sock

# Verificar permisos de usuario apache
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### Error: "File not found" o "Primary script unknown"
```bash
# Verificar configuración de DocumentRoot
grep DocumentRoot /etc/httpd/conf.d/*.conf

# Verificar que los archivos PHP existan
ls -la /var/www/html/info81.php
ls -la /var/www/html/php83-test/info83.php

# Verificar configuración de FastCGI
sudo httpd -D DUMP_VHOSTS
```

### Problemas con módulos específicos

```bash
# Si Redis no funciona correctamente
sudo yum reinstall php83-php-redis
sudo systemctl restart php83-fpm
php83 -m | grep redis

# Si Imagick presenta problemas
sudo yum reinstall php83-php-imagick ImageMagick
convert --version
php83 --ri imagick

# Verificar dependencias faltantes
ldd /opt/remi/php83/root/usr/lib64/php/modules/imagick.so
```

### Problemas de rendimiento

```bash
# Monitorear procesos PHP-FPM
sudo ps aux | grep php-fpm

# Verificar uso de memoria
free -h
ps aux --sort=-%mem | head

# Ajustar configuración de PHP-FPM si es necesario
sudo nano /opt/remi/php83/root/etc/php-fpm.d/www83.conf
# Modificar pm.max_children, pm.start_servers según recursos
```

## Integración con Composer para Red Hat 7.9

```bash
# Descargar e instalar Composer
cd /tmp
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
sudo chmod +x /usr/local/bin/composer

# Crear aliases para usar con diferentes versiones de PHP
echo 'alias composer81="php /usr/local/bin/composer"' | sudo tee -a /etc/bashrc
echo 'alias composer83="php83 /usr/local/bin/composer"' | sudo tee -a /etc/bashrc

# Recargar configuración de bash
source /etc/bashrc

# Verificar instalación
composer81 --version
composer83 --version

# Uso en proyectos Laravel
cd /var/www/html/app-laravel
composer83 install
composer83 require laravel/framework
```

## Configuración avanzada para entornos de producción

### Optimización de OPcache

```bash
# Editar configuración de OPcache para PHP 8.3
sudo nano /opt/remi/php83/root/etc/php.d/10-opcache.ini
```

Configuración optimizada para producción:

```ini
; Configuración OPcache optimizada para Red Hat 7.9
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.max_wasted_percentage=10
opcache.use_cwd=1
opcache.validate_timestamps=0
opcache.revalidate_freq=0
opcache.save_comments=1
opcache.enable_file_override=0
opcache.huge_code_pages=1
opcache.file_cache=/tmp/opcache
opcache.file_cache_only=0
```

### Configuración de PHP para Laravel en producción

```bash
# Editar php.ini para PHP 8.3
sudo nano /opt/remi/php83/root/etc/php.ini
```

Configuraciones recomendadas:

```ini
; Configuración general
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
post_max_size = 50M
upload_max_filesize = 50M
max_file_uploads = 20

; Configuración de sesiones
session.gc_maxlifetime = 1440
session.gc_probability = 1
session.gc_divisor = 1000
session.cookie_httponly = 1
session.cookie_secure = 1

; Configuración de errores (producción)
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php83-errors.log

; Configuración de seguridad
expose_php = Off
allow_url_fopen = Off
allow_url_include = Off
```

### Configuración de logrotate para logs PHP

```bash
# Crear configuración de logrotate para PHP 8.3
sudo nano /etc/logrotate.d/php83-fpm
```

Contenido:

```
/var/log/php83-*.log /opt/remi/php83/root/var/log/php-fpm/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 apache apache
    postrotate
        /bin/kill -USR1 `cat /opt/remi/php83/root/var/run/php-fpm/php-fpm.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

## Monitoreo y mantenimiento

### Script de monitoreo de servicios

```bash
# Crear script de monitoreo
sudo nano /usr/local/bin/monitor-php-services.sh
```

Contenido del script:

```bash
#!/bin/bash

LOG_FILE="/var/log/php-services-monitor.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Iniciando monitoreo de servicios PHP" >> $LOG_FILE

# Función para verificar servicio
check_service() {
    local service=$1
    local name=$2
    
    if systemctl is-active --quiet $service; then
        echo "[$DATE] ✓ $name está activo" >> $LOG_FILE
        return 0
    else
        echo "[$DATE] ✗ $name está inactivo - Intentando reiniciar" >> $LOG_FILE
        systemctl start $service
        if systemctl is-active --quiet $service; then
            echo "[$DATE] ✓ $name reiniciado correctamente" >> $LOG_FILE
        else
            echo "[$DATE] ✗ Error al reiniciar $name" >> $LOG_FILE
        fi
        return 1
    fi
}

# Verificar servicios
check_service "php-fpm" "PHP 8.1 FPM"
check_service "php83-fpm" "PHP 8.3 FPM"
check_service "httpd" "Apache"

# Verificar sockets
if [ -S "/var/run/php-fpm/www.sock" ]; then
    echo "[$DATE] ✓ Socket PHP 8.1 existe" >> $LOG_FILE
else
    echo "[$DATE] ✗ Socket PHP 8.1 no existe" >> $LOG_FILE
fi

if [ -S "/var/run/php83-fpm.sock" ]; then
    echo "[$DATE] ✓ Socket PHP 8.3 existe" >> $LOG_FILE
else
    echo "[$DATE] ✗ Socket PHP 8.3 no existe" >> $LOG_FILE
fi

# Verificar uso de memoria
MEMORY_USAGE=$(free | grep Mem | awk '{printf("%.2f", ($3/$2) * 100.0)}')
echo "[$DATE] Uso de memoria: ${MEMORY_USAGE}%" >> $LOG_FILE

if (( $(echo "$MEMORY_USAGE > 90" | bc -l) )); then
    echo "[$DATE] ⚠ Advertencia: Uso de memoria alto (${MEMORY_USAGE}%)" >> $LOG_FILE
fi

echo "[$DATE] Monitoreo completado" >> $LOG_FILE
echo "----------------------------------------" >> $LOG_FILE
```

Hacer ejecutable y programar en crontab:

```bash
sudo chmod +x /usr/local/bin/monitor-php-services.sh

# Agregar a crontab para ejecutar cada 5 minutos
echo "*/5 * * * * /usr/local/bin/monitor-php-services.sh" | sudo crontab -
```

### Backup de configuraciones

```bash
# Crear script de backup de configuraciones
sudo nano /usr/local/bin/backup-php-configs.sh
```

Contenido del script:

```bash
#!/bin/bash

BACKUP_DIR="/backup/php-configs/$(date +%Y-%m-%d)"
sudo mkdir -p $BACKUP_DIR

# Backup de configuraciones PHP
sudo cp -r /etc/php.ini $BACKUP_DIR/
sudo cp -r /etc/php-fpm.d/ $BACKUP_DIR/php-fpm.d-81/
sudo cp -r /opt/remi/php83/root/etc/php.ini $BACKUP_DIR/php83.ini
sudo cp -r /opt/remi/php83/root/etc/php-fpm.d/ $BACKUP_DIR/php-fpm.d-83/

# Backup de configuraciones Apache
sudo cp -r /etc/httpd/conf/ $BACKUP_DIR/httpd-conf/
sudo cp -r /etc/httpd/conf.d/ $BACKUP_DIR/httpd-conf.d/
sudo cp -r /etc/httpd/conf.modules.d/ $BACKUP_DIR/httpd-conf.modules.d/

# Crear archivo de información del sistema
cat << EOF > $BACKUP_DIR/system-info.txt
Fecha: $(date)
Sistema: $(cat /etc/redhat-release)
PHP 8.1: $(php --version | head -1)
PHP 8.3: $(php83 --version | head -1)
Apache: $(httpd -v | head -1)
SELinux: $(getenforce)

Servicios activos:
$(systemctl is-active php-fpm php83-fpm httpd)
EOF

echo "Backup completado en: $BACKUP_DIR"
```

## Diferencias específicas de Red Hat 7.9

### Comparación con CentOS 7 y Rocky Linux

| Aspecto | Red Hat 7.9 | CentOS 7 | Rocky Linux |
|---------|-------------|----------|-------------|
| Repositorios | Requiere suscripción | Gratuitos | Gratuitos |
| Soporte | Comercial hasta 2024 | Comunidad | Comunidad |
| Actualizaciones | Via subscription-manager | Via yum | Via dnf |
| Certificación | Oficialmente certificado | Compatible | Compatible |
| SELinux | Políticas estrictas | Estándar | Estándar |

### Comandos específicos de Red Hat

```bash
# Gestión de suscripciones
subscription-manager status
subscription-manager list --available
subscription-manager attach --pool=POOL_ID

# Verificar entitlements
subscription-manager list --consumed

# Actualizar sistema via Red Hat Network
sudo yum update --security

# Ver información de soporte
redhat-support-tool
```

## Notas importantes específicas para Red Hat 7.9

1. **Suscripción requerida**: Red Hat 7.9 requiere una suscripción válida para acceder a repositorios oficiales

2. **Fin de soporte**: Red Hat 7.9 tiene soporte extendido hasta junio 2024, considera migración a versiones más nuevas

3. **SELinux por defecto**: Red Hat viene con SELinux habilitado y políticas más estrictas

4. **Certificaciones**: Ideal para entornos empresariales que requieren certificación oficial

5. **Repositorio Remi**: Fundamental para múltiples versiones de PHP, ya que los repositorios oficiales tienen limitaciones

6. **Firewalld**: Por defecto en Red Hat 7.9, diferente configuración que iptables tradicional

7. **systemd**: Gestión de servicios moderna, todos los servicios deben configurarse correctamente

8. **Seguridad**: Configuraciones de seguridad más estrictas por defecto

Esta configuración te permite ejecutar múltiples versiones de PHP en Red Hat 7.9 de manera segura y eficiente, con todas las consideraciones específicas del sistema operativo y los módulos necesarios para proyectos Laravel modernos.