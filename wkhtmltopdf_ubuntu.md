# Instalación de wkhtmltopdf en Ubuntu

## Requisitos previos

- Ubuntu 18.04, 20.04, 22.04 o 24.04
- Acceso de administrador (sudo)
- Conexión a internet

## Opción 1: Instalación desde repositorios oficiales (Básica)

### Paso 1: Actualizar repositorios

```bash
sudo apt update
```

### Paso 2: Instalar wkhtmltopdf

```bash
sudo apt install wkhtmltopdf -y
```

**Nota:** Esta versión puede tener limitaciones y no incluir soporte completo para JavaScript.

## Opción 2: Instalación desde paquete oficial (Recomendado)

### Paso 1: Descargar el paquete DEB

Para Ubuntu 22.04/24.04:
```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
```

Para Ubuntu 20.04:
```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.focal_amd64.deb
```

Para Ubuntu 18.04:
```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.bionic_amd64.deb
```

### Paso 2: Instalar dependencias

```bash
sudo apt install -y \
    fontconfig \
    libfontconfig1 \
    libfreetype6 \
    libx11-6 \
    libxext6 \
    libxrender1 \
    libjpeg-turbo8 \
    libpng16-16 \
    xfonts-75dpi \
    xfonts-base \
    xvfb
```

### Paso 3: Instalar el paquete DEB

```bash
# Para Ubuntu 22.04/24.04
sudo dpkg -i wkhtmltox_0.12.6.1-3.jammy_amd64.deb

# Para Ubuntu 20.04
sudo dpkg -i wkhtmltox_0.12.6.1-3.focal_amd64.deb

# Para Ubuntu 18.04
sudo dpkg -i wkhtmltox_0.12.6.1-3.bionic_amd64.deb
```

### Paso 4: Resolver dependencias faltantes (si es necesario)

```bash
sudo apt install -f
```

## Opción 3: Instalación usando Snap

```bash
sudo snap install wkhtmltopdf
```

## Opción 4: Compilación desde código fuente

### Paso 1: Instalar herramientas de desarrollo

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    cmake \
    git \
    qt5-qmake \
    qtbase5-dev \
    qtbase5-dev-tools \
    qtwebkit5-dev \
    libqt5webkit5-dev \
    libqt5svg5-dev \
    libqt5xmlpatterns5-dev
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
echo "<h1>Prueba de wkhtmltopdf</h1><p>Instalación exitosa en Ubuntu</p>" > test.html

# Generar PDF
wkhtmltopdf test.html output.pdf
```

### Prueba con contenido web

```bash
wkhtmltopdf https://www.google.com google.pdf
```

## Solución de problemas

### Error de display en servidores sin GUI

Para servidores sin interfaz gráfica:

```bash
# Usar con Xvfb
xvfb-run -a wkhtmltopdf input.html output.pdf
```

### Error "cannot connect to X server"

```bash
# Instalar y configurar Xvfb
sudo apt install xvfb -y

# Crear script wrapper
sudo tee /usr/local/bin/wkhtmltopdf.sh > /dev/null <<EOF
#!/bin/bash
xvfb-run -a --server-args="-screen 0, 1024x768x24" /usr/local/bin/wkhtmltopdf $*
EOF

sudo chmod +x /usr/local/bin/wkhtmltopdf.sh
```

### Problemas con fuentes

```bash
# Instalar fuentes adicionales
sudo apt install -y \
    fonts-liberation \
    fonts-dejavu-core \
    fontconfig \
    fonts-liberation2 \
    fonts-noto-core

# Actualizar cache de fuentes
sudo fc-cache -fv
```

### Error de dependencias en instalación DEB

```bash
# Forzar instalación y resolver dependencias
sudo apt --fix-broken install
sudo dpkg --configure -a
```

## Configuración adicional

### Configurar fuentes personalizadas

```bash
# Crear directorio de fuentes de usuario
mkdir -p ~/.fonts

# Copiar fuentes personalizadas
cp /path/to/your/fonts/* ~/.fonts/

# Actualizar cache de fuentes
fc-cache -fv
```

### Configuración para aplicaciones web

Para Django/Flask u otras aplicaciones web:

```bash
# Agregar al PATH del sistema
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Configurar variables de entorno
echo 'export QT_QPA_PLATFORM=offscreen' >> ~/.bashrc
```

### Script de servicio systemd

Para usar wkhtmltopdf como servicio:

```bash
# Crear archivo de servicio
sudo tee /etc/systemd/system/wkhtmltopdf.service > /dev/null <<EOF
[Unit]
Description=wkhtmltopdf Service
After=network.target

[Service]
Type=forking
User=www-data
Group=www-data
Environment=DISPLAY=:99
ExecStart=/usr/bin/Xvfb :99 -screen 0 1024x768x24
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable wkhtmltopdf.service
sudo systemctl start wkhtmltopdf.service
```

## Uso avanzado

### Opciones comunes de línea de comandos

```bash
# PDF con márgenes personalizados
wkhtmltopdf --margin-top 0.75in --margin-right 0.75in --margin-bottom 0.75in --margin-left 0.75in input.html output.pdf

# PDF con orientación horizontal
wkhtmltopdf --orientation Landscape input.html output.pdf

# PDF con tamaño de página específico
wkhtmltopdf --page-size A4 input.html output.pdf

# PDF con encabezado y pie de página
wkhtmltopdf --header-center "Mi Documento" --footer-center "Página [page] de [topage]" input.html output.pdf
```

### Integración con Python

```python
import subprocess

def html_to_pdf(html_content, output_path):
    cmd = [
        'wkhtmltopdf',
        '--page-size', 'A4',
        '--margin-top', '0.75in',
        '--margin-right', '0.75in',
        '--margin-bottom', '0.75in',
        '--margin-left', '0.75in',
        '-', output_path
    ]
    
    process = subprocess.Popen(cmd, stdin=subprocess.PIPE)
    process.communicate(input=html_content.encode('utf-8'))
    return process.returncode == 0
```

## Desinstalación

### Si se instaló desde repositorios

```bash
sudo apt remove wkhtmltopdf -y
sudo apt autoremove -y
```

### Si se instaló desde paquete DEB

```bash
sudo dpkg -r wkhtmltox
```

### Si se instaló desde Snap

```bash
sudo snap remove wkhtmltopdf
```

### Si se compiló desde código fuente

```bash
sudo rm -f /usr/local/bin/wkhtmltopdf
sudo rm -f /usr/local/bin/wkhtmltoimage
```

## Notas importantes

- La versión de repositorios es más estable pero puede carecer de características
- El paquete oficial incluye soporte completo para JavaScript y CSS
- Para servidores de producción, siempre usa Xvfb
- Verifica la compatibilidad de versiones con tu distribución Ubuntu
- Para aplicaciones críticas, considera usar Docker para encapsular wkhtmltopdf

## Recursos adicionales

- [Documentación oficial](https://wkhtmltopdf.org/)
- [Repositorio GitHub](https://github.com/wkhtmltopdf/wkhtmltopdf)
- [Opciones de línea de comandos](https://wkhtmltopdf.org/usage/wkhtmltopdf.txt)
- [Problemas conocidos](https://github.com/wkhtmltopdf/wkhtmltopdf/issues)