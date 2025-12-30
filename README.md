# 📂 Integración de Google Drive en Ubuntu con rclone

## 0. ¿Qué hace exactamente esta configuración?

Tras la configuración, aparecerá en tu Carpeta personal una carpeta llamada `GoogleDrive`. Esta actúa como un enlace directo a tu nube: puedes trabajar con los archivos desde tu computadora y rclone sincronizará los cambios automáticamente con Google Drive.

### Comportamiento general

Esta configuración sigue reglas claras y consistentes:

- 💻 **Edición desde la computadora (uso principal)**  
  Cuando abres o editas un archivo dentro de la carpeta `GoogleDrive`:
  - el archivo se descarga
  - lo editas **con los programas instalados en tu computadora**
  - el cambio se guarda primero en tu disco duro (SSD)
  - si hay conexión, rclone **sube el archivo modificado** a Google Drive  
  (Para ti, el archivo se comporta como cualquier archivo local.)

- 📥 **Descarga bajo demanda**  
  Ningún archivo se descarga automáticamente. Un archivo solo se descarga cuando:
  - lo abres
  - lo copias
  - lo editas
  - un programa lo necesita

- 💾 **Caché local limitada (60 GB)**  
  Los archivos usados se almacenan temporalmente en tu computadora en un espacio oculto llamado **caché**. La caché puede ocupar **hasta 60 GB**.  
  Cuando se alcanza ese límite, rclone elimina los archivos **más antiguos**, sin afectar los archivos en Google Drive.

- ⏳ **Disponibilidad sin internet (14 días)**  
  Una vez descargado, un archivo permanece disponible **hasta 14 días**, incluso sin conexión.  
  Si lo vuelves a usar dentro de ese periodo, el tiempo se renueva automáticamente.

- 🌐 **Tolerancia a fallos de red**  
  Si la conexión se interrumpe mientras trabajas:
  - los cambios se guardan localmente
  - la sincronización se reanuda automáticamente cuando vuelve el internet  
  No necesitas intervenir.

### ¿Qué pasa si editas desde otro lugar?

Esta configuración **sí detecta cambios externos**, pero con un comportamiento específico:

- 🌍 **Edición desde la página web de Google Drive**  
  Si editas un archivo desde el navegador:
  - el cambio se guarda directamente en la nube
  - rclone **no descarga el archivo automáticamente** a tu computadora
  - el archivo se descargará **la próxima vez que lo abras o lo uses desde `~/GoogleDrive`**  
  Es decir, verás la versión actualizada cuando accedas al archivo.

- 📱 **Edición desde el teléfono o tablet**  
  El comportamiento es el mismo que desde el navegador:
  - los cambios se guardan en Google Drive
  - rclone no los baja hasta que los necesites localmente
  - cuando abras el archivo en tu computadora, se descargará la versión más reciente

- ⚠️ **Ediciones simultáneas**  
  Si editas el mismo archivo **al mismo tiempo**:
  - una copia local en tu computadora
  - y otra desde la web o el teléfono  
  Google Drive puede crear un archivo duplicado para evitar sobrescritura.  
  Esto es un comportamiento normal del servicio, no un error de rclone.

### Qué NO hace esta configuración

Para evitar malentendidos:

- ❌ No descarga todo tu Drive
- ❌ No mantiene todos los archivos siempre en tu computadora
- ❌ No sincroniza cambios en tiempo real
- ❌ No es edición colaborativa en vivo
- ❌ No sustituye un sistema de respaldo independiente

### En pocas palabras

> Trabajas **desde tu computadora** como siempre.  
> Rclone se encarga de descargar lo necesario, guardar los cambios localmente y sincronizarlos cuando puede, sin que tengas que preocuparte por la conexión.

---

## 1. Requisitos

- Ubuntu 24.04 LTS o superior
- Cuenta de Google Drive
- Conexión a internet (solo para configurar)
- Al menos 60 GB libres en disco

---

## 2. Instalar rclone

  Ejecuta la siguiente línea en la terminal:

    sudo -v && curl https://rclone.org/install.sh | sudo bash

  Para verificar la instalación, comprueba la versión instalada ejecutando el comando:

    rclone version

Debe mostrar un número de versión (`rclone v1.xx.x`).

---

## 3. Conectar Google Drive

Ejecuta lo siguiente:

    rclone config

Responde exactamente así en el asistente interactivo:

1. `n` → New remote  
2. **name:** `remote`  
3. **Storage:** Selecciona Google Drive (número 22, pero verifica) 
4. **Client ID:** Enter  
5. **Client Secret:** Enter  
6. **Scope:** opción `1` (Full access)
7. **service_account_file:** Enter
8. **Advanced config:** `n`  
9. **Auto config:** `y`  
   (Se abrirá el navegador, inicia sesión y autoriza)
10. Confirma con `y`
11. Confirma con `y`
12. Sal con `q`

---

## 4. Crear carpetas necesarias

Ejecuta las siguientes líneas, una por una:

    mkdir -p ~/GoogleDrive
    mkdir -p ~/.config/systemd/user
    mkdir -p ~/.cache/rclone

- `~/GoogleDrive` → punto de montaje  
- `systemd/user` → servicios automáticos del usuario  
- `~/.cache/rclone` → caché y registros  

---

## 5. Crear el servicio automático (systemd)

Este servicio monta Drive automáticamente al iniciar sesión.

### 5.1 Abrir el archivo del servicio

Ejecuta en la terminal

    nano ~/.config/systemd/user/rclone-mount.service

Esto abre un editor de texto. Copia y pega todo el contenido siguiente dentro del editor:

    [Unit]
    Description=Google Drive (montaje estable con rclone)
    After=network-online.target
    Wants=network-online.target
    AssertPathIsDirectory=%h/GoogleDrive

    [Service]
    Type=notify

    ExecStart=/usr/bin/rclone mount \
        remote: %h/GoogleDrive \
        --config=%h/.config/rclone/rclone.conf \
        --vfs-cache-mode full \
        --vfs-cache-max-size 60G \
        --vfs-cache-max-age 14d \
        --dir-cache-time 10m \
        --poll-interval 1m \
        --drive-use-trash \
        --umask 022 \
        --log-file %h/.cache/rclone/mount.log \
        --log-level INFO

    ExecStop=/bin/fusermount -uz %h/GoogleDrive

    Restart=always
    RestartSec=15

    [Install]
    WantedBy=default.target

### Para guardar y salir de nano:

- Ctrl + O → guardar  
- Enter → confirmar  
- Ctrl + X → salir  

---

## 6. Activar el servicio

Ejecuta ambos comandos, uno tras otro

    systemctl --user daemon-reload
    systemctl --user enable --now rclone-mount.service

### Verificar estado

    systemctl --user status rclone-mount.service

Debe indicar:

    active (running)

---

## 7. Comprobación final

### 7.1 Ver archivos

    ls ~/GoogleDrive

También puedes abrir el explorador de archivos y entrar a `GoogleDrive`.

---

### 7.2 Prueba de escritura

Escribe en la terminal

    echo "Prueba de sincronización" > ~/GoogleDrive/test_rclone.txt

Espera unos segundos y verifica que el archivo aparezca en Google Drive
(desde el navegador o celular).

---

## 🔧 Comandos útiles

### Ver actividad y errores

    tail -f ~/.cache/rclone/mount.log

### Reiniciar el montaje

    systemctl --user restart rclone-mount.service

### Detener temporalmente

    systemctl --user stop rclone-mount.service

---

## 8.0 🗑️ Desinstalación y limpieza total

Si deseas revertir todos los cambios y eliminar rclone de tu sistema, sigue estos pasos en orden estricto.

**Nota técnica:** Dado que instalamos rclone manualmente con el script oficial (Paso 2), no podemos usar `apt remove`. Debemos eliminar los binarios manualmente.

### 1. Detener y limpiar el servicio
Primero detenemos el proceso para liberar los archivos. Ejecuta las siguientes instrucciones (una por una):

    systemctl --user stop rclone-mount.service
    systemctl --user disable rclone-mount.service

Ahora elimina el archivo de configuración del servicio y recarga el gestor:

    rm ~/.config/systemd/user/rclone-mount.service
    systemctl --user daemon-reload

### 2. Eliminar directorios de configuración y caché
Borramos las credenciales de Google, la caché de disco y los logs.

Ejecuta:

    rm -rf ~/.config/rclone
    rm -rf ~/.cache/rclone

### 3. Eliminar la carpeta de montaje
**⚠️ Precaución:** Asegúrate de que la carpeta `~/GoogleDrive` esté vacía. Si ves tus archivos dentro, significa que el **Paso 1** falló (el disco sigue montado).

Si la carpeta está vacía, elimínala con:

    rmdir ~/GoogleDrive

*(Usamos `rmdir` por seguridad: si la carpeta tiene archivos, el comando fallará para evitar borrados accidentales).*

### 4. Limpieza de carpetas del sistema (Opcional)
En la instalación creamos la ruta para los servicios de usuario. Si esa carpeta ha quedado vacía tras el paso 1, podemos borrarla.

Ejecuta:

    rmdir ~/.config/systemd/user 2>/dev/null

*(Si el comando no hace nada o da error, ignóralo; significa que tienes otros servicios de usuario ajenos a esta guía y no debemos borrar la carpeta).*

### 5. Eliminar el programa rclone
Finalmente, eliminamos el ejecutable instalado por el script y su documentación.

Ejecuta:

    sudo rm /usr/bin/rclone
    sudo rm /usr/local/share/man/man1/rclone.1

Tras esto, no queda rastro de la configuración ni del programa en tu sistema.
