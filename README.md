
# Guía de Instalación y Despliegue de WordPress en Ubuntu

Esta guía recopila los pasos necesarios para desplegar un entorno completo para WordPress, incluyendo la verificación de la pila LAMP, la instalación mediante WP-CLI y la configuración avanzada de WordPress en su propio directorio.

---

## 1. Verificación e Instalación del Entorno LAMP + WP-CLI

Esta primera parte permite comprobar si los servicios necesarios para ejecutar WordPress están instalados en una máquina o servidor con Ubuntu y proporciona los comandos para instalarlos en caso de que falten.

### 0. Actualización previa del repositorio
Antes de comprobar o instalar paquetes, es buena práctica actualizar el índice de repositorios del sistema:
```bash
sudo apt update && sudo apt upgrade -y

```

### 1. Apache2 (Servidor Web)

* **Comprobar si está instalado y en ejecución:**
```bash
sudo systemctl status apache2

```


* **Si NO está instalado, ejecutar:**
```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2

```



### 2. PHP y Extensiones necesarias para WordPress

WordPress requiere una versión reciente de PHP y una serie de extensiones críticas para funcionar correctamente con plugins, procesamiento de imágenes y conexiones seguras.

* **Comprobar versión de PHP instalada:**
```bash
php -v

```


* **Si NO está instalado (o para instalar PHP completo con las extensiones requeridas por WordPress):**
```bash
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y

```


* **Reiniciar Apache tras instalar o configurar módulos de PHP:**
```bash
sudo systemctl restart apache2

```



### 3. MySQL / MariaDB (Base de Datos)

* **Comprobar si el servicio está instalado y activo:**
```bash
sudo systemctl status mysql

```


* **Si NO está instalado:**
```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation

```



> **Nota:** `mysql_secure_installation` permite establecer la contraseña de root del motor de BD y eliminar accesos anónimos o bases de datos de prueba por seguridad.

### 4. phpMyAdmin (Gestión visual de Bases de Datos)

* **Comprobar si existe el directorio de phpMyAdmin:**
```bash
ls -la /usr/share/phpmyadmin

```


* **Si NO está instalado:**
```bash
sudo apt install phpmyadmin php-mbstring php-zip php-gd php-json php-curl -y

```



> **Nota:** Durante el asistente de instalación interactivo, presiona la tecla **Espacio** para seleccionar `apache2` y luego confirma la configuración automática de la base de datos (`dbconfig-common`).

### 5. WP-CLI (Interfaz de Línea de Comandos para WordPress)

* **Comprobar si WP-CLI está disponible en el sistema:**
```bash
wp --info --allow-root

```


* **Si NO está instalado (Descarga y configuración binaria global):**
```bash
curl -O [https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar](https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar)
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

```


* **Verificar la correcta instalación:**
```bash
wp --version

```



---

## 2. Instalación de WordPress en su propio directorio

En esta sección se explican los pasos que hay que llevar a cabo para instalar WordPress en su propio directorio (por ejemplo: `/var/www/html/wordpress`).

1. Descargamos la última versión de WordPress con el comando `wget`:
```bash
wget [http://wordpress.org/latest.tar.gz](http://wordpress.org/latest.tar.gz) -P /tmp

```


*El parámetro `-P` indica la ruta donde se guardará el archivo.*

2. Descomprimimos el archivo `.tar.gz` que acabamos de descargar con el comando `tar`:
   
```bash
tar -xzvf /tmp/latest.tar.gz -C /tmp

```


Utilizamos los siguientes parámetros:
* `-x`: Indica que queremos extraer el contenido del archivo.
* `-z`: Indica que queremos descomprimir el archivo.
* `-v`: Habilita el modo verboso para mostrar por pantalla el proceso de descompresión.
* `-f`: Se utiliza para indicar cuál es el nombre del archivo de entrada.
* `-C`: Se utiliza para indicar cuál es el directorio destino.


3. El contenido se ha descomprimido en una carpeta llamada `wordpress`. Ahora, movemos el contenido de `/tmp/wordpress` a `/var/www/html`:
```bash
mv -f /tmp/wordpress /var/www/html

```


4. Creamos la base de datos y el usuario para WordPress:
```bash
mysql -u root <<< "DROP DATABASE IF EXISTS $WORDPRESS_DB_NAME"
mysql -u root <<< "CREATE DATABASE $WORDPRESS_DB_NAME"
mysql -u root <<< "DROP USER IF EXISTS $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"
mysql -u root <<< "CREATE USER $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL IDENTIFIED BY '$WORDPRESS_DB_PASSWORD'"
mysql -u root <<< "GRANT ALL PRIVILEGES ON $WORDPRESS_DB_NAME.* TO $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"

```


> **Tenga en cuenta** que las variables `$WORDPRESS_DB_NAME`, `$WORDPRESS_DB_USER`, `$WORDPRESS_DB_PASSWORD` y `$IP_CLIENTE_MYSQL` estarán definidas en el archivo `.env`.
> El valor de la variable `$IP_CLIENTE_MYSQL` puede ser:
> * `localhost`: Para permitir que el usuario sólo pueda conectarse desde el servidor MySQL.
> * `%`: Para permitir que el usuario pueda conectarse desde cualquier dirección IP.
> * Una dirección IP concreta (ej. `172.31.80.67`).
> * Una dirección IP con un comodín (ej. `172.31.%`).
> 
> 


5. Creamos un archivo de configuración `wp-config.php` a partir del archivo de ejemplo `wp-config-sample.php`:
```bash
cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php

```


6. Configuramos las credenciales de la base de datos en el archivo `wp-config.php`. El contenido original incluye:
```php
define( 'DB_NAME', 'database_name_here' );
define( 'DB_USER', 'username_here' );
define( 'DB_PASSWORD', 'password_here' );
define( 'DB_HOST', 'localhost' );

```


Reemplazaremos los textos por los valores de nuestras variables de entorno utilizando el comando `sed`:
```bash
sed -i "s/database_name_here/$WORDPRESS_DB_NAME/" /var/www/html/wordpress/wp-config.php
sed -i "s/username_here/$WORDPRESS_DB_USER/" /var/www/html/wordpress/wp-config.php
sed -i "s/password_here/$WORDPRESS_DB_PASSWORD/" /var/www/html/wordpress/wp-config.php
sed -i "s/localhost/$WORDPRESS_DB_HOST/" /var/www/html/wordpress/wp-config.php

```


7. Configuración de las variables `WP_SITEURL` y `WP_HOME` al instalar WordPress en su propio directorio:
* **Dirección de WordPress (`WP_SITEURL`):** URL que incluye el directorio donde está instalado el código fuente.
* **Dirección del sitio (`WP_HOME`):** URL principal para acceder a la web.


Para automatizarlo mediante `sed`:
```bash
sed -i "/DB_COLLATE/a define('WP_SITEURL', 'https://$CERTIFICATE_DOMAIN/wordpress');" /var/www/html/wordpress/wp-config.php
sed -i "/WP_SITEURL/a define('WP_HOME', 'https://$CERTIFICATE_DOMAIN');" /var/www/html/wordpress/wp-config.php

```


*(Nota: Si realiza pruebas sin dominio, puede usar `http://IP_PUBLICA_SERVIDOR_WEB/wordpress` e `http://IP_PUBLICA_SERVIDOR_WEB`)*.
8. Copiamos el archivo `index.php` raíz de WordPress al directorio principal `/var/www/html`:
```bash
cp /var/www/html/wordpress/index.php /var/www/html

```


9. Modificamos el archivo `index.php` copiado para que cargue correctamente la ruta del subdirectorio:
Reemplazamos:
```php
require( dirname( __FILE__ ) . '/wp-blog-header.php' );

```


Por:
```php
require( dirname( __FILE__ ) . '/wordpress/wp-blog-header.php' );

```


Comando automatizado con `sed`:
```bash
sed -i "s#wp-blog-header.php#wordpress/wp-blog-header.php#" /var/www/html/index.php

```


10. Preparamos la configuración para los enlaces permanentes creando un archivo `.htaccess` en `/var/www/html`:
```apache
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress

```


*Explicación de directivas:*
* `RewriteEngine On`: Habilita el motor de reescritura de URLs.
* `RewriteBase /`: Define la ruta base.
* `RewriteCond`: Comprueban si la petición no es un archivo ni un directorio existente antes de redirigir a `index.php`.


> **Nota importante:** Para que Apache lea el archivo `.htaccess`, asegúrese de configurar la directiva `AllowOverride All` en el archivo de configuración del VirtualHost.


11. Habilitamos el módulo `mod_rewrite` de Apache y reiniciamos el servicio:
```bash
a2enmod rewrite
sudo systemctl restart apache2

```



---

## 3. Configuración de las security keys de WordPress

Se recomienda que cada instalación genere unas claves de seguridad únicas para mejorar la seguridad de las sesiones y cookies.

1. Eliminamos los valores por defecto del archivo de configuración:
```bash
sed -i "/AUTH_KEY/d" /var/www/html/wordpress/wp-config.php
sed -i "/SECURE_AUTH_KEY/d" /var/www/html/wordpress/wp-config.php
sed -i "/LOGGED_IN_KEY/d" /var/www/html/wordpress/wp-config.php
sed -i "/NONCE_KEY/d" /var/www/html/wordpress/wp-config.php
sed -i "/AUTH_SALT/d" /var/www/html/wordpress/wp-config.php
sed -i "/SECURE_AUTH_SALT/d" /var/www/html/wordpress/wp-config.php
sed -i "/LOGGED_IN_SALT/d" /var/www/html/wordpress/wp-config.php
sed -i "/NONCE_SALT/d" /var/www/html/wordpress/wp-config.php

```


2. Obtenemos las claves de seguridad desde la API oficial de WordPress y las guardamos en una variable:
```bash
SECURITY_KEYS=$(curl [https://api.wordpress.org/secret-key/1.1/salt/](https://api.wordpress.org/secret-key/1.1/salt/))

```


3. Reemplazamos el carácter `/` por `_` para evitar conflictos al usar el comando `sed`:
```bash
SECURITY_KEYS=$(echo$SECURITY_KEYS | tr / _)

```


4. Añadimos las nuevas security keys al archivo de configuración:
```bash
sed -i "/@-/a $SECURITY_KEYS" /var/www/html/wordpress/wp-config.php

```


5. Finalmente, ajustamos los permisos cambiando el propietario y el grupo de todo el directorio web:
```bash
chown -R www-data:www-data /var/www/html/

```



```

---

### ¿Cómo se incluye esto en tu repositorio de Git?

1. **Solo debe haber un archivo `README.md`** en la raíz de tu repositorio de GitHub. 
2. Para incluirlo:
   - Crea un archivo llamado exactamente `README.md` en la carpeta principal de tu proyecto en tu equipo local.
   - Pega todo el contenido del bloque de código anterior dentro de ese archivo.
   - Súbelo a tu repositorio de GitHub con los siguientes comandos en tu terminal:
     ```bash
     git add README.md
     git commit -m "Añadida la guía completa de instalación de LAMP y WordPress"
     git push origin main
     ```
GitHub detectará automáticamente el archivo `README.md` y lo mostrará de manera visual y formateada directamente en la página principal de tu repositorio.

```
