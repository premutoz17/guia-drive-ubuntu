# 📂 Integración de Google Drive en Ubuntu

## 0. ¿Qué hace exactamente esta configuración?

Al finalizar, tendrás una carpeta especial llamada `GoogleDrive`. Esa carpeta **no es una copia completa** de tu Google Drive, es una carpeta que te permite **abrir, editar y guardar archivos desde tu computadora**, mientras rclone se encarga de sincronizarlos con Google Drive de forma segura.

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
  Ningún archivo se descarga automáticamente.  
  Un archivo solo se descarga cuando:
  - lo abres
  - lo copias
  - lo editas
  - un programa lo necesita

- 💾 **Caché local limitada (60 GB)**  
  Los archivos usados se almacenan temporalmente en tu disco.  
  La caché puede ocupar **hasta 60 GB**.  
  Cuando se alcanza ese límite, rclone elimina del disco local los archivos **menos usados o más antiguos**, sin afectar los archivos en Google Drive.

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

### 🟢 COMANDO ÚNICO  
(Ejecuta una sola línea)

    sudo -v && curl https://rclone.org/install.sh | sudo bash

### Verificación (comando único)

    rclone version

Debe mostrar un número de versión (`rclone v1.xx.x`).

---

## 3. Conectar Google Drive

### 🟢 COMANDO ÚNICO

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

### 🟦 BLOQUE DE VARIOS COMANDOS  
(Ejecuta todas las líneas, una por una)

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

### 🟢 COMANDO ÚNICO

    nano ~/.config/systemd/user/rclone-mount.service

### 5.2 Contenido del archivo

⚠️ IMPORTANTE  
Las líneas con `\` NO son comandos separados.  
Forman un solo comando largo (`ExecStart`).

Copia y pega todo el contenido dentro del editor:

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

### Guardar y salir de nano

- Ctrl + O → guardar  
- Enter → confirmar  
- Ctrl + X → salir  

---

## 6. Activar el servicio

### 🟦 BLOQUE DE VARIOS COMANDOS  
(Ejecuta ambos comandos, uno tras otro)

    systemctl --user daemon-reload
    systemctl --user enable --now rclone-mount.service

### Verificar estado

### 🟢 COMANDO ÚNICO

    systemctl --user status rclone-mount.service

Debe indicar:

    active (running)

---

## 7. Comprobación final

### 7.1 Ver archivos

### 🟢 COMANDO ÚNICO

    ls ~/GoogleDrive

También puedes abrir el explorador de archivos y entrar a `GoogleDrive`.

---

### 7.2 Prueba de escritura

### 🟢 COMANDO ÚNICO

    echo "Prueba de sincronización" > ~/GoogleDrive/test_rclone.txt

Espera unos segundos y verifica que el archivo aparezca en Google Drive
(navegador o celular).

---

## 8. Comportamiento diario esperado

- Abrir archivos → descarga bajo demanda
- Guardar → primero en disco local, luego en la nube
- Sin internet → acceso a archivos usados en los últimos 14 días
- Caché llena → rclone borra automáticamente lo más antiguo

---

## 🔧 Comandos útiles

### Ver actividad y errores

    tail -f ~/.cache/rclone/mount.log

### Reiniciar el montaje

    systemctl --user restart rclone-mount.service

### Detener temporalmente

    systemctl --user stop rclone-mount.service

---

## ✅ Conclusión

Esta configuración es:

- Estable
- Conservadora con los datos
- Adecuada para uso diario
- Pensada para funcionar durante años sin mantenimiento

Es la forma más segura y limpia de usar Google Drive en Linux.
