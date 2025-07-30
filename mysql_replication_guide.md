# Guía Completa: Configuración Replicación MySQL 8.x Master-to-Master

## Datos del Proyecto
- **Productivo**: 10.69.34.160
- **Reserva**: 10.69.34.161  
- **Usuario replicación**: replication
- **Password**: zXsAOLP-w7tVCXsUHWlq

---

## FASE 1: LIMPIEZA Y PREPARACIÓN

### PASO 1: Detener replicación existente
**En ambos servidores (Productivo y Reserva):**
```sql
STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;
```

### PASO 2: Limpiar usuarios de replicación existentes
**En ambos servidores:**
```sql
-- Verificar usuarios existentes
SELECT user, host FROM mysql.user WHERE user = 'replication';

-- Eliminar todos los usuarios de replicación existentes
DROP USER IF EXISTS 'replication'@'%';
DROP USER IF EXISTS 'replication'@'localhost';
DROP USER IF EXISTS 'replication'@'10.69.34.160';
DROP USER IF EXISTS 'replication'@'10.69.34.161';

FLUSH PRIVILEGES;
```

---

## FASE 2: CREACIÓN DE USUARIOS CON AUTENTICACIÓN LEGACY

### PASO 3: Crear usuarios de replicación con mysql_native_password

**En Productivo (10.69.34.160):**
```sql
-- Crear usuario para que Reserva se conecte a Productivo
CREATE USER 'replication'@'10.69.34.161' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'10.69.34.161';

-- Usuario comodín por si hay problemas de resolución DNS
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

FLUSH PRIVILEGES;
```

**En Reserva (10.69.34.161):**
```sql
-- Crear usuario para que Productivo se conecte a Reserva
CREATE USER 'replication'@'10.69.34.160' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'10.69.34.160';

-- Usuario comodín por si hay problemas de resolución DNS
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

FLUSH PRIVILEGES;
```

### PASO 4: Verificar usuarios creados
**En ambos servidores:**
```sql
SELECT user, host, plugin FROM mysql.user WHERE user = 'replication';
```

**Resultado esperado:**
```
+-------------+---------------+-----------------------+
| user        | host          | plugin                |
+-------------+---------------+-----------------------+
| replication | 10.69.34.160  | mysql_native_password |
| replication | 10.69.34.161  | mysql_native_password |
| replication | %             | mysql_native_password |
+-------------+---------------+-----------------------+
```

---

## FASE 3: RESPALDO Y SINCRONIZACIÓN

### PASO 5: Obtener tamaño de bases de datos
**En Productivo:**
```sql
SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS 'Total_Size_GB'
FROM information_schema.tables 
WHERE table_schema NOT IN ('information_schema','performance_schema','mysql','sys');
```

### PASO 6: Generar respaldo completo desde Productivo

**A. Poner Productivo en modo solo lectura:**
```sql
FLUSH TABLES WITH READ LOCK;
```

**B. Obtener posición del binlog (¡MUY IMPORTANTE!):**
```sql
SHOW MASTER STATUS;
```
**📝 ANOTAR: File y Position**

**C. En otra terminal del servidor Productivo:**
```bash
# Generar backup completo
mysqldump -u root -p --all-databases --single-transaction --routines --triggers --events > backup_completo_$(date +%Y%m%d_%H%M%S).sql
```

**D. Liberar Productivo:**
```sql
UNLOCK TABLES;
```

### PASO 7: Transferir backup a Reserva
```bash
# Desde Productivo hacia Reserva
scp backup_completo_*.sql root@10.69.34.161:/tmp/
```

### PASO 8: Restaurar en Reserva
**En servidor Reserva:**
```bash
# Restaurar backup
mysql -u root -p < /tmp/backup_completo_*.sql
```

---

## FASE 4: CONFIGURACIÓN DE REPLICACIÓN MASTER-TO-MASTER

### PASO 9: Obtener posiciones actuales de binlog

**En Productivo:**
```sql
SHOW MASTER STATUS;
```
**📝 ANOTAR: File_productivo y Position_productivo**

**En Reserva:**
```sql
SHOW MASTER STATUS;
```
**📝 ANOTAR: File_reserva y Position_reserva**

### PASO 10: Configurar replicación bidireccional

**En Productivo (10.69.34.160) - configurar para leer desde Reserva:**
```sql
CHANGE MASTER TO
    MASTER_HOST='10.69.34.161',
    MASTER_USER='replication',
    MASTER_PASSWORD='zXsAOLP-w7tVCXsUHWlq',
    MASTER_LOG_FILE='File_reserva',
    MASTER_LOG_POS=Position_reserva;
```

**En Reserva (10.69.34.161) - configurar para leer desde Productivo:**
```sql
CHANGE MASTER TO
    MASTER_HOST='10.69.34.160',
    MASTER_USER='replication',
    MASTER_PASSWORD='zXsAOLP-w7tVCXsUHWlq',
    MASTER_LOG_FILE='File_productivo',
    MASTER_LOG_POS=Position_productivo;
```

### PASO 11: Iniciar replicación
**En ambos servidores:**
```sql
START SLAVE;
```

---

## FASE 5: VERIFICACIÓN Y PRUEBAS

### PASO 12: Verificar estado de replicación
**En ambos servidores ejecutar:**
```sql
SHOW SLAVE STATUS\G
```

**✅ Verificar que aparezca:**
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Last_IO_Error:` (vacío)
- `Last_SQL_Error:` (vacío)
- `Seconds_Behind_Master: 0` (o número muy pequeño)

### PASO 13: Prueba de funcionamiento bidireccional

**A. Prueba desde Productivo hacia Reserva:**
```sql
-- En Productivo
CREATE DATABASE test_replication;
USE test_replication;
CREATE TABLE prueba (id INT, servidor VARCHAR(50), timestamp DATETIME DEFAULT NOW());
INSERT INTO prueba (id, servidor) VALUES (1, 'PRODUCTIVO');
```

**Verificar en Reserva:**
```sql
USE test_replication;
SELECT * FROM prueba;
```

**B. Prueba desde Reserva hacia Productivo:**
```sql
-- En Reserva
USE test_replication;
INSERT INTO prueba (id, servidor) VALUES (2, 'RESERVA');
```

**Verificar en Productivo:**
```sql
SELECT * FROM prueba;
```

**✅ Resultado esperado:**
```
+----+------------+---------------------+
| id | servidor   | timestamp           |
+----+------------+---------------------+
|  1 | PRODUCTIVO | 2025-01-XX XX:XX:XX |
|  2 | RESERVA    | 2025-01-XX XX:XX:XX |
+----+------------+---------------------+
```

### PASO 14: Limpieza final
```sql
-- En ambos servidores
DROP DATABASE test_replication;
```

```bash
# En Reserva, eliminar archivo de backup
rm /tmp/backup_completo_*.sql
```

---

## TROUBLESHOOTING COMÚN

### Error de autenticación:
```sql
-- Si aparece error de plugin de autenticación
ALTER USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY 'zXsAOLP-w7tVCXsUHWlq';
FLUSH PRIVILEGES;
```

### Error de conexión:
```sql
-- Verificar usuarios y permisos
SELECT user, host, plugin FROM mysql.user WHERE user = 'replication';
SHOW GRANTS FOR 'replication'@'%';
```

### Reiniciar replicación si hay problemas:
```sql
STOP SLAVE;
START SLAVE;
SHOW SLAVE STATUS\G
```

---

## MONITOREO CONTINUO

### Comando para verificar estado regularmente:
```sql
-- Ejecutar periódicamente en ambos servidores
SELECT 
    @@hostname as Servidor,
    CASE 
        WHEN @@read_only = 1 THEN 'SLAVE'
        ELSE 'MASTER'
    END as Modo,
    (SELECT COUNT(*) FROM INFORMATION_SCHEMA.PROCESSLIST WHERE COMMAND = 'Binlog Dump') as Conexiones_Slave;

SHOW SLAVE STATUS\G
```

---

## ✅ CHECKLIST FINAL

- [ ] Replicación existente eliminada
- [ ] Usuarios creados con mysql_native_password
- [ ] Backup generado y transferido exitosamente
- [ ] Backup restaurado en Reserva
- [ ] Replicación bidireccional configurada
- [ ] Estado de replicación verificado (IO y SQL Running = Yes)
- [ ] Pruebas bidireccionales exitosas
- [ ] Archivos temporales eliminados

**🎉 ¡Replicación Master-to-Master configurada exitosamente!**