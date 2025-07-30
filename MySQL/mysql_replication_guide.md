<h1 id="gu%C3%ADacompleta%3Aconfiguraci%C3%B3nreplicaci%C3%B3nmysql8.xmaster-to-master">Guía Completa: Configuración Replicación MySQL 8.x Master-to-Master</h1>

<h2 id="datosdelproyecto">Datos del Proyecto</h2>

<ul>
<li><strong>Productivo</strong>: 10.69.34.160</li>
<li><strong>Reserva</strong>: 10.69.34.161</li>
<li><strong>Usuario replicación</strong>: replication</li>
<li><strong>Password</strong>: zXsAOLP-w7tVCXsUHWlq</li>
</ul>

<hr />

<h2 id="fase1%3Alimpiezaypreparaci%C3%93n">FASE 1: LIMPIEZA Y PREPARACIÓN</h2>

<h3 id="paso1%3Adetenerreplicaci%C3%B3nexistente">PASO 1: Detener replicación existente</h3>

<p><strong>En ambos servidores (Productivo y Reserva):</strong></p>
<pre><code class="sql">STOP SLAVE;
RESET SLAVE ALL;
RESET MASTER;</code></pre>

<h3 id="paso2%3Alimpiarusuariosdereplicaci%C3%B3nexistentes">PASO 2: Limpiar usuarios de replicación existentes</h3>

<p><strong>En ambos servidores:</strong></p>
<pre><code class="sql">-- Verificar usuarios existentes
SELECT user, host FROM mysql.user WHERE user = &apos;replication&apos;;

-- Eliminar todos los usuarios de replicación existentes
DROP USER IF EXISTS &apos;replication&apos;@&apos;%&apos;;
DROP USER IF EXISTS &apos;replication&apos;@&apos;localhost&apos;;
DROP USER IF EXISTS &apos;replication&apos;@&apos;10.69.34.160&apos;;
DROP USER IF EXISTS &apos;replication&apos;@&apos;10.69.34.161&apos;;

FLUSH PRIVILEGES;</code></pre>

<hr />

<h2 id="fase2%3Acreaci%C3%93ndeusuariosconautenticaci%C3%93nlegacy">FASE 2: CREACIÓN DE USUARIOS CON AUTENTICACIÓN LEGACY</h2>

<h3 id="paso3%3Acrearusuariosdereplicaci%C3%B3nconmysql_native_password">PASO 3: Crear usuarios de replicación con mysql_native_password</h3>

<p><strong>En Productivo (10.69.34.160):</strong></p>
<pre><code class="sql">-- Crear usuario para que Reserva se conecte a Productivo
CREATE USER &apos;replication&apos;@&apos;10.69.34.161&apos; IDENTIFIED WITH mysql_native_password BY &apos;zXsAOLP-w7tVCXsUHWlq&apos;;
GRANT REPLICATION SLAVE ON *.* TO &apos;replication&apos;@&apos;10.69.34.161&apos;;

-- Usuario comodín por si hay problemas de resolución DNS
CREATE USER &apos;replication&apos;@&apos;%&apos; IDENTIFIED WITH mysql_native_password BY &apos;zXsAOLP-w7tVCXsUHWlq&apos;;
GRANT REPLICATION SLAVE ON *.* TO &apos;replication&apos;@&apos;%&apos;;

FLUSH PRIVILEGES;</code></pre>

<p><strong>En Reserva (10.69.34.161):</strong></p>
<pre><code class="sql">-- Crear usuario para que Productivo se conecte a Reserva
CREATE USER &apos;replication&apos;@&apos;10.69.34.160&apos; IDENTIFIED WITH mysql_native_password BY &apos;zXsAOLP-w7tVCXsUHWlq&apos;;
GRANT REPLICATION SLAVE ON *.* TO &apos;replication&apos;@&apos;10.69.34.160&apos;;

-- Usuario comodín por si hay problemas de resolución DNS
CREATE USER &apos;replication&apos;@&apos;%&apos; IDENTIFIED WITH mysql_native_password BY &apos;zXsAOLP-w7tVCXsUHWlq&apos;;
GRANT REPLICATION SLAVE ON *.* TO &apos;replication&apos;@&apos;%&apos;;

FLUSH PRIVILEGES;</code></pre>

<h3 id="paso4%3Averificarusuarioscreados">PASO 4: Verificar usuarios creados</h3>

<p><strong>En ambos servidores:</strong></p>
<pre><code class="sql">SELECT user, host, plugin FROM mysql.user WHERE user = &apos;replication&apos;;</code></pre>

<p><strong>Resultado esperado:</strong></p>
<pre><code>+-------------+---------------+-----------------------+
| user        | host          | plugin                |
+-------------+---------------+-----------------------+
| replication | 10.69.34.160  | mysql_native_password |
| replication | 10.69.34.161  | mysql_native_password |
| replication | %             | mysql_native_password |
+-------------+---------------+-----------------------+</code></pre>

<hr />

<h2 id="fase3%3Arespaldoysincronizaci%C3%93n">FASE 3: RESPALDO Y SINCRONIZACIÓN</h2>

<h3 id="paso5%3Aobtenertama%C3%B1odebasesdedatos">PASO 5: Obtener tamaño de bases de datos</h3>

<p><strong>En Productivo:</strong></p>
<pre><code class="sql">SELECT 
    ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS &apos;Total_Size_GB&apos;
FROM information_schema.tables 
WHERE table_schema NOT IN (&apos;information_schema&apos;,&apos;performance_schema&apos;,&apos;mysql&apos;,&apos;sys&apos;);</code></pre>

<h3 id="paso6%3Agenerarrespaldocompletodesdeproductivo">PASO 6: Generar respaldo completo desde Productivo</h3>

<p><strong>A. Poner Productivo en modo solo lectura:</strong></p>
<pre><code class="sql">FLUSH TABLES WITH READ LOCK;</code></pre>

<p><strong>B. Obtener posición del binlog (¡MUY IMPORTANTE!):</strong></p>
<pre><code class="sql">SHOW MASTER STATUS;</code></pre>

<p><strong>📝 ANOTAR: File y Position</strong></p>

<p><strong>C. En otra terminal del servidor Productivo:</strong></p>
<pre><code class="bash"># Generar backup completo
mysqldump -u root -p --all-databases --single-transaction --routines --triggers --events &gt; backup_completo_$(date +%Y%m%d_%H%M%S).sql</code></pre>

<p><strong>D. Liberar Productivo:</strong></p>
<pre><code class="sql">UNLOCK TABLES;</code></pre>

<h3 id="paso7%3Atransferirbackupareserva">PASO 7: Transferir backup a Reserva</h3>
<pre><code class="bash"># Desde Productivo hacia Reserva
scp backup_completo_*.sql root@10.69.34.161:/tmp/</code></pre>

<h3 id="paso8%3Arestaurarenreserva">PASO 8: Restaurar en Reserva</h3>

<p><strong>En servidor Reserva:</strong></p>
<pre><code class="bash"># Restaurar backup
mysql -u root -p &lt; /tmp/backup_completo_*.sql</code></pre>

<hr />

<h2 id="fase4%3Aconfiguraci%C3%93ndereplicaci%C3%93nmaster-to-master">FASE 4: CONFIGURACIÓN DE REPLICACIÓN MASTER-TO-MASTER</h2>

<h3 id="paso9%3Aobtenerposicionesactualesdebinlog">PASO 9: Obtener posiciones actuales de binlog</h3>

<p><strong>En Productivo:</strong></p>
<pre><code class="sql">SHOW MASTER STATUS;</code></pre>

<p><strong>📝 ANOTAR: File_productivo y Position_productivo</strong></p>

<p><strong>En Reserva:</strong></p>
<pre><code class="sql">SHOW MASTER STATUS;</code></pre>

<p><strong>📝 ANOTAR: File_reserva y Position_reserva</strong></p>

<h3 id="paso10%3Aconfigurarreplicaci%C3%B3nbidireccional">PASO 10: Configurar replicación bidireccional</h3>

<p><strong>En Productivo (10.69.34.160) - configurar para leer desde Reserva:</strong></p>
<pre><code class="sql">CHANGE MASTER TO
    MASTER_HOST=&apos;10.69.34.161&apos;,
    MASTER_USER=&apos;replication&apos;,
    MASTER_PASSWORD=&apos;zXsAOLP-w7tVCXsUHWlq&apos;,
    MASTER_LOG_FILE=&apos;File_reserva&apos;,
    MASTER_LOG_POS=Position_reserva;</code></pre>

<p><strong>En Reserva (10.69.34.161) - configurar para leer desde Productivo:</strong></p>
<pre><code class="sql">CHANGE MASTER TO
    MASTER_HOST=&apos;10.69.34.160&apos;,
    MASTER_USER=&apos;replication&apos;,
    MASTER_PASSWORD=&apos;zXsAOLP-w7tVCXsUHWlq&apos;,
    MASTER_LOG_FILE=&apos;File_productivo&apos;,
    MASTER_LOG_POS=Position_productivo;</code></pre>

<h3 id="paso11%3Ainiciarreplicaci%C3%B3n">PASO 11: Iniciar replicación</h3>

<p><strong>En ambos servidores:</strong></p>
<pre><code class="sql">START SLAVE;</code></pre>

<hr />

<h2 id="fase5%3Averificaci%C3%93nypruebas">FASE 5: VERIFICACIÓN Y PRUEBAS</h2>

<h3 id="paso12%3Averificarestadodereplicaci%C3%B3n">PASO 12: Verificar estado de replicación</h3>

<p><strong>En ambos servidores ejecutar:</strong></p>
<pre><code class="sql">SHOW SLAVE STATUS\G</code></pre>

<p><strong>✅ Verificar que aparezca:</strong><br/>
- <code>Slave_IO_Running: Yes</code><br/>
- <code>Slave_SQL_Running: Yes</code><br/>
- <code>Last_IO_Error:</code> (vacío)<br/>
- <code>Last_SQL_Error:</code> (vacío)<br/>
- <code>Seconds_Behind_Master: 0</code> (o número muy pequeño)</p>

<h3 id="paso13%3Apruebadefuncionamientobidireccional">PASO 13: Prueba de funcionamiento bidireccional</h3>

<p><strong>A. Prueba desde Productivo hacia Reserva:</strong></p>
<pre><code class="sql">-- En Productivo
CREATE DATABASE test_replication;
USE test_replication;
CREATE TABLE prueba (id INT, servidor VARCHAR(50), timestamp DATETIME DEFAULT NOW());
INSERT INTO prueba (id, servidor) VALUES (1, &apos;PRODUCTIVO&apos;);</code></pre>

<p><strong>Verificar en Reserva:</strong></p>
<pre><code class="sql">USE test_replication;
SELECT * FROM prueba;</code></pre>

<p><strong>B. Prueba desde Reserva hacia Productivo:</strong></p>
<pre><code class="sql">-- En Reserva
USE test_replication;
INSERT INTO prueba (id, servidor) VALUES (2, &apos;RESERVA&apos;);</code></pre>

<p><strong>Verificar en Productivo:</strong></p>
<pre><code class="sql">SELECT * FROM prueba;</code></pre>

<p><strong>✅ Resultado esperado:</strong></p>
<pre><code>+----+------------+---------------------+
| id | servidor   | timestamp           |
+----+------------+---------------------+
|  1 | PRODUCTIVO | 2025-01-XX XX:XX:XX |
|  2 | RESERVA    | 2025-01-XX XX:XX:XX |
+----+------------+---------------------+</code></pre>

<h3 id="paso14%3Alimpiezafinal">PASO 14: Limpieza final</h3>
<pre><code class="sql">-- En ambos servidores
DROP DATABASE test_replication;</code></pre>
<pre><code class="bash"># En Reserva, eliminar archivo de backup
rm /tmp/backup_completo_*.sql</code></pre>

<hr />

<h2 id="troubleshootingcom%C3%9An">TROUBLESHOOTING COMÚN</h2>

<h3 id="errordeautenticaci%C3%B3n%3A">Error de autenticación:</h3>
<pre><code class="sql">-- Si aparece error de plugin de autenticación
ALTER USER &apos;replication&apos;@&apos;%&apos; IDENTIFIED WITH mysql_native_password BY &apos;zXsAOLP-w7tVCXsUHWlq&apos;;
FLUSH PRIVILEGES;</code></pre>

<h3 id="errordeconexi%C3%B3n%3A">Error de conexión:</h3>
<pre><code class="sql">-- Verificar usuarios y permisos
SELECT user, host, plugin FROM mysql.user WHERE user = &apos;replication&apos;;
SHOW GRANTS FOR &apos;replication&apos;@&apos;%&apos;;</code></pre>

<h3 id="reiniciarreplicaci%C3%B3nsihayproblemas%3A">Reiniciar replicación si hay problemas:</h3>
<pre><code class="sql">STOP SLAVE;
START SLAVE;
SHOW SLAVE STATUS\G</code></pre>

<hr />

<h2 id="monitoreocontinuo">MONITOREO CONTINUO</h2>

<h3 id="comandoparaverificarestadoregularmente%3A">Comando para verificar estado regularmente:</h3>
<pre><code class="sql">-- Ejecutar periódicamente en ambos servidores
SELECT 
    @@hostname as Servidor,
    CASE 
        WHEN @@read_only = 1 THEN &apos;SLAVE&apos;
        ELSE &apos;MASTER&apos;
    END as Modo,
    (SELECT COUNT(*) FROM INFORMATION_SCHEMA.PROCESSLIST WHERE COMMAND = &apos;Binlog Dump&apos;) as Conexiones_Slave;

SHOW SLAVE STATUS\G</code></pre>

<hr />

<h2 id="%E2%9C%85checklistfinal">✅ CHECKLIST FINAL</h2>

<ul>
<li>[ ] Replicación existente eliminada</li>
<li>[ ] Usuarios creados con mysql_native_password</li>
<li>[ ] Backup generado y transferido exitosamente</li>
<li>[ ] Backup restaurado en Reserva</li>
<li>[ ] Replicación bidireccional configurada</li>
<li>[ ] Estado de replicación verificado (IO y SQL Running = Yes)</li>
<li>[ ] Pruebas bidireccionales exitosas</li>
<li>[ ] Archivos temporales eliminados</li>
</ul>

<p><strong>🎉 ¡Replicación Master-to-Master configurada exitosamente!</strong></p>
