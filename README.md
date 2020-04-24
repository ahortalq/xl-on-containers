# Imágenes XL para ejecución con Docker

Vamos a ver lo necesario para configurar XLD, Istio y K8s.

## Máquinas virtuales con DockerEngine para XLR y XLD
### Creación de máquina virtual.
```
$ docker-machine create --driver virtualbox xld
```
### Configuración del cliente Docker
Para conectar con el nuevo Docker Engine que hemos creado, tenemos que configurar nuestro cliente Docker. Para ello ejecutamos el comando:
```
$ docker-machine env xld
```
Y obtendremos la instrucciones para conectar, en nuestro caso:
```
eval $(docker-machine env xld)
```
### Acceder vía ssh al host
Para acceder vía ssh ejecutamos:
```
$ docker-machine ssh xld
```
### Conocer la IP
```
$ docker-machine ip xld
```
### Comandos de inicio y parada
```
$ docker-machine stop xld
$ docker-machine start xld
$ docker-machine restart xld
$ docker-machine rm xld
```

## Comandos básicos de Docker
### Descarga de imagen
```
$ docker image pull nginx
```
### Acciones sobre un contenedor
```
$ docker container run -d --name nginx-test -p 8080:80 nginx
$ docker container run -d --name nginx-test -p 8080:80 -e var1=val1 -e var2=val2 nginx
$ docker container run -it --name alpine-test alpine /bin/sh
$ docker container attach nginx-test
$ docker container attach --sig-proxy=false nginx-test
$ docker container exec -i -t nginx-test /bin/sh
$ docker container logs -f nginx-test
$ docker container top nginx-test
$ docker container stats
$ docker container stats nginx-test
$ docker container stop nginx-test
$ docker container rm nginx-test
$ docker container prune
$ docker container port nginx-test
```

### Listar imágenes
```
$ docker image ls
```

### Construir imágenes Docker
```
$ docker image build -t <REPOSITORY>:<TAG> .
```

### Crear una red
```
$ docker network create my-network
```
Para utilizarla
```
$ docker container run -d --name redis --network my-network redis:alpine
```

## Dockerfile
```
# Indicamos la imagen base de la que partimos
# Alpine Linux se ha convertido en la estándar
FROM alpine:latest

# Agregamos una descripcion a la imagen
# Podemos utilizarlo para poner la version
LABEL maintainer="Jose Carlos <josecarlos.lopezayala@gmail.com>"
LABEL description="Este Dockerfile de ejemplo instala NGINX"

# El comando RUN es el que nos permite interactuar con nuestra imagen
# para instalar software, ejecuar scripts, etc.
# Con este comando
# - Instalamos nginx utilizando el Alpine Package Manager
# - Eliminamos ficheros temporales
# - Creamos la carpeta /tmp/nginx para que nginx pueda iniciarse
RUN apk add --update nginx && \
        rm -rf /var/cache/apk/* && \
        mkdir -p /tmp/nginx/

# COPY y ADD son parecidos. COPY es más sencillo
# Con este comando configuramos un virtual host
COPY files/default.conf /etc/nginx/conf.d/default.conf

# ADD descomprime el fichero en el directorio indicado
# Con ADD tambien se pueden referenciar ficheros remotos
ADD files/html.tar.gz /usr/share/nginx/

# Con esto decimos a Docker que cuando se ejecute la imagen
# estará disponible el puerto 80. No hace ningún mapeo con el host
# sino que abre el puerto para permitir el acceso al servicio en la red
# del contenedor
EXPOSE 80/tcp

# Comandos que se van a ejecuar cuando arranquemos el contenedor
# Si solo usamos CMD, el ENTRYOINT asociado sera '/bin/sh -c'
# Si usamos ENTRYPOINT, ese sera el ejecutable y lo que pongamos en
# CMD seran los parametros. Por ejemplo:
# 
#    FROM alpine:latest
#    CMD ["/bin/date"]
#
# Si construimos con 'docker build -t test .' y ejecutamos con
# 'docker run test' obtendremos la fecha
# Ahora podemos anular lo especificado en el comando CMD si lo
# pasamos como parametro al iniciar el contendor. Por ejemplo
# si ejecutamos 'docker run test /bin/hostname'
# Si utilizamos ENTRYPOINT será el ejecutable de entrada y CMD seran
# los parametros. Por ejemplo
#
#    FROM alpine:latest
#    ENTRYPOINT ["/bin/echo"]
#    CMD ["Hello"]
#
# Al ejecutarlo escribira 'Hello'. Pero si ejecutamos lo siguiente:
# 'docker run test hola' escribirá 'hola' y se sobreescribiran los
# parametros por defecto.
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# Para incluir variables de entorno tenemos que utilizar lo siguiente
#
#    ENV key value
#    ENV key1=value1 key2=value2
#
# Con la segunda opción podemos asignar varias variables en una misma línea
# Se pueden sobreescribir estas variables de entorno al crear el contenedor
#
#    docker container run -d --name nginx-test -p 8080:80 -e var1=val1 -e var2=val2 nginx
#
```

Otro ejemplo con Apache
```
FROM alpine:latest
LABEL maintainer="Jose Carlos <josecarlos.lopezayala@gmail.com>"
LABEL description="Este Dockerfile instala Apache y PHP"

ENV PHPVERSION 7

RUN apk add --update apache2 php${PHPVERSION}-apache2 php${PHPVERSION} && \
  rm -rf /var/cache/apk/* && \
  mkdir /run/apache2/ && \
  rm -rf /var/www/localhost/htdocs/index.html && \
  echo "<?php phpinfo(); ?>" > /var/www/localhost/htdocs/index.php && \
  chmod 755 /var/www/localhost/htdocs/index.php

EXPOSE 80/tcp

ENTRYPOINT ["httpd"]
CMD ["-D", "FOREGROUND"]
```


## Volúmenes en Docker

Si en el Docker file utilizamos `VOLUME` se creará automáticamente un volumen.

### Crear un volumen
```
$ docker volume create myvolume
```

### Listar volúmenes existentes
```
$ docker volume ls
$ docker volume inspect volume-name
```

### Arrancar un contenedor utilizando un volumen existente
Suponemos que en el Dockerfile tenemos algo como `VOLUME /data`

```
$ docker container run --name nginx-test -v 719d0cc415dba31f0dd2d2a:/data -p 8080:1234 test:9
$ docker container run --name nginx-test -v myvolume:/data -p 8080:1234 test:9
```

# XL Release

## Variables de entorno

* `ADMIN_PASSWORD`. Mejora pasarla para no tener que buscarla luego en los logs.
* `REPOSITORY_KEYSTORE_PASSPHRASE`
* `ACCEPT_EULA`. Esta variable tenemos que pasarla para iniciar XL Release.

## Volúmenes

### Explicación de los volúmenes

* `/opt/xebialabs/xl-release-server/archive`. Para la base de datos embebida.
* `/opt/xebialabs/xl-release-server/conf`. La primera vez que se inicia el contenedor se copiará aquí el contenido del directorio `default-conf`. Los ficheros que ya existan aquí no se sobreescribirán. La imagen también tiene el directorio `node-conf` que contiene un fichero `xl-release.conf` con dos propiedades que se actualizarán cada vez que se arranque el contenedor. Esta información sobreescribe lo configurado en `conf/xl-release.conf` y puede utilizarse para configurar XLR en alta disponibilidad.
  * `xl.cluster.node.id`
  * `xl.cluster.node.hostname`
* `/opt/xebialabs/xl-release-server/ext`. Para personalizaciones.
* `/opt/xebialabs/xl-release-server/hotfix`. Para personalizaciones.
* `/opt/xebialabs/xl-release-server/plugins`. Se copia el contenido del directorio `default-plugins` la primera vez que se arranca el contenedor. No se sobreescriben los plugins ya existentes.
* `/opt/xebialabs/xl-release-server/repository`. Para la base de datos embebida.
* `/opt/xebialabs/xl-release-server/reports`. Debería ser un directorio compartido por todas las instancias XLR. En una configuración de cluster este volumen debería apuntar a un sistema de ficheros compartido por todas las instancias de XLR.

### Creación de volúmenes
```
docker volume create archive
docker volume create conf
docker volume create ext
docker volume create hotfix
docker volume create plugins
docker volume create repository
docker volume create reports
```

### Ejemplo de ejecución con configuración persistente

```
$ docker run --rm -p 5516:5516 -e ACCEPT_EULA=y -e ADMIN_PASSWORD=admin \
  -v conf:/opt/xebialabs/xl-release-server/conf:rw \
  -v repository:/opt/xebialabs/xl-release-server/repository:rw \
  -v archive:/opt/xebialabs/xl-release-server/archive:rw \
  -v reports:/opt/xebialabs/xl-release-server/reports:rw \
  -v ext:/opt/xebialabs/xl-release-server/ext:rw \
  -v hotfix:/opt/xebialabs/xl-release-server/hotfix:rw \
  -v plugins:/opt/xebialabs/xl-release-server/plugins:rw \
  --name xlr xebialabs/xl-release:9.6
```
Una vez hecho esto, podemos detener el contenedor, copiar la licencia definitiva en el directorio `conf` y reiniciar el contenedor. En este caso ya tendría la licencia definitiva y actualizada.

```
sudo cp /opt/xebialabs/xl-release-9.5.2-server/conf/xl-release-license.lic /var/lib/docker/volumes/conf/_data
```

Las próximas veces que arranquemos el contenedor no será necesario pasar la password ni otras variables de entorno. Ya existen los ficheros de configuración creados y no se sobreescribirán.

# XL Deploy