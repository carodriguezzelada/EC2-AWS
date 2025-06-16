<h1 id="%F0%9F%92%BEagregarvolumenebsde100gbainstanciaairviro">💾 Agregar Volumen EBS de 100GB a Instancia Airviro</h1>

<h2 id="%F0%9F%93%8Binformaci%C3%B3nprevianecesaria">📋 Información Previa Necesaria</h2>

<ul>
<li><strong>IP de tu instancia</strong>: 44.195.187.47</li>
<li><strong>Directorio destino</strong>: <code>/usr/airviro</code></li>
<li><strong>Tamaño volumen</strong>: 100 GB</li>
</ul>

<hr />

<h2 id="%F0%9F%8E%AFparte1%3Acrearelvolumenebsenawsconsole">🎯 Parte 1: Crear el Volumen EBS en AWS Console</h2>

<h3 id="paso1.1%3Aaccederavol%C3%BAmenesebs">Paso 1.1: Acceder a Volúmenes EBS</h3>

<ol>
<li><strong>Ir a EC2 Console</strong> → <code>https://console.aws.amazon.com/ec2/</code></li>
<li>En el menú lateral izquierdo, bajo <strong>&#8220;Elastic Block Store&#8221;</strong></li>
<li>Click en <strong>&#8220;Volumes&#8221;</strong></li>
</ol>

<h3 id="paso1.2%3Acrearnuevovolumen">Paso 1.2: Crear Nuevo Volumen</h3>

<ol>
<li>Click en <strong>&#8220;Create volume&#8221;</strong> (botón azul)</li>
<li><strong>Configuración del volumen</strong>:</li>
</ol>

<p><pre><code>Volume type: gp3 (General Purpose SSD)
Size (GiB): 100
IOPS: 3000 (default para gp3)
Throughput (MiB/s): 125 (default para gp3)</code></pre></p>

<h3 id="paso1.3%3Aconfigurarubicaci%C3%B3nyencriptaci%C3%B3n">Paso 1.3: Configurar Ubicación y Encriptación</h3>

<ol>
<li><p><strong>Availability Zone</strong>: </p></li>
<li><p>⚠️ <strong>CRÍTICO</strong>: Debe ser la MISMA AZ que tu instancia Airviro </p></li>
<li>Para verificar AZ de tu instancia:

<ul>
<li>Ve a <strong>Instances</strong> → Selecciona tu instancia Airviro</li>
<li>En la pestaña <strong>Details</strong>, busca <strong>&#8220;Availability Zone&#8221;</strong></li>
<li>Ejemplo: <code>us-east-1a</code></li>
</ul></li>
<li><p><strong>Snapshot ID</strong>: Dejar en blanco (volumen nuevo)</p></li>
<li><p><strong>Encryption</strong>: </p></li>
<li><p>✅ <strong>Encrypt this volume</strong>: Activado </p></li>
<li><p><strong>KMS key</strong>: Usar <code>(default) aws/ebs</code></p></li>
</ol>

<h3 id="paso1.4%3Aagregartags">Paso 1.4: Agregar Tags</h3>
<pre><code>Key: Name           Value: Airviro-Data-Volume
Key: Purpose        Value: Airviro Application Data
Key: Size           Value: 100GB
Key: AttachedTo     Value: Airviro-Production</code></pre>

<h3 id="paso1.5%3Acrearvolumen">Paso 1.5: Crear Volumen</h3>

<ol>
<li><strong>Revisar configuración</strong></li>
<li>Click <strong>&#8220;Create volume&#8221;</strong></li>
<li>✅ <strong>Confirmación</strong>: &#8220;Successfully created volume vol-xxxxxxxxx&#8221;</li>
<li><strong>Anotar el Volume ID</strong> (lo necesitarás): <code>vol-xxxxxxxxx</code></li>
</ol>

<hr />

<h2 id="%F0%9F%94%97parte2%3Aattacharelvolumenalainstancia">🔗 Parte 2: Attachar el Volumen a la Instancia</h2>

<h3 id="paso2.1%3Aseleccionarelvolumen">Paso 2.1: Seleccionar el Volumen</h3>

<ol>
<li>En la lista de <strong>Volumes</strong>, buscar tu volumen recién creado</li>
<li><strong>Estado</strong> debe ser: <code>available</code></li>
<li><strong>Seleccionar</strong> el volumen (checkbox)</li>
</ol>

<h3 id="paso2.2%3Aattacharalainstancia">Paso 2.2: Attachar a la Instancia</h3>

<ol>
<li>Click <strong>&#8220;Actions&#8221;</strong> → <strong>&#8220;Attach volume&#8221;</strong></li>
<li><strong>Instance ID</strong>:</li>
<li>Click en el campo y buscar tu instancia Airviro</li>
<li>Debería aparecer como <code>i-xxxxxxxxx (Airviro-Production)</code></li>
<li><strong>Device name</strong>:</li>
<li>Usar <code>/dev/sdf</code> (recomendado)</li>
<li>⚠️ <strong>Nota</strong>: En la instancia aparecerá como <code>/dev/xvdf</code></li>
<li>Click <strong>&#8220;Attach volume&#8221;</strong></li>
</ol>

<h3 id="paso2.3%3Averificarattachment">Paso 2.3: Verificar Attachment</h3>

<ul>
<li><strong>Estado</strong> cambiará a: <code>in-use</code></li>
<li><strong>Attached instances</strong> mostrará tu instancia</li>
</ul>

<hr />

<h2 id="%F0%9F%92%BBparte3%3Aconfigurarelvolumenenlainstanciassh">💻 Parte 3: Configurar el Volumen en la Instancia (SSH)</h2>

<h3 id="paso3.1%3Aconectarsealainstancia">Paso 3.1: Conectarse a la Instancia</h3>
<pre><code class="bash">ssh -i tu-keypair.pem ec2-user@44.195.187.47</code></pre>

<h3 id="paso3.2%3Averificarqueelvolumenest%C3%A1visible">Paso 3.2: Verificar que el Volumen está Visible</h3>
<pre><code class="bash"># Listar todos los discos
lsblk

# Deberías ver algo como:
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda    202:0    0  50G  0 disk 
# └─xvda1 202:1    0  50G  0 part /
# xvdf    202:80   0 100G  0 disk    &lt;- Nuevo volumen</code></pre>

<h3 id="paso3.3%3Averificarsielvolumentienesistemadearchivos">Paso 3.3: Verificar si el Volumen tiene Sistema de Archivos</h3>
<pre><code class="bash"># Verificar si ya tiene formato
sudo file -s /dev/xvdf

# Si muestra &quot;data&quot;, significa que está vacío y necesita formato
# Si muestra un sistema de archivos, ya está formateado</code></pre>

<h3 id="paso3.4%3Aformatearelvolumensiesnecesario">Paso 3.4: Formatear el Volumen (si es necesario)</h3>
<pre><code class="bash"># Formatear con ext4 (recomendado para Linux)
sudo mkfs -t ext4 /dev/xvdf

# Salida esperada:
# mke2fs 1.45.6 (20-Mar-2020)
# Creating filesystem with 26214400 4k blocks and 6553600 inodes
# ...</code></pre>

<h3 id="paso3.5%3Acreareldirectoriodemontaje">Paso 3.5: Crear el Directorio de Montaje</h3>
<pre><code class="bash"># Crear directorio /usr/airviro
sudo mkdir -p /usr/airviro

# Verificar que se creó
ls -la /usr/airviro</code></pre>

<h3 id="paso3.6%3Amontarelvolumentemporalmente">Paso 3.6: Montar el Volumen Temporalmente</h3>
<pre><code class="bash"># Montar el volumen
sudo mount /dev/xvdf /usr/airviro

# Verificar que está montado
df -h /usr/airviro

# Salida esperada:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvdf        98G   61M   93G   1% /usr/airviro</code></pre>

<h3 id="paso3.7%3Aconfigurarpermisos">Paso 3.7: Configurar Permisos</h3>
<pre><code class="bash"># Cambiar propietario (opcional, según necesidades de Airviro)
sudo chown -R ec2-user:ec2-user /usr/airviro

# O si Airviro necesita un usuario específico:
# sudo chown -R airviro:airviro /usr/airviro

# Establecer permisos apropiados
sudo chmod 755 /usr/airviro</code></pre>

<hr />

<h2 id="%F0%9F%94%84parte4%3Aconfigurarmontajeautom%C3%A1ticopermanente">🔄 Parte 4: Configurar Montaje Automático (Permanente)</h2>

<h3 id="paso4.1%3Aobteneruuiddelvolumen">Paso 4.1: Obtener UUID del Volumen</h3>
<pre><code class="bash"># Obtener UUID único del volumen
sudo blkid /dev/xvdf

# Salida ejemplo:
# /dev/xvdf: UUID=&quot;12345678-1234-1234-1234-123456789abc&quot; TYPE=&quot;ext4&quot;

# Copiar el UUID completo</code></pre>

<h3 id="paso4.2%3Ahacerbackupdelfstab">Paso 4.2: Hacer Backup del fstab</h3>
<pre><code class="bash"># Crear backup del archivo fstab
sudo cp /etc/fstab /etc/fstab.backup

# Verificar backup
ls -la /etc/fstab*</code></pre>

<h3 id="paso4.3%3Aagregarentradaalfstab">Paso 4.3: Agregar Entrada al fstab</h3>
<pre><code class="bash"># Editar fstab
sudo nano /etc/fstab

# Agregar al final del archivo (reemplaza UUID con el tuyo):
UUID=12345678-1234-1234-1234-123456789abc /usr/airviro ext4 defaults,nofail 0 2

# Guardar y salir (Ctrl+X, Y, Enter)</code></pre>

<h3 id="paso4.4%3Averificarconfiguraci%C3%B3nfstab">Paso 4.4: Verificar Configuración fstab</h3>
<pre><code class="bash"># Probar la configuración sin reiniciar
sudo mount -a

# Si no hay errores, está correcto
# Verificar que sigue montado
df -h /usr/airviro</code></pre>

<h3 id="paso4.5%3Averificarmontajepermanente">Paso 4.5: Verificar Montaje Permanente</h3>
<pre><code class="bash"># Desmontar temporalmente
sudo umount /usr/airviro

# Montar usando fstab
sudo mount /usr/airviro

# Verificar
df -h /usr/airviro</code></pre>

<hr />

<h2 id="%E2%9C%85parte5%3Averificaci%C3%B3nfinalypruebas">✅ Parte 5: Verificación Final y Pruebas</h2>

<h3 id="paso5.1%3Acreararchivosdeprueba">Paso 5.1: Crear Archivos de Prueba</h3>
<pre><code class="bash"># Crear archivo de prueba
echo &quot;Volumen Airviro funcionando correctamente - $(date)&quot; | sudo tee /usr/airviro/test.txt

# Verificar
cat /usr/airviro/test.txt</code></pre>

<h3 id="paso5.2%3Averificarespaciodisponible">Paso 5.2: Verificar Espacio Disponible</h3>
<pre><code class="bash"># Ver espacio total y disponible
df -h /usr/airviro

# Ver detalles del sistema de archivos
sudo tune2fs -l /dev/xvdf | grep -E &quot;(Block count|Block size|Free blocks)&quot;</code></pre>

<h3 id="paso5.3%3Aprobarreinicioopcional">Paso 5.3: Probar Reinicio (Opcional)</h3>
<pre><code class="bash"># Reiniciar instancia para verificar montaje automático
sudo reboot

# Después del reinicio, reconectar y verificar:
df -h /usr/airviro
ls -la /usr/airviro/</code></pre>

<hr />

<h2 id="%F0%9F%93%8Ainformaci%C3%B3ndelvolumenconfigurado">📊 Información del Volumen Configurado</h2>

<table>
<colgroup>
<col style="text-align:left;"/>
<col style="text-align:left;"/>
</colgroup>

<thead>
<tr>
	<th style="text-align:left;">Aspecto</th>
	<th style="text-align:left;">Configuración</th>
</tr>
</thead>

<tbody>
<tr>
	<td style="text-align:left;"><strong>Tamaño</strong></td>
	<td style="text-align:left;">100 GB</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Tipo</strong></td>
	<td style="text-align:left;">gp3 (General Purpose SSD)</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>IOPS</strong></td>
	<td style="text-align:left;">3000</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Throughput</strong></td>
	<td style="text-align:left;">125 MiB/s</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Encriptación</strong></td>
	<td style="text-align:left;">✅ Habilitada</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Punto de montaje</strong></td>
	<td style="text-align:left;"><code>/usr/airviro</code></td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Sistema de archivos</strong></td>
	<td style="text-align:left;">ext4</td>
</tr>
<tr>
	<td style="text-align:left;"><strong>Montaje automático</strong></td>
	<td style="text-align:left;">✅ Configurado</td>
</tr>
</tbody>
</table>

<hr />

<h2 id="%F0%9F%9A%A8troubleshootingcom%C3%BAn">🚨 Troubleshooting Común</h2>

<h3 id="problema%3Anospaceleftondevice">Problema: &#8220;No space left on device&#8221;</h3>
<pre><code class="bash"># Verificar espacio disponible
df -h

# Verificar inodos disponibles
df -i</code></pre>

<h3 id="problema%3Amount%3Awrongfstype">Problema: &#8220;mount: wrong fs type&#8221;</h3>
<pre><code class="bash"># Verificar tipo de sistema de archivos
sudo blkid /dev/xvdf

# Re-formatear si es necesario
sudo mkfs.ext4 /dev/xvdf</code></pre>

<h3 id="problema%3Avolumennoaparecedespu%C3%A9sdereboot">Problema: Volumen no aparece después de reboot</h3>
<pre><code class="bash"># Verificar fstab
cat /etc/fstab

# Montar manualmente
sudo mount -a

# Ver logs de errores
dmesg | tail -20</code></pre>

<h3 id="problema%3Apermisosincorrectos">Problema: Permisos incorrectos</h3>
<pre><code class="bash"># Verificar permisos actuales
ls -la /usr/airviro

# Corregir permisos
sudo chown -R ec2-user:ec2-user /usr/airviro
sudo chmod 755 /usr/airviro</code></pre>

<hr />

<h2 id="%F0%9F%93%9Dcomandosderesumenparaverificaci%C3%B3n">📝 Comandos de Resumen para Verificación</h2>
<pre><code class="bash"># Verificación completa del estado del volumen
echo &quot;=== VERIFICACIÓN VOLUMEN AIRVIRO ===&quot;
echo &quot;1. Discos disponibles:&quot;
lsblk | grep -E &quot;(xvd|nvme)&quot;

echo -e &quot;\n2. Espacio disponible:&quot;
df -h /usr/airviro

echo -e &quot;\n3. Punto de montaje:&quot;
mount | grep airviro

echo -e &quot;\n4. UUID del volumen:&quot;
sudo blkid /dev/xvdf

echo -e &quot;\n5. Entrada en fstab:&quot;
grep airviro /etc/fstab

echo -e &quot;\n6. Contenido del directorio:&quot;
ls -la /usr/airviro/

echo -e &quot;\n7. Archivo de prueba:&quot;
cat /usr/airviro/test.txt 2&gt;/dev/null || echo &quot;Archivo de prueba no encontrado&quot;</code></pre>

<hr />

<h2 id="%E2%9A%A0%EF%B8%8Fnotasimportantes">⚠️ Notas Importantes</h2>

<ol>
<li><strong>Backup</strong>: Siempre haz backup antes de modificar <code>/etc/fstab</code></li>
<li><strong>AZ Matching</strong>: El volumen DEBE estar en la misma Availability Zone que la instancia</li>
<li><strong>Costo</strong>: Un volumen gp3 de 100GB cuesta aproximadamente $8&#8211;10/mes</li>
<li><strong>Snapshots</strong>: Considera crear snapshots regulares para backup</li>
<li><strong>Permisos</strong>: Ajusta los permisos según los requerimientos específicos de Airviro</li>
</ol>
