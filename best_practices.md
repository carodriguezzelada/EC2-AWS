# Mejores Prácticas para Lanzamiento de Instancias EC2

## 🔐 Seguridad

### Key Pairs y Acceso
- **Nunca compartas tu key pair privado**
- Usa key pairs diferentes para producción y desarrollo
- Considera usar AWS Systems Manager Session Manager para acceso sin SSH
- Implementa rotación regular de keys

### Security Groups
- **Principio de menor privilegio**: Solo abre los puertos necesarios
- Usa rangos de IP específicos en lugar de 0.0.0.0/0 cuando sea posible
- Separa security groups por función (web, database, admin)
- Documenta cada regla y su propósito

### Encriptación
- **Siempre encripta volúmenes EBS** (incluido en el script)
- Usa AWS KMS para gestión de claves
- Habilita encriptación en tránsito cuando sea posible

## 💰 Gestión de Costos

### Dimensionamiento Correcto
- **Airviro (c5.xlarge)**: Adecuado para cargas computacionales intensivas
- **Laravel (t3.large)**: Burstable performance, ideal para apps web
- Monitorea utilización de CPU/memoria para optimizar tamaños

### Optimización de Costos
- Usa **Reserved Instances** para cargas predecibles (hasta 75% descuento)
- Considera **Spot Instances** para workloads tolerantes a interrupciones
- Implementa **Auto Scaling** para ajustar capacidad automáticamente
- Programa paradas automáticas para entornos de desarrollo

## 🏗️ Arquitectura y Diseño

### Alta Disponibilidad
```
Recomendación: Distribuir en múltiples AZ
- AZ-1a: Instancia Airviro principal
- AZ-1b: Instancia Laravel + backup Airviro
```

### Red y Conectividad
- Usa **VPC dedicada** con subnets públicas y privadas
- Implementa **NAT Gateway** para instancias en subnets privadas
- Configura **Route 53** para DNS personalizado
- Considera **Load Balancer** para Laravel si esperas alto tráfico

## 📊 Monitoreo y Logging

### CloudWatch
```bash
# Habilitar métricas detalladas (incluido en script)
--monitoring Enabled=true
```

### Logs Centralizados
- Configura **CloudWatch Logs Agent**
- Implementa log rotation
- Monitorea logs de aplicación y sistema

### Alertas Críticas
- CPU > 80% por 5 minutos
- Memoria > 85%
- Espacio en disco < 15%
- Errores de aplicación

## 🔄 Backup y Recovery

### Snapshots Automatizados
```bash
# Crear snapshot diario
aws ec2 create-snapshot \
    --volume-id vol-xxxxxxxxx \
    --description "Daily backup $(date +%Y-%m-%d)"
```

### Estrategia 3-2-1
- 3 copias de datos críticos
- 2 medios de almacenamiento diferentes
- 1 copia off-site (S3 Glacier)

## 🚀 Deployment y CI/CD

### Para Laravel
```bash
# Post-configuración Laravel
composer install --optimize-autoloader --no-dev
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### Para Airviro
- Documenta dependencias específicas
- Crea scripts de deployment automatizado
- Implementa blue-green deployment

## 🔧 Configuración Post-Launch

### Hardening del Sistema
```bash
# Deshabilitar servicios innecesarios
systemctl disable bluetooth
systemctl disable cups

# Configurar firewall
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# Configurar fail2ban
yum install fail2ban -y
systemctl enable fail2ban
```

### Actualizaciones Automáticas
```bash
# Configurar yum-cron para actualizaciones de seguridad
yum install yum-cron -y
sed -i 's/apply_updates = no/apply_updates = yes/' /etc/yum/yum-cron.conf
systemctl enable yum-cron
```

## 📝 Tagging y Organización

### Tags Recomendados
```json
{
  "Name": "Airviro-Production",
  "Environment": "Production",
  "Application": "Airviro",
  "Owner": "DevOps-Team",
  "CostCenter": "IT-001",
  "Backup": "Daily",
  "Maintenance": "Sunday-2AM"
}
```

## 🔍 Troubleshooting Común

### Problemas de Conexión SSH
```bash
# Verificar security group
aws ec2 describe-security-groups --group-ids sg-xxxxxxxxx

# Verificar status de instancia
aws ec2 describe-instances --instance-ids i-xxxxxxxxx
```

### Problemas de Performance
```bash
# Monitorear recursos
top
iotop
nethogs
```

## 📋 Checklist Pre-Launch

- [ ] AMI de Rocky Linux 9 verificada
- [ ] Key pair creado y almacenado seguramente
- [ ] Security groups configurados correctamente
- [ ] VPC y subnets configuradas
- [ ] IAM roles y policies definidos
- [ ] Monitoreo y alertas configurados
- [ ] Estrategia de backup definida
- [ ] Documentación actualizada

## 🚨 Consideraciones de Producción

### Antes de ir a producción:
1. **Testing exhaustivo** en entorno staging
2. **Plan de rollback** documentado
3. **Ventana de mantenimiento** programada
4. **Equipo de soporte** disponible
5. **Monitoreo intensivo** durante las primeras 24-48 horas

### Métricas a monitorear post-launch:
- Tiempo de respuesta de aplicaciones
- Utilización de recursos
- Errores de aplicación
- Conectividad de red
- Espacio disponible en disco