# Despliegue Web Estatica con Nginx

---

## 1. Instalacion servidor web Nginx

He instalado Nginx con los siguientes comandos:

```bash
sudo apt update
sudo apt install nginx
```

He verificado que el servicio esta activo:

```bash
systemctl status nginx
```

![instalacion nginx](docs/screenshots/instalacion-nginx.png)

---

## 2. Creacion de las carpetas del sitio web

He creado el directorio para mi sitio web:

```bash
sudo mkdir -p /var/www/webpracticadespliegue41/html
```

He clonado el repositorio con el sitio web de ejemplo:

```bash
cd /var/www/webpracticadespliegue41/html
sudo git clone https://github.com/cloudacademy/static-website-example .
```

He asignado el propietario correcto:

```bash
sudo chown -R www-data:www-data /var/www/webpracticadespliegue41/html
```

He configurado los permisos:

```bash
sudo chmod -R 755 /var/www/webpracticadespliegue41
```

### 3.1 Estructura de Nginx

La estructura de directorios de Nginx es la siguiente:

- `/etc/nginx/`: Aqui configuraciones nginx
- `/var/www/`: Donde insertar las paginas web estaticas
- `/var/log/nginx/`: Logs de nginx

### 3.2 Configuracion basica de Nginx

El archivo principal es `/etc/nginx/nginx.conf` y contiene bloques `server` para cada sitio.

![puntos 2 y 3](docs/screenshots/2_clonacionwebgit.png)

---

## 4. Configuracion de servidor web NGINX

He creado el archivo de configuracion para mi sitio:

```bash
sudo nano /etc/nginx/sites-available/webpracticadespliegue41.light
```

Con el contenido:

```nginx
server {
    listen 80;
    server_name webpracticadespliegue41.light;

    root /var/www/webpracticadespliegue41/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

He creado el enlace simbolico:

```bash
sudo ln -s /etc/nginx/sites-available/webpracticadespliegue41.light /etc/nginx/sites-enabled/
```

He recargado Nginx:

```bash
sudo systemctl reload nginx
```

He editado `/etc/hosts` para acceder con el nombre de dominio:

```
127.0.0.1   webpracticadespliegue41.light
```

![configuracion nginx](docs/screenshots/4_configuracionnginx.png)

He accedido a la web:

![captura web nginx](docs/screenshots/4_capturaweb.png)

---

## 5. Archivos de log

He comprobado los logs de Nginx:

**Access log:**
```bash
sudo tail /var/log/nginx/access.log
```

**Error log:**
```bash
sudo tail /var/log/nginx/error.log
```

![logs nginx](docs/screenshots/5_archivos_logs.png)

---

## 6. FTP - Transferencia de archivos (Teoria)

### 6.1 Diferencia entre FTP y SFTP

- **FTP (File Transfer Protocol):** Envia datos en texto plano, incluyendo usuario y contrasena. Es inseguro.
- **SFTP (SSH File Transfer Protocol):** Usa SSH para cifrar toda la comunicacion. Es seguro.

En esta practica utilizare SFTP a traves del contenedor Docker que configurare en la siguiente seccion.

### 6.2 Cliente SFTP

Para conectarme al servidor SFTP utilizare Filezilla, un cliente grafico que permite transferir archivos de forma sencilla. La configuracion practica la realizare en la seccion 7 con Docker.

---

## 7. Infraestructura inmutable con Docker

### 7.1 Requisitos de la infraestructura

He creado la estructura de carpetas:

```bash
mkdir -p web/html web/reloj nginx certs
```

He creado el archivo [docker-compose.yml](/docker-compose.yml):

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-server
    network_mode: host
    volumes:
      - ./web/html:/usr/share/nginx/html
      - ./web/reloj:/usr/share/nginx/html/reloj
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
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

He usado `network_mode: host` en ambos contenedores para que usen directamente la red del host. Nginx escucha en los puertos 8080 y 8443, y el servidor SFTP en el puerto 2222.

He creado el archivo [nginx/default.conf](/nginx/default.conf):

```nginx
server {
    listen 8080;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    location /reloj {
        alias /usr/share/nginx/html/reloj;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
}
```

### 7.2 Tarea de despliegue

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
docker compose up -d
```

He verificado que estan activos:

```bash
docker compose ps
```

![levantando docker](docs/screenshots/7_dockerinit.png)

He accedido a `http://localhost:8080`:

![web deplegada :8080](docs/screenshots/7_webdesplegada_dimension.png)

He accedido a `http://localhost:8080/reloj`:

![web deplegada reloj](docs/screenshots/7_webdesplegada_reloj.png)

En este punto la web funciona correctamente por HTTP. En la siguiente sección configuraré HTTPS con certificados SSL.

### 7.3 Conexion SFTP con Filezilla

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

## 8. HTTPS

### 8.1 Generacion de certificados y Volumenes

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

### 8.2 Inyeccion de la configuracion de Nginx

He copiado la configuracion HTTPS a `nginx/default.conf`:

```bash
cp nginx/default-https.conf nginx/default.conf
```

El contenido de la configuracion SSL es:

```nginx
server {
    listen 8080;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}

server {
    listen 8443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/certs/nginx-selfsigned.key;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    location /reloj {
        alias /usr/share/nginx/html/reloj;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
}
```

He reiniciado nginx para aplicar los cambios:

```bash
docker compose restart nginx
```

### 8.3 Redireccion HTTP a HTTPS

He configurado dos bloques server:
- Puerto 8080: redirige a HTTPS con codigo 301
- Puerto 8443: sirve el contenido con SSL

He verificado HTTPS en `https://localhost:8443`:

![web https](docs/screenshots/8_web_https.png)

He verificado la redireccion accediendo a `http://localhost:8080` y comprobando en las herramientas de desarrollador (F12 -> Red) que devuelve codigo 301:

![redireccion 301](docs/screenshots/8_redireccion_301.png)

---

## 9. Checklist y evidencias

| # | Requisito | Evidencia |
|---|-----------|-----------|
| 1 | Nginx activo | ![nginx activo](docs/screenshots/7_dockerinit.png) |
| 2 | Configuracion cargada | ![config cargada](docs/screenshots/4_configuracionnginx.png) |
| 3 | Resolucion de nombres | ![resolucion nombres](docs/screenshots/4_capturaweb.png) |
| 4 | Contenido web visible | ![contenido web](docs/screenshots/7_webdesplegada_dimension.png) |
| 5 | Conexion SFTP | ![conexion sftp](docs/screenshots/filezilla_connected.png) |
| 6 | Permisos de escritura | ![permisos escritura](docs/screenshots/filezilla_browsing_inserting_website_web_html.png) |
| 7 | Contenedores activos | ![contenedores activos](docs/screenshots/7_dockerinit.png) |
| 8 | Volumen compartido | ![volumen compartido](docs/screenshots/filezilla_nuevaweb_verificado.png) |
| 9 | Multi-sitio (reloj) | ![multi-sitio](docs/screenshots/7_webdesplegada_reloj.png) |
| 10 | HTTPS activo | ![https activo](docs/screenshots/8_web_https.png) |
| 11 | Redireccion HTTP->HTTPS | ![redireccion](docs/screenshots/8_redireccion_301.png) |
