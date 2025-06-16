# Instalación y Configuración Multi-versión PHP en Rocky Linux 9

## Introducción

Esta guía te permitirá instalar múltiples versiones de PHP (8.2, 8.3 y 8.4) en Rocky Linux 9, configurando Apache para que diferentes directorios utilicen versiones específicas de PHP. Esta configuración es especialmente útil para:

- **Desarrollo y testing**: Probar aplicaciones en diferentes versiones de PHP
- **Migración gradual**: Actualizar proyectos uno por uno sin afectar otros
- **Compatibilidad**: Mantener aplicaciones legacy mientras desarrollas con versiones más recientes
- **Ambientes de producción**: Servir múltiples aplicaciones con diferentes requerimientos

### Arquitectura de la Solución

- **PHP por defecto del sistema**: PHP 8.4
- **Versiones adicionales**: PHP 8.2, 8.3 (vía PHP-FPM)
- **Servidor web**: Apache con mod_proxy_fcgi
- **Configuración**: Virtual hosts específicos por versión
- **Gestión**: Scripts para cambio rápido de versiones

## Requisitos Previos

```bash
# Verificar sistema operativo
cat /etc/rocky-release

# Actualizar el sistema
sudo dnf update -y

# Instalar herramientas necesarias
sudo dnf install -y wget curl vim git
```

## Paso 1: Configurar Repositorios Adicionales

```bash
# Habilitar repositorio EPEL
sudo dnf install -y epel-release

# Instalar repositorio Remi (necesario para múltiples versiones PHP)
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm

# Verificar repositorios disponibles
sudo dnf repolist | grep -i remi
```

## Paso 2: Instalar Apache

```bash
# Instalar Apache
sudo dnf install -y httpd httpd-tools

# Iniciar y habilitar Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# Verificar estado
sudo systemctl status httpd

# Configurar firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## Paso 3: Instalar PHP 8.4 (Versión por Defecto)

```bash
# Habilitar módulo PHP 8.4 de Remi
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.4 -y

# Instalar PHP 8.4 y extensiones comunes
sudo dnf install -y php php-cli php-fpm php-common php-mysqlnd php-pdo \
    php-gd php-mbstring php-curl php-xml php-zip php-json php-openssl \
    php-fileinfo php-tokenizer php-opcache php-redis php-intl

# Verificar instalación
php -v
```

## Paso 4: Instalar PHP 8.3

```bash
# Instalar PHP 8.3 con prefijo específico
sudo dnf install -y php83-php php83-php-cli php83-php-fpm php83-php-common \
    php83-php-mysqlnd php83-php-pdo php83-php-gd php83-php-mbstring \
    php83-php-curl php83-php-xml php83-php-zip php83-php-json \
    php83-php-opcache php83-php-redis php83-php-intl

# Verificar instalación
/usr/bin/php83 -v

# Configurar PHP-FPM 8.3
sudo systemctl start php83-php-fpm
sudo systemctl enable php83-php-fpm
```

## Paso 5: Instalar PHP 8.2

```bash
# Instalar PHP 8.2 con prefijo específico
sudo dnf install -y php82-php php82-php-cli php82-php-fpm php82-php-common \
    php82-php-mysqlnd php82-php-pdo php82-php-gd php82-php-mbstring \
    php82-php-curl php82-php-xml php82-php-zip php82-php-json \
    php82-php-opcache php82-php-redis php82-php-intl

# Verificar instalación
/usr/bin/php82 -v

# Configurar PHP-FPM 8.2
sudo systemctl start php82-php-fpm
sudo systemctl enable php82-php-fpm
```

## Paso 6: Configurar PHP-FPM para cada versión

### Configuración PHP-FPM 8.4 (por defecto)
```bash
# Editar configuración principal
sudo vim /etc/php-fpm.d/www.conf

# Modificar estas líneas:
# listen = /run/php-fpm/php84.sock
# listen.owner = apache
# listen.group = apache
# listen.mode = 0660

# Reiniciar servicio
sudo systemctl restart php-fpm
sudo systemctl enable php-fpm
```

### Configuración PHP-FPM 8.3
```bash
# Editar configuración
sudo vim /etc/opt/remi/php83/php-fpm.d/www.conf

# Modificar socket específico:
# listen = /run/php-fpm/php83.sock
# listen.owner = apache
# listen.group = apache
# listen.mode = 0660

# Reiniciar servicio
sudo systemctl restart php83-php-fpm
```

### Configuración PHP-FPM 8.2
```bash
# Editar configuración
sudo vim /etc/opt/remi/php82/php-fpm.d/www.conf

# Modificar socket específico:
# listen = /run/php-fpm/php82.sock
# listen.owner = apache
# listen.group = apache
# listen.mode = 0660

# Reiniciar servicio
sudo systemctl restart php82-php-fpm
```

## Paso 7: Configurar Apache para Multi-versión PHP

### Habilitar módulos necesarios
```bash
# Cargar módulos de Apache
sudo vim /etc/httpd/conf/httpd.conf

# Agregar estas líneas si no existen:
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
```

### Crear configuración base para PHP-FPM
```bash
# Crear archivo de configuración PHP-FPM
sudo tee /etc/httpd/conf.d/php-fpm.conf << 'EOF'
# Configuración para múltiples versiones de PHP via PHP-FPM

# Definir handlers para cada versión
<Proxy "unix:/run/php-fpm/php84.sock|fcgi://php84-fpm">
    ProxySet connectiontimeout=5
</Proxy>

<Proxy "unix:/run/php-fpm/php83.sock|fcgi://php83-fpm">
    ProxySet connectiontimeout=5
</Proxy>

<Proxy "unix:/run/php-fpm/php82.sock|fcgi://php82-fpm">
    ProxySet connectiontimeout=5
</Proxy>

# Configuración por defecto (PHP 8.4)
<FilesMatch \.php$>
    SetHandler "proxy:fcgi://php84-fpm"
</FilesMatch>
EOF
```

## Paso 8: Crear Directorios de Proyectos

```bash
# Crear directorios para diferentes versiones
sudo mkdir -p /var/www/project84
sudo mkdir -p /var/www/project83
sudo mkdir -p /var/www/project82

# Establecer permisos
sudo chown -R baseuser:devteam /var/www/project84
sudo chown -R baseuser:devteam /var/www/project83
sudo chown -R baseuser:devteam /var/www/project82

sudo chmod -R 2775 /var/www/project84
sudo chmod -R 2775 /var/www/project83
sudo chmod -R 2775 /var/www/project82
```

## Paso 9: Configurar Virtual Hosts Específicos

### Virtual Host para PHP 8.4
```bash
sudo tee /etc/httpd/conf.d/project84.conf << 'EOF'
<VirtualHost *:80>
    ServerName project84.local
    DocumentRoot /var/www/project84
    
    <Directory /var/www/project84>
        AllowOverride All
        Require all granted
        
        # Forzar PHP 8.4 para este directorio
        <FilesMatch \.php$>
            SetHandler "proxy:fcgi://php84-fpm"
        </FilesMatch>
    </Directory>
    
    ErrorLog /var/log/httpd/project84_error.log
    CustomLog /var/log/httpd/project84_access.log combined
</VirtualHost>
EOF
```

### Virtual Host para PHP 8.3
```bash
sudo tee /etc/httpd/conf.d/project83.conf << 'EOF'
<VirtualHost *:80>
    ServerName project83.local
    DocumentRoot /var/www/project83
    
    <Directory /var/www/project83>
        AllowOverride All
        Require all granted
        
        # Forzar PHP 8.3 para este directorio
        <FilesMatch \.php$>
            SetHandler "proxy:fcgi://php83-fpm"
        </FilesMatch>
    </Directory>
    
    ErrorLog /var/log/httpd/project83_error.log
    CustomLog /var/log/httpd/project83_access.log combined
</VirtualHost>
EOF
```

### Virtual Host para PHP 8.2
```bash
sudo tee /etc/httpd/conf.d/project82.conf << 'EOF'
<VirtualHost *:80>
    ServerName project82.local
    DocumentRoot /var/www/project82
    
    <Directory /var/www/project82>
        AllowOverride All
        Require all granted
        
        # Forzar PHP 8.2 para este directorio
        <FilesMatch \.php$>
            SetHandler "proxy:fcgi://php82-fpm"
        </FilesMatch>
    </Directory>
    
    ErrorLog /var/log/httpd/project82_error.log
    CustomLog /var/log/httpd/project82_access.log combined
</VirtualHost>
EOF
```

## Paso 10: Crear Archivos de Prueba

### Archivo de prueba PHP 8.4
```bash
sudo tee /var/www/project84/index.php << 'EOF'
<?php
echo "<h1>Proyecto PHP 8.4</h1>";
echo "<p>Versión de PHP: " . phpversion() . "</p>";
echo "<p>Fecha: " . date('Y-m-d H:i:s') . "</p>";

// Mostrar información detallada
phpinfo();
?>
EOF
```

### Archivo de prueba PHP 8.3
```bash
sudo tee /var/www/project83/index.php << 'EOF'
<?php
echo "<h1>Proyecto PHP 8.3</h1>";
echo "<p>Versión de PHP: " . phpversion() . "</p>";
echo "<p>Fecha: " . date('Y-m-d H:i:s') . "</p>";

// Mostrar información detallada
phpinfo();
?>
EOF
```

### Archivo de prueba PHP 8.2
```bash
sudo tee /var/www/project82/index.php << 'EOF'
<?php
echo "<h1>Proyecto PHP 8.2</h1>";
echo "<p>Versión de PHP: " . phpversion() . "</p>";
echo "<p>Fecha: " . date('Y-m-d H:i:s') . "</p>";

// Mostrar información detallada
phpinfo();
?>
EOF
```

## Paso 11: Configurar Hosts Locales (Opcional)

```bash
# Agregar entradas al archivo hosts
sudo tee -a /etc/hosts << 'EOF'
127.0.0.1 project84.local
127.0.0.1 project83.local
127.0.0.1 project82.local
EOF
```

## Paso 12: Reiniciar Servicios y Verificar

```bash
# Verificar configuración de Apache
sudo httpd -t

# Reiniciar todos los servicios
sudo systemctl restart php-fpm
sudo systemctl restart php83-php-fpm
sudo systemctl restart php82-php-fpm
sudo systemctl restart httpd

# Verificar estado de servicios
sudo systemctl status php-fpm php83-php-fpm php82-php-fpm httpd
```

## Verificación y Testing

### Verificar versiones instaladas
```bash
echo "=== Versiones PHP instaladas ==="
echo "PHP 8.4 (por defecto):"
php -v

echo -e "\nPHP 8.3:"
/usr/bin/php83 -v

echo -e "\nPHP 8.2:"
/usr/bin/php82 -v
```

### Verificar sockets PHP-FPM
```bash
echo "=== Sockets PHP-FPM ==="
ls -la /run/php-fpm/
sudo ss -tulpn | grep php-fpm
```

### Probar en navegador
```bash
# Probar cada proyecto
curl -s http://project84.local | grep "PHP Version"
curl -s http://project83.local | grep "PHP Version"
curl -s http://project82.local | grep "PHP Version"
```

## Scripts de Utilidad

### Script para cambiar versión PHP por defecto
```bash
sudo tee /usr/local/bin/php-switch.sh << 'EOF'
#!/bin/bash

case $1 in
    8.4)
        sudo alternatives --set php /usr/bin/php
        echo "PHP 8.4 establecido como predeterminado"
        ;;
    8.3)
        sudo alternatives --set php /usr/bin/php83
        echo "PHP 8.3 establecido como predeterminado"
        ;;
    8.2)
        sudo alternatives --set php /usr/bin/php82
        echo "PHP 8.2 establecido como predeterminado"
        ;;
    *)
        echo "Uso: $0 {8.4|8.3|8.2}"
        echo "Versión actual: $(php -v | head -n1)"
        ;;
esac
EOF

sudo chmod +x /usr/local/bin/php-switch.sh
```

### Script de monitoreo de servicios PHP-FPM
```bash
sudo tee /usr/local/bin/php-status.sh << 'EOF'
#!/bin/bash

echo "=== Estado de servicios PHP-FPM ==="
for service in php-fpm php83-php-fpm php82-php-fpm; do
    echo -n "$service: "
    if systemctl is-active --quiet $service; then
        echo "✅ Activo"
    else
        echo "❌ Inactivo"
    fi
done

echo -e "\n=== Procesos PHP-FPM ==="
ps aux | grep -E "php-fpm|php83-fpm|php82-fpm" | grep -v grep

echo -e "\n=== Sockets disponibles ==="
ls -la /run/php-fpm/
EOF

sudo chmod +x /usr/local/bin/php-status.sh
```

## Mantenimiento y Troubleshooting

### Logs importantes
```bash
# Logs de Apache
sudo tail -f /var/log/httpd/error_log
sudo tail -f /var/log/httpd/project84_error.log

# Logs de PHP-FPM
sudo tail -f /var/log/php-fpm/error.log
sudo tail -f /var/opt/remi/php83/log/php-fpm/error.log
sudo tail -f /var/opt/remi/php82/log/php-fpm/error.log
```

### Comandos de diagnóstico
```bash
# Verificar módulos de Apache cargados
sudo httpd -M | grep -E "proxy|fcgi"

# Verificar configuración PHP-FPM
sudo php-fpm -t
sudo /usr/bin/php83-fpm -t
sudo /usr/bin/php82-fpm -t

# Verificar conectividad a sockets
sudo -u apache test -w /run/php-fpm/php84.sock && echo "PHP 8.4 socket OK"
sudo -u apache test -w /run/php-fpm/php83.sock && echo "PHP 8.3 socket OK"
sudo -u apache test -w /run/php-fpm/php82.sock && echo "PHP 8.2 socket OK"
```

## Configuración Avanzada

### Crear proyecto con versión específica de PHP
```bash
sudo tee /usr/local/bin/create-php-project.sh << 'EOF'
#!/bin/bash

if [ $# -ne 2 ]; then
    echo "Uso: $0 <nombre-proyecto> <version-php>"
    echo "Versiones disponibles: 8.2, 8.3, 8.4"
    exit 1
fi

PROJECT_NAME=$1
PHP_VERSION=$2

case $PHP_VERSION in
    8.2|8.3|8.4)
        PROJECT_DIR="/var/www/project${PHP_VERSION//.}"
        PHP_HANDLER="php${PHP_VERSION//.}-fpm"
        ;;
    *)
        echo "Versión PHP no válida: $PHP_VERSION"
        exit 1
        ;;
esac

# Crear directorio
mkdir -p "$PROJECT_DIR/$PROJECT_NAME"
chown baseuser:devteam "$PROJECT_DIR/$PROJECT_NAME"
chmod 2775 "$PROJECT_DIR/$PROJECT_NAME"

echo "Proyecto $PROJECT_NAME creado en $PROJECT_DIR"
echo "Configurado para usar PHP $PHP_VERSION"
EOF

sudo chmod +x /usr/local/bin/create-php-project.sh
```

Esta configuración te permite ejecutar múltiples versiones de PHP de manera simultánea y controlada, ideal para entornos de desarrollo y producción que requieren compatibilidad con diferentes versiones de PHP.