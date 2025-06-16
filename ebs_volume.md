# 💾 Agregar Volumen EBS de 100GB a Instancia Airviro

## 📋 Información Previa Necesaria
- **IP de tu instancia**: 44.195.187.47
- **Directorio destino**: `/usr/airviro`
- **Tamaño volumen**: 100 GB

---

## 🎯 Parte 1: Crear el Volumen EBS en AWS Console

### Paso 1.1: Acceder a Volúmenes EBS
1. **Ir a EC2 Console** → `https://console.aws.amazon.com/ec2/`
2. En el menú lateral izquierdo, bajo **"Elastic Block Store"**
3. Click en **"Volumes"**

### Paso 1.2: Crear Nuevo Volumen
1. Click en **"Create volume"** (botón azul)
2. **Configuración del volumen**:
   ```
   Volume type: gp3 (General Purpose SSD)
   Size (GiB): 100
   IOPS: 3000 (default para gp3)
   Throughput (MiB/s): 125 (default para gp3)
   ```

### Paso 1.3: Configurar Ubicación y Encriptación
1. **Availability Zone**: 
   - ⚠️ **CRÍTICO**: Debe ser la MISMA AZ que tu instancia Airviro
   - Para verificar AZ de tu instancia:
     - Ve a **Instances** → Selecciona tu instancia Airviro
     - En la pestaña **Details**, busca **"Availability Zone"**
     - Ejemplo: `us-east-1a`
   
2. **Snapshot ID**: Dejar en blanco (volumen nuevo)

3. **Encryption**: 
   - ✅ **Encrypt this volume**: Activado
   - **KMS key**: Usar `(default) aws/ebs`

### Paso 1.4: Agregar Tags
```
Key: Name           Value: Airviro-Data-Volume
Key: Purpose        Value: Airviro Application Data
Key: Size           Value: 100GB
Key: AttachedTo     Value: Airviro-Production
```

### Paso 1.5: Crear Volumen
1. **Revisar configuración**
2. Click **"Create volume"**
3. ✅ **Confirmación**: "Successfully created volume vol-xxxxxxxxx"
4. **Anotar el Volume ID** (lo necesitarás): `vol-xxxxxxxxx`

---

## 🔗 Parte 2: Attachar el Volumen a la Instancia

### Paso 2.1: Seleccionar el Volumen
1. En la lista de **Volumes**, buscar tu volumen recién creado
2. **Estado** debe ser: `available`
3. **Seleccionar** el volumen (checkbox)

### Paso 2.2: Attachar a la Instancia
1. Click **"Actions"** → **"Attach volume"**
2. **Instance ID**: 
   - Click en el campo y buscar tu instancia Airviro
   - Debería aparecer como `i-xxxxxxxxx (Airviro-Production)`
3. **Device name**: 
   - Usar `/dev/sdf` (recomendado)
   - ⚠️ **Nota**: En la instancia aparecerá como `/dev/xvdf`
4. Click **"Attach volume"**

### Paso 2.3: Verificar Attachment
- **Estado** cambiará a: `in-use`
- **Attached instances** mostrará tu instancia

---

## 💻 Parte 3: Configurar el Volumen en la Instancia (SSH)

### Paso 3.1: Conectarse a la Instancia
```bash
ssh -i tu-keypair.pem ec2-user@44.195.187.47
```

### Paso 3.2: Verificar que el Volumen está Visible
```bash
# Listar todos los discos
lsblk

# Deberías ver algo como:
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda    202:0    0  50G  0 disk 
# └─xvda1 202:1    0  50G  0 part /
# xvdf    202:80   0 100G  0 disk    <- Nuevo volumen
```

### Paso 3.3: Verificar si el Volumen tiene Sistema de Archivos
```bash
# Verificar si ya tiene formato
sudo file -s /dev/xvdf

# Si muestra "data", significa que está vacío y necesita formato
# Si muestra un sistema de archivos, ya está formateado
```

### Paso 3.4: Formatear el Volumen (si es necesario)
```bash
# Formatear con ext4 (recomendado para Linux)
sudo mkfs -t ext4 /dev/xvdf

# Salida esperada:
# mke2fs 1.45.6 (20-Mar-2020)
# Creating filesystem with 26214400 4k blocks and 6553600 inodes
# ...
```

### Paso 3.5: Crear el Directorio de Montaje
```bash
# Crear directorio /usr/airviro
sudo mkdir -p /usr/airviro

# Verificar que se creó
ls -la /usr/airviro
```

### Paso 3.6: Montar el Volumen Temporalmente
```bash
# Montar el volumen
sudo mount /dev/xvdf /usr/airviro

# Verificar que está montado
df -h /usr/airviro

# Salida esperada:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvdf        98G   61M   93G   1% /usr/airviro
```

### Paso 3.7: Configurar Permisos
```bash
# Cambiar propietario (opcional, según necesidades de Airviro)
sudo chown -R ec2-user:ec2-user /usr/airviro

# O si Airviro necesita un usuario específico:
# sudo chown -R airviro:airviro /usr/airviro

# Establecer permisos apropiados
sudo chmod 755 /usr/airviro
```

---

## 🔄 Parte 4: Configurar Montaje Automático (Permanente)

### Paso 4.1: Obtener UUID del Volumen
```bash
# Obtener UUID único del volumen
sudo blkid /dev/xvdf

# Salida ejemplo:
# /dev/xvdf: UUID="12345678-1234-1234-1234-123456789abc" TYPE="ext4"

# Copiar el UUID completo
```

### Paso 4.2: Hacer Backup del fstab
```bash
# Crear backup del archivo fstab
sudo cp /etc/fstab /etc/fstab.backup

# Verificar backup
ls -la /etc/fstab*
```

### Paso 4.3: Agregar Entrada al fstab
```bash
# Editar fstab
sudo nano /etc/fstab

# Agregar al final del archivo (reemplaza UUID con el tuyo):
UUID=12345678-1234-1234-1234-123456789abc /usr/airviro ext4 defaults,nofail 0 2

# Guardar y salir (Ctrl+X, Y, Enter)
```

### Paso 4.4: Verificar Configuración fstab
```bash
# Probar la configuración sin reiniciar
sudo mount -a

# Si no hay errores, está correcto
# Verificar que sigue montado
df -h /usr/airviro
```

### Paso 4.5: Verificar Montaje Permanente
```bash
# Desmontar temporalmente
sudo umount /usr/airviro

# Montar usando fstab
sudo mount /usr/airviro

# Verificar
df -h /usr/airviro
```

---

## ✅ Parte 5: Verificación Final y Pruebas

### Paso 5.1: Crear Archivos de Prueba
```bash
# Crear archivo de prueba
echo "Volumen Airviro funcionando correctamente - $(date)" | sudo tee /usr/airviro/test.txt

# Verificar
cat /usr/airviro/test.txt
```

### Paso 5.2: Verificar Espacio Disponible
```bash
# Ver espacio total y disponible
df -h /usr/airviro

# Ver detalles del sistema de archivos
sudo tune2fs -l /dev/xvdf | grep -E "(Block count|Block size|Free blocks)"
```

### Paso 5.3: Probar Reinicio (Opcional)
```bash
# Reiniciar instancia para verificar montaje automático
sudo reboot

# Después del reinicio, reconectar y verificar:
df -h /usr/airviro
ls -la /usr/airviro/
```

---

## 📊 Información del Volumen Configurado

| Aspecto | Configuración |
|---------|---------------|
| **Tamaño** | 100 GB |
| **Tipo** | gp3 (General Purpose SSD) |
| **IOPS** | 3000 |
| **Throughput** | 125 MiB/s |
| **Encriptación** | ✅ Habilitada |
| **Punto de montaje** | `/usr/airviro` |
| **Sistema de archivos** | ext4 |
| **Montaje automático** | ✅ Configurado |

---

## 🚨 Troubleshooting Común

### Problema: "No space left on device"
```bash
# Verificar espacio disponible
df -h

# Verificar inodos disponibles
df -i
```

### Problema: "mount: wrong fs type"
```bash
# Verificar tipo de sistema de archivos
sudo blkid /dev/xvdf

# Re-formatear si es necesario
sudo mkfs.ext4 /dev/xvdf
```

### Problema: Volumen no aparece después de reboot
```bash
# Verificar fstab
cat /etc/fstab

# Montar manualmente
sudo mount -a

# Ver logs de errores
dmesg | tail -20
```

### Problema: Permisos incorrectos
```bash
# Verificar permisos actuales
ls -la /usr/airviro

# Corregir permisos
sudo chown -R ec2-user:ec2-user /usr/airviro
sudo chmod 755 /usr/airviro
```

---

## 📝 Comandos de Resumen para Verificación

```bash
# Verificación completa del estado del volumen
echo "=== VERIFICACIÓN VOLUMEN AIRVIRO ==="
echo "1. Discos disponibles:"
lsblk | grep -E "(xvd|nvme)"

echo -e "\n2. Espacio disponible:"
df -h /usr/airviro

echo -e "\n3. Punto de montaje:"
mount | grep airviro

echo -e "\n4. UUID del volumen:"
sudo blkid /dev/xvdf

echo -e "\n5. Entrada en fstab:"
grep airviro /etc/fstab

echo -e "\n6. Contenido del directorio:"
ls -la /usr/airviro/

echo -e "\n7. Archivo de prueba:"
cat /usr/airviro/test.txt 2>/dev/null || echo "Archivo de prueba no encontrado"
```

---

## ⚠️ Notas Importantes

1. **Backup**: Siempre haz backup antes de modificar `/etc/fstab`
2. **AZ Matching**: El volumen DEBE estar en la misma Availability Zone que la instancia
3. **Costo**: Un volumen gp3 de 100GB cuesta aproximadamente $8-10/mes
4. **Snapshots**: Considera crear snapshots regulares para backup
5. **Permisos**: Ajusta los permisos según los requerimientos específicos de Airviro