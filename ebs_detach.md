# 🔗 Guía: Desvincular Disco EBS de Instancia EC2

## ⚠️ ADVERTENCIAS CRÍTICAS

### 🚨 Antes de Desvincular:
- **NUNCA desvincules el volumen root** (/) - destruirá tu instancia
- **HAZ BACKUP** de datos importantes antes de proceder
- **DETÉN aplicaciones** que usen el disco a desvincular
- **VERIFICA** que no sea un volumen crítico para el sistema

---

## 📋 Información Necesaria Antes de Empezar

### Datos a Recopilar:
1. **Instance ID**: `i-xxxxxxxxx`
2. **Volume ID**: `vol-xxxxxxxxx` 
3. **Punto de montaje**: `/usr/airviro` (ejemplo)
4. **Device name**: `/dev/xvdf` (ejemplo)

---

## 🛡️ Parte 1: Preparación y Verificación (SSH)

### Paso 1.1: Conectarse a la Instancia
```bash
ssh -i tu-keypair.pem ec2-user@tu-ip-publica
```

### Paso 1.2: Identificar el Volumen a Desvincular
```bash
# Ver todos los discos montados
df -h

# Ver todos los discos disponibles
lsblk

# Identificar cuál es tu volumen objetivo
# Ejemplo de salida:
# NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvda    202:0    0  50G  0 disk 
# └─xvda1 202:1    0  50G  0 part /          <- VOLUMEN ROOT (NO TOCAR)
# xvdf    202:80   0 100G  0 disk 
# └─xvdf1 202:81   0 100G  0 part /usr/airviro  <- ESTE ES EL QUE QUIERES
```

### Paso 1.3: Verificar Procesos Usando el Disco
```bash
# Ver qué procesos están usando el directorio
sudo lsof +D /usr/airviro

# Ver procesos usando el dispositivo
sudo fuser -m /dev/xvdf
```

### Paso 1.4: Crear Backup (Recomendado)
```bash
# Opción 1: Backup con tar (si el volumen es pequeño)
sudo tar -czf /tmp/airviro-backup-$(date +%Y%m%d).tar.gz -C /usr/airviro .

# Opción 2: Crear snapshot desde AWS Console (más seguro)
# Se hará desde la consola AWS en el siguiente paso
```

---

## 📸 Parte 2: Crear Snapshot (Backup) - Consola AWS

### Paso 2.1: Acceder a Volúmenes EBS
1. **AWS Console** → **EC2** → **Volumes**
2. **Buscar** tu volumen por Volume ID o filtrar por instancia

### Paso 2.2: Crear Snapshot
1. **Seleccionar** el volumen a desvincular
2. Click **"Actions"** → **"Create snapshot"**
3. **Configurar snapshot**:
   ```
   Description: Backup antes de detach - /usr/airviro - [FECHA]
   Tags:
     - Key: Name, Value: Backup-Airviro-PreDetach
     - Key: Purpose, Value: Safety backup before detachment
     - Key: Date, Value: [FECHA ACTUAL]
   ```
4. Click **"Create snapshot"**
5. **Esperar** que el estado cambie a `completed` (puede tomar varios minutos)

---

## 🔄 Parte 3: Desmontar el Volumen (SSH)

### Paso 3.1: Detener Aplicaciones
```bash
# Detener servicios que usen el directorio (ejemplo para Airviro)
sudo systemctl stop airviro  # Ajustar según tu aplicación
sudo systemctl stop apache2  # Si aplica
sudo systemctl stop nginx    # Si aplica

# Verificar que no hay procesos usando el directorio
sudo lsof +D /usr/airviro
```

### Paso 3.2: Sync y Flush
```bash
# Sincronizar datos pendientes al disco
sync

# Flush buffers específicos del dispositivo
sudo blockdev --flushbufs /dev/xvdf
```

### Paso 3.3: Desmontar el Sistema de Archivos
```bash
# Desmontar el volumen
sudo umount /usr/airviro

# Si da error "device is busy", forzar (CUIDADO):
# sudo umount -l /usr/airviro  # Lazy unmount
# sudo umount -f /usr/airviro  # Force unmount (último recurso)
```

### Paso 3.4: Verificar Desmontaje
```bash
# Verificar que ya no está montado
df -h | grep airviro
mount | grep airviro

# No debe mostrar resultados
```

### Paso 3.5: Remover del fstab (si está configurado)
```bash
# Hacer backup del fstab
sudo cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d)

# Editar fstab
sudo nano /etc/fstab

# COMENTAR (agregar # al inicio) o ELIMINAR la línea del volumen:
# UUID=12345678-1234-1234-1234-123456789abc /usr/airviro ext4 defaults,nofail 0 2

# Guardar y salir (Ctrl+X, Y, Enter)
```

### Paso 3.6: Verificar fstab
```bash
# Probar configuración del fstab
sudo mount -a

# No debe dar errores relacionados con el volumen removido
```

---

## 🔌 Parte 4: Desvincular desde AWS Console

### Paso 4.1: Identificar el Volumen en AWS
1. **AWS Console** → **EC2** → **Volumes**
2. **Filtrar** por tu Instance ID o buscar por Volume ID
3. **Verificar**:
   - State: `in-use`
   - Attachment information muestra tu instancia
   - Device: `/dev/sdf` (aparece como `/dev/xvdf` en la instancia)

### Paso 4.2: Desvincular el Volumen
1. **Seleccionar** el volumen objetivo
2. Click **"Actions"** → **"Detach volume"**
3. **Confirmar detachment**:
   ```
   Instance: i-xxxxxxxxx (tu instancia)
   Device: /dev/sdf
   
   ⚠️ WARNING: Detaching may cause data loss if the volume 
   is not properly unmounted from within the instance.
   ```
4. **Verificar** que desmontaste correctamente en los pasos anteriores
5. Click **"Detach volume"**

### Paso 4.3: Verificar Estado del Volumen
- **Estado** cambiará de `in-use` a `available`
- **Attached instances** estará vacío
- El proceso puede tomar 1-2 minutos

---

## ✅ Parte 5: Verificación Post-Detachment

### Paso 5.1: Verificar desde la Instancia (SSH)
```bash
# Verificar que el dispositivo ya no está disponible
lsblk

# El dispositivo /dev/xvdf ya no debe aparecer

# Verificar que el directorio existe pero está vacío
ls -la /usr/airviro/
# Debe mostrar un directorio vacío
```

### Paso 5.2: Verificar desde AWS Console
1. **Volumes** → Tu volumen debe mostrar:
   - **State**: `available`
   - **Attachment information**: `Not attached`
   - **Volume ID**: Conserva su ID original

### Paso 5.3: Probar Reinicio de Instancia (Opcional)
```bash
# Reiniciar para verificar que todo funciona sin el volumen
sudo reboot

# Después del reinicio, verificar:
df -h
lsblk
# El volumen no debe aparecer
```

---

## 🗂️ Parte 6: Opciones Post-Detachment

### Opción A: Eliminar el Volumen
```
⚠️ PELIGRO: Esta acción es IRREVERSIBLE
```
1. **Seleccionar** volumen en estado `available`
2. **Actions** → **"Delete volume"**
3. **Confirmar eliminación**
4. 💰 **Ahorro**: Se detiene el costo del volumen

### Opción B: Conservar el Volumen
- **Mantener** en estado `available`
- **Usar** para otra instancia en el futuro
- **Costo**: Continúa cobrando storage (~$10/mes por 100GB)

### Opción C: Crear AMI del Volumen
1. **Actions** → **"Create image from EBS snapshot"**
2. **Usar** como template para futuras instancias

---

## 🚨 Troubleshooting

### Problema: "Device is busy" al desmontar
```bash
# Identificar procesos que usan el directorio
sudo lsof +D /usr/airviro

# Matar procesos específicos (CUIDADO)
sudo kill -9 [PID]

# O usar lazy unmount como último recurso
sudo umount -l /usr/airviro
```

### Problema: Error al detach desde AWS Console
**Posibles causas**:
- Volumen aún montado en la instancia
- Procesos activos usando el volumen
- Volumen es el root volume

**Solución**:
1. Volver al SSH y verificar desmontaje
2. Usar `sudo fuser -km /dev/xvdf` (mata todos los procesos)
3. Intentar detach nuevamente

### Problema: Instancia no bootea después del detach
**Si removiste por error el volumen root**:
1. **PARAR** la instancia (NO terminar)
2. **Crear** nueva instancia con AMI similar
3. **Attach** volumen root original a nueva instancia
4. **Modificar** `/etc/fstab` si es necesario

### Problema: Pérdida de datos
**Si no hiciste backup**:
- 💔 Los datos están perdidos si eliminaste el volumen
- 🔄 Si aún existe, puedes re-attach y montar
- 📸 Siempre crear snapshot antes de operaciones riesgosas

---

## 📝 Checklist de Verificación

### Antes del Detachment:
- [ ] ✅ Identificado volumen correcto (NO es root)
- [ ] ✅ Backup/snapshot creado
- [ ] ✅ Aplicaciones detenidas
- [ ] ✅ Procesos que usan el disco identificados
- [ ] ✅ Sistema sincronizado (sync)

### Durante el Desmontaje:
- [ ] ✅ Directorio desmontado exitosamente
- [ ] ✅ fstab editado (si aplicaba)
- [ ] ✅ No hay errores en `mount -a`
- [ ] ✅ Dispositivo no aparece en `lsblk`

### Durante el Detachment:
- [ ] ✅ Volumen en estado `available`
- [ ] ✅ No errores en AWS Console
- [ ] ✅ Attachment information vacía

### Post-Detachment:
- [ ] ✅ Instancia funciona normalmente
- [ ] ✅ Aplicaciones principales funcionan
- [ ] ✅ No errores en logs del sistema
- [ ] ✅ Decisión tomada sobre el volumen (eliminar/conservar)

---

## 📊 Comandos de Resumen para Verificación

```bash
#!/bin/bash
echo "=== VERIFICACIÓN PRE-DETACHMENT ==="
echo "1. Discos montados:"
df -h

echo -e "\n2. Todos los discos:"
lsblk

echo -e "\n3. Procesos usando /usr/airviro:"
sudo lsof +D /usr/airviro 2>/dev/null || echo "Ningún proceso encontrado"

echo -e "\n4. Configuración fstab:"
grep -v "^#" /etc/fstab

echo -e "\n=== EJECUTAR DESPUÉS DE DESMONTAR ==="
echo "sudo umount /usr/airviro"
echo "df -h | grep airviro  # No debe mostrar resultados"
echo "lsblk  # Verificar que xvdf no aparece después del detach"
```

¿En qué volumen específico necesitas trabajar? ¿Es el volumen de 100GB que mencionaste antes o otro diferente?