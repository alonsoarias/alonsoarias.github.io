### Guía para Desplegar una Solución de Correo Electrónico en Azure y Publicar en GitHub Pages

#### Índice
1. [Creación de Recursos en Microsoft Azure](#1-creación-de-recursos-en-microsoft-azure)
2. [Configuración de DNS](#2-configuración-de-dns)
3. [Instalación y Configuración de Servidores de Correo](#3-instalación-y-configuración-de-servidores-de-correo)
4. [Instalación y Configuración de Certificados SSL con Certbot](#4-instalación-y-configuración-de-certificados-ssl-con-certbot)
5. [Sincronización de Servidores](#5-sincronización-de-servidores)
6. [Configuración de Roundcube](#6-configuración-de-roundcube)
7. [Seguridad](#7-seguridad)
8. [Pruebas y Creación de Datos de Prueba](#8-pruebas-y-creación-de-datos-de-prueba)
9. [Alta Disponibilidad](#9-alta-disponibilidad)
10. [Publicación en GitHub Pages](#10-publicación-en-github-pages)

### 1. Creación de Recursos en Microsoft Azure

#### Máquinas Virtuales:

1. **Inicia sesión en el portal de Azure.**
2. **Crea las máquinas virtuales:**
    - **VM1 (mail-server-1):**
      - Navega a **Máquinas Virtuales** y selecciona **Agregar**.
      - Configura:
        - **Nombre**: `mail-server-1`
        - **Región**: (Selecciona la región adecuada, ej. East US)
        - **Tamaño**: Estándar B2s (2 vCPUs, 4GB RAM)
        - **Imagen**: Ubuntu Server 20.04 LTS
        - **Autenticación**: Clave SSH (Genera una nueva clave o usa una existente)
        - **Red virtual**: Crea una nueva red virtual (`mail-vnet`) y subred (`mail-subnet`)
        - **Disco de sistema**: SSD Premium, 64GB
      - **Red**:
        - **Grupo de Seguridad de Red (NSG)**: Crea uno nuevo y permite los siguientes puertos: 
          - **SMTP**: 25
          - **IMAP**: 143, 993
          - **POP3**: 110, 995
          - **HTTPS**: 443
        - **Dirección IP pública**: Asigna una nueva.
    - **VM2 (mail-server-2):**
      - Repite el proceso anterior con las mismas configuraciones, cambiando el nombre a `mail-server-2` y usando la misma red virtual (`mail-vnet`) y subred (`mail-subnet`).

#### Redes y Seguridad:

1. **Red Virtual (VNet):**
    - Ya creada durante la configuración de las máquinas virtuales.
    - VNet: `mail-vnet`
    - Subred: `mail-subnet`
2. **Grupo de Seguridad de Red (NSG):**
    - **Configuraciones de NSG (inicialmente creadas con las VMs):**
      - Reglas de entrada:
        - **SMTP**: 25
        - **IMAP**: 143, 993
        - **POP3**: 110, 995
        - **HTTPS**: 443

#### Balanceador de Carga:

1. **Crear el Balanceador de Carga:**
    - Navega a **Balanceadores de Carga** y selecciona **Agregar**.
    - Configura:
      - **Nombre**: `mail-loadbalancer`
      - **SKU**: Estándar
      - **Tipo**: Público
      - **Región**: Misma región que las VMs
      - **IP pública**: Crea una nueva IP pública.
2. **Configurar Backend Pool:**
    - En el balanceador de carga, selecciona **Backend Pools** y **Agregar**.
    - Nombre: `mail-backend-pool`
    - Añadir `mail-server-1` y `mail-server-2` al pool.
3. **Configurar Reglas de Balanceo de Carga:**
    - Selecciona **Reglas de carga** y **Agregar**.
    - Configura para los puertos necesarios:
      - **HTTPS**: 443
      - **IMAP**: 143, 993
      - **POP3**: 110, 995

### 2. Configuración de DNS

1. **Configurar Registros DNS en tu proveedor de dominio:**
    - **Registro A**:
      - **Nombre**: `mail`
      - **Tipo**: A
      - **Valor**: Dirección IP pública del balanceador de carga.
    - **Registro MX**:
      - **Nombre**: vacío
      - **Tipo**: MX
      - **Valor**: `mail.orioncloud.com.co` con prioridad 10.
    - **Registro CNAME**:
      - **Nombre**: `webmail`
      - **Tipo**: CNAME
      - **Valor**: `mail.orioncloud.com.co`.

### 3. Instalación y Configuración de Servidores de Correo

#### Instalación de Dovecot y Exim:

1. **Conectar a las Máquinas Virtuales:**
    - Utiliza SSH para conectar a `mail-server-1` y `mail-server-2`.

2. **Instalar Dovecot y Exim en ambas máquinas:**

    ```bash
    sudo apt update
    sudo apt install dovecot-core dovecot-imapd dovecot-pop3d exim4 -y
    sudo apt install spamassassin clamav clamav-daemon sieve -y
    ```

3. **Configurar Dovecot:**

    - Edita `/etc/dovecot/dovecot.conf`:

    ```bash
    sudo nano /etc/dovecot/dovecot.conf
    ```

    - Añade o ajusta las siguientes líneas:

    ```bash
    protocols = imap pop3
    mail_location = maildir:~/Maildir
    ```

4. **Configurar Exim:**

    - Edita `/etc/exim4/update-exim4.conf.conf`:

    ```bash
    sudo nano /etc/exim4/update-exim4.conf.conf
    ```

    - Configura las siguientes opciones:

    ```bash
    dc_eximconfig_configtype='internet'
    dc_other_hostnames='mail.orioncloud.com.co'
    dc_local_interfaces='0.0.0.0'
    dc_readhost='mail.orioncloud.com.co'
    dc_relay_domains=''
    dc_minimaldns='false'
    dc_relay_nets=''
    dc_smarthost=''
    CFILEMODE='644'
    dc_use_split_config='false'
    dc_hide_mailname='true'
    dc_mailname_in_oh='true'
    dc_localdelivery='maildir_home'
    ```

5. **Reinicia los servicios:**

    ```bash
    sudo systemctl restart dovecot
    sudo systemctl restart exim4
    ```

#### Configuración de SpamAssassin y ClamAV:

1. **Configurar SpamAssassin:**

    - Edita `/etc/default/spamassassin` y habilita el servicio:

    ```bash
    sudo nano /etc/default/spamassassin
    ```

    - Ajusta lo siguiente:

    ```bash
    ENABLED=1
    ```

    - Habilita y arranca el servicio:

    ```bash
    sudo systemctl enable spamassassin
    sudo systemctl start spamassassin
    ```

2. **Configurar ClamAV:**

    - Asegúrate de que ClamAV está actualizado y habilitado:

    ```bash
    sudo systemctl enable clamav-daemon
    sudo systemctl start clamav-daemon
    sudo freshclam
    ```

3. **Configurar Sieve:**

    - Edita `/etc/dovecot/conf.d/90-sieve.conf`:

    ```bash
    sudo nano /etc/dovecot/conf.d/90-sieve.conf
    ```

    - Añade:

    ```bash
    plugin {
      sieve = ~/.dovecot.sieve
      sieve_global_path = /var/lib/dovecot/sieve/default.sieve
      sieve_dir = ~/sieve
      sieve_global_dir = /var/lib/dovecot/sieve/
    }
    ```

### 4. Instalación y Configuración de Certificados SSL con Certbot

1. **Instalar Certbot en ambos servidores:**

    ```bash
    sudo apt install certbot python3-certbot-apache -y
    ```

2. **Obtener certificados SSL:**

    ```bash
    sudo certbot --apache -d mail.orioncloud.com.co
    ```

    - Sigue las instrucciones en pantalla para completar el proceso. Certbot configurará automáticamente Apache para usar los certificados obtenidos.

3. **Renovar certificados automáticamente:**

    ```bash
    sudo crontab -e
    ```

    - Añade la siguiente línea para renovar los certificados automáticamente:

    ```bash
    0 3 * * * /usr/bin/certbot renew --quiet
    ```

### 5. Sincronización de Servidores

1. **Configura la replicación de correo:**

    - Edita `/etc/dovecot/conf.d/10-mail.conf`

:

    ```bash
    sudo nano /etc/dovecot/conf.d/10-mail.conf
    ```

    - Añade o ajusta:

    ```bash
    mail_home = /var/mail/vhosts/%d/%n
    mail_location = maildir:~/Maildir
    ```

2. **Configura la sincronización con `dovecot` director:**

    - En ambos servidores, edita `/etc/dovecot/conf.d/90-plugin.conf`:

    ```bash
    sudo nano /etc/dovecot/conf.d/90-plugin.conf
    ```

    - Añade:

    ```bash
    plugin {
      mail_replica = tcp:other.mail.server.ip:port
    }
    ```

### 6. Configuración de Roundcube

1. **Instala Apache y PHP:**

    ```bash
    sudo apt install apache2 php php-mysql libapache2-mod-php -y
    ```

2. **Descarga y configura Roundcube:**

    ```bash
    cd /var/www/html
    sudo wget https://github.com/roundcube/roundcubemail/releases/download/1.5.0/roundcubemail-1.5.0.tar.gz
    sudo tar xvf roundcubemail-1.5.0.tar.gz
    sudo mv roundcubemail-1.5.0 roundcube
    cd roundcube
    sudo composer install --no-dev
    sudo cp config/config.inc.php.sample config/config.inc.php
    sudo nano config/config.inc.php
    ```

    - Configura la base de datos y otras opciones necesarias en `config.inc.php`.

3. **Configura Apache:**

    - Edita `/etc/apache2/sites-available/roundcube.conf`:

    ```bash
    sudo nano /etc/apache2/sites-available/roundcube.conf
    ```

    - Añade:

    ```apache
    <VirtualHost *:443>
        ServerName mail.orioncloud.com.co
        DocumentRoot /var/www/html/roundcube
        SSLEngine on
        SSLCertificateFile /etc/letsencrypt/live/mail.orioncloud.com.co/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/mail.orioncloud.com.co/privkey.pem

        <Directory /var/www/html/roundcube>
            Options -Indexes
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>
    ```

4. **Habilita el sitio y reinicia Apache:**

    ```bash
    sudo a2ensite roundcube
    sudo a2enmod ssl
    sudo systemctl restart apache2
    ```

### 7. Seguridad

1. **Implementar TLS/SSL:**

    - Ya configurado en Dovecot y Exim.

2. **Autenticación Segura:**

    - Edita `/etc/dovecot/conf.d/10-auth.conf`:

    ```bash
    sudo nano /etc/dovecot/conf.d/10-auth.conf
    ```

    - Asegúrate de que está configurado para `login` y `PLAIN`.

### 8. Pruebas y Creación de Datos de Prueba

#### Crear Usuarios de Email de Prueba:

1. **Crea un usuario de correo en ambos servidores:**

    ```bash
    sudo adduser usuario1
    sudo adduser usuario2
    ```

2. **Configura las carpetas de correo:**

    ```bash
    sudo -u usuario1 maildirmake.dovecot /home/usuario1/Maildir
    sudo -u usuario2 maildirmake.dovecot /home/usuario2/Maildir
    ```

#### Pruebas de Disponibilidad y Funcionamiento:

1. **Prueba envío y recepción de correos utilizando `telnet`:**

    - **Envía un correo de prueba:**

    ```bash
    telnet mail.orioncloud.com.co 25
    ```

    - Introduce los siguientes comandos en la sesión de telnet:

    ```bash
    EHLO mail.orioncloud.com.co
    MAIL FROM:<usuario1@mail.orioncloud.com.co>
    RCPT TO:<usuario2@mail.orioncloud.com.co>
    DATA
    Subject: Prueba de correo
    Este es un correo de prueba.
    .
    QUIT
    ```

2. **Verifica la recepción del correo:**

    - Conéctate al servidor IMAP utilizando telnet para verificar que el correo se ha recibido:

    ```bash
    telnet mail.orioncloud.com.co 143
    ```

    - Introduce los siguientes comandos:

    ```bash
    a login usuario2 <password>
    a select inbox
    a fetch 1 body[header]
    a logout
    ```

3. **Acceso Webmail a través de Roundcube:**
    - Abre `https://mail.orioncloud.com.co` en un navegador web.
    - Inicia sesión con los usuarios de prueba (`usuario1` y `usuario2`).

4. **Verificación de Configuraciones de Seguridad:**
    - Utiliza herramientas como `SSL Labs` para verificar la configuración TLS.
    - Verifica los logs de Dovecot y Exim para asegurarte de que no hay problemas de seguridad o configuración.

### 9. Alta Disponibilidad

1. **Configurar el balanceador de carga o sistema de failover:**

    - Asegúrate de que el balanceador de carga de Azure está correctamente configurado y funcionando.
    - Configura los chequeos de salud para los servicios en cada VM.
    - Prueba desconectar una de las VMs y asegúrate de que el servicio sigue funcionando sin interrupciones.

### 10. Publicación en GitHub Pages

1. **Crear un nuevo repositorio en GitHub:**

    - Navega a [GitHub](https://github.com) y crea un nuevo repositorio. Dale un nombre, por ejemplo, `mail-server-guide`.

2. **Clona el repositorio a tu máquina local:**

    ```bash
    git clone https://github.com/tu-usuario/mail-server-guide.git
    cd mail-server-guide
    ```

3. **Crea el archivo de la guía:**

    - Crea un archivo `index.md` y añade el contenido de la guía paso a paso:

    ```bash
    nano index.md
    ```

    - Copia y pega el contenido de esta guía en el archivo `index.md`.

4. **Configura GitHub Pages:**

    - Navega a la configuración del repositorio en GitHub.
    - En la sección **GitHub Pages**, selecciona la rama `main` (o `master`) y la carpeta `/root`.

5. **Sube los cambios y despliega:**

    ```bash
    git add .
    git commit -m "Guía de despliegue de servidor de correo en Azure"
    git push origin main
    ```

6. **Accede a tu GitHub Page:**

    - GitHub Pages generará una URL para tu página. Será algo como `https://tu-usuario.github.io/mail-server-guide`.

### Conclusión

Este paso a paso proporciona una guía completa para desplegar una solución de correo electrónico en Microsoft Azure utilizando dos máquinas virtuales con Ubuntu, asegurando alta disponibilidad y seguridad para el dominio `mail.orioncloud.com.co`. Incluye la creación de datos de prueba (usuarios de correo) y métodos de prueba para asegurar el correcto funcionamiento del servicio. La guía está diseñada para ser publicada en GitHub Pages, permitiendo fácil acceso y compartición.
