# Deployar una aplicación Django con Gunicorn, Nginx, Supervisor en Ubuntu 16.04

A continuación explicaré paso a paso como deployar una aplicación Django con Gunicorn, Nginx y administrarlo usando Supervisord en Ubuntu 16.04, ambiente productivo. (DigitalOcean)

## Inicio

En mi caso, he creado un Droplet en www.digitalocean.com con las siguientes características 2 GB Memory / 50 GB Disk / NYC1 - Ubuntu 16.04.4 x64. Durante el proceso de creación seleccione las opciones de "Monitoring" y agregué acceso a través de ssh.

#### Primero, actualicemos nuestros paquetes:

    sudo apt-get update
    sudo apt-get upgrade

#### Segundo, instalemos  `pip`. para python 2 copia lo siguiente:

    sudo apt-get install python-pip

Si estas usando python3, copia lo siguiente:

    sudo apt-get install python3-pip

#### Tercero, instalemos  `virtualenv`: 
Para python2:

    sudo pip install virtualenv

Para python3:

    sudo pip3 install virtualenv
    
En este [link](https://virtualenv.pypa.io/en/stable/) puedes encontrar la documentación oficial de `virtualenv` 

#### Cuarto, ahora debemos crear nuestro ambiente virtual en `/opt`. 
Tu puedes hacerlo donde quieras y usar nombres descriptivos para tu `virtualenv` 

    virtualenv /opt/myvirtualenv/

Activemos nuestro ambiente:

    source /opt/myvirtualenv/bin/activate

Ahora al inicio de la linea de comandos vas a encontrar algo como  `(myvirtualenv)`

#### Quinto, debemos clonar nuestro proyecto  
`git clone https://gitlab.com/myuser/myrepository.git,`

a continuación

`pip install -r requirements.txt`

Nota: en caso de no haber clonado un repositorio es necesario instalar django y crear un nuevo proyecto para ellos debes copiar

`pip install django`
`django-admin startproject myproject`

#### Sexto, debemos aplicar migraciones y correr el servidor en modo development:

`cd myproject` o `cd myrepositorio`
`./manage.py migrate`
`./manage.py runserver`

Vamos al navegador y tipeamos lo siguiente  `miip:8000/admin`  y nos aseguramos que esta corriendo. Luego de esto vamos a reemplazar el servidor de development por `gunicorn`

#### Septimo, Instalar gunicorn 
Estando con el virtualenv activo (o sino debes escribir `source /opt/myvirtualenv/bin/activate` procedemos a Instalar gunicorn:

`pip install gunicorn`

Luego ejecutamos gunicorn y vamos a  `miip:8000` y veamos la magia

`gunicorn project.wsgi`

Observemos algunas configuraciones.

Podemos agregar un puerto especifico

`gunicorn --bind 0.0.0.0:8030 project.wsgi`

Podemos incrementar el numero de workers para servir las peticiones (Normalmente cuando la cantidad de usuarios aumentas, requieres mas workers)

`gunicorn --workers 3 project.wsgi`

Ejecutar en modo demonio:

`gunicorn --daemon project.wsgi`

Incluso, podemos unir todas estas configuraciones: 

`gunicorn -d -b 0.0.0.0:8030 -w 3 project.wsgi`

Puedes encontrar mayor información  [en la documentación](http://docs.gunicorn.org/en/stable/run.html#commonly-used-arguments)

Observemos que no están los estilos, esto se debe a que gunicorn es un servidor de aplicaciones y no provee archivos estatios, para solucionar este problema utilizaremos  `Nginx`  como un reverse proxy para gunicorn. 

Hasta este punto si deseas tener un mayor detalle sobre django y gunicorn puedes ir a la  [documentación](http://docs.gunicorn.org/en/stable/) .

**Django y Gunicorn funcionando!!!**

## Segunda parte
Hasta el momento tenemos Django y Gunicorn listos, ahora continuemos con Nginx. 

#### Primero, instalemos Nginx
En caso de encontrarnos dentro de nuestro `virtualenv` es necesario tipear `deactivate`

`sudo apt-get install nginx`

#### Segundo, configuremos Nginx para manejar el tráfico

Debemos crear un archivo en la siguiente ruta  `/etc/nginx/sites-available/myname`, para ello tipeamos:

`nano /etc/nginx/sites-available/myname`

Luego vamos a colocar dentro del archivo lo siguiente:

    server {
        listen 8000;
        server_name MY_IP_OR_DOMAIN;
    
        location = /favicon.ico { access_log off; log_not_found off; }
    
        location /static/ {
                root /opt/myvirtualenv/myproyect;
        }
    
        location / {
                include proxy_params;
                proxy_pass http://unix:/opt/myvirtualenv/myproyect/project.sock;
        }
    }

Debes ajustar el path de forma que  `/opt/myvirtualenv/myproject`  apunte a tu directorio

Que quiere decir cada instrucción: 

 - Primeras dos lineas indican que se va a escuchar por el puerto 8000 y a través de la iP o dominio,
 - Tercera linea, ignoras problemas con el favicon y Nginx
 - Siguiente bloque indica donde se encuentran los archivos estáticos, los cuales tienen el prefijo `static/`
 - El último bloque permite manejar todos los request diferentes a los estáticos. Una cosa a tener en cuenta aquí es que Nginx y Gunicorn "hablan" entre sí a través de un socket de Unix. Es por eso que uniremos a nuestro gunicorn con un socket, como veremos pronto.

#### Tercero, vamos a habilitar este archivo haciendo un link con el directorio `sites-enabled`

`sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled`

revisemos si el archivo de configuración fue escrito correctamente con el siguiente comando:

`sudo nginx -t`

Si todo esta bien, deberíamos observar lo siguiente:

    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful

#### Cuarto, actualicemos la ubicación de nuestros archivos estáticos

Debemos mover nuestros archivos a la siguiente ubicación  `~/myproject/static/`  ya que configuramos para que Nginx fuera allí a buscarlos. para ellos necesitamos abrir el siguiente archivo  `myproject/settings.py`  y agrega lo siguiente:

`STATIC_ROOT = os.path.join(BASE_DIR, 'static/')`

Graba y cierra, luego hagamos collect hacia esa carpeta:

`./manage.py collectstatic` o `python manage.py collectstatic`

En caso de que solicite confirmación, procede a confirmar y de esta forma estarán disponibles para que Nginx los encuentre.

Ahora, ejecutemos nuestra aplicación:

`gunicorn --daemon --workers 3 --bind unix:/opt/myvirtualenv/myproject/project.sock project.wsgi`

Ahora estamos iniciando gunicorn un poco diferente, ya que estamos bindeando este a un archivo socket el cual es necesario para comunicarnos con Nginx. Este archivo sera ceardo y habilidato para que Nginx y Gunicorn se comuniquen el uno con el otro. Nginx se encarga de los puerto, recodemos que habiamos configurado para que escuchara a través del `MY_IP_OR_DOMAIN:8000`

####	Quinto, reiniciemos Nginx

`sudo service nginx restart`

Ahora podemos ir a la dirección  `MY_IP_OR_DOMAIN:8000`. Excelente, nuestra aplicación esta corriendo. Nuestra aplicación esta corrriendo, podemos confirmarlo al acceder a nuestro panel de administración a través de `MY_IP_OR_DOMAIN:8000/admin`. Finalmente, podemos observar los estilos.

**Nginx Funcionando!!!**

## Tercera parte

Es muy complicado manejar unicorn recordando largas lineas de comandos, es por ellos que haremos uso de  [Supervisord](http://supervisord.org/)  el cual nos permite monitorear y controlar diferentes procesos en nuestro SO UNIX.

De esta forma iniciamos nuestra aplicación: 

`gunicorn --daemon --workers 3 --bind unix:/opt/myvirtualenv/myproject/project.sock project.wsgi`

Es un texto largo y complicado de recordar, por ello con supervisord sera mucho mas conveniente y fácil ejecutar estos comandos

    supervisorctl start myproject
    supervisorctl stop myproject
    supervisorctl restart myproject

#### Primero, instalemos supervisor 
Para Instalar:

`sudo apt-get install supervisor`

Ahora reiniciemos: 

`sudo service supervisor restart`

El archivo principal de configuración de supervisord esta en  `/etc/supervisor/supervisord.conf`. Si revisamos el mismo podemos observar que contiene las siguiente lineas

    [include]
    files = /etc/supervisor/conf.d/*.conf

Esto significa que los archivos de configuración de cada proyecto pueden ser almacenados en It means that config files of specific projects can be stored here  `/etc/supervisor/conf.d/`  y ellos serán incluidos en el archivo principal

Entonces, creamos el archivo `project.conf`  en la carpeta `/etc/supervisor/conf.d/` :

`sudo nano /etc/supervisor/conf.d/project.conf`

#### Segundo, configuremos nuestro proyecto

    [program:myproject]
    command=/opt/myvirtualenv/bin/gunicorn --workers 3 --bind unix:/opt/myvirtualenv/myproject/project.sock project.wsgi
    directory=/opt/myvirtualenv/myproject
    autostart=true
    autorestart=true
    stderr_logfile=/var/log/myproject.err.log
    stdout_logfile=/var/log/myproject.out.log

Analicemos el significado de las instrucciones

 1. `[program:myproject]` Aquí, definimos el nombre del programa que usaremos en este tipo de comandos `sudo supervisorctl start myproject`
 2. `command=/opt/myvirtualenv/bin/gunicorn --workers 3 --bind unix:/opt/myvirtualenv/myproject/project.sock project.wsgi`  permite definir el comando a ser usado cuando iniciamos o reiniciamos nuestro proyecto
 3. `directory=/opt/myvirtualenv/myproject` indica el path desde donde el comando sera ejecutado
 4. `autostart=true` y `autorestart=true` estas lineas definen el comportamiento del script en diferentes condiciones. `Autostart`  le dice al script que debe iniciar cuando el sistema inicie y  `autorestart`  dice al script que debe reiniciar si existe alguna razón.
 5. `stderr_logfile=/var/log/myproject.err.log` y `stdout_logfile=/var/log/myproject.out.log` indican donde los diferentes logs serán almacenados. Log por errores serán almacenamos en `myproject.err.log` y otros en `myproject.out.log`

Ahora grabemos el archivos y ejecutemos el siguiente comando para que los cambios tomen efecto:

    sudo supervisorctl reread
    sudo supervisorctl update

Para verificar que todo esta funcionando, podemos tipear: 
`ps ax | grep gunicorn`

Deberíamos ver diversos procesos gunicorn ejecutandose o podemos ir a `MY_IP_OR_DOMAIN:8000` y veremos la aplicación django funcionando. 

También podemos usar  `supervisor`  para verificar si la aplicación esta corriendo

`sudo supervisorctl status myproject`

#### Tercero, implementemos la interfaz web de supervisord
Ideal en caso de que no queramos utilizar la linea de comandos para controlar los procesos, para ello abrimos  `/etc/supervisor/supervisor.conf`  y colocamos las siguientes lineas al inicio del archivo

    [inet_http_server]
    port=MY_IP_OR_DOMAIN:9001

Esto indica que la interfaz web del supervisor va a correr en  `MY_IP_OR_DOMAIN:9001`.

Grabemos el archivo y recarguemos al supervisor con el siguiente comando

`sudo supervisorctl reload`

Vamos al navegador y coloquemos  `MY_IP_OR_DOMAIN:9001`

**Funciona: Django, Nginx, Gunicorn y los administramos con supervisord**

Fuente: 

 1. [Parte 1](http://rahmonov.me/posts/run-a-django-app-with-gunicorn-in-ubuntu-16-04/)
 2. [Parte 2](http://rahmonov.me/posts/run-a-django-app-with-nginx-and-gunicorn/)
 3. [Parte 3](http://rahmonov.me/posts/run-a-django-app-with-nginx-gunicorn-and-supervisor/)
