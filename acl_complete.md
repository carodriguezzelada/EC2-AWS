# Guía Completa: ACL (Access Control Lists) en Rocky Linux 9 y Ubuntu

## Introducción

Las ACL (Access Control Lists) son una extensión del sistema tradicional de permisos Unix que permite un control de acceso mucho más granular y flexible. Mientras que el sistema tradicional solo permite definir permisos para propietario, grupo y otros, las ACL permiten asignar permisos específicos a múltiples usuarios y grupos individuales.

### ¿Por qué usar ACL?

**Limitaciones del sistema tradicional:**
```bash
# Sistema tradicional: Solo 3 niveles
-rwxrwxr-x  usuario grupo otros
```

**Ventajas de ACL:**
- Permisos específicos para múltiples usuarios
- Permisos específicos para múltiples grupos
- Permisos por defecto para archivos nuevos
- Máscaras de permisos efectivos
- Herencia automática de permisos
- Control granular sin modificar estructura de grupos

## Conceptos Fundamentales

### Tipos de Entradas ACL

1. **user (u)**: Permisos para usuarios específicos
2. **group (g)**: Permisos para grupos específicos
3. **other (o)**: Permisos para otros usuarios
4. **mask (m)**: Máscara que limita permisos efectivos
5. **default**: Plantilla para archivos/directorios nuevos

### Permisos ACL

- **r (read)**: Lectura
- **w (write)**: Escritura
- **x (execute)**: Ejecución
- **- (none)**: Sin permiso

## Instalación y Verificación

### Rocky Linux 9

```bash
# Verificar si ACL está instalado
rpm -qa | grep acl

# Instalar herramientas ACL si no están disponibles
sudo dnf install -y acl

# Verificar que el kernel soporta ACL
grep -i acl /boot/config-$(uname -r)

# Verificar módulos cargados
lsmod | grep -i acl
```

### Ubuntu

```bash
# Verificar si ACL está instalado
dpkg -l | grep acl

# Instalar herramientas ACL si no están disponibles
sudo apt update
sudo apt install -y acl

# Verificar soporte del kernel
grep -i acl /boot/config-$(uname -r)
```

### Verificar soporte del sistema de archivos

```bash
# Verificar si el filesystem soporta ACL
tune2fs -l /dev/sda1 | grep -i acl  # Para ext2/3/4
xfs_info /dev/sda1 | grep -i acl    # Para XFS

# Verificar montaje con soporte ACL
mount | grep acl
```

## Configuración del Sistema de Archivos

### Habilitar ACL en montajes

#### Temporal (hasta reinicio)
```bash
# Remontar con soporte ACL
sudo mount -o remount,acl /

# Verificar
mount | grep "on / " | grep acl
```

#### Permanente (editar /etc/fstab)
```bash
# Hacer backup del fstab
sudo cp /etc/fstab /etc/fstab.backup

# Editar fstab
sudo vim /etc/fstab

# Ejemplo de línea con ACL habilitado:
# UUID=xxx-xxx-xxx / ext4 defaults,acl 0 1

# Aplicar cambios
sudo mount -o remount /
```

### Verificación de configuración
```bash
# Crear archivo de prueba
touch /tmp/test-acl

# Intentar establecer ACL
setfacl -m u:nobody:r /tmp/test-acl

# Si funciona, ACL está habilitado
getfacl /tmp/test-acl
```

## Comandos Básicos

### getfacl - Mostrar ACL

```bash
# Sintaxis básica
getfacl archivo_o_directorio

# Opciones útiles
getfacl -R directorio/          # Recursivo
getfacl -t archivo             # Formato tabular
getfacl -n archivo             # Sin resolver nombres a IDs
getfacl -p archivo             # Solo permisos, sin comentarios
getfacl -s archivo             # Omitir archivos que no tienen ACL extendidas
```

### setfacl - Establecer ACL

```bash
# Sintaxis básica
setfacl -m entrada archivo_o_directorio

# Opciones principales
setfacl -m u:usuario:permisos archivo    # Modificar ACL de usuario
setfacl -m g:grupo:permisos archivo      # Modificar ACL de grupo
setfacl -x u:usuario archivo             # Eliminar entrada específica
setfacl -b archivo                       # Eliminar todas las ACL extendidas
setfacl -k archivo                       # Eliminar ACL por defecto
setfacl -R -m entrada directorio/        # Aplicar recursivamente
setfacl -d -m entrada directorio/        # Establecer ACL por defecto
```

## Ejemplos Prácticos

### Ejemplo 1: Configuración Básica

```bash
# Crear directorio de prueba
mkdir /tmp/acl-demo
cd /tmp/acl-demo

# Crear usuarios de prueba (opcional)
sudo useradd testuser1
sudo useradd testuser2

# Crear archivo
echo "Contenido de prueba" > archivo.txt

# Ver permisos tradicionales
ls -la archivo.txt
# -rw-rw-r-- 1 usuario grupo 18 fecha archivo.txt

# Ver ACL actual
getfacl archivo.txt
```

### Ejemplo 2: Asignar Permisos a Usuario Específico

```bash
# Dar permiso de lectura a testuser1
setfacl -m u:testuser1:r archivo.txt

# Dar permisos de lectura y escritura a testuser2
setfacl -m u:testuser2:rw archivo.txt

# Ver resultado
getfacl archivo.txt
# # file: archivo.txt
# # owner: usuario
# # group: grupo
# user::rw-
# user:testuser1:r--
# user:testuser2:rw-
# group::rw-
# mask::rw-
# other::r--

# Nota el "+" en ls -la indicando ACL extendidas
ls -la archivo.txt
# -rw-rw-r--+ 1 usuario grupo 18 fecha archivo.txt
```

### Ejemplo 3: Permisos para Grupos

```bash
# Crear grupos de prueba
sudo groupadd desarrolladores
sudo groupadd testers

# Asignar permisos a grupos
setfacl -m g:desarrolladores:rwx archivo.txt
setfacl -m g:testers:r archivo.txt

# Ver resultado
getfacl archivo.txt
```

### Ejemplo 4: ACL por Defecto para Directorios

```bash
# Crear directorio
mkdir proyecto/

# Establecer ACL por defecto
setfacl -d -m u:testuser1:rwx proyecto/
setfacl -d -m g:desarrolladores:rwx proyecto/

# Ver ACL por defecto
getfacl proyecto/

# Crear archivo dentro del directorio
touch proyecto/nuevo_archivo.txt

# El archivo hereda permisos por defecto
getfacl proyecto/nuevo_archivo.txt
```

## Escenarios Avanzados

### Escenario 1: Servidor Web con Múltiples Desarrolladores

```bash
# Crear estructura
sudo mkdir -p /var/www/proyecto
sudo chown apache:apache /var/www/proyecto

# Permitir acceso a desarrolladores específicos
sudo setfacl -m u:dev1:rwx /var/www/proyecto
sudo setfacl -m u:dev2:rwx /var/www/proyecto
sudo setfacl -m u:dev3:r-x /var/www/proyecto  # Solo lectura

# Permitir al servidor web leer y escribir
sudo setfacl -m u:apache:rwx /var/www/proyecto

# Establecer permisos por defecto para archivos nuevos
sudo setfacl -d -m u:dev1:rwx /var/www/proyecto
sudo setfacl -d -m u:dev2:rwx /var/www/proyecto
sudo setfacl -d -m u:dev3:r-x /var/www/proyecto
sudo setfacl -d -m u:apache:rwx /var/www/proyecto

# Verificar configuración
getfacl /var/www/proyecto
```

### Escenario 2: Compartir Archivos entre Departamentos

```bash
# Crear directorio compartido
sudo mkdir /shared/interdepartamental

# Crear grupos departamentales
sudo groupadd ventas
sudo groupadd marketing
sudo groupadd desarrollo

# Permisos específicos por departamento
sudo setfacl -m g:ventas:rwx /shared/interdepartamental
sudo setfacl -m g:marketing:r-x /shared/interdepartamental
sudo setfacl -m g:desarrollo:rwx /shared/interdepartamental

# Subdirectorio específico para ventas
sudo mkdir /shared/interdepartamental/ventas-privado
sudo setfacl -m g:ventas:rwx /shared/interdepartamental/ventas-privado
sudo setfacl -m g:marketing:--- /shared/interdepartamental/ventas-privado
sudo setfacl -m g:desarrollo:--- /shared/interdepartamental/ventas-privado
```

### Escenario 3: Control de Acceso Temporal

```bash
# Crear archivo sensible
sudo touch /etc/config-importante.conf
sudo chmod 600 /etc/config-importante.conf

# Dar acceso temporal a usuario específico
sudo setfacl -m u:admin-temp:r /etc/config-importante.conf

# Verificar acceso
sudo -u admin-temp cat /etc/config-importante.conf

# Revocar acceso temporal
sudo setfacl -x u:admin-temp /etc/config-importante.conf
```

## Máscara de Permisos (Mask)

### Entender la Máscara

```bash
# La máscara limita los permisos efectivos
# Ejemplo: usuario tiene rwx, pero máscara es r--, permisos efectivos = r--

# Ver máscara actual
getfacl archivo.txt | grep mask

# Establecer máscara específica
setfacl -m m:rw archivo.txt

# Ver permisos efectivos
getfacl archivo.txt
```

### Cálculo de Permisos Efectivos

```bash
# Permisos efectivos = Permisos asignados AND Máscara
# Ejemplo práctico:

# Crear archivo y asignar permisos
touch ejemplo.txt
setfacl -m u:testuser1:rwx ejemplo.txt

# Ver permisos sin máscara restrictiva
getfacl ejemplo.txt

# Aplicar máscara restrictiva
setfacl -m m:r-- ejemplo.txt

# Ahora testuser1 solo puede leer, aunque tenía rwx
getfacl ejemplo.txt
```

## Backup y Restauración de ACL

### Backup de ACL

```bash
# Backup de ACL de un archivo
getfacl archivo.txt > archivo.txt.acl

# Backup recursivo de directorio
getfacl -R directorio/ > directorio.acl

# Backup de todo el sistema (cuidado con el tamaño)
getfacl -R / > sistema-completo.acl 2>/dev/null

# Backup selectivo con find
find /home -type f -exec getfacl {} \; > home.acl 2>/dev/null
```

### Restauración de ACL

```bash
# Restaurar ACL de archivo específico
setfacl --restore=archivo.txt.acl

# Restaurar ACL de directorio
setfacl --restore=directorio.acl

# Verificar antes de restaurar
cat archivo.txt.acl  # Ver contenido del backup
```

### Script de Backup Automatizado

```bash
#!/bin/bash
# Script: backup-acl.sh

BACKUP_DIR="/backup/acl"
DATE=$(date +%Y%m%d_%H%M%S)

# Crear directorio de backup
mkdir -p "$BACKUP_DIR"

# Directorios importantes a respaldar
DIRS=("/home" "/var/www" "/opt")

for dir in "${DIRS[@]}"; do
    if [ -d "$dir" ]; then
        echo "Respaldando ACL de $dir..."
        getfacl -R "$dir" > "$BACKUP_DIR/$(basename $dir)_${DATE}.acl" 2>/dev/null
    fi
done

echo "Backup completado en $BACKUP_DIR"
```

## Herramientas de Monitoreo y Auditoría

### Script de Auditoría ACL

```bash
#!/bin/bash
# Script: audit-acl.sh

echo "=== AUDITORÍA DE ACL ==="
echo "Fecha: $(date)"
echo

# Buscar archivos con ACL extendidas
echo "Archivos con ACL extendidas:"
find /home /var/www /opt -type f -exec ls -la {} \; 2>/dev/null | grep "+" | head -20

echo -e "\nDirectorios con ACL extendidas:"
find /home /var/www /opt -type d -exec ls -lad {} \; 2>/dev/null | grep "+" | head -10

echo -e "\nUsuarios con permisos ACL específicos:"
find /home /var/www /opt -exec getfacl {} \; 2>/dev/null | grep "^user:" | grep -v "user::" | sort -u

echo -e "\nGrupos con permisos ACL específicos:"
find /home /var/www /opt -exec getfacl {} \; 2>/dev/null | grep "^group:" | grep -v "group::" | sort -u
```

### Monitoreo en Tiempo Real

```bash
# Usar inotify para monitorear cambios en ACL
sudo dnf install -y inotify-tools  # Rocky Linux
sudo apt install -y inotify-tools  # Ubuntu

# Script de monitoreo
#!/bin/bash
# monitor-acl.sh

WATCH_DIR="/var/www"

inotifywait -m -r -e attrib "$WATCH_DIR" --format '%w%f %e' |
while read file event; do
    if [[ $event == "ATTRIB" ]]; then
        # Verificar si tiene ACL extendidas
        if ls -la "$file" 2>/dev/null | grep -q "+"; then
            echo "$(date): ACL modificada en $file"
            getfacl "$file" 2>/dev/null
            echo "---"
        fi
    fi
done
```

## Troubleshooting y Problemas Comunes

### Problema 1: ACL no funciona

```bash
# Verificar soporte del filesystem
tune2fs -l /dev/sda1 | grep -i acl

# Verificar montaje
mount | grep acl

# Verificar herramientas instaladas
which getfacl setfacl

# Probar en directorio temporal
mkdir /tmp/test-acl
setfacl -m u:nobody:r /tmp/test-acl
getfacl /tmp/test-acl
```

### Problema 2: Permisos inesperados

```bash
# Verificar máscara
getfacl archivo | grep mask

# Recalcular máscara automáticamente
setfacl -m m:: archivo  # Máscara vacía = recalcular

# Ver permisos efectivos
getfacl archivo | grep -E "(user:|group:)" | grep -v "::"
```

### Problema 3: ACL no se heredan

```bash
# Verificar ACL por defecto del directorio padre
getfacl directorio-padre/

# Establecer ACL por defecto si no existe
setfacl -d -m u:usuario:rwx directorio-padre/

# Verificar herencia en archivo nuevo
touch directorio-padre/test-herencia.txt
getfacl directorio-padre/test-herencia.txt
```

### Problema 4: Rendimiento con muchas ACL

```bash
# Verificar cantidad de ACL en el sistema
find / -type f -exec ls -la {} \; 2>/dev/null | grep -c "+"

# Limpiar ACL innecesarias
find /path -type f -exec setfacl -b {} \; 2>/dev/null  # ¡CUIDADO!

# Optimizar con scripts selectivos
# Solo limpiar ACL que coincidan con criterios específicos
```

## Integración con Servicios

### Apache/Nginx

```bash
# Configurar ACL para servidor web
sudo setfacl -R -m u:apache:rx /var/www/
sudo setfacl -R -d -m u:apache:rx /var/www/

# Para archivos que Apache debe escribir
sudo setfacl -R -m u:apache:rwx /var/www/uploads/
sudo setfacl -R -d -m u:apache:rwx /var/www/uploads/
```

### SSH/SFTP

```bash
# Configurar ACL para usuarios SFTP
sudo setfacl -m u:sftpuser:rx /home/shared/
sudo setfacl -R -m u:sftpuser:rw /home/shared/upload/

# Restringir acceso a directorios sensibles
sudo setfacl -m u:sftpuser:--- /home/shared/private/
```

### Samba

```bash
# ACL para shares de Samba
sudo setfacl -R -m g:sambashare:rwx /srv/samba/
sudo setfacl -R -d -m g:sambashare:rwx /srv/samba/

# Configurar en smb.conf
# [share]
# path = /srv/samba
# acl group control = yes
# map acl inherit = yes
```

## Mejores Prácticas

### 1. Planificación
- Documentar estructura de permisos antes de implementar
- Usar grupos cuando sea posible en lugar de usuarios individuales
- Establecer convenciones de nomenclatura consistentes

### 2. Seguridad
- Aplicar principio de menor privilegio
- Auditar ACL regularmente
- Usar ACL por defecto para automatizar permisos
- Respaldar ACL antes de cambios importantes

### 3. Mantenimiento
- Limpiar ACL obsoletas periódicamente
- Monitorear rendimiento en sistemas con muchas ACL
- Documentar cambios significativos

### 4. Testing
- Probar cambios en entorno de desarrollo
- Verificar herencia en directorios
- Validar permisos efectivos con máscara

## Scripts de Utilidad

### Script de Reporte ACL

```bash
#!/bin/bash
# acl-report.sh

echo "REPORTE DE ACL - $(date)"
echo "================================="

# Función para mostrar ACL de un directorio
show_acl_summary() {
    local dir=$1
    echo "Directorio: $dir"
    
    if [ -d "$dir" ]; then
        # Contar archivos con ACL
        local count=$(find "$dir" -exec ls -la {} \; 2>/dev/null | grep -c "+")
        echo "Archivos/directorios con ACL extendidas: $count"
        
        # Mostrar usuarios únicos con permisos ACL
        echo "Usuarios con permisos específicos:"
        find "$dir" -exec getfacl {} \; 2>/dev/null | \
        grep "^user:" | grep -v "user::" | \
        cut -d: -f2 | sort -u | sed 's/^/  /'
        
        echo "Grupos con permisos específicos:"
        find "$dir" -exec getfacl {} \; 2>/dev/null | \
        grep "^group:" | grep -v "group::" | \
        cut -d: -f2 | sort -u | sed 's/^/  /'
    else
        echo "Directorio no existe o no es accesible"
    fi
    echo "---"
}

# Directorios a analizar
DIRS=("/home" "/var/www" "/opt" "/srv")

for dir in "${DIRS[@]}"; do
    show_acl_summary "$dir"
done
```

### Script de Limpieza ACL

```bash
#!/bin/bash
# clean-acl.sh

# ADVERTENCIA: Este script elimina ACL. Usar con precaución.

read -p "¿Estás seguro de que quieres limpiar ACL? (y/N): " confirm
if [[ $confirm != [yY] ]]; then
    echo "Operación cancelada"
    exit 1
fi

# Backup antes de limpiar
echo "Creando backup..."
getfacl -R /home > /tmp/acl-backup-$(date +%Y%m%d_%H%M%S).acl

# Limpiar ACL específicas (ejemplo: usuario que ya no existe)
USUARIO_ELIMINADO="olduser"

find /home -exec setfacl -x u:$USUARIO_ELIMINADO {} \; 2>/dev/null

echo "Limpieza completada para usuario: $USUARIO_ELIMINADO"
```

Esta guía completa te permitirá implementar y gestionar ACL de manera efectiva en Rocky Linux 9 y Ubuntu, proporcionando un control de acceso granular y flexible para tus sistemas.