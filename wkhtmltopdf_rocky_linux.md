# Instalación de wkhtmltopdf en Rocky Linux 9

## Requisitos previos

- Rocky Linux 9.x
- Acceso de administrador (sudo)
- Conexión a internet

## Opción 1: Instalación desde repositorios EPEL (Recomendado)

### Paso 1: Habilitar el repositorio EPEL

```bash
sudo dnf install epel-release -y
```

### Paso 2: Instalar wkhtmltopdf

```bash
sudo dnf install wkhtmltopdf -y
```

### Paso 3: Verificar la instalación

```bash
wkhtmltopdf --version
```

## Opción 2: Instalación desde paquete oficial

### Paso 1: Descargar el paquete RPM

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox-0.12.6.1-3.almalinux9.x86_64.rpm
```

### Paso 2: Instalar dependencias

```bash
sudo dnf install -y \
    xorg-x11-fonts-75dpi \
    xorg-x11-fonts-Type1 \
    xorg-x11-fonts-truetype \
    liberation-fonts \
    fontconfig \
    freetype \
    libX11 \
    libXext \
    libXrender \
    libjpeg-turbo \
    libpng \
    xorg-x11-server-Xvfb
```

### Paso 3: Instalar el paquete RPM

```bash
sudo rpm -Uvh wkhtmltox-0.12.6.1-3.almalinux9.x86_64.rpm
```

## Opción 3: Compilación desde código fuente

### Paso 1: Instalar herramientas de desarrollo

```bash
sudo dnf groupinstall "Development Tools" -y
sudo dnf install -y \
    qt5-qtbase-devel \
    qt5-qtwebkit-devel \
    qt5-qtsvg-devel \
    qt5-qtxmlpatterns-devel \
    cmake \
    git
```

### Paso 2: Clonar el repositorio

```bash
git clone https://github.com/wkhtmltopdf/wkhtmltopdf.git
cd wkhtmltopdf
git submodule update --init --recursive
```

### Paso 3: Compilar e instalar

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
```

## Verificación y pruebas

### Verificar versión instalada

```bash
wkhtmltopdf --version
```

### Crear un PDF de prueba

```bash
# Crear archivo HTML de prueba
echo "<h1>Prueba de wkhtmltopdf</h1><p>Instalación exitosa en Rocky Linux 9</p>" > test.html

# Generar PDF
wkhtmltopdf test.html output.pdf
```

### Prueba con contenido web

```bash
wkhtmltopdf https://www.google.com google.pdf
```

## Solución de problemas

### Error de display en servidores sin GUI

Si usas un servidor sin interfaz gráfica:

```bash
# Instalar Xvfb
sudo dnf install xorg-x11-server-Xvfb -y

# Usar con Xvfb
xvfb-run -a wkhtmltopdf input.html output.pdf
```

### Error de dependencias

Si encuentras errores de dependencias:

```bash
# Instalar dependencias adicionales
sudo dnf install -y \
    glibc \
    libgcc \
    libstdc++ \
    zlib
```

### Uso con aplicaciones web

Para aplicaciones web (como Django, Flask, etc.):

```bash
# Agregar al PATH del sistema
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Configuración adicional

### Configurar fuentes personalizadas

```bash
# Crear directorio de fuentes
sudo mkdir -p /usr/share/fonts/custom

# Copiar fuentes personalizadas
sudo cp /path/to/your/fonts/* /usr/share/fonts/custom/

# Actualizar cache de fuentes
sudo fc-cache -fv
```

### Configuración de seguridad

Para entornos de producción, considera estas configuraciones:

```bash
# Crear usuario específico para wkhtmltopdf
sudo useradd -r -s /bin/false wkhtmltopdf-user

# Configurar permisos
sudo chown -R wkhtmltopdf-user:wkhtmltopdf-user /tmp/wkhtmltopdf/
```

## Notas importantes

- La versión EPEL puede ser más antigua pero es más estable
- El paquete oficial incluye todas las características
- La compilación desde código fuente permite personalización completa
- Siempre verifica la integridad de los archivos descargados
- Para servidores de producción, usa Xvfb para evitar problemas de display

## Recursos adicionales

- [Documentación oficial](https://wkhtmltopdf.org/)
- [Repositorio GitHub](https://github.com/wkhtmltopdf/wkhtmltopdf)
- [Opciones de línea de comandos](https://wkhtmltopdf.org/usage/wkhtmltopdf.txt)