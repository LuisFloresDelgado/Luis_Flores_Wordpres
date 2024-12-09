PRIMERO HAY QUE CREAR LA INFRAESTRUCTURA EN AWS
TENDREMOS 5 INSTANCIAS QUE SON LAS SIGUIENTES:
![image](https://github.com/user-attachments/assets/2ca4d423-61c4-4b24-94d2-a937d3d3e7ad)
Cada una tendra una ip privada y la del balanceador además trendrá una ip páblica que sera con la que veremos la página Wordpress
También tenemos unos grupos de seguridad asociados a cada instancia
![image](https://github.com/user-attachments/assets/6e073155-4166-4ff2-8621-79e28552038a)

Gracias a esto nos podremos conectar a la máquina del balanceador y en esta nos podremos conectar al resto, y hacer la siguiente configuración para que nos aparezca la paguina de wordpress:

BALANCEADOR DE CARGA

sudo apt install apache2 -y
CONFIGURAR EL SERVIDOR WEB APACHE COMO PROXY INVERSO. TENEMOS QUE ACTIVAR ESTOS MÓDULOS COMO MÍNIMO
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
ESTE MÉTODO DE BALANCEO DE CARGA CONSISTE EN DISTRIBUIR LAS PETICIONES ENTRE LOS SERVIDORES DE FORMA SECUENCIAL, DE FORMA QUE CADA VEZ QUE LLEGUE UNA NUEVA PETICIÓN SE ENVÍA AL SIGUIENTE SERVIDOR DE LA LISTA DE SERVIDORES CONFIGURADOS EN EL SERVIDOR APACHE
sudo a2enmod lbmethod_byrequests
OTROS MÓDULOS NECESARIOS
sudo a2enmod rewrite
sudo a2enmod headers
REINICIAMOS APACHE PARA APLICAR CAMBIOS
sudo systemctl restart apache2
CREAMOS UN NUEVO ARCHIVO DE CONFIGURACIÓN PARA CREAR UN VIRTUALHOST CON LA CONFIGURACIÓN DEL PROXY
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
HABILITAMOS EL VIRTUALHOST QUE ACABAMOS DE CREAR
sudo a2ensite load-balancer.conf
DESHABILITAMOS EL VIRTUALHOST QUE TIENE APACHE CONFIGURADO POR DEFECTO
sudo a2dissite 000-default.conf
REINICIAMOS EL SERVICIO PARA APLICAR LOS CAMBIOS
sudo systemctl restart apache2
REINICIAMOS EL SERVICIO PARA APLICAR LOS CAMBIOS
sudo systemctl reload apache2


EN EL NFS

ACTUALIZAMOS REPOSITORIOS
sudo apt update -y
INSTALAMOS EL SERVIDOR NFS
sudo apt install nfs-kernel-server -y
INSTALAMOS UNZIP PARA DESCOMPRIMIR ARCHIVOS
sudo apt install unzip -y
INSTALAMOS CURL PARA DESCARGAR ARCHIVOS DESDE INTERNET
sudo apt install curl -y
INSTALAMOS PHP Y SU CONECTOR PARA MYSQL
sudo apt install php php-mysql -y
INSTALAMOS EL CLIENTE MYSQL
sudo apt install mysql-client -y
CREAMOS EL DIRECTORIO PARA COMPARTIR ARCHIVOS
sudo mkdir -p /var/nfs/compartir
DAMOS PERMISOS GENÉRICOS AL DIRECTORIO NFS
sudo chown nobody:nogroup /var/nfs/compartir 
CONFIGURAMOS LA CARPETA EN EL ARCHIVO /ETC/EXPORTS PARA COMPARTIRLA CON LA RED
sudo sed -i '$a /var/nfs/compartir    10.0.2.0/24(rw,sync,no_subtree_check)' /etc/exports
DESCARGAMOS WORDPRESS
sudo curl -O https://wordpress.org/latest.zip
DESCOMPRIMIMOS WORDPRESS EN EL DIRECTORIO COMPARTIDO
sudo unzip -o latest.zip -d /var/nfs/compartir/
DAMOS PERMISOS DE EJECUCIÓN AL DIRECTORIO Y ARCHIVOS COMPARTIDOS
sudo chmod 755 -R /var/nfs/compartir/
ASIGNAMOS COMO PROPIETARIO DE LOS ARCHIVOS AL USUARIO Y GRUPO DE APACHE
sudo chown -R www-data:www-data /var/nfs/compartir/*
REASIGNAMOS PERMISOS GENÉRICOS AL DIRECTORIO COMPARTIDO PARA NFS
sudo chown -R nobody:nogroup /var/nfs/compartir/
REINICIAMOS EL SERVIDOR NFS PARA APLICAR LOS CAMBIOS
sudo systemctl restart nfs-kernel-server

EN LOS BACKENDS
ACTUALIZAMOS REPOSITORIOS
sudo apt update -y
INSTALAMOS APACHE PARA USARLO COMO BALANCEADOR DE CARGA
sudo apt install apache2 -y
INSTALAMOS EL CLIENTE NFS PARA MONTAR DIRECTORIOS REMOTOS
sudo apt install nfs-common -y
INSTALAMOS PHP Y LAS DEPENDENCIAS NECESARIAS PARA SERVIR WORDPRESS
sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-xml php-mbstring php-xmlrpc php-zip php-soap php -y
CREAMOS UNA COPIA DE LA CONFIGURACIÓN POR DEFECTO DE APACHE PARA USARLA COMO BASE
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/web.conf
CONFIGURAMOS EL DIRECTORIO RAÍZ PARA QUE APACHE SIRVA LOS ARCHIVOS DE WORDPRESS
sudo sed -i 's|DocumentRoot .*|DocumentRoot /var/nfs/compartir/wordpress|g' /etc/apache2/sites-available/web.conf
AÑADIMOS PERMISOS PARA ACCEDER AL DIRECTORIO DE WORDPRESS
sudo sed -i '/<\/VirtualHost>/i \
<Directory /var/nfs/compartir/wordpress>\
    Options Indexes FollowSymLinks\
    AllowOverride All\
    Require all granted\
</Directory>' /etc/apache2/sites-available/web.conf
MONTAMOS EL DIRECTORIO COMPARTIDO NFS
sudo mount 10.0.2.17:/var/nfs/compartir /var/nfs/compartir
DESHABILITAMOS EL SITIO POR DEFECTO DE APACHE
sudo a2dissite 000-default.conf
HABILITAMOS LA NUEVA CONFIGURACIÓN DEL SITIO WEB
sudo a2ensite web.conf
REINICIAMOS APACHE PARA APLICAR LOS CAMBIOS
sudo systemctl restart apache2
RECARGAMOS APACHE POR SI HAY CAMBIOS ADICIONALES PENDIENTES
sudo systemctl reload apache2
AÑADIMOS LA CONFIGURACIÓN DE MONTAJE NFS AL ARCHIVO FSTAB PARA QUE SEA PERSISTENTE
sudo echo "10.0.2.17:/var/nfs/compartir    /var/nfs/compartir   nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" | sudo tee -a /etc/fstab


EN LA BASE DE DATOS

ACTUALIZAMOS LOS REPOSITORIOS DEL SISTEMA
sudo apt update -y
INSTALAMOS EL SERVIDOR MYSQL
sudo apt install mysql-server -y
INSTALAMOS PHPMYADMIN PARA GESTIONAR MYSQL DESDE UNA INTERFAZ WEB
sudo apt install -y phpmyadmin
EDITAMOS EL BIND-ADDRESS = 0.0.0.0 DEL /ETC/MYSQL/MYSQL.CONF.D/MYSQLD.CNF
REINICIAMOS EL SERVICIO MYSQL PARA APLICAR LOS CAMBIOS
sudo systemctl restart mysql
CONFIGURAMOS LA BASE DE DATOS Y EL USUARIO PARA WORDPRESS
sudo mysql 

PARA FINALIZAR PONEMOS LA SIGUIENTE IP Y COMPLETAMOS LA INSTALACION, AL ACABARLA, YA NAOS APARECERA LA PÁGUINA.
http://34.207.204.225/
![image](https://github.com/user-attachments/assets/fc5d2d65-b256-4777-b907-2f104afa0e58)

