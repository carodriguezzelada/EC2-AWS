<h1 id="mejorespr%C3%A1cticasparalanzamientodeinstanciasec2">Mejores Prácticas para Lanzamiento de Instancias EC2</h1>

<h2 id="%F0%9F%94%90seguridad">🔐 Seguridad</h2>

<h3 id="keypairsyacceso">Key Pairs y Acceso</h3>

<ul>
<li><strong>Nunca compartas tu key pair privado</strong></li>
<li>Usa key pairs diferentes para producción y desarrollo</li>
<li>Considera usar AWS Systems Manager Session Manager para acceso sin SSH</li>
<li>Implementa rotación regular de keys</li>
</ul>

<h3 id="securitygroups">Security Groups</h3>

<ul>
<li><strong>Principio de menor privilegio</strong>: Solo abre los puertos necesarios</li>
<li>Usa rangos de IP específicos en lugar de 0.0.0.0/0 cuando sea posible</li>
<li>Separa security groups por función (web, database, admin)</li>
<li>Documenta cada regla y su propósito</li>
</ul>

<h3 id="encriptaci%C3%B3n">Encriptación</h3>

<ul>
<li><strong>Siempre encripta volúmenes EBS</strong> (incluido en el script)</li>
<li>Usa AWS KMS para gestión de claves</li>
<li>Habilita encriptación en tránsito cuando sea posible</li>
</ul>

<h2 id="%F0%9F%92%B0gesti%C3%B3ndecostos">💰 Gestión de Costos</h2>

<h3 id="dimensionamientocorrecto">Dimensionamiento Correcto</h3>

<ul>
<li><strong>Airviro (c5.xlarge)</strong>: Adecuado para cargas computacionales intensivas</li>
<li><strong>Laravel (t3.large)</strong>: Burstable performance, ideal para apps web</li>
<li>Monitorea utilización de CPU/memoria para optimizar tamaños</li>
</ul>

<h3 id="optimizaci%C3%B3ndecostos">Optimización de Costos</h3>

<ul>
<li>Usa <strong>Reserved Instances</strong> para cargas predecibles (hasta 75% descuento)</li>
<li>Considera <strong>Spot Instances</strong> para workloads tolerantes a interrupciones</li>
<li>Implementa <strong>Auto Scaling</strong> para ajustar capacidad automáticamente</li>
<li>Programa paradas automáticas para entornos de desarrollo</li>
</ul>

<h2 id="%F0%9F%8F%97%EF%B8%8Farquitecturaydise%C3%B1o">🏗️ Arquitectura y Diseño</h2>

<h3 id="altadisponibilidad">Alta Disponibilidad</h3>
<pre><code>Recomendación: Distribuir en múltiples AZ
- AZ-1a: Instancia Airviro principal
- AZ-1b: Instancia Laravel + backup Airviro</code></pre>

<h3 id="redyconectividad">Red y Conectividad</h3>

<ul>
<li>Usa <strong>VPC dedicada</strong> con subnets públicas y privadas</li>
<li>Implementa <strong>NAT Gateway</strong> para instancias en subnets privadas</li>
<li>Configura <strong>Route 53</strong> para DNS personalizado</li>
<li>Considera <strong>Load Balancer</strong> para Laravel si esperas alto tráfico</li>
</ul>

<h2 id="%F0%9F%93%8Amonitoreoylogging">📊 Monitoreo y Logging</h2>

<h3 id="cloudwatch">CloudWatch</h3>
<pre><code class="bash"># Habilitar métricas detalladas (incluido en script)
--monitoring Enabled=true</code></pre>

<h3 id="logscentralizados">Logs Centralizados</h3>

<ul>
<li>Configura <strong>CloudWatch Logs Agent</strong></li>
<li>Implementa log rotation</li>
<li>Monitorea logs de aplicación y sistema</li>
</ul>

<h3 id="alertascr%C3%ADticas">Alertas Críticas</h3>

<ul>
<li>CPU &gt; 80% por 5 minutos</li>
<li>Memoria &gt; 85%</li>
<li>Espacio en disco &lt; 15%</li>
<li>Errores de aplicación</li>
</ul>

<h2 id="%F0%9F%94%84backupyrecovery">🔄 Backup y Recovery</h2>

<h3 id="snapshotsautomatizados">Snapshots Automatizados</h3>
<pre><code class="bash"># Crear snapshot diario
aws ec2 create-snapshot \
    --volume-id vol-xxxxxxxxx \
    --description &quot;Daily backup $(date +%Y-%m-%d)&quot;</code></pre>

<h3 id="estrategia3-2-1">Estrategia 3&#8211;2&#8211;1</h3>

<ul>
<li>3 copias de datos críticos</li>
<li>2 medios de almacenamiento diferentes</li>
<li>1 copia off-site (S3 Glacier)</li>
</ul>

<h2 id="%F0%9F%9A%80deploymentycicd">🚀 Deployment y CI/CD</h2>

<h3 id="paralaravel">Para Laravel</h3>
<pre><code class="bash"># Post-configuración Laravel
composer install --optimize-autoloader --no-dev
php artisan config:cache
php artisan route:cache
php artisan view:cache</code></pre>

<h3 id="paraairviro">Para Airviro</h3>

<ul>
<li>Documenta dependencias específicas</li>
<li>Crea scripts de deployment automatizado</li>
<li>Implementa blue-green deployment</li>
</ul>

<h2 id="%F0%9F%94%A7configuraci%C3%B3npost-launch">🔧 Configuración Post-Launch</h2>

<h3 id="hardeningdelsistema">Hardening del Sistema</h3>
<pre><code class="bash"># Deshabilitar servicios innecesarios
systemctl disable bluetooth
systemctl disable cups

# Configurar firewall
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Configurar fail2ban
yum install fail2ban -y
systemctl enable fail2ban</code></pre>

<h3 id="actualizacionesautom%C3%A1ticas">Actualizaciones Automáticas</h3>
<pre><code class="bash"># Configurar yum-cron para actualizaciones de seguridad
yum install yum-cron -y
sed -i &apos;s/apply_updates = no/apply_updates = yes/&apos; /etc/yum/yum-cron.conf
systemctl enable yum-cron</code></pre>

<h2 id="%F0%9F%93%9Dtaggingyorganizaci%C3%B3n">📝 Tagging y Organización</h2>

<h3 id="tagsrecomendados">Tags Recomendados</h3>
<pre><code class="json">{
  &quot;Name&quot;: &quot;Airviro-Production&quot;,
  &quot;Environment&quot;: &quot;Production&quot;,
  &quot;Application&quot;: &quot;Airviro&quot;,
  &quot;Owner&quot;: &quot;DevOps-Team&quot;,
  &quot;CostCenter&quot;: &quot;IT-001&quot;,
  &quot;Backup&quot;: &quot;Daily&quot;,
  &quot;Maintenance&quot;: &quot;Sunday-2AM&quot;
}</code></pre>

<h2 id="%F0%9F%94%8Dtroubleshootingcom%C3%BAn">🔍 Troubleshooting Común</h2>

<h3 id="problemasdeconexi%C3%B3nssh">Problemas de Conexión SSH</h3>
<pre><code class="bash"># Verificar security group
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# Verificar status de instancia
aws ec2 describe-instances --instance-ids i-xxxxxxxxx</code></pre>

<h3 id="problemasdeperformance">Problemas de Performance</h3>
<pre><code class="bash"># Monitorear recursos
top
iotop
nethogs</code></pre>

<h2 id="%F0%9F%93%8Bchecklistpre-launch">📋 Checklist Pre-Launch</h2>

<ul>
<li>[ ] AMI de Rocky Linux 9 verificada</li>
<li>[ ] Key pair creado y almacenado seguramente</li>
<li>[ ] Security groups configurados correctamente</li>
<li>[ ] VPC y subnets configuradas</li>
<li>[ ] IAM roles y policies definidos</li>
<li>[ ] Monitoreo y alertas configurados</li>
<li>[ ] Estrategia de backup definida</li>
<li>[ ] Documentación actualizada</li>
</ul>

<h2 id="%F0%9F%9A%A8consideracionesdeproducci%C3%B3n">🚨 Consideraciones de Producción</h2>

<h3 id="antesdeiraproducci%C3%B3n%3A">Antes de ir a producción:</h3>

<ol>
<li><strong>Testing exhaustivo</strong> en entorno staging</li>
<li><strong>Plan de rollback</strong> documentado</li>
<li><strong>Ventana de mantenimiento</strong> programada</li>
<li><strong>Equipo de soporte</strong> disponible</li>
<li><strong>Monitoreo intensivo</strong> durante las primeras 24&#8211;48 horas</li>
</ol>

<h3 id="m%C3%A9tricasamonitorearpost-launch%3A">Métricas a monitorear post-launch:</h3>

<ul>
<li>Tiempo de respuesta de aplicaciones</li>
<li>Utilización de recursos</li>
<li>Errores de aplicación</li>
<li>Conectividad de red</li>
<li>Espacio disponible en disco</li>
</ul>
