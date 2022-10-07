# Configuración de un servidor Linux Para despliegue de aplicación Laravel.

### Indice

* [Instalar y actualizar dependencias del servidor](#instalar-y-actualizar-dependencias-del-servidor).
* [Instalar Apache](#instalar-apache).
* [Instalar PHP](#instalar-php).
* [Instalar Mysql](#instalar-mysql).
* [Instalar Composer](#instalar-composer).
* [Instalar Nodejs](#instalar-nodejs).
* [Clonar repositorio](#clonar-repositorio).
* [Correr migraciones y semillas](#correr-migraciones-y-semillas).
* [Configurar PHP.ini](#configurar-phpini).
* [Configurar Apache](#configurar-apache).
* [Configurar Certificado SSL](#configurar-certificado-ssl).

### Instalar y actualizar dependencias del servidor
Instalamos los paquetes que trae **Ubuntu** por defecto, los actualizamos y revalidamos la instalación de los paquetes actualizados.
```sh
sudo apt-get update
sudo apt-get upgrade
sudo apt-get update
```
### Instalar Apache
```sh
sudo apt-get install apache2
```

### Instalar PHP
Instalamos **PHP** y sus dependencias basicas
```sh
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update
sudo apt-get install php7.3
sudo apt install php7.3-cli php7.3-fpm php7.3-json php7.3-pdo php7.3-mysql php7.3-zip php7.3-gd  php7.3-mbstring php7.3-curl php7.3-xml php7.3-bcmath php7.3-json
```
Instalamos dependencias especificas que pueden variar dependiendo de las librerias que use el proyecto de Laravel `sudo apt install php8.X-sqlite3 `

> Para verificar que se haya instalado correctamente, corremos el comando `php -v`.

### Instalar Mysql
```sh
sudo apt install mysql-server
```
> Para verificar que se haya instalado correctamente ejecutamos `sudo systemctl status mysql`.

El siguiente paso es establecer la contraseña del usuario **root** y establecer la configuración basica de seguridad, ejecutamos el siguiente comando.
```sh
mysql_secure_installation
```

Creamos la base de datos que usara nuestro proyecto, ingresamos a **Mysql** con el comando `mysql -u root -p`.
```sh
create database DB_NAME;
```

Ahora creamos el usuario que usaremos en nuestro proyecto para conectarnos a la base de datos y le concedemos los permisos necesarios.
```sh
CREATE USER 'app_user'@'localhost' IDENTIFIED WITH mysql_native_password BY '*****';
GRANT ALL PRIVILEGES ON DB_NAME.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

### Instalar Composer
Descargamos el instalador y generamos el ejecutable.
```sh
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '55ce33d7678c5a611085589f1f3ddf8b3c52d662cd01d4ba75c0ee0459970c2200a51f492d557530c71c15d8dba01eae') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
```
Pasamos el ejecutable de composer a la capeta **bin** del usuario para que se ejecute de manera global.
```sh
sudo mv composer.phar /usr/local/bin/composer
```

> Para verificar que se haya instalado correctamente, corremos el comando `composer`

### Instalar Nodejs
Instalamos **Nodejs** el cual incluira **NPM**
```sh
curl -s https://deb.nodesource.com/setup_16.x | sudo bash
sudo apt install nodejs -y
```

> Para verificar que se haya instalado correctamente ejecutamos `node -v` y `npm -v`.

Opcionalmente podemos instalar **Yarn** como altervaitva a **NPM**.
```
npm install --global yarn
```

## Clonar repositorio
Nos ubicamos en la carpeta **WWW**
```
cd /var/www
```

Clonamos el repositorio, ingresamos en la carpeta del mismo y nos cambiamos a la rama que se va a desplegar.
```
sudo git clone "REPO"
cd "DIR"
git checkout "Branch"
```

Instalamos las dependencias de **PHP** via composer.
```
sudo composer install
```

Concedemos los permisos necesarios a la carpeta del proyecto.
```
sudo chown -R www-data:www-data .
sudo usermod -a -G www-data ubuntu
sudo find . -type f -exec chmod 644 {} \;
sudo find . -type d -exec chmod 755 {} \;
sudo chgrp -R www-data storage bootstrap/cache
sudo chmod -R ug+rwx storage bootstrap/cache
```

Debemos configurar nuestras variables de entorno, ejecutamos 
```
cp .env.example .env
```
En el archivo **`.env`** establecemos el valor de las variables de conexión a base de datos y generamos la llave de encriptación de la aplicación con el comando `php artisan key:gen`.
 
### Correr migraciones y semillas
```
php artisan migrate --seed
```
 
### Configurar php.ini
Ahora vamos a sobreescribir unas configuraciones basicas de PHP que nos permitira incrementar el tamaño del cuerpo de las peticiones **HTTP** que podremos recibir asi como el tamaño de los archivos que suban al servidor, para esto ingresamos al archivo de configuración de **PHP** que usara **Apache** para establecer las reglas de las llamadas

```
sudo nano /etc/php/7.3/apache2/php.ini
```

Y sobreescribimos las siguientes variables

```
memory_limit = 2048M
post_max_size = 2048M
upload_max_filesize = 2048M
```
### Configurar apache
Ahora vamos a configurar apache para que nuestra aplicación se pueda ver desde internet.

Activar el modo de reescritura de rutas de apache
```
sudo a2enmod rewrite
```

Configuramos el usuario y el grupo de apache, ingresamos al archivo de configuracion con el comando `sudo nano /etc/apache2/apache2.conf` y sobreescribimos las siguientes variables.

```
User ubuntu
Group ubuntu
```

Copiamos la configuración por defecto de apache a un archivo con el nombre de nuestro dominio
```
sudo cp /etc/apache2/sites-enabled/000-default.conf /etc/apache2/sites-enabled/mydomain.com.conf
```

Ingresamos al archivo de configuración que creamos y sobreescribimos las siguientes variables

```
ServerName mydomain.com
ServerAlias www.mydomain.com

ServerAdmin my_email@mydomain.com
DocumentRoot /var/www/`DIR`/public
```

Ahora desabilitamos el sitio por defecto y habilitamos la configuracion de nuestro dominio.
```
 a2dissite 000-default
 a2ensite mydomain.com
```

## Configurar Certificado SSL

Habilitar protocolo https en el firewall
```
sudo ufw allow 'Apache Full'
sudo ufw delete allow 'Apache'
sudo ufw status
```

### Instalar Certbot
Certbot es un cliente que se utiliza para solicitar un certificado SSL con protocolo ACME e implementarlo en un servidor web, estos certificados son vigentes por 3 meses y es necesario renovarlos.

```
sudo apt install certbot python3-certbot-apache
sudo certbot --apache
```

Certbot utiliza un cron job para renovar los certificados, podemos consultar el estado de este cron ejecutando 
```
sudo systemctl status certbot.timer
```

