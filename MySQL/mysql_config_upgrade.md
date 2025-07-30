# Actualización Segura de Configuración MySQL Existente

## CONFIGURACIÓN MEJORADA PARA PRODUCTIVO (10.69.34.160)

### Archivo: `/etc/my.cnf` o `/etc/mysql/mysql.conf.d/mysqld.cnf`

```ini
[mysqld]
# =====================================
# CONFIGURACIÓN EXISTENTE (MANTENER)
# =====================================
# Replication Server Master A
server-id = 10
replicate-same-server-id = 0
auto-increment-increment = 2
auto-increment-offset = 1

# Logs y binarios (mantener rutas existentes)
relay-log = /var/lib/mysql/node1-relay-bin
relay-log-index = /var/lib/mysql/node1-relay-bin.index
master-info-file = /var/lib/mysql/node1-relay-log.info
log-bin = /var/lib/mysql/node1-bin

# Configuración de socket y data
bind-address = 0.0.0.0
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid

# =====================================
# MEJORAS NUEVAS (AGREGAR)
# =====================================
# Puerto explícito
port = 3306

# Formato de binlog (crítico para consistencia)
binlog-format = ROW

# Seguridad y rendimiento
skip-name-resolve
default-authentication-plugin = mysql_native_password

# Configuración de logs mejorada
expire-logs-days = 10
max-binlog-size = 500M
sync-binlog = 1
log-slave-updates = 1

# =====================================
# OPTIMIZACIÓN DE RENDIMIENTO
# =====================================
# InnoDB (ajustar según RAM disponible)
innodb-buffer-pool-size = 1G
innodb-log-file-size = 256M
innodb-log-buffer-size = 16M
innodb-flush-log-at-trx-commit = 1
innodb-file-per-table = 1
innodb-flush-method = O_DIRECT

# Conexiones y cache
max-connections = 200
max-connect-errors = 10000
table-open-cache = 2000
table-definition-cache = 1400

# Tablas temporales
tmp-table-size = 64M
max-heap-table-size = 64M

# Query lento
slow-query-log = 1
slow-query-log-file = /var/log/mysql/slow.log
long-query-time = 2

# Threads InnoDB
innodb-read-io-threads = 4
innodb-write-io-threads = 4
innodb-io-capacity = 200
```

---

## CONFIGURACIÓN MEJORADA PARA RESERVA (10.69.34.161)

### Archivo: `/etc/my.cnf` o `/etc/mysql/mysql.conf.d/mysqld.cnf`

```ini
[mysqld]
# =====================================
# CONFIGURACIÓN EXISTENTE (MANTENER)
# =====================================
# Replication Server Master B
server-id = 20
replicate-same-server-id = 0
auto-increment-increment = 2
auto-increment-offset = 2

# Logs y binarios (mantener rutas existentes)
relay-log = /var/lib/mysql/node2-relay-bin
relay-log-index = /var/lib/mysql/node2-relay-bin.index
master-info-file = /var/lib/mysql/node2-relay-log.info
log-bin = /var/lib/mysql/node2-bin

# Configuración de socket y data
bind-address = 0.0.0.0
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid

# =====================================
# MEJORAS NUEVAS (AGREGAR)
# =====================================
# Puerto explícito
port = 3306

# Formato de binlog (crítico para consistencia)
binlog-format = ROW

# Seguridad y rendimiento
skip-name-resolve
default-authentication-plugin = mysql_native_password

# Configuración de logs mejorada
expire-logs-days = 10
max-binlog-size = 500M
sync-binlog = 1
log-slave-updates = 1

# =====================================
# OPTIMIZACIÓN DE RENDIMIENTO
# =====================================
# InnoDB (ajustar según RAM disponible)
innodb-buffer-pool-size = 1G
innodb-log-file-size = 256M
innodb-log-buffer-size = 16M
innodb-flush-log-at-trx-commit = 1
innodb-file-per-table = 1
innodb-flush-method = O_DIRECT

# Conexiones y cache
max-connections = 200
max-connect-errors = 10000
table-open-cache = 2000
table-definition-cache = 1400

# Tablas temporales
tmp-table-size = 64M
max-heap-table-size = 64M

# Query lento
slow-query-log = 1
slow-query-log-file = /var/log/mysql/slow.log
long-query-time = 2

# Threads InnoDB
innodb-read-io-threads = 4
innodb-write-io-threads = 4
innodb-io-capacity = 200
```

---

## PROCEDIMIENTO DE ACTUALIZACIÓN SEGURA

### PASO 1: Verificar estado actual de replicación
```sql
-- En ambos servidores
SHOW SLAVE STATUS\G
SHOW MASTER STATUS;
```

**📝 Anotar:** ¿Está `Slave_IO_Running: Yes` y `Slave_SQL_Running: Yes`?

### PASO 2: Hacer backup de configuración actual
```bash
# En ambos servidores
sudo cp /etc/my.cnf /etc/my.cnf.backup.$(date +%Y%m%d_%H%M%S)
# O si usas el otro formato:
sudo cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.backup.$(date +%Y%m%d_%H%M_S)
```

### PASO 3: Actualizar configuración (UN SERVIDOR A LA VEZ)

#### 3A. Empezar con RESERVA primero:
```bash
# En reserva (10.69.34.161)
sudo nano /etc/my.cnf
# Aplicar la configuración mejorada de reserva

# Verificar sintaxis antes de reiniciar
sudo mysqld --help --verbose > /dev/null
```

#### 3B. Reiniciar MySQL en reserva:
```bash
sudo systemctl restart mysql
sudo systemctl status mysql

# Verificar que no hay errores
sudo tail -20 /var/log/mysqld.log
```

#### 3C. Verificar que replicación sigue funcionando:
```sql
-- En reserva
SHOW SLAVE STATUS\G
-- Debe seguir mostrando Slave_IO_Running: Yes y Slave_SQL_Running: Yes
```

### PASO 4: Aplicar mismos cambios a PRODUCTIVO
```bash
# En productivo (10.69.34.160)
sudo nano /etc/my.cnf
# Aplicar la configuración mejorada de productivo

sudo mysqld --help --verbose > /dev/null
sudo systemctl restart mysql
sudo systemctl status mysql
```

### PASO 5: Verificación final
```sql
-- En ambos servidores
SHOW SLAVE STATUS\G
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'skip_name_resolve';
```

---

## MEJORAS QUE OBTIENES

### 🚀 Rendimiento:
- **InnoDB optimizado**: Buffer pool de 1GB para mejor cache
- **Skip name resolve**: Conexiones más rápidas
- **Tablas temporales optimizadas**: Mejor manejo de consultas complejas

### 🔒 Seguridad y estabilidad:
- **binlog-format = ROW**: Mayor consistencia en replicación
- **mysql_native_password**: Evita problemas de autenticación
- **Log rotation**: Los binlogs se eliminan automáticamente después de 10 días

### 📊 Monitoreo:
- **Slow query log**: Identificar consultas lentas
- **Mejor logging**: Logs más organizados y útiles

---

## VALORES AJUSTABLES SEGÚN TU HARDWARE

### Si tus servidores tienen menos de 2GB RAM:
```ini
innodb-buffer-pool-size = 512M
innodb-log-file-size = 128M
max-connections = 100
```

### Si tus servidores tienen más de 4GB RAM:
```ini
innodb-buffer-pool-size = 2G
innodb-log-file-size = 512M
max-connections = 300
```

---

## ⚠️ PRECAUCIONES IMPORTANTES

### NO cambiar estos valores (romperían replicación):
- ❌ `server-id`
- ❌ `auto-increment-increment`
- ❌ `auto-increment-offset`
- ❌ Rutas de `log-bin`, `relay-log`

### Cambios seguros que agregamos:
- ✅ Optimizaciones de rendimiento
- ✅ Configuración de logs
- ✅ Parámetros de seguridad

---

## 🎯 RESULTADO ESPERADO

Después de aplicar estos cambios:
- **Replicación sigue funcionando** exactamente igual
- **Rendimiento mejorado** significativamente
- **Mayor estabilidad** y consistencia
- **Mejor monitoreo** de la base de datos
- **Preparado** para MySQL 8.x moderno

**¿Quieres proceder con esta actualización segura?**
