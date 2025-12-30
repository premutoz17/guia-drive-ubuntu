# 📂 Integración definitiva de Google Drive en Ubuntu

## Acceso estable bajo demanda con rclone  
**Caché local: 60 GB · Persistencia offline: 14 días**

Guía paso a paso para montar Google Drive en Linux como una carpeta local,
priorizando **integridad de datos**, **tolerancia a fallos de red** y **uso a largo plazo**.

---

## 0. ¿Qué hace exactamente esta configuración?

Al finalizar, tendrás una carpeta:

    ~/GoogleDrive

Esa carpeta funciona como un **disco híbrido**:

- Los archivos se descargan solo cuando los usas
- Se guardan primero en tu SSD local
- Permanecen disponibles 14 días, incluso sin internet
- Los cambios se suben a Google Drive automáticamente cuando hay conexión

### ❌ Esto NO es

- No descarga todo tu Drive
- No es sincronización tipo Dropbox
- No es edición en tiempo real como Google Docs
- No reemplaza un respaldo (backup)

---

## 1. Requisitos

- Ubuntu 22.04 LTS o 24.04 LTS
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
3. **Storage:** Google Drive  
4. **Client ID:** Enter  
5. **Client Secret:** Enter  
6. **Scope:** opción `1` (Full access)  
7. **Advanced config:** `n`  
8. **Auto config:** `y`  
   (Se abrirá el navegador, inicia sesión y autoriza)
9. Confirma con `y`
10. Sal con `q`

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
