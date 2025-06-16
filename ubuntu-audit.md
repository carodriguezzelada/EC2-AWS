# Guía de Auditoría para Ubuntu
## Sistema de Monitoreo: auditd + SSH Logging

### Requisitos previos
- Ubuntu 20.04 LTS / 22.04 LTS / 24.04 LTS
- Acceso root o sudo
- Desarrolladores con usuarios individuales configurados

---

## 1. INSTALACIÓN Y CONFIGURACIÓN DE AUDITD

### Instalar auditd
```bash
# Actualizar sistema
sudo apt update

# Instalar auditd y herramientas
sudo apt install -y auditd audispd-plugins

# Habilitar e iniciar el servicio
sudo systemctl enable auditd
sudo systemctl start auditd

# Verificar estado
sudo systemctl status auditd
```

### Configurar auditd principal
```bash
# Editar configuración principal
sudo nano /etc/audit/auditd.conf
```

**Configuración recomendada en `/etc/audit/auditd.conf`:**
```conf
# Directorio de logs
log_file = /var/log/audit/audit.log

# Tamaño máximo de archivos de log (MB)
max_log_file = 100

# Número de archivos de log a rotar
num_logs = 10

# Acción cuando se llena el disco
disk_full_action = ROTATE

# Acción cuando hay poco espacio
space_left_action = SYSLOG

# Acción cuando hay error de escritura
disk_error_action = SYSLOG

# Formato de log (RAW es más detallado)
log_format = RAW

# Frecuencia de flush a disco
freq = 50

# Prioridad del proceso auditd
priority_boost = 4
```

### Configurar reglas de auditoría
```bash
# Crear archivo de reglas personalizadas
sudo nano /etc/audit/rules.d/developers.rules
```

**Contenido de `/etc/audit/rules.d/developers.rules`:**
```bash
# === REGLAS DE AUDITORÍA PARA DESARROLLADORES ===

# Eliminar reglas existentes
-D

# Buffer para eventos
-b 8192

# Fallos cuando el buffer está lleno
-f 1

# === COMANDOS EJECUTADOS ===
# Auditar todas las llamadas execve (comandos ejecutados)
-a always,exit -F arch=b64 -S execve -k commands
-a always,exit -F arch=b32 -S execve -k commands

# === ESCALACIÓN DE PRIVILEGIOS ===
# Monitorear uso de sudo
-w /usr/bin/sudo -p x -k sudo_usage
-w /etc/sudoers -p wa -k sudo_config
-w /etc/sudoers.d/ -p wa -k sudo_config

# Monitorear comando su
-w /bin/su -p x -k su_usage

# === ARCHIVOS SENSIBLES ===
# Configuraciones críticas del sistema
-w /etc/passwd -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/shadow -p wa -k shadow_modification
-w /etc/gshadow -p wa -k shadow_modification

# Configuración SSH
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /etc/ssh/ -p wa -k ssh_config

# Logs del sistema
-w /var/log/ -p wa -k log_access

# === CONFIGURACIONES DE RED ===
# Monitorear cambios en configuración de red
-w /etc/hosts -p wa -k network_config
-w /etc/resolv.conf -p wa -k dns_config
-w /etc/netplan/ -p wa -k network_config

# === PROCESOS Y SERVICIOS ===
# Monitorear systemctl
-w /bin/systemctl -p x -k service_management
-w /usr/bin/systemctl -p x -k service_management

# === TRANSFERENCIA DE ARCHIVOS ===
# Monitorear herramientas de transferencia
-w /usr/bin/scp -p x -k file_transfer
-w /usr/bin/sftp -p x -k file_transfer
-w /usr/bin/rsync -p x -k file_transfer
-w /usr/bin/wget -p x -k file_transfer
-w /usr/bin/curl -p x -k file_transfer

# === INSTALACIÓN DE SOFTWARE ===
# Monitorear instalación de paquetes
-w /usr/bin/apt -p x -k package_management
-w /usr/bin/apt-get -p x -k package_management
-w /usr/bin/dpkg -p x -k package_management
-w /usr/bin/snap -p x -k package_management

# === CONFIGURACIONES ESPECÍFICAS DE UBUNTU ===
# Monitorear cambios en AppArmor
-w /etc/apparmor/ -p wa -k apparmor_config
-w /etc/apparmor.d/ -p wa -k apparmor_config

# Hacer reglas inmutables (opcional, requiere reinicio para cambios)
# -e 2
```

### Aplicar configuración
```bash
# Reiniciar auditd para aplicar cambios
sudo systemctl restart auditd

# Verificar reglas aplicadas
sudo auditctl -l

# Verificar estado
sudo auditctl -s
```

---

## 2. CONFIGURACIÓN DE SSH LOGGING DETALLADO

### Configurar SSH para logging extendido
```bash
# Hacer backup de configuración SSH
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Editar configuración SSH
sudo nano /etc/ssh/sshd_config
```

**Agregar/modificar en `/etc/ssh/sshd_config`:**
```bash
# Logging detallado
LogLevel VERBOSE
SyslogFacility AUTH

# Logs de login/logout
PrintLastLog yes
PrintMotd yes

# Banner informativo (opcional)
Banner /etc/ssh/banner

# Configuraciones de sesión
ClientAliveInterval 300
ClientAliveCountMax 2

# Logging de SFTP (si se usa)
Subsystem sftp /usr/lib/openssh/sftp-server -l INFO -f AUTH
```

### Crear banner informativo (opcional)
```bash
sudo nano /etc/ssh/banner
```

**Contenido del banner:**
```
==========================================
SERVIDOR QA - ACCESO MONITOREADO
==========================================
* Todas las actividades son registradas
* Use solo para propósitos autorizados
* Contacte al administrador para dudas
==========================================
```

### Configurar rsyslog para SSH
```bash
# Crear configuración específica para SSH
sudo nano /etc/rsyslog.d/50-ssh-audit.conf
```

**Contenido de `/etc/rsyslog.d/50-ssh-audit.conf`:**
```bash
# === CONFIGURACIÓN SSH AUDIT ===

# Template para logs SSH detallados
$template SSHLogFormat,"%timegenerated% %HOSTNAME% %syslogtag%%msg%\n"

# Logs de autenticación SSH (Ubuntu usa auth en lugar de authpriv)
auth,authpriv.*                    /var/log/ssh-audit.log;SSHLogFormat

# Separar logs de auditd si es necesario
local6.*                          /var/log/audit-commands.log;SSHLogFormat

# Evitar duplicados en syslog
& stop
```

### Reiniciar servicios
```bash
# Reiniciar SSH
sudo systemctl restart ssh

# Reiniciar rsyslog
sudo systemctl restart rsyslog

# Verificar servicios
sudo systemctl status ssh
sudo systemctl status rsyslog
```

---

## 3. SCRIPTS DE MONITOREO

### Script para analizar comandos ejecutados
```bash
sudo nano /usr/local/bin/audit-commands.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Script para analizar comandos ejecutados por desarrolladores

LOG_FILE="/var/log/audit/audit.log"
OUTPUT_FILE="/var/log/audit-summary.log"

echo "=== RESUMEN DE COMANDOS EJECUTADOS ===" > $OUTPUT_FILE
echo "Generado: $(date)" >> $OUTPUT_FILE
echo "========================================" >> $OUTPUT_FILE

# Extraer comandos del día actual
TODAY=$(date +%Y-%m-%d)

echo -e "\n--- COMANDOS POR USUARIO ---" >> $OUTPUT_FILE

# Buscar comandos ejecutados
ausearch -ts $TODAY -k commands 2>/dev/null | \
    ausearch --input-logs --format csv 2>/dev/null | \
    grep EXECVE | \
    while IFS=',' read -r node type timestamp serial uid user cmd rest; do
        if [ ! -z "$cmd" ] && [ ! -z "$user" ]; then
            # Limpiar y formatear timestamp
            clean_time=$(echo $timestamp | tr -d '"')
            formatted_time=$(date -d "@${clean_time}" '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo $clean_time)
            
            # Limpiar comando
            clean_cmd=$(echo $cmd | tr -d '"' | head -c 100)
            clean_user=$(echo $user | tr -d '"')
            
            echo "$formatted_time - Usuario: $clean_user - Comando: $clean_cmd" >> $OUTPUT_FILE
        fi
    done

# Método alternativo si el anterior no funciona
if [ ! -s $OUTPUT_FILE ] || [ $(wc -l < $OUTPUT_FILE) -lt 10 ]; then
    echo -e "\n--- COMANDOS (MÉTODO ALTERNATIVO) ---" >> $OUTPUT_FILE
    ausearch -ts $TODAY -k commands 2>/dev/null | \
        grep -E "type=EXECVE|type=USER_CMD" | \
        while read line; do
            if echo "$line" | grep -q "EXECVE"; then
                USER=$(echo "$line" | grep -o 'uid=[0-9]*' | cut -d= -f2)
                CMD=$(echo "$line" | grep -o 'a0="[^"]*"' | cut -d'"' -f2)
                TIME=$(echo "$line" | grep -o 'msg=audit([^)]*' | cut -d'(' -f2 | cut -d: -f1)
                
                if [ ! -z "$CMD" ] && [ ! -z "$USER" ]; then
                    USER_NAME=$(getent passwd $USER 2>/dev/null | cut -d: -f1)
                    [ -z "$USER_NAME" ] && USER_NAME="UID:$USER"
                    echo "$(date -d @$TIME '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo $TIME) - Usuario: $USER_NAME - Comando: $CMD" >> $OUTPUT_FILE
                fi
            fi
        done
fi

echo -e "\n--- ESCALACIÓN DE PRIVILEGIOS ---" >> $OUTPUT_FILE
ausearch -ts $TODAY -k sudo_usage 2>/dev/null | \
    grep -v "type=PATH" | \
    sed 's/^/SUDO: /' >> $OUTPUT_FILE

echo -e "\n--- ACCESO A ARCHIVOS SENSIBLES ---" >> $OUTPUT_FILE
ausearch -ts $TODAY -k log_access,user_modification,shadow_modification 2>/dev/null | \
    grep -v "type=PATH" | \
    sed 's/^/FILE_ACCESS: /' >> $OUTPUT_FILE

echo -e "\n--- INSTALACIÓN DE SOFTWARE ---" >> $OUTPUT_FILE
ausearch -ts $TODAY -k package_management 2>/dev/null | \
    grep -v "type=PATH" | \
    sed 's/^/PACKAGE: /' >> $OUTPUT_FILE

# Hacer el script ejecutable
chmod +x /usr/local/bin/audit-commands.sh
```

### Script para analizar conexiones SSH
```bash
sudo nano /usr/local/bin/ssh-sessions.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Script para analizar sesiones SSH

SSH_LOG="/var/log/ssh-audit.log"
AUTH_LOG="/var/log/auth.log"
OUTPUT_FILE="/var/log/ssh-summary.log"

echo "=== RESUMEN DE CONEXIONES SSH ===" > $OUTPUT_FILE
echo "Generado: $(date)" >> $OUTPUT_FILE
echo "===================================" >> $OUTPUT_FILE

# Fecha actual en formato de log
TODAY=$(date +"%b %d")
TODAY_ALT=$(date +"%b  %d")  # Para días de un dígito

echo -e "\n--- CONEXIONES EXITOSAS ---" >> $OUTPUT_FILE

# Buscar en ssh-audit.log primero
if [ -f "$SSH_LOG" ]; then
    grep -E "($TODAY|$TODAY_ALT)" $SSH_LOG | grep "Accepted" | \
        awk '{print $1 " " $2 " " $3 " - Usuario: " $9 " - IP: " $11 " - Puerto: " $13}' >> $OUTPUT_FILE
fi

# Buscar también en auth.log como respaldo
if [ -f "$AUTH_LOG" ]; then
    grep -E "($TODAY|$TODAY_ALT)" $AUTH_LOG | grep "sshd.*Accepted" | \
        awk '{print $1 " " $2 " " $3 " - Usuario: " $9 " - IP: " $11}' >> $OUTPUT_FILE
fi

echo -e "\n--- DESCONEXIONES ---" >> $OUTPUT_FILE
if [ -f "$SSH_LOG" ]; then
    grep -E "($TODAY|$TODAY_ALT)" $SSH_LOG | grep -E "(session closed|Connection closed)" | \
        awk '{print $1 " " $2 " " $3 " - " $0}' >> $OUTPUT_FILE
fi

if [ -f "$AUTH_LOG" ]; then
    grep -E "($TODAY|$TODAY_ALT)" $AUTH_LOG | grep -E "sshd.*session closed|sshd.*Connection closed" | \
        awk '{print $1 " " $2 " " $3 " - " $0}' >> $OUTPUT_FILE
fi

echo -e "\n--- INTENTOS FALLIDOS ---" >> $OUTPUT_FILE
for log_file in "$SSH_LOG" "$AUTH_LOG"; do
    if [ -f "$log_file" ]; then
        grep -E "($TODAY|$TODAY_ALT)" $log_file | grep -E "(Failed|Invalid|authentication failure)" | \
            awk '{print $1 " " $2 " " $3 " - " $0}' >> $OUTPUT_FILE
    fi
done

echo -e "\n--- ESTADÍSTICAS DEL DÍA ---" >> $OUTPUT_FILE

# Contar conexiones exitosas
TOTAL_CONNECTIONS=0
for log_file in "$SSH_LOG" "$AUTH_LOG"; do
    if [ -f "$log_file" ]; then
        count=$(grep -E "($TODAY|$TODAY_ALT)" $log_file | grep -c "Accepted" 2>/dev/null || echo 0)
        TOTAL_CONNECTIONS=$((TOTAL_CONNECTIONS + count))
    fi
done

# Contar intentos fallidos
FAILED_ATTEMPTS=0
for log_file in "$SSH_LOG" "$AUTH_LOG"; do
    if [ -f "$log_file" ]; then
        count=$(grep -E "($TODAY|$TODAY_ALT)" $log_file | grep -E -c "(Failed|Invalid|authentication failure)" 2>/dev/null || echo 0)
        FAILED_ATTEMPTS=$((FAILED_ATTEMPTS + count))
    fi
done

# Contar usuarios únicos
UNIQUE_USERS=0
for log_file in "$SSH_LOG" "$AUTH_LOG"; do
    if [ -f "$log_file" ]; then
        users=$(grep -E "($TODAY|$TODAY_ALT)" $log_file | grep "Accepted" | awk '{print $9}' | sort | uniq | wc -l)
        [ $users -gt $UNIQUE_USERS ] && UNIQUE_USERS=$users
    fi
done

echo "Total conexiones exitosas: $TOTAL_CONNECTIONS" >> $OUTPUT_FILE
echo "Intentos fallidos: $FAILED_ATTEMPTS" >> $OUTPUT_FILE
echo "Usuarios únicos: $UNIQUE_USERS" >> $OUTPUT_FILE

echo -e "\n--- USUARIOS ACTIVOS ---" >> $OUTPUT_FILE
who >> $OUTPUT_FILE

chmod +x /usr/local/bin/ssh-sessions.sh
```

### Script combinado de monitoreo en tiempo real
```bash
sudo nano /usr/local/bin/live-monitor.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Monitor en tiempo real de actividad

echo "=== MONITOR EN TIEMPO REAL ==="
echo "Presiona Ctrl+C para salir"
echo "=============================="

# Función para limpiar al salir
cleanup() {
    echo -e "\n\nMonitoreo terminado."
    exit 0
}

trap cleanup SIGINT

# Mostrar actividad en tiempo real
tail -f /var/log/audit/audit.log /var/log/ssh-audit.log /var/log/auth.log 2>/dev/null | \
while read line; do
    # Filtrar líneas relevantes
    if echo "$line" | grep -E "(EXECVE|Accepted|Failed|session)" >/dev/null; then
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        
        if echo "$line" | grep "EXECVE" >/dev/null; then
            user=$(echo "$line" | grep -o 'uid=[0-9]*' | cut -d= -f2)
            cmd=$(echo "$line" | grep -o 'a0="[^"]*"' | cut -d'"' -f2)
            user_name=$(getent passwd $user 2>/dev/null | cut -d: -f1)
            [ ! -z "$cmd" ] && echo "[$timestamp] COMANDO: $user_name ejecutó: $cmd"
            
        elif echo "$line" | grep "Accepted" >/dev/null; then
            user=$(echo "$line" | awk '{print $9}')
            ip=$(echo "$line" | awk '{print $11}')
            echo "[$timestamp] LOGIN: $user desde $ip"
            
        elif echo "$line" | grep "session closed" >/dev/null; then
            user=$(echo "$line" | awk '{print $8}')
            echo "[$timestamp] LOGOUT: $user"
            
        elif echo "$line" | grep "Failed" >/dev/null; then
            ip=$(echo "$line" | awk '{print $11}')
            echo "[$timestamp] FALLO: Intento fallido desde $ip"
        fi
    fi
done

chmod +x /usr/local/bin/live-monitor.sh
```

---

## 4. AUTOMATIZACIÓN CON CRON

### Configurar tareas automáticas
```bash
# Editar crontab del sistema
sudo crontab -e
```

**Agregar estas líneas:**
```bash
# Generar reportes de auditoría cada hora
0 * * * * /usr/local/bin/audit-commands.sh >/dev/null 2>&1
30 * * * * /usr/local/bin/ssh-sessions.sh >/dev/null 2>&1

# Reporte diario por email (opcional)
0 8 * * * /usr/local/bin/daily-report.sh

# Limpiar logs antiguos semanalmente
0 2 * * 0 find /var/log -name "audit-*.log" -mtime +30 -delete
0 3 * * 0 find /var/log -name "ssh-*.log" -mtime +30 -delete

# Rotar logs de auditd manualmente si es necesario
0 1 * * * /sbin/service auditd rotate >/dev/null 2>&1
```

### Script de reporte diario opcional
```bash
sudo nano /usr/local/bin/daily-report.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Reporte diario de actividad

REPORT_FILE="/tmp/daily-audit-report.txt"
EMAIL="admin@empresa.com"  # Cambiar por tu email

echo "=== REPORTE DIARIO DE AUDITORÍA ===" > $REPORT_FILE
echo "Fecha: $(date)" >> $REPORT_FILE
echo "Servidor: $(hostname)" >> $REPORT_FILE
echo "====================================" >> $REPORT_FILE

# Ejecutar scripts de monitoreo
/usr/local/bin/audit-commands.sh
/usr/local/bin/ssh-sessions.sh

# Incluir resúmenes en reporte
echo -e "\n--- RESUMEN DE COMANDOS ---" >> $REPORT_FILE
tail -20 /var/log/audit-summary.log >> $REPORT_FILE

echo -e "\n--- RESUMEN DE CONEXIONES ---" >> $REPORT_FILE
tail -20 /var/log/ssh-summary.log >> $REPORT_FILE

# Estadísticas del sistema
echo -e "\n--- ESTADÍSTICAS DEL SISTEMA ---" >> $REPORT_FILE
echo "Usuarios conectados: $(who | wc -l)" >> $REPORT_FILE
echo "Procesos auditd: $(ps aux | grep auditd | grep -v grep | wc -l)" >> $REPORT_FILE
echo "Espacio en logs: $(du -sh /var/log/audit/)" >> $REPORT_FILE

# Enviar por email (requiere configurar sendmail/postfix)
# mail -s "Reporte Diario Auditoría - $(hostname)" $EMAIL < $REPORT_FILE

# Limpiar archivo temporal
rm -f $REPORT_FILE

chmod +x /usr/local/bin/daily-report.sh
```

---

## 5. CONFIGURACIÓN DE LOGROTATE

### Configurar rotación de logs
```bash
sudo nano /etc/logrotate.d/audit-custom
```

**Contenido:**
```bash
/var/log/ssh-audit.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 syslog adm
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}

/var/log/audit-commands.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 root root
}

/var/log/audit-summary.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 640 root root
}

/var/log/ssh-summary.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 640 root root
}
```

### Configurar logrotate para auditd
```bash
sudo nano /etc/logrotate.d/auditd
```

**Modificar o crear:**
```bash
/var/log/audit/*.log {
    weekly
    rotate 10
    compress
    delaycompress
    missingok
    notifempty
    create 640 root root
    postrotate
        /sbin/service auditd restart > /dev/null 2>&1 || true
    endscript
}
```

---

## 6. COMANDOS ÚTILES PARA CONSULTA

### Consultas de auditd
```bash
# Ver comandos ejecutados hoy
ausearch -ts today -k commands

# Ver comandos de un usuario específico
ausearch -ts today -ui username

# Ver uso de sudo
ausearch -ts today -k sudo_usage

# Ver acceso a archivos sensibles
ausearch -ts today -k log_access

# Ver instalación de paquetes
ausearch -ts today -k package_management

# Buscar por rango de fechas
ausearch -ts 01/15/2025 -te 01/16/2025

# Ver eventos en tiempo real
autail -f

# Buscar por proceso específico
ausearch -p 1234

# Ver todos los tipos de eventos
ausearch -ts today --format csv | cut -d',' -f2 | sort | uniq
```

### Consultas de SSH
```bash
# Ver conexiones SSH del día
grep "$(date +"%b %d")" /var/log/ssh-audit.log | grep Accepted

# Ver también en auth.log
grep "$(date +"%b %d")" /var/log/auth.log | grep "sshd.*Accepted"

# Ver intentos fallidos
grep "$(date +"%b %d")" /var/log/auth.log | grep "sshd.*Failed"

# Ver último login de usuarios
lastlog

# Ver usuarios conectados actualmente
who
w

# Historial de logins
last
last -f /var/log/wtmp

# Ver intentos de login por IP
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr
```

### Consultas específicas de Ubuntu
```bash
# Ver logs de AppArmor
ausearch -ts today -k apparmor_config

# Ver cambios en configuración de red
ausearch -ts today -k network_config

# Ver actividad de snap
ausearch -ts today -k package_management | grep snap

# Monitorear journald también
journalctl -f -u ssh
journalctl -f -u auditd
```

---

## 7. CONFIGURACIÓN DE ALERTAS (OPCIONAL)

### Configurar alertas por email
```bash
sudo nano /usr/local/bin/alert-monitor.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Monitor de alertas críticas

ALERT_EMAIL="admin@empresa.com"
LOG_FILE="/var/log/alert-monitor.log"

# Función para enviar alerta
send_alert() {
    local subject="$1"
    local message="$2"
    
    echo "$(date): $subject - $message" >> $LOG_FILE
    
    # Enviar email (requiere postfix/sendmail configurado)
    # echo "$message" | mail -s "$subject - $(hostname)" $ALERT_EMAIL
    
    # Alternativamente, escribir a syslog
    logger -t AUDIT_ALERT "$subject: $message"
}

# Monitorear intentos de acceso root
if grep "$(date +"%b %d")" /var/log/auth.log | grep -q "Failed password for root"; then
    count=$(grep "$(date +"%b %d")" /var/log/auth.log | grep -c "Failed password for root")
    if [ $count -gt 5 ]; then
        send_alert "ALERTA SEGURIDAD" "Más de 5 intentos fallidos de login como root hoy ($count intentos)"
    fi
fi

# Monitorear uso de sudo en horarios no laborales
current_hour=$(date +%H)
if [ $current_hour -lt 8 ] || [ $current_hour -gt 18 ]; then
    sudo_count=$(ausearch -ts today -k sudo_usage 2>/dev/null | grep -c "type=USER_CMD" || echo 0)
    if [ $sudo_count -gt 0 ]; then
        send_alert "ACTIVIDAD FUERA DE HORARIO" "Uso de sudo detectado fuera de horario laboral ($sudo_count comandos)"
    fi
fi

# Monitorear cambios en archivos críticos
if ausearch -ts today -k user_modification,shadow_modification 2>/dev/null | grep -q "type=PATH"; then
    send_alert "MODIFICACIÓN CRÍTICA" "Cambios detectados en archivos de usuarios/contraseñas"
fi

chmod +x /usr/local/bin/alert-monitor.sh
```

### Agregar a cron para ejecutar cada 30 minutos
```bash
# Agregar a crontab
*/30 * * * * /usr/local/bin/alert-monitor.sh >/dev/null 2>&1
```

---

## 8. VERIFICACIÓN Y TESTING

### Verificar que todo funciona
```bash
# 1. Verificar auditd
sudo auditctl -l
sudo systemctl status auditd

# 2. Verificar reglas aplicadas
sudo auditctl -s

# 3. Verificar SSH logging
sudo systemctl status ssh
tail -f /var/log/ssh-audit.log &
# (conectarse por SSH desde otra terminal)

# 4. Probar scripts de monitoreo
sudo /usr/local/bin/audit-commands.sh
sudo /usr/local/bin/ssh-sessions.sh

# 5. Verificar archivos generados
ls -la /var/log/audit-summary.log
ls -la /var/log/ssh-summary.log

# 6. Verificar rotación de logs
sudo logrotate -d /etc/logrotate.d/audit-custom
```

### Test de funcionalidad completo
```bash
# Como usuario de prueba, ejecutar:
echo "Probando auditoría" > /tmp/test.txt
ls -la /etc/passwd
sudo whoami
apt list --installed | head -5

# Luego verificar que se registró:
sudo ausearch -ts today -k commands | grep -E "(echo|ls|sudo|apt)"
grep "$(date +"%b %d")" /var/log/ssh-audit.log | tail -5
```

### Monitoreo en tiempo real
```bash
# Terminal 1: Monitor en tiempo real
sudo /usr/local/bin/live-monitor.sh

# Terminal 2: Ejecutar comandos de prueba como diferentes usuarios
su - dev1 -c "ls -la"
su - dev2 -c "sudo whoami"
```

---

## 9. TROUBLESHOOTING

### Problemas comunes en Ubuntu

**auditd no registra comandos:**
```bash
# Verificar que auditd está corriendo
sudo systemctl status auditd

# Verificar reglas cargadas
sudo auditctl -l

# Verificar espacio en disco
df -h /var/log

# Verificar permisos
ls -la /var/log/audit/

# Reiniciar auditd si es necesario
sudo systemctl restart auditd
```

**SSH logs no aparecen:**
```bash
# Verificar configuración SSH
sudo sshd -T | grep -i log

# Verificar que SSH usa el facility correcto
grep -i syslogfacility /etc/ssh/sshd_config

# Verificar rsyslog
sudo systemctl status rsyslog
sudo journalctl -u rsyslog -f

# Verificar permisos de archivo de log
ls -la /var/log/ssh-audit.log
```

**Scripts no ejecutan desde cron:**
```bash
# Verificar cron está corriendo
sudo systemctl status cron

# Verificar logs de cron
grep audit /var/log/syslog
sudo journalctl -u cron

# Verificar permisos de scripts
ls -la /usr/local/bin/audit-*.sh

# Probar script manualmente
sudo /usr/local/bin/audit-commands.sh
```

**Logs de auth.log no rotan:**
```bash
# Verificar configuración de logrotate
sudo logrotate -d /etc/logrotate.d/rsyslog

# Ejecutar rotación manual
sudo logrotate -f /etc/logrotate.d/rsyslog
```

**Ausearch no encuentra eventos:**
```bash
# Verificar formato de fecha
ausearch -ts $(date +%m/%d/%Y)

# Verificar con timestamp Unix
ausearch -ts $(date -d "today 00:00" +%s)

# Buscar todos los eventos del día
ausearch -ts today

# Verificar índices de audit
sudo aureport
```

---

## 10. OPTIMIZACIÓN Y MANTENIMIENTO

### Optimizar rendimiento
```bash
# Ajustar buffer de auditd si hay pérdida de eventos
sudo nano /etc/audit/auditd.conf
# Incrementar: buffer_size = 16384

# Verificar estadísticas de auditd
sudo auditctl -s

# Si hay pérdidas, ajustar:
sudo auditctl -b 16384
```

### Mantenimiento regular
```bash
# Script de mantenimiento semanal
sudo nano /usr/local/bin/audit-maintenance.sh
```

**Contenido:**
```bash
#!/bin/bash
# Mantenimiento semanal de auditoría

echo "=== MANTENIMIENTO AUDITORÍA $(date) ===" >> /var/log/audit-maintenance.log

# Verificar espacio en disco
DISK_USAGE=$(df /var/log | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "ADVERTENCIA: Uso de disco alto: $DISK_USAGE%" >> /var/log/audit-maintenance.log
    # Forzar rotación si es necesario
    /usr/sbin/logrotate -f /etc/logrotate.conf
fi

# Verificar que auditd está funcionando
if ! systemctl is-active --quiet auditd; then
    echo "ERROR: auditd no está corriendo" >> /var/log/audit-maintenance.log
    systemctl start auditd
fi

# Generar estadísticas semanales
echo "Estadísticas de la semana:" >> /var/log/audit-maintenance.log
ausearch -ts week-ago | wc -l >> /var/log/audit-maintenance.log

chmod +x /usr/local/bin/audit-maintenance.sh

# Agregar a cron para ejecutar semanalmente
0 6 * * 0 /usr/local/bin/audit-maintenance.sh
```

---

## UBICACIONES IMPORTANTES DE ARCHIVOS

- **Configuración auditd:** `/etc/audit/auditd.conf`
- **Reglas auditd:** `/etc/audit/rules.d/developers.rules`
- **Logs auditd:** `/var/log/audit/audit.log`
- **Configuración SSH:** `/etc/ssh/sshd_config`
- **Logs SSH personalizados:** `/var/log/ssh-audit.log`
- **Logs de autenticación:** `/var/log/auth.log`
- **Scripts de monitoreo:** `/usr/local/bin/`
- **Reportes generados:** `/var/log/audit-summary.log` y `/var/log/ssh-summary.log`
- **Configuración rsyslog:** `/etc/rsyslog.d/50-ssh-audit.conf`
- **Configuración logrotate:** `/etc/logrotate.d/audit-custom`