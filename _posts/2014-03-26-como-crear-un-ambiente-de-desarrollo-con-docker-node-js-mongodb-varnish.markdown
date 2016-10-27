---
layout: post
title: ¿Cómo crear un ambiente de desarrollo con Docker + Node.js + MongoDB + Varnish?
date: '2014-03-26 17:21:22'
tags:
- docker
- espanol
- mongodb
- node-js
- nodejs
---

<p>El fin de semana pasado empecé a aprender <a href="https://www.docker.io/" 
target="_blank">Docker.io</a> y debo decir que es fantástico. Desafortunadamente, al ser un proyecto relativamente nuevo, no hay mucha documentación ni tutoriales en línea. Al principio, batallé un poco pero aquí está toda la explicación para que tu no tengas que batallar. Lo que vamos a hacer es crear un blog sencillo escrito en Node.js, corriendo en una base de datos MongoDB junto con Varnish como balanceador de carga (cubierto en otro post). Vamos a usar Docker y recomiendo ampliamente que uses Vagrant para que no destruyas tu ambiente principal. En este tutorial, no voy a cubrir cómo instalar Docker o las cosas más básicas ya que hay suficiente información al respecto en la web.</p>

<p>Lo primero que necesitamos hacer es crear un nuevo directorio, en este directorio voy a descargar 3 proyectos Git.</p>

<ul>
<li><a
 target="_blank" 
href="https://github.com/luiselizondo/blog-example">Nuestra aplicación de Blog hecha con Node.js y Express.js MVC</a></li>
<li><a
 target="_blank" 
href="https://github.com/luiselizondo/docker-nodejs">La imagen de Docker para 
Node.js</a></li>
<li><a target="_blank" 
href="https://github.com/luiselizondo/docker-mongo">La imagen de Docker para MongoDB</a></li>
</ul>

<p>Después de que descargues todo, puedes revisar los archivos Dockerfile. A continuación, voy a explicar qué están haciendo cada uno y cómo correrlos.</p>

<h3>Imagen Docker para Node.js</h3>

<blockcode>
FROM ubuntu:12.04
MAINTAINER Luis Elizondo "lelizondo@gmail.com"
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
RUN apt-get updateRUN apt-get install -y python-software-properties curl git
RUN add-apt-repository -y ppa:chris-lea/node.js
RUN apt-get -qq update
RUN apt-get install -y nodejs
RUN npm install -g expressjsmvc express nodemon bower
EXPOSE 3000
ADD start.sh /start.sh
RUN chmod +x /start.sh
CMD ["/start.sh"]
</blockcode>

Este proyecto tiene un archivo Dockerfile, un README, un archivo run.sh y un archivo start.sh. El archivo start.sh se usa *dentro* del contenedor, así que no lo vas a estar usando realmente pero es importante que no lo modifiques a menos que sepas lo que estas haciendo.

El archivo run.sh está ahí para que puedas escribir "sh run.sh" en lugar de todo el comando de docker que puede ser muy largo.

El archivo Dockerfile va a instalar Node.js, usar npm para instalar expressjsvmc, express, bower y nodemon; y después va a exponer el puerto 3000 antes de agregar 'start.sh' al contenedor para después correrlo.

Antes de continuar, tengo que explicar algo con lo que batallé un poco. Cuando estas usando archivos Dockerfiles (y para ahora deberías saber qué son) básicamente construyes una imagen para que puedas correr contenedores en base a esta imagen y este contenedor va a correr como está especificado en el archivo Dockerfile. Lo que esto significa es que puedes crear un contenedor que va a correr un comando tan pronto como se crea *o* puedes hacer este comando opcional.

Aquí hay una gran diferencia y realmente depende del servicio que estés configurando. Cuando usas la propiedad ENTRYPOINT en tu archivo Dockerfile, básicamente le estás diciendo a la imagen que corra eses comando tan pronto el contenedor es creado, así que si haces algo como lo siguiente:

Dockerfile:

<code>
ENTRYPOINT ["/start.sh"]
</code>

Significa que el contenedor va a correr el archivo 'start.sh' cuando se cree. Si después queires hacer algo como:

<code>
$ docker run -it luis/nodejs bash
</code>

Para acceder al contenedor, simplemente no va a funcionar, ya que el contenedor va a ignorar cualquier comando que pases (en este caso bash) y va a correr 'start.sh'.

Así que para poder tener ambos, a veces necesitas usar CMD en lugar de ENTRYPOINT, de esta manera, el contenedor va a correr el comando si no le pasas ningún otro comando, pero si le pasas un comando, va a correrlo.

Si reemplazo mi Dockerfile con:

<code>
CMD ["/start.sh"]
</code>

Entonces

<code>
$ docker run -it luis/nodejs bash
</code>

Va a correr bash, y:

<code>
$ docker run -it luis/nodejs
</code>

Va a correr start.sh

<h3>Imagen de Docker para MongoDB</h3>
<code>
FROM ubuntu:12.04
MAINTAINER Luis Elizondo, lelizondo@gmail.com
RUN apt-get update

################## BEGIN INSTALLATION
####################### Install MongoDB Following the Instructions at MongoDB Docs
# Ref: http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/

# Add the package verification key
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10

# Add MongoDB to the repository sources list
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/mongodb.list

# Update the repository sources list once more
RUN apt-get update

# Install MongoDB package (.deb)
RUN apt-get install -y mongodb-10gen

# Create the default data directory
RUN mkdir -p /data/db

##################### INSTALLATION END #####################
# Expose the default port
EXPOSE 27017

# Default port to execute the entrypoint (MongoDB)
CMD ["--port 27017"]

# Set default container command
ENTRYPOINT /usr/bin/mongod
</code>

Esta imagen es más simple, va a exponer dos puertos, y después va a correr el servicio mongod usando ENTRYPOINT, así que realmente no se puede acceder al contenedor a menos que reescribamos el entrypoint.

<h3>Construir las imágenes</h3>
El siguiente paso es construir las imágenes, ve a cada directorio que contenga un Dockerfile y corre:

<code>
$ docker build -t myname/image-name .
</code>

Este comando va a construir la imagen y nombrarla con un nombre, los comandos que yo use son:

<code>
$ docker build -t luis/nodejs .
$ docker build -t luis/mongodb .
</code>

Si quiero listar todas mis imágenes, solo hago:

<code>
$ docker images

REPOSITORY          TAG                 IMAGE ID                        
luis/nodejs         latest              3e9589892ef9                 
luis/mongodb        latest              79868a4506c7                 
</code>

<h3>¿Cómo enlazar mis contenedores?</h3>

Para este momento, ya deberías de poder correr contenedores y deberían funcionar, pero no están enlazados unos a otros, necesitamos que el contenedor de Node pueda acceder al contenedor de MongoDB. Lo primero que vamos a hacer es iniciar el contenedor de MongoDB.

<code>
$ docker run -itd -p 27017 --name mongodb luis/mongodb
</code>

Cuando corremos este comando, Docker va a crear un nuevo contendor con la imagen "luis/mongodb", y nombrar ese contenedor como "mongodb", también va a detectar el puerto 27017 y va a correr este comando como un daemon. Esto último es muy importante ya que deseamos crear el contenedor y que siga corriendo.

 <h3>Espera, ¿y los archivos?</h3>
Tanto MongoDB como la aplicación, van a necesitar leer/escribir datos en el disco duro, y probablemente quieres que esos datos persistan fuera del contenedor, ya que este es desechable. La solución es enlazar volúmenes. Primero hagamos esto con MongoDB.

De manera predeterminada, MongoDB, dentro del contenedor, va a guardar los datos en /data/db, y esto está bien, de hecho nosotros creamos ese directorio en el archivo Dockerfile. MongoDB va a pensar que está guardando la información en /data/db pero en realidad, la va a estar guardando fuera del contenedor en un directorio que nosotros especifiquemos.

Creemos un nuevo directorio para guardar los datos de MongoDB *fuera* del contenedor.

<code>
$ sudo mkdir -p /var/mongodb
</code>

Y ahora, engañemos a MongoDB para que guarde los datos en /var/mongodb

<code>
$ docker run -itd -p 27017 -v /var/mongodb:/data/db --name mongodb luis/mongodb
</code>

Lo que estamos haciendo diferente aquí es enlazar /var/mongodb (fuera del contenedor) a /data/db (dentro del contenedor). 

El mismo principio va a aplicar para los archivos de la aplicación.

<h3>Enlazando contenedores</h3>
Ahora podemos regresar a enlazar nuestros contenedores. Primero, tenemos que clonar la aplicación, en este caso yo estoy usando una aplicación en Node.js que cree antes ,mis archivos estarán en /home/luis/Docker/blog-example.

Ahora, para correr el contenedor de node.js enlazado a MongoDB debo hacer:

<code>
$ docker run -itd -p 8000:3000 --name nodejs --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs
</code>

Expliquemos qué es lo que estamos haciendo con ese comando. Primero, se que mi contenedor está exponiendo el puerto 3000, así que estoy redirigiendo ese puerto al puerto 8000 (eventualmente tendremos que modificar esto pero por ahora está bien dejarlo así). Segundo, le damos un nombre a nuestro contenedor. Tercero, enlazamos el contenedor de MongoDB, que básicamente permite al contenedor de Node.js acceder al contenedor de MongoDB. Finalmente, establecemos la ruta real de /var/www, engañando a nuestro contenedor sobre la ubicación real de nuestros archivos.

<h3>Mi aplicación no está funcionando</h3>
La aplicación necesita instalar varias cosas antes de correr, así que vamos a instalar dependencias antes de correrla.

Matemos a nuestro contenedor primero:

<code>
$ docker rm -f nodejs
</code>

Y ahora, entremos usando bash:

<code>
$ docker run -it -p 8000:3000 --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs bash
</code>

Nota que no estamos daemonizing (demonizando) nuestro contenedor para que podamos acceder a él.

Si listamos todos los archivos en /var/www veremos que nuestra aplicación ahí está:

<code>
$ ls -la /var/www
drwxr-xr-x  7 1000 1000 4096 Mar 25 05:09 .
drwxr-xr-x 20 root root 4096 Mar 25 05:20 ..
-rw-r--r--  1 1000 1000   34 Mar 25 05:09 .bowerrc
drwxr-xr-x  8 1000 1000 4096 Mar 25 05:09 .git
-rw-r--r--  1 1000 1000   44 Mar 25 05:09 .gitignore
-rw-r--r--  1 1000 1000 1377 Mar 25 05:09 app.js
-rw-r--r--  1 1000 1000  260 Mar 25 05:09 bower.json
drwxr-xr-x  4 1000 1000 4096 Mar 25 05:09 components
-rw-r--r--  1 1000 1000  169 Mar 25 05:09 expressjsmvc.json
drwxr-xr-x  2 1000 1000 4096 Mar 25 05:09 lib
-rw-r--r--  1 1000 1000  327 Mar 25 05:09 package.json
drwxr-xr-x  4 1000 1000 4096 Mar 25 05:09 public
-rw-r--r--  1 1000 1000  824 Mar 25 05:09 start.js
drwxr-xr-x  2 1000 1000 4096 Mar 25 05:09 views
</code>

<h3>Variables de ambiente o Environment variables</h3>
Antes de instalar todo, quiero mostrarte algo muy interesante llamado variables de ambiente o environment variables. Estas variables pueden ser accesadas por nuestro sistema, solo tenemos que hacer:

<code>
$ env
HOSTNAME=7921e0543e40
MONGODB_NAME=/focused_engelbart/mongodb
MONGODB_PORT_27017_TCP=tcp://172.17.0.2:27017
TERM=xterm
MONGODB_PORT=tcp://172.17.0.2:27017
MONGODB_PORT_27017_TCP_PORT=27017
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/
MONGODB_PORT_27017_TCP_PROTO=tcp
SHLVL=1
HOME=/
MONGODB_PORT_27017_TCP_ADDR=172.17.0.2
_=/usr/bin/env
</code>

Como puedes ver, tengo múltiples variables que referencian a MongoDB, esto es por el enlace que creamos. Si echamos un vistazo a nuestra aplicación, veremos que estamos utilizando estas mismas variables:

<code>
var address = process.env.MONGODB_PORT_27017_TCP_ADDR;
var port = process.env.MONGODB_PORT_27017_TCP_PORT;
mongoose.connect("mongodb://" + address + ":" + port + "/blog");
</code>

Ahora instalemos nuestras dependencias:

<code>
$ cd /var/www
$ npm install ; expressjsmvc install
</code>

Y salgamos de nuestro contenedor:

<code>
$ exit
</code>

Corramos de nuevo el contenedor:

<code>
$ docker run -itd -p 8000:3000 --name nodejs --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs
</code>

Y ahora veamos qué está pasando dentro de nuestro contenedor:

<code>
$ docker logs nodejs
</code>

Si no vemos ningún error, podemos finalmente abrir el navegador e ir a http://localhost:8000

Deberías ver la aplicación corriendo. Puedes agregar un nuevo blog si vas a http://localhost:8000/blogs/add

<h3>¿Y Varnish?</h3>
Este post quedó muy largo, así que va a tener que esperar para otro día.
