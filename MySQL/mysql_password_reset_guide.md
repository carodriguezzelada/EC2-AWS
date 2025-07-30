# Instructivo: Restablecimiento de Contraseña MySQL Root

## CONTRASEÑA GENERADA
```
Usuario: root
Contraseña: nK8#mQ2$vL9@xR4!pW6
```

---

## MÉTODO 1: USANDO mysqld_safe (ESTÁNDAR)

### PASO 1: Detener MySQL completamente
```bash
sudo systemctl stop mysql
sudo systemctl stop mysqld

# Verificar que no hay procesos corriendo
sudo ps aux | grep mysql
sudo pkill mysqld  # Solo si hay procesos residuales
```

### PASO 2: Iniciar MySQL en modo seguro
```bash
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

**Salida esperada:**
```
Logging to '/var/log/mysqld.log'.
Starting mysqld daemon with databases from /var/lib/mysql
```

### PASO 3: Conectar sin contraseña
```bash
mysql -u root
```

### PASO 4: Cambiar contraseña dentro de MySQL
```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nK8#mQ2$vL9@xR4!pW6';
FLUSH PRIVILEGES;
EXIT;
```

### PASO 5: Detener modo seguro y reiniciar normal
```bash
# Detener procesos temporales
sudo pkill mysqld_safe
sudo pkill mysqld

# Esperar a que terminen
sleep 3

# Reiniciar MySQL normalmente
sudo systemctl start mysql
```

### PASO 6: Verificar nueva contraseña
```bash
mysql -u root -p
# Introducir: nK8#mQ2$vL9@xR4!pW6
```

---

## MÉTODO 2: SI mysqld_safe NO ESTÁ DISPONIBLE

### PASO 1: Detener MySQL
```bash
sudo systemctl stop mysql
```

### PASO 2: Crear archivo de configuración temporal
```bash
sudo nano /tmp/mysql-reset.cnf
```

**Contenido del archivo:**
```ini
[mysqld]
skip-grant-tables
skip-networking
user = mysql
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
pid-file = /var/run/mysqld/mysqld.pid
log-error = /var/log/mysqld.log
```

### PASO 3: Iniciar con configuración temporal
```bash
sudo mysqld --defaults-file=/tmp/mysql-reset.cnf &
```

### PASO 4: Conectar y cambiar contraseña
```bash
sleep 5  # Esperar a que inicie
mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nK8#mQ2$vL9@xR4!pW6';
FLUSH PRIVILEGES;
EXIT;
```

### PASO 5: Limpiar y reiniciar normal
```bash
sudo pkill mysqld
sudo rm /tmp/mysql-reset.cnf
sudo systemctl start mysql
```

---

## MÉTODO 3: USANDO ARCHIVO DE INICIALIZACIÓN

### PASO 1: Crear archivo SQL de reset
```bash
sudo nano /tmp/reset-password.sql
```

**Contenido:**
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nK8#mQ2$vL9@xR4!pW6';
FLUSH PRIVILEGES;
```

### PASO 2: Detener MySQL y ejecutar reset
```bash
sudo systemctl stop mysql
sudo mysqld --init-file=/tmp/reset-password.sql --user=mysql &
```

### PASO 3: Esperar, limpiar y reiniciar
```bash
sleep 10
sudo pkill mysqld
sudo rm /tmp/reset-password.sql
sudo systemctl start mysql
```

---

## MÉTODO 4: USANDO SYSTEMD OVERRIDE

### PASO 1: Crear configuración override
```bash
sudo systemctl edit mysql
```

**Agregar en el editor:**
```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/mysqld --skip-grant-tables --skip-networking --user=mysql
```

### PASO 2: Reiniciar con override
```bash
sudo systemctl start mysql
```

### PASO 3: Conectar y cambiar contraseña
```bash
mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nK8#mQ2$vL9@xR4!pW6';
FLUSH PRIVILEGES;
EXIT;
```

### PASO 4: Eliminar override y reiniciar normal
```bash
sudo systemctl revert mysql
sudo systemctl restart mysql
```

---

## TROUBLESHOOTING COMÚN

### Si no encuentra mysqld_safe:
```bash
# Buscar ubicación
sudo find / -name mysqld_safe 2>/dev/null

# Alternativas comunes
sudo /usr/bin/mysqld_safe --skip-grant-tables --skip-networking &
sudo /usr/local/bin/mysqld_safe --skip-grant-tables --skip-networking &
```

### Si hay errores de permisos:
```bash
sudo chown -R mysql:mysql /var/lib/mysql
sudo chown mysql:mysql /var/log/mysqld.log
sudo chmod 755 /var/lib/mysql
```

### Si MySQL no inicia:
```bash
# Verificar logs
sudo tail -20 /var/log/mysqld.log
sudo tail -20 /var/log/mysql/error.log

# Verificar espacio en disco
df -h

# Verificar procesos MySQL residuales
sudo ps aux | grep mysql
sudo pkill -9 mysqld
```

### Si no puedes conectar después del cambio:
```bash
# Verificar estado del servicio
sudo systemctl status mysql

# Probar conexión local
mysql -u root -p -S /var/lib/mysql/mysql.sock

# Verificar usuarios existentes
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user='root';"
```

---

## VERIFICACIÓN POST-CAMBIO

### Comandos de verificación:
```sql
-- Conectar con nueva contraseña
mysql -u root -p

-- Verificar usuario actual
SELECT USER(), @@hostname;

-- Verificar permisos
SHOW GRANTS;

-- Listar bases de datos
SHOW DATABASES;

-- Verificar plugin de autenticación
SELECT user, host, plugin FROM mysql.user WHERE user = 'root';
```

### Resultado esperado plugin:
```
+------+-----------+-----------------------+
| user | host      | plugin                |
+------+-----------+-----------------------+
| root | localhost | mysql_native_password |
+------+-----------+-----------------------+
```

---

## CREAR USUARIOS ADICIONALES (OPCIONAL)

### Usuario administrativo sin contraseña (desarrollo):
```sql
CREATE USER 'admin'@'localhost' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
```

### Usuario administrativo con contraseña (producción):
```sql
CREATE USER 'admin'@'%' IDENTIFIED BY 'nK8#mQ2$vL9@xR4!pW6';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
```

### Usuario solo para replicación:
```sql
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
```

### Aplicar cambios:
```sql
FLUSH PRIVILEGES;
```

---

## ORDEN DE PREFERENCIA DE MÉTODOS

1. **MÉTODO 1**: mysqld_safe (más confiable y estándar)
2. **MÉTODO 2**: mysqld con archivo temporal (universal)
3. **MÉTODO 3**: archivo de inicialización (simple)
4. **MÉTODO 4**: systemd override (moderno)

---

## INFORMACIÓN IMPORTANTE

### ⚠️ Precauciones:
- **Nunca** ejecutar skip-grant-tables en producción sin skip-networking
- **Siempre** verificar que no hay conexiones activas antes del reset
- **Hacer backup** de mysql.user antes de cambios masivos

### 🔒 Seguridad:
- La contraseña generada incluye: mayúsculas, minúsculas, números y símbolos
- Longitud: 19 caracteres (muy segura)
- **Cambiar** esta contraseña después de completar la configuración si es necesario

### 📝 Documentar:
- **Guardar** la nueva contraseña en un lugar seguro
- **Actualizar** scripts de backup que usen credenciales de root
- **Informar** al equipo sobre el cambio de credenciales

---

## ✅ CHECKLIST FINAL

- [ ] MySQL detenido completamente
- [ ] Método de reset ejecutado sin errores
- [ ] Contraseña cambiada exitosamente
- [ ] MySQL reiniciado en modo normal
- [ ] Conexión con nueva contraseña verificada
- [ ] Usuarios adicionales creados (si necesario)
- [ ] Documentación actualizada
- [ ] Scripts de backup actualizados

**🎯 ¡Contraseña de root restablecida exitosamente!**
