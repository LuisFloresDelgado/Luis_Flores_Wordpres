PRIMERO HAY QUE CREAR LA INFRAESTRUCTURA EN AWS
TENDREMOS 5 INSTANCIAS QUE SON LAS SIGUIENTES:
![image](https://github.com/user-attachments/assets/2ca4d423-61c4-4b24-94d2-a937d3d3e7ad)
Cada una tendra una ip privada y la del balanceador además trendrá una ip páblica que sera con la que veremos la página Wordpress
También tenemos unos grupos de seguridad asociados a cada instancia
![image](https://github.com/user-attachments/assets/6e073155-4166-4ff2-8621-79e28552038a)

Gracias a esto nos podremos conectar a la máquina del balanceador y en esta nos podremos conectar al resto, y hacer la siguiente configuración para que nos aparezca la paguina de wordpress:
# En el Balanceador
# 1. Instalar Apache en el balanceador de carga
sudo apt install apache2 -y

# 2. Activar los módulos necesarios para configurar Apache como proxy inverso
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer

# 3. Configurar el método de balanceo de carga para distribuir las peticiones de forma secuencial
sudo a2enmod lbmethod_byrequests

# 4. Activar otros módulos necesarios
sudo a2enmod rewrite
sudo a2enmod headers

# 5. Reiniciar Apache para aplicar los cambios
sudo systemctl restart apache2

# 6. Crear un archivo de configuración para el VirtualHost con la configuración del proxy
sudo bash -c "cat > /etc/apache2/sites-available/load-balancer.conf <<EOL
<VirtualHost *:80>
    <Proxy balancer://mycluster>
        BalancerMember http://10.0.2.191
        BalancerMember http://10.0.2.195
    </Proxy>
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
EOL"

# 7. Habilitar el VirtualHost creado
sudo a2ensite load-balancer.conf

# 8. Deshabilitar el VirtualHost predeterminado de Apache
sudo a2dissite 000-default.conf

# 9. Reiniciar Apache para aplicar los cambios
sudo systemctl restart apache2

# 10. Recargar Apache por si hay cambios adicionales pendientes
sudo systemctl reload apache2


# EN EL NFS

# 1. Actualizar los repositorios del sistema
sudo apt update -y

# 2. Instalar el servidor NFS
sudo apt install nfs-kernel-server -y

# 3. Instalar unzip para descomprimir archivos
sudo apt install unzip -y

# 4. Instalar curl para descargar archivos desde Internet
sudo apt install curl -y

# 5. Instalar PHP y su conector para MySQL
sudo apt install php php-mysql -y

# 6. Instalar el cliente MySQL
sudo apt install mysql-client -y

# 7. Crear el directorio para compartir archivos
sudo mkdir -p /var/nfs/compartir

# 8. Dar permisos genéricos al directorio NFS
sudo chown nobody:nogroup /var/nfs/compartir

# 9. Configurar la carpeta en el archivo /etc/exports para compartirla con la red
sudo sed -i '$a /var/nfs/compartir    10.0.2.0/24(rw,sync,no_subtree_check)' /etc/exports

# 10. Descargar la última versión de WordPress
sudo curl -O https://wordpress.org/latest.zip

# 11. Descomprimir WordPress en el directorio compartido
sudo unzip -o latest.zip -d /var/nfs/compartir/

# 12. Dar permisos de ejecución al directorio y archivos compartidos
sudo chmod 755 -R /var/nfs/compartir/

# 13. Asignar como propietario de los archivos al usuario y grupo de Apache
sudo chown -R www-data:www-data /var/nfs/compartir/*

# 14. Reasignar permisos genéricos al directorio compartido para NFS
sudo chown -R nobody:nogroup /var/nfs/compartir/

# 15. Reiniciar el servidor NFS para aplicar los cambios
sudo systemctl restart nfs-kernel-server


# EN LOS BACKENDS
# 1. Actualizar los repositorios del sistema
sudo apt update -y

# 2. Instalar Apache para usarlo como balanceador de carga en el backend
sudo apt install apache2 -y

# 3. Instalar el cliente NFS para montar directorios remotos
sudo apt install nfs-common -y

# 4. Instalar PHP y las dependencias necesarias para servir WordPress
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-xml php-mbstring php-xmlrpc php-zip php-soap php -y

# 5. Crear una copia de la configuración por defecto de Apache para usarla como base
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/web.conf

# 6. Configurar el directorio raíz para que Apache sirva los archivos de WordPress
sudo sed -i 's|DocumentRoot .*|DocumentRoot /var/nfs/compartir/wordpress|g' /etc/apache2/sites-available/web.conf

# 7. Añadir permisos para acceder al directorio de WordPress
sudo sed -i '/<\/VirtualHost>/i \
<Directory /var/nfs/compartir/wordpress>\
    Options Indexes FollowSymLinks\
    AllowOverride All\
    Require all granted\
</Directory>' /etc/apache2/sites-available/web.conf

# 8. Montar el directorio compartido NFS
sudo mount 10.0.2.17:/var/nfs/compartir /var/nfs/compartir

# 9. Deshabilitar el sitio por defecto de Apache
sudo a2dissite 000-default.conf

# 10. Habilitar la nueva configuración del sitio web
sudo a2ensite web.conf

# 11. Reiniciar Apache para aplicar los cambios
sudo systemctl restart apache2

# 12. Recargar Apache por si hay cambios adicionales pendientes
sudo systemctl reload apache2

# 13. Añadir la configuración de montaje NFS al archivo fstab para que sea persistente
sudo echo "10.0.2.17:/var/nfs/compartir    /var/nfs/compartir   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a /etc/fstab



# EN LA BASE DE DATOS

# 1. Actualizar los repositorios del sistema
sudo apt update -y

# 2. Instalar el servidor MySQL
sudo apt install mysql-server -y

# 3. Instalar phpMyAdmin para gestionar MySQL desde una interfaz web
sudo apt install -y phpmyadmin

# 4. Editar el archivo de configuración de MySQL para permitir conexiones desde cualquier IP
# Cambiar la configuración del bind-address a 0.0.0.0 en /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i 's/bind-address\s*=.*$/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf

# 5. Reiniciar el servicio MySQL para aplicar los cambios
sudo systemctl restart mysql

# 6. Acceder a MySQL para crear la base de datos y el usuario para WordPress
sudo mysql

# 7. Dentro de MySQL, ejecutar los siguientes comandos para configurar la base de datos y el usuario
![image](https://github.com/user-attachments/assets/2de06262-fc74-4e63-b0ab-4207618f0ad1)
![image](https://github.com/user-attachments/assets/672cf818-f5e5-4fbf-a3f9-6e5532dc455b)
Vemos que tenemos acceso


PARA FINALIZAR PONEMOS LA SIGUIENTE IP Y COMPLETAMOS LA INSTALACION, AL ACABARLA, YA NAOS APARECERA LA PÁGUINA.
(http://98.85.172.193/)
![image](https://github.com/user-attachments/assets/1374455b-fd2f-42e8-93a4-65e4e29f3489)


