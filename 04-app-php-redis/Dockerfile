##Descargamos una versión concreta de UBUNTU, a través del tag
FROM ubuntu:22.04
LABEL maintainer="jho.jim.01@gmail.com"
##Actualizamos el sistema
RUN apt-get update
##Instalamos HTTPD Apache 2
RUN apt-get install -y apache2
##Creamos un fichero index.html en el directorio por defecto de nginx
ADD web /var/www/html
##Exponemos el Puerto 80
EXPOSE 80
##Arrancamos Apache a través de ENTRYPOINT para que no pueda ser modificado en la creación del contenedor
CMD ["apachectl", "-D", "FOREGROUND"]
