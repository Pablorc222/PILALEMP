# Despliegue de WordPress con Balanceador de Carga, Backends, NFS y MySQL

Este proyecto describe cómo desplegar un sitio de WordPress utilizando un balanceador de carga, un servidor NFS para compartir archivos, servidores backend y una base de datos MySQL. El documento incluye los pasos detallados para configurar cada componente y asegurar un despliegue.

---

## Tabla de Contenidos
1. [Infraestructura](#infraestructura)
2. [Configuración del Balanceador de Carga](#configuración-del-balanceador-de-carga)
3. [Configuración del Servidor NFS](#configuración-del-servidor-nfs)
4. [Configuración de los Backends](#configuración-de-los-backends)
5. [Configuración del Servidor de Base de Datos](#configuración-del-servidor-de-base-de-datos)

---

## Infraestructura
Aqui están creadas todas las máquinas y asociadas cada una a su correspondiente grupo de seguridad.
![image](https://github.com/user-attachments/assets/c94c658b-0c09-491e-84da-c3b30226cfc6)



---

## Configuración del Balanceador de Carga

### Pasos:
1. **Actualizar los repositorios**:
    ```bash
    sudo apt update -y
    ```

2. **Instalar Apache**:
    ```bash
    sudo apt install apache2 -y
    ```

3. **Habilitar los módulos necesarios de Apache**:
    ```bash
    sudo a2enmod proxy
    sudo a2enmod proxy_http
    sudo a2enmod proxy_balancer
    sudo a2enmod lbmethod_byrequests
    sudo a2enmod rewrite
    sudo a2enmod headers
    ```

4. **Reiniciar Apache**:
    ```bash
    sudo systemctl restart apache2
    ```

5. **Crear el archivo de configuración para el VirtualHost**:
    ```bash
    sudo bash -c "cat > /etc/apache2/sites-available/load-balancer.conf <<EOL
    <VirtualHost *:80>
        <Proxy balancer://mycluster>
            BalancerMember http://10.0.2.161
            BalancerMember http://10.0.2.135
        </Proxy>
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/
    </VirtualHost>
    EOL"
    ```

6. **Habilitar la nueva configuración**:
    ```bash
    sudo a2ensite load-balancer.conf
    sudo a2dissite 000-default.conf
    sudo systemctl reload apache2
    ```

---

## Configuración del Servidor NFS

### Pasos:
1. **Actualizar los repositorios**:
    ```bash
    sudo apt update -y
    ```

2. **Instalar los paquetes necesarios**:
    ```bash
    sudo apt install nfs-kernel-server unzip curl php php-mysql mysql-client -y
    ```

3. **Crear y configurar el directorio compartido**:
    ```bash
    sudo mkdir -p /var/nfs/compartir
    sudo chown nobody:nogroup /var/nfs/compartir
    sudo sed -i '$a /var/nfs/compartir    10.0.2.0/24(rw,sync,no_subtree_check)' /etc/exports
    ```

4. **Descargar y configurar WordPress**:
    ```bash
    sudo curl -O https://wordpress.org/latest.zip
    sudo unzip -o latest.zip -d /var/nfs/compartir/
    sudo chmod 755 -R /var/nfs/compartir/
    sudo chown -R www-data:www-data /var/nfs/compartir/*
    sudo chown -R nobody:nogroup /var/nfs/compartir/
    ```

5. **Reiniciar el servicio NFS**:
    ```bash
    sudo systemctl restart nfs-kernel-server
    ```

---

## Configuración de los Backends

### Pasos:
1. **Actualizar los repositorios**:
    ```bash
    sudo apt update -y
    ```

2. **Instalar los paquetes necesarios**:
    ```bash
    sudo apt install apache2 nfs-common php libapache2-mod-php php-mysql php-curl php-gd php-xml php-mbstring php-xmlrpc php-zip php-soap php -y
    ```

3. **Configurar Apache**:
    ```bash
    sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/web.conf
    sudo sed -i 's|DocumentRoot .*|DocumentRoot /var/nfs/compartir/wordpress|g' /etc/apache2/sites-available/web.conf
    sudo sed -i '/<\/VirtualHost>/i \
    <Directory /var/nfs/compartir/wordpress>\
        Options Indexes FollowSymLinks\
        AllowOverride All\
        Require all granted\
    </Directory>' /etc/apache2/sites-available/web.conf
    ```

4. **Montar el directorio compartido NFS**:
    ```bash
    sudo mount 10.0.2.61:/var/nfs/compartir /var/nfs/compartir
    ```

5. **Habilitar la nueva configuración de Apache**:
    ```bash
    sudo a2dissite 000-default.conf
    sudo a2ensite web.conf
    sudo systemctl reload apache2
    ```

6. **Hacer persistente el montaje NFS**:
    ```bash
    echo "10.0.2.61:/var/nfs/compartir    /var/nfs/compartir   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a /etc/fstab
    ```

---

## Configuración del Servidor de Base de Datos

### Pasos:
1. **Actualizar los repositorios**:
    ```bash
    sudo apt update -y
    ```

2. **Instalar MySQL y phpMyAdmin**:
    ```bash
    sudo apt install mysql-server phpmyadmin -y
    ```

3. **Configurar MySQL para escuchar en una IP específica**:
    ```bash
    sudo sed -i "s/^bind-address\s*=.*/bind-address = 10.0.3.88/" /etc/mysql/mysql.conf.d/mysqld.cnf
    sudo systemctl restart mysql
    ```

4. **Configurar la base de datos y el usuario para WordPress**:
    ```bash
    sudo mysql -u root 
    CREATE DATABASE db_wordpresspablo;
    CREATE USER 'Pablo'@'10.0.2.%' IDENTIFIED BY '1234';
    GRANT ALL PRIVILEGES ON db_wordpress.* TO 'Pablo'@'10.0.2.%';
    FLUSH PRIVILEGES;
    ```
## Proceso de Instalación de WordPress

Finalmente, al poner nuestra IP en el navegador (por ejemplo, `http://52.204.26.215/`), nos encontraremos con la página de instalación de WordPress:

![Pantalla de bienvenida de WordPress](https://github.com/user-attachments/assets/63d9d51c-5361-4e26-a9ae-ea0482bdb79e)

---

### Paso 1: Configuración de la Base de Datos

Una vez en la página de instalación, nos dirigimos a nuestro servidor **BACKEND** y realizamos la siguiente configuración:

![Configuración en el Backend](https://github.com/user-attachments/assets/f6bf470e-386c-4613-ad59-3898024d8957)

---

### Paso 2: Instalación y Datos de WordPress

Después de realizar la configuración, continuamos con la instalación de WordPress. Nos encontraremos con una pantalla similar a la siguiente, donde debemos ingresar los datos necesarios para finalizar la instalación y configurar el sitio:

![Pantalla de instalación de WordPress](https://github.com/user-attachments/assets/0ded3bc9-ebcb-4665-a97a-45639ff2c170)

---

### Paso 3: ¡Listo!

Finalmente, después de completar la instalación, podremos acceder a nuestra página de WordPress funcionando correctamente. ¡El proyecto estará listo!

![WordPress funcionando correctamente](https://github.com/user-attachments/assets/8b52ac1e-35bc-4800-884f-cda34c4cf8ca)

Y así es como se ve el resultado final:

![Página de inicio de WordPress](https://github.com/user-attachments/assets/31a54620-3943-4f30-a373-016327ce8aa9)

---

## Notas
- Hay que asegurarse de que las IPs y rutas coincidan con la configuración de red.
- Para solucionar problemas, revisar los logs en:
  - Apache: `/var/log/apache2/error.log`
  - MySQL: `/var/log/mysql/error.log`



