# Deployar una aplicación Django con Gunicorn, Nginx, Supervisor en Ubuntu 16.04

A continuación explicaré paso a paso como deployar una aplicación Django con Gunicorn, Nginx y administrarlo usando Supervisord en Ubuntu 16.04, ambiente productivo. (DigitalOcean)

## Inicio

En mi caso, he creado un Droplet en www.digitalocean.com con las siguientes características 2 GB Memory / 50 GB Disk / NYC1 - Ubuntu 16.04.4 x64. Durante el proceso de creación seleccione las opciones de "Monitoring" y agregué acceso a través de ssh.

Primero, actualicemos nuestros paquetes:

    sudo apt-get update
    sudo apt-get upgrade

Segundo, instalemos  `pip`. para python 2 copia lo siguiente:

    sudo apt-get install python-pip

Si estas usando python3, copia lo siguiente:

    sudo apt-get install python3-pip

Tercero, instalemos  `virtualenv`: 
Para python2:

    sudo pip install virtualenv

Para python3:

    sudo pip3 install virtualenv
    
En este [link](https://virtualenv.pypa.io/en/stable/) puedes encontrar la documentación oficial de `virtualenv` 

Cuarto, ahora debemos crear nuestro ambiente virtual en `/opt`. Tu puedes hacerlo donde quieras y usar nombres descriptivos para tu `virtualenv` 

    virtualenv /opt/myvirtualenv/

Activemos nuestro ambiente:

    source /opt/myvirtualenv/bin/activate

Ahora al inicio de la linea de comandos vas a encontrar algo como  `(myvirtualenv)`

Luego, debemos clonar nuestro proyecto  `git clone https://gitlab.com/myuser/myrepository.git,`

a continuación

`pip install -r requirements.txt`

Nota: en caso de no haber clonado un repositorio es necesario instalar django y crear un nuevo proyecto para ellos debes copiar

`pip install django`
`django-admin startproject myproject`

Luego, debemos aplicar migraciones y correr el servidor en modo development:

`cd myproject` o `cd myrepositorio`
`./manage.py migrate`
`./manage.py runserver`

Vamos al navegador y tipeamos lo siguiente  `miip:8000/admin`  y nos aseguramos que esta corriendo. Luego de esto vamos a reemplazar el servidor de development por `gunicorn`

Estando con el virtualenv activo (o sino debes escribir `source /opt/myvirtualenv/bin/activate` procedemos a Instalar gunicorn:

`pip install gunicorn`

Luego ejecutamos gunicorn y vamos a  `miip:8000` y veamos la magia

`gunicorn project.wsgi`

**Funciona!!!**

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
