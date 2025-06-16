# Guía de Auditoría para Rocky Linux
## Sistema de Monitoreo: auditd + SSH Logging

### Requisitos previos
- Rocky Linux 8/9
- Acceso root o sudo
- Desarrolladores con usuarios individuales configurados

---

## 1. INSTALACIÓN Y CONFIGURACIÓN DE AUDITD

### Instalar auditd
```bash
# Instalar auditd y herramientas
sudo dnf install -y audit audit-libs

# Habilitar e iniciar el servicio
sudo systemctl enable auditd
sudo systemctl start auditd

# Verificar estado
sudo systemctl status auditd
```

### Configurar auditd principal
```bash
# Editar configuración principal
sudo vi /etc/audit/auditd.conf
```

**Configuración recomendada en `/etc/audit/auditd.conf`:**
```conf
# Tamaño máximo de archivos de log (MB)
max_log_file = 100

# Número de archivos de log a rotar
num_logs = 10

# Acción cuando se llena el disco
disk_full_action = ROTATE

# Acción cuando hay error de escritura
disk_error_action = SYSLOG

# Formato de log (RAW es más detallado)
log_format = RAW

# Frecuencia de flush a disco
freq = 50
```

### Configurar reglas de auditoría
```bash
# Crear archivo de reglas personalizadas
sudo vi /etc/audit/rules.d/developers.rules
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

# Monitorear comando su
-w /usr/bin/su -p x -k su_usage

# === ARCHIVOS SENSIBLES ===
# Configuraciones críticas del sistema
-w /etc/passwd -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/shadow -p wa -k shadow_modification
-w /etc/ssh/sshd_config -p wa -k ssh_config

# Logs del sistema
-w /var/log/ -p wa -k log_access

# === CONEXIONES DE RED ===
# Monitorear cambios en configuración de red
-w /etc/hosts -p wa -k network_config
-w /etc/resolv.conf -p wa -k dns_config

# === PROCESOS Y SERVICIOS ===
# Monitorear systemctl
-w /usr/bin/systemctl -p x -k service_management

# === TRANSFERENCIA DE ARCHIVOS ===
# Monitorear herramientas de transferencia
-w /usr/bin/scp -p x -k file_transfer
-w /usr/bin/sftp -p x -k file_transfer
-w /usr/bin/rsync -p x -k file_transfer

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
sudo vi /etc/ssh/sshd_config
```

**Agregar/modificar en `/etc/ssh/sshd_config`:**
```bash
# Logging detallado
LogLevel VERBOSE
SyslogFacility AUTHPRIV

# Logs de login/logout
PrintLastLog yes
PrintMotd yes

# Configuraciones de sesión
ClientAliveInterval 300
ClientAliveCountMax 2

# Logging de comandos (opcional, más detallado)
# Subsystem sftp /usr/libexec/openssh/sftp-server -l INFO
```

### Configurar rsyslog para SSH
```bash
# Crear configuración específica para SSH
sudo vi /etc/rsyslog.d/ssh-audit.conf
```

**Contenido de `/etc/rsyslog.d/ssh-audit.conf`:**
```bash
# === CONFIGURACIÓN SSH AUDIT ===

# Template para logs SSH detallados
$template SSHLogFormat,"%timegenerated% %HOSTNAME% %syslogtag%%msg%\n"

# Logs de autenticación SSH
authpriv.*                    /var/log/ssh-audit.log;SSHLogFormat

# Logs de auditd a archivo separado
local6.*                      /var/log/audit-commands.log;SSHLogFormat

# Rotar logs diariamente
$RotateFiles 30
$RotateInterval daily
```

### Reiniciar servicios
```bash
# Reiniciar SSH
sudo systemctl restart sshd

# Reiniciar rsyslog
sudo systemctl restart rsyslog

# Verificar servicios
sudo systemctl status sshd
sudo systemctl status rsyslog
```

---

## 3. SCRIPTS DE MONITOREO

### Script para analizar comandos ejecutados
```bash
sudo vi /usr/local/bin/audit-commands.sh
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
ausearch -ts $TODAY -k commands 2>/dev/null | \
    grep -E "(type=EXECVE|type=USER_CMD)" | \
    while read line; do
        if echo "$line" | grep -q "EXECVE"; then
            USER=$(echo "$line" | grep -o 'uid=[0-9]*' | cut -d= -f2)
            CMD=$(echo "$line" | grep -o 'a0="[^"]*"' | cut -d'"' -f2)
            TIME=$(echo "$line" | grep -o 'msg=audit([^)]*' | cut -d'(' -f2 | cut -d: -f1)
            
            if [ ! -z "$CMD" ] && [ ! -z "$USER" ]; then
                USER_NAME=$(getent passwd $USER | cut -d: -f1)
                echo "$(date -d @$TIME '+%Y-%m-%d %H:%M:%S') - Usuario: $USER_NAME - Comando: $CMD" >> $OUTPUT_FILE
            fi
        fi
    done

echo -e "\n--- ESCALACIÓN DE PRIVILEGIOS ---" >> $OUTPUT_FILE
ausearch -ts $TODAY -k sudo_usage 2>/dev/null | \
    grep -v "type=PATH" | \
    sed 's/^/SUDO: /' >> $OUTPUT_FILE

echo -e "\n--- ACCESO A ARCHIVOS SENSIBLES ---" >> $OUTPUT_FILE
ausearch -ts $TODAY -k log_access,user_modification,shadow_modification 2>/dev/null | \
    grep -v "type=PATH" | \
    sed 's/^/FILE_ACCESS: /' >> $OUTPUT_FILE

chmod +x /usr/local/bin/audit-commands.sh
```

### Script para analizar conexiones SSH
```bash
sudo vi /usr/local/bin/ssh-sessions.sh
```

**Contenido del script:**
```bash
#!/bin/bash
# Script para analizar sesiones SSH

SSH_LOG="/var/log/ssh-audit.log"
OUTPUT_FILE="/var/log/ssh-summary.log"

echo "=== RESUMEN DE CONEXIONES SSH ===" > $OUTPUT_FILE
echo "Generado: $(date)" >> $OUTPUT_FILE
echo "===================================" >> $OUTPUT_FILE

# Conexiones del día actual
TODAY=$(date +"%b %d")

echo -e "\n--- CONEXIONES EXITOSAS ---" >> $OUTPUT_FILE
grep "$TODAY" $SSH_LOG | grep "Accepted" | \
    awk '{print $1 " " $2 " " $3 " - Usuario: " $9 " - IP: " $11 " - Puerto: " $13}' >> $OUTPUT_FILE

echo -e "\n--- DESCONEXIONES ---" >> $OUTPUT_FILE
grep "$TODAY" $SSH_LOG | grep "session closed" | \
    awk '{print $1 " " $2 " " $3 " - Usuario: " $8}' >> $OUTPUT_FILE

echo -e "\n--- INTENTOS FALLIDOS ---" >> $OUTPUT_FILE
grep "$TODAY" $SSH_LOG | grep -E "(Failed|Invalid)" | \
    awk '{print $1 " " $2 " " $3 " - " $0}' >> $OUTPUT_FILE

echo -e "\n--- ESTADÍSTICAS DEL DÍA ---" >> $OUTPUT_FILE
TOTAL_CONNECTIONS=$(grep "$TODAY" $SSH_LOG | grep "Accepted" | wc -l)
FAILED_ATTEMPTS=$(grep "$TODAY" $SSH_LOG | grep -E "(Failed|Invalid)" | wc -l)
UNIQUE_USERS=$(grep "$TODAY" $SSH_LOG | grep "Accepted" | awk '{print $9}' | sort | uniq | wc -l)

echo "Total conexiones exitosas: $TOTAL_CONNECTIONS" >> $OUTPUT_FILE
echo "Intentos fallidos: $FAILED_ATTEMPTS" >> $OUTPUT_FILE
echo "Usuarios únicos: $UNIQUE_USERS" >> $OUTPUT_FILE

chmod +x /usr/local/bin/ssh-sessions.sh
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
0 * * * * /usr/local/bin/audit-commands.sh
30 * * * * /usr/local/bin/ssh-sessions.sh

# Limpiar logs antiguos semanalmente
0 2 * * 0 find /var/log -name "audit-*.log" -mtime +30 -delete
0 3 * * 0 find /var/log -name "ssh-*.log" -mtime +30 -delete
```

---

## 5. CONFIGURACIÓN DE LOGROTATE

### Configurar rotación de logs
```bash
sudo vi /etc/logrotate.d/audit-custom
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
    postrotate
        /bin/kill -HUP `cat /var/run/rsyslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}

/var/log/audit-commands.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
}

/var/log/audit-summary.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
}

/var/log/ssh-summary.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
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

# Buscar por rango de fechas
ausearch -ts 01/15/2025 -te 01/16/2025

# Ver eventos en tiempo real
autail -f
```

### Consultas de SSH
```bash
# Ver conexiones SSH del día
grep "$(date +"%b %d")" /var/log/ssh-audit.log | grep Accepted

# Ver intentos fallidos
grep "$(date +"%b %d")" /var/log/ssh-audit.log | grep Failed

# Ver último login de usuarios
lastlog

# Ver usuarios conectados actualmente
who

# Historial de logins
last
```

---

## 7. VERIFICACIÓN Y TESTING

### Verificar que todo funciona
```bash
# 1. Verificar auditd
sudo auditctl -l
sudo ausearch -ts today -k commands | head -10

# 2. Verificar SSH logging
tail -f /var/log/ssh-audit.log

# 3. Probar scripts de monitoreo
sudo /usr/local/bin/audit-commands.sh
sudo /usr/local/bin/ssh-sessions.sh

# 4. Verificar archivos generados
ls -la /var/log/audit-summary.log
ls -la /var/log/ssh-summary.log
```

### Test de funcionalidad
```bash
# Como usuario de prueba, ejecutar:
ls -la
sudo whoami
cat /etc/passwd

# Luego verificar que se registró:
sudo ausearch -ts today -k commands | grep "ls -la"
```

---

## 8. TROUBLESHOOTING

### Problemas comunes

**auditd no registra comandos:**
```bash
# Verificar reglas
sudo auditctl -l

# Verificar espacio en disco
df -h /var/log

# Verificar permisos
ls -la /var/log/audit/
```

**SSH logs no aparecen:**
```bash
# Verificar configuración SSH
sudo sshd -T | grep -i log

# Verificar rsyslog
sudo systemctl status rsyslog
tail -f /var/log/messages
```

**Scripts no ejecutan:**
```bash
# Verificar permisos
ls -la /usr/local/bin/audit-commands.sh

# Verificar cron
sudo systemctl status crond
sudo grep audit /var/log/cron
```

---

## UBICACIONES IMPORTANTES DE ARCHIVOS

- **Configuración auditd:** `/etc/audit/auditd.conf`
- **Reglas auditd:** `/etc/audit/rules.d/developers.rules`
- **Logs auditd:** `/var/log/audit/audit.log`
- **Configuración SSH:** `/etc/ssh/sshd_config`
- **Logs SSH personalizados:** `/var/log/ssh-audit.log`
- **Scripts de monitoreo:** `/usr/local/bin/`
- **Reportes generados:** `/var/log/audit-summary.log` y `/var/log/ssh-summary.log`