# Guía: Instalación de Múltiples Versiones de PHP en Ubuntu/Debian

## Objetivo
Instalar PHP 8.3 junto a PHP 8.1 existente, manteniendo 8.1 como versión por defecto y configurando Apache para usar PHP 8.3 en módulos específicos.

## Prerrequisitos
- Sistema Ubuntu/Debian (adaptable a otras distribuciones)
- Apache2 instalado y funcionando
- PHP 8.1 ya instalado y configurado
- Permisos de administrador (sudo)

## Paso 1: Agregar el repositorio de PHP

```bash
# Actualizar el sistema
sudo apt update

# Instalar software-properties-common si no está instalado
sudo apt install software-properties-common

# Agregar el repositorio de Ondrej Sury (contiene múltiples versiones de PHP)
sudo add-apt-repository ppa:ondrej/php

# Actualizar la lista de paquetes
sudo apt update
```

## Paso 2: Instalar PHP 8.3 con módulos completos para Laravel

```bash
# Instalar PHP 8.3 y módulos esenciales para Laravel
sudo apt install php8.3 php8.3-fpm php8.3-cli php8.3-common php8.3-mysql \
php8.3-pgsql php8.3-sqlite3 php8.3-xml php8.3-xmlrpc php8.3-curl \
php8.3-gd php8.3-imagick php8.3-dev php8.3-imap php8.3-opcache \
php8.3-soap php8.3-zip php8.3-intl php8.3-bcmath php8.3-json \
php8.3-readline php8.3-mbstring php8.3-tokenizer php8.3-fileinfo \
php8.3-dom php8.3-simplexml php8.3-xmlwriter php8.3-session \
php8.3-ctype php8.3-filter php8.3-hash php8.3-openssl \
php8.3-pcre php8.3-redis php8.3-memcached php8.3-iconv

# Verificar que ambas versiones están instaladas
php8.1 --version
php8.3 --version
```

## Paso 3: Buscar e instalar módulos adicionales de PHP

### Buscar módulos disponibles

```bash
# Buscar módulos PHP 8.3 disponibles
apt search php8.3-

# Ver todos los módulos instalados para PHP 8.3
php8.3 -m

# Ver módulos instalados con detalles
php8.3 -m | sort
```

### Instalar módulos específicos

#### ImageMagick (Imagick)
```bash
# Instalar ImageMagick y el módulo PHP
sudo apt install php8.3-imagick imagemagick

# Verificar instalación
php8.3 -m | grep imagick
```

#### Otros módulos útiles para desarrollo

```bash
# Módulos para desarrollo y depuración
sudo apt install php8.3-xdebug php8.3-dev

# Módulos para diferentes bases de datos
sudo apt install php8.3-odbc php8.3-mongodb

# Módulos para procesamiento adicional
sudo apt install php8.3-gmp php8.3-ldap php8.3-snmp

# Módulos para caching
sudo apt install php8.3-apcu
```

### Módulos esenciales para Laravel (verificación)

Laravel requiere las siguientes extensiones PHP. Verifica que estén instaladas:

```bash
# Verificar módulos requeridos por Laravel
php8.3 -m | grep -E "(openssl|pdo|mbstring|tokenizer|xml|ctype|json|bcmath|fileinfo)"

# Si falta algún módulo, instálalo:
# sudo apt install php8.3-[nombre-modulo]
```

#### Lista completa de módulos recomendados para Laravel:

| Módulo | Paquete | Propósito |
|--------|---------|-----------|
| OpenSSL | php8.3-openssl | Encriptación y seguridad |
| PDO | php8.3-mysql/pgsql/sqlite3 | Acceso a base de datos |
| Mbstring | php8.3-mbstring | Manejo de strings multibyte |
| Tokenizer | php8.3-tokenizer | Análisis de tokens PHP |
| XML | php8.3-xml | Procesamiento XML |
| Ctype | php8.3-ctype | Validación de caracteres |
| JSON | php8.3-json | Manejo de JSON |
| BCMath | php8.3-bcmath | Matemáticas de precisión arbitraria |
| Fileinfo | php8.3-fileinfo | Información de archivos |
| GD | php8.3-gd | Manipulación de imágenes |
| Curl | php8.3-curl | Cliente HTTP |
| Zip | php8.3-zip | Compresión de archivos |
| Intl | php8.3-intl | Internacionalización |
| Redis | php8.3-redis | Cache y sesiones |
| Imagick | php8.3-imagick | Procesamiento avanzado de imágenes |

### Instalar todos los módulos recomendados para Laravel de una vez:

```bash
sudo apt install php8.3-openssl php8.3-mysql php8.3-mbstring \
php8.3-tokenizer php8.3-xml php8.3-ctype php8.3-json \
php8.3-bcmath php8.3-fileinfo php8.3-gd php8.3-curl \
php8.3-zip php8.3-intl php8.3-redis php8.3-imagick \
php8.3-dom php8.3-xmlwriter php8.3-simplexml php8.3-session \
php8.3-filter php8.3-hash php8.3-pcre php8.3-iconv
```

## Paso 4: Verificar instalaciones y módulos

```bash
# Verificar PHP 8.1 (versión actual)
php --version

# Verificar PHP 8.3 (nueva instalación)
php8.3 --version

# Ver módulos cargados en ambas versiones
php8.1 -m
php8.3 -m
```

## Paso 5: Configurar Apache para múltiples versiones de PHP

### Habilitar módulos necesarios

```bash
# Habilitar mod_rewrite y módulos de PHP-FPM
sudo a2enmod rewrite
sudo a2enmod proxy_fcgi
sudo a2enmod setenvif

# Habilitar configuraciones de PHP-FPM
sudo a2enconf php8.1-fpm
sudo a2enconf php8.3-fpm
```

### Verificar que PHP-FPM esté ejecutándose

```bash
# Iniciar y habilitar PHP-FPM para ambas versiones
sudo systemctl start php8.1-fpm
sudo systemctl start php8.3-fpm
sudo systemctl enable php8.1-fpm
sudo systemctl enable php8.3-fpm

# Verificar estado
sudo systemctl status php8.1-fpm
sudo systemctl status php8.3-fpm
```

## Paso 6: Configuración por Virtual Host

### Opción A: Configurar un sitio completo con PHP 8.3

Crear un nuevo virtual host:

```bash
sudo nano /etc/apache2/sites-available/sitio-php83.conf
```

Contenido del archivo:

```apache
<VirtualHost *:80>
    ServerName sitio-php83.local
    DocumentRoot /var/www/sitio-php83
    
    # Configurar PHP 8.3 para este sitio
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost"
    </FilesMatch>
    
    <Directory /var/www/sitio-php83>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/sitio-php83_error.log
    CustomLog ${APACHE_LOG_DIR}/sitio-php83_access.log combined
</VirtualHost>
```

Habilitar el sitio:

```bash
# Crear directorio del sitio
sudo mkdir -p /var/www/sitio-php83
sudo chown -R www-data:www-data /var/www/sitio-php83

# Habilitar el sitio
sudo a2ensite sitio-php83.conf
sudo systemctl reload apache2
```

### Opción B: Configurar directorios específicos con PHP 8.3

Para usar PHP 8.3 solo en ciertos directorios de un sitio existente:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Agregar dentro del VirtualHost:

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html
    
    # PHP 8.1 por defecto (ya configurado)
    
    # Configurar PHP 8.3 para directorios específicos
    <Directory "/var/www/html/app-php83">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost"
        </FilesMatch>
    </Directory>
    
    <Directory "/var/www/html/modulo-especial">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost"
        </FilesMatch>
    </Directory>
</VirtualHost>
```

## Paso 7: Configurar versión por defecto del sistema

```bash
# Verificar versión actual por defecto
php --version

# Si necesitas cambiar la versión por defecto del sistema
sudo update-alternatives --install /usr/bin/php php /usr/bin/php8.1 81
sudo update-alternatives --install /usr/bin/php php /usr/bin/php8.3 83

# Configurar PHP 8.1 como predeterminado
sudo update-alternatives --config php
# Seleccionar la opción correspondiente a PHP 8.1
```

## Paso 8: Crear archivos de prueba

### Archivo de información PHP para verificar versiones

```bash
# Para el sitio principal (PHP 8.1)
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info81.php

# Para directorios con PHP 8.3
sudo mkdir -p /var/www/html/app-php83
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/app-php83/info83.php
```

## Paso 9: Configuración de php.ini específica

Cada versión tiene su propio archivo de configuración:

```bash
# PHP 8.1
sudo nano /etc/php/8.1/fpm/php.ini

# PHP 8.3
sudo nano /etc/php/8.3/fpm/php.ini
```

Después de cualquier cambio en php.ini, reinicia el servicio correspondiente:

```bash
sudo systemctl restart php8.1-fpm
sudo systemctl restart php8.3-fpm
sudo systemctl reload apache2
```

## Comandos de gestión específicos para Ubuntu/Debian

### Gestión de paquetes y módulos

```bash
# Ver paquetes PHP instalados
dpkg -l | grep php

# Ver información detallada de un módulo específico
apt show php8.3-imagick

# Buscar módulos específicos
apt search imagick | grep php8.3

# Actualizar PHP 8.1
sudo apt update && sudo apt upgrade php8.1*

# Actualizar PHP 8.3
sudo apt update && sudo apt upgrade php8.3*

# Remover un módulo específico
sudo apt remove php8.3-xdebug

# Purgar completamente un módulo
sudo apt purge php8.3-xdebug
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
    if php8.1 -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php8.1 -m | grep -q "^$module$"; then
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
    if php8.3 -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "✗ $module - FALTANTE"
    fi
done

echo ""
echo "Módulos opcionales recomendados:"
for module in "${optional_modules[@]}"; do
    if php8.3 -m | grep -q "^$module$"; then
        echo "✓ $module - Instalado"
    else
        echo "○ $module - No instalado"
    fi
done

echo ""
echo "=== Versiones de PHP ==="
echo "PHP 8.1: $(php8.1 --version | head -1)"
echo "PHP 8.3: $(php8.3 --version | head -1)"
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
sudo apt install php8.3-dev php-pear build-essential

# Usar PECL de PHP 8.3 para instalar módulos
sudo pecl -d php_suffix=8.3 install redis
sudo pecl -d php_suffix=8.3 install imagick

# Habilitar módulos en php.ini
echo "extension=redis.so" | sudo tee -a /etc/php/8.3/fpm/php.ini
echo "extension=imagick.so" | sudo tee -a /etc/php/8.3/fpm/php.ini

# También habilitarlo para CLI
echo "extension=redis.so" | sudo tee -a /etc/php/8.3/cli/php.ini
echo "extension=imagick.so" | sudo tee -a /etc/php/8.3/cli/php.ini

# Reiniciar PHP-FPM después de cambios
sudo systemctl restart php8.3-fpm
```

### Configuración específica de Imagick

Si necesitas configuraciones específicas para Imagick:

```bash
# Verificar que ImageMagick esté instalado en el sistema
convert --version

# Si no está instalado
sudo apt install imagemagick

# Verificar configuración de Imagick en PHP
php8.3 -m | grep imagick
php8.3 --ri imagick

# Configurar políticas de ImageMagick si es necesario
sudo nano /etc/ImageMagick-6/policy.xml
```

## Comandos útiles para gestión

### Verificar versiones instaladas
```bash
# Listar todas las versiones de PHP instaladas
ls /etc/php/

# Verificar qué módulos están habilitados
php8.1 -m
php8.3 -m

# Comparar módulos entre versiones
echo "=== Módulos en PHP 8.1 ===" && php8.1 -m | sort
echo "=== Módulos en PHP 8.3 ===" && php8.3 -m | sort
```

### Gestionar servicios PHP-FPM
```bash
# Ver estado de todos los servicios PHP-FPM
sudo systemctl status php*-fpm

# Reiniciar servicios específicos
sudo systemctl restart php8.1-fpm
sudo systemctl restart php8.3-fpm

# Ver procesos PHP-FPM activos
ps aux | grep php-fpm
```

### Logs para troubleshooting
```bash
# Ver logs de PHP-FPM
sudo tail -f /var/log/php8.1-fpm.log
sudo tail -f /var/log/php8.3-fpm.log

# Ver logs de Apache
sudo tail -f /var/log/apache2/error.log

# Ver logs específicos del sitio
sudo tail -f /var/log/apache2/sitio-php83_error.log
```

### Herramientas de debugging

```bash
# Verificar configuración de PHP-FPM
sudo php8.3-fpm -t

# Ver configuración actual de PHP
php8.3 --ini

# Información detallada de configuración
php8.3 -i | grep "Configuration File"
```

## Solución de problemas comunes

### Error: "File not found" o "Primary script unknown"
- Verificar que DocumentRoot y rutas en la configuración sean correctas
- Asegurar que www-data tenga permisos de lectura en los directorios

```bash
# Verificar permisos
ls -la /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### PHP-FPM no responde
```bash
# Verificar que los sockets existan
ls -la /var/run/php/

# Reiniciar servicios
sudo systemctl restart php8.1-fpm php8.3-fpm apache2

# Verificar configuración de pools
sudo php8.1-fpm -t
sudo php8.3-fpm -t
```

### Verificar configuración de Apache
```bash
# Probar configuración
sudo apache2ctl configtest

# Ver módulos habilitados
apache2ctl -M | grep php
apache2ctl -M | grep proxy
```

### Problemas con módulos específicos

```bash
# Si Imagick no funciona correctamente
sudo apt install --reinstall php8.3-imagick
sudo systemctl restart php8.3-fpm

# Verificar dependencias de un módulo
apt depends php8.3-imagick

# Ver información detallada de un módulo instalado
php8.3 --ri imagick
```

## Integración con Composer

Para usar Composer con diferentes versiones de PHP:

```bash
# Usar Composer con PHP 8.1
php8.1 /usr/local/bin/composer install

# Usar Composer con PHP 8.3
php8.3 /usr/local/bin/composer install

# Crear alias para facilitar el uso
echo 'alias composer81="php8.1 /usr/local/bin/composer"' >> ~/.bashrc
echo 'alias composer83="php8.3 /usr/local/bin/composer"' >> ~/.bashrc
source ~/.bashrc
```

## Notas importantes

1. **Seguridad**: Mantén ambas versiones actualizadas regularmente con `sudo apt update && sudo apt upgrade`
2. **Rendimiento**: PHP-FPM es más eficiente que mod_php para múltiples versiones
3. **Composer**: Usa la versión correcta de PHP para cada proyecto
4. **Extensiones**: Instala las extensiones necesarias para cada versión por separado
5. **Configuración**: Cada versión tiene archivos de configuración independientes
6. **Logs**: Cada versión genera logs separados para facilitar el debugging

Esta configuración te permite mantener PHP 8.1 como predeterminado mientras usas PHP 8.3 en sitios o directorios específicos según tus necesidades, con todos los módulos necesarios para proyectos Laravel.