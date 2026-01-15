# Despliegue Web Estatica con Apache (Docker)

---

## 1. FTP - Transferencia de archivos (Teoria)

### 1.1 Diferencia entre FTP y SFTP

- **FTP (File Transfer Protocol):** Envia datos en texto plano, incluyendo usuario y contrasena. Es inseguro.
- **SFTP (SSH File Transfer Protocol):** Usa SSH para cifrar toda la comunicacion. Es seguro.

En esta practica utilizare SFTP a traves del contenedor Docker.

### 1.2 Cliente SFTP

Para conectarme al servidor SFTP utilizare Filezilla, un cliente grafico que permite transferir archivos de forma sencilla.

---

## 2. Infraestructura inmutable con Docker

### 2.1 Requisitos de la infraestructura

He creado la estructura de carpetas:

```bash
mkdir -p web/html web/reloj apache certs
```

He creado el archivo [docker-compose-apache.yml](/docker-compose-apache.yml):

```yaml
services:
  apache:
    image: httpd:latest
    container_name: apache-server
    network_mode: host
    volumes:
      - ./web/html:/usr/local/apache2/htdocs
      - ./web/reloj:/usr/local/apache2/htdocs/reloj
      - ./apache/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
      - ./apache/httpd-ssl.conf:/usr/local/apache2/conf/extra/httpd-ssl.conf:ro
      - ./certs:/usr/local/apache2/conf/certs:ro
    restart: unless-stopped

  sftp:
    image: linuxserver/openssh-server
    container_name: sftp-server
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - USER_NAME=usuario
      - USER_PASSWORD=password
      - PASSWORD_ACCESS=true
    volumes:
      - ./web:/config/web
    restart: unless-stopped
```

He usado `network_mode: host` en ambos contenedores para que usen directamente la red del host. Apache escucha en los puertos 8080 y 8443, y el servidor SFTP en el puerto 2222.

He creado el archivo [apache/httpd.conf](/apache/httpd.conf) con la configuracion basica de Apache:

```apache
ServerRoot "/usr/local/apache2"
Listen 8080

# Modulos necesarios
LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule authz_core_module modules/mod_authz_core.so
LoadModule dir_module modules/mod_dir.so
LoadModule alias_module modules/mod_alias.so
LoadModule ssl_module modules/mod_ssl.so
LoadModule rewrite_module modules/mod_rewrite.so
# ... otros modulos

DocumentRoot "/usr/local/apache2/htdocs"

<Directory "/usr/local/apache2/htdocs">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

# Alias para la web del reloj
Alias /reloj "/usr/local/apache2/htdocs/reloj"

<Directory "/usr/local/apache2/htdocs/reloj">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

DirectoryIndex index.html index.htm

Include conf/extra/httpd-ssl.conf
```

### 2.2 Tarea de despliegue

He clonado la web principal:

```bash
git clone https://github.com/cloudacademy/static-website-example
mv static-website-example/* web/html/
rm -rf static-website-example
```

He clonado la web del reloj:

```bash
git clone https://github.com/ArchiDep/static-clock-website
mv static-clock-website/* web/reloj/
rm -rf static-clock-website
```

He levantado los contenedores:

```bash
docker compose -f docker-compose-apache.yml up -d
```

He verificado que estan activos:

```bash
docker compose -f docker-compose-apache.yml ps
```

![levantando docker](docs/screenshots/7_dockerinit_apache.png)

He accedido a `http://localhost:8080`:

![web deplegada :8080](docs/screenshots/7_webdesplegada_dimension.png)

He accedido a `http://localhost:8080/reloj`:

![web deplegada reloj](docs/screenshots/7_webdesplegada_reloj.png)

En este punto la web funciona correctamente por HTTP. En la siguiente seccion configurare HTTPS con certificados SSL.

### 2.3 Conexion SFTP con Filezilla

He conectado con Filezilla al servidor SFTP del contenedor:

1. He abierto Filezilla
2. He configurado la conexion:
   - **Protocolo:** SFTP
   - **Servidor:** localhost
   - **Puerto:** 2222
   - **Usuario:** usuario
   - **Contrasena:** password

![filezilla connected](docs/screenshots/filezilla_connected.png)

He navegado a la carpeta `/config/web/html` y he subido un archivo de prueba:

![filezilla config web html](docs/screenshots/filezilla_browsing_inserting_website_web_html.png)

He comprobado que el archivo aparece automaticamente en la web (http://localhost:8080):

![filezilla nueva web](docs/screenshots/filezilla_nuevaweb_verificado.png)

---

## 3. HTTPS

### 3.1 Generacion de certificados

He generado los certificados autofirmados en la carpeta `certs/`:

```bash
cd certs

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx-selfsigned.key \
  -out nginx-selfsigned.crt \
  -subj "/C=ES/ST=Andalucia/L=Sevilla/O=IES/OU=DAW/CN=localhost"

cd ..
```

![generacion certificados](docs/screenshots/8_generacion_certificados.png)

### 3.2 Configuracion SSL de Apache

La configuracion SSL esta en [apache/httpd-ssl.conf](/apache/httpd-ssl.conf):

```apache
Listen 8443

SSLCipherSuite HIGH:MEDIUM:!MD5:!RC4:!3DES
SSLProtocol all -SSLv3

# Virtual Host HTTPS
<VirtualHost *:8443>
    ServerName localhost:8443

    DocumentRoot "/usr/local/apache2/htdocs"

    SSLEngine on
    SSLCertificateFile "/usr/local/apache2/conf/certs/nginx-selfsigned.crt"
    SSLCertificateKeyFile "/usr/local/apache2/conf/certs/nginx-selfsigned.key"

    <Directory "/usr/local/apache2/htdocs">
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    Alias /reloj "/usr/local/apache2/htdocs/reloj"
</VirtualHost>

# Virtual Host HTTP - Redireccion a HTTPS
<VirtualHost *:8080>
    ServerName localhost:8080

    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}:8443$1 [R=301,L]
</VirtualHost>
```

He reiniciado Apache para aplicar los cambios:

```bash
docker compose -f docker-compose-apache.yml restart apache
```

### 3.3 Redireccion HTTP a HTTPS

He configurado dos VirtualHosts:
- Puerto 8080: redirige a HTTPS con codigo 301 usando mod_rewrite
- Puerto 8443: sirve el contenido con SSL

He verificado HTTPS en `https://localhost:8443`:

![web https](docs/screenshots/8_web_https.png)

He verificado la redireccion accediendo a `http://localhost:8080` y comprobando en las herramientas de desarrollador (F12 -> Red) que devuelve codigo 301:

![redireccion 301](docs/screenshots/8_redireccion_301.png)
