# ¿Como configurar varios proyectos django en una plataforma apache bajo un servidor inverso?



## Introducción

En ocasiones tenemos un sólo dominio, un sólo servidor y deseamos montar sobre él varios proyectos django. Una de las cosas que no me agradan de django en el momento de liberar a producción en un servidor apache (deploy),  es que deben recolectarse todos los archivos static bajo un directorio (python manage.py collectstatic) y este directorio debe ser colocado en el directorio de static que posee el servidor apache. Si hay varios  proyectos estos deberán compartir un único directorio static. Además los logs `access.log` y `error.log` son compartidos por todos los proyectos y aplicaciones (sean django o no).

Una alternativa es levantar un servidor virtual por cada proyecto, de esta manera se aísla el comportamiento de cada proyecto en su propio espacio de trabajo, reduciendo, desde mi punto de vista, la posibilidad de que un proyecto interfiera inadvertidamente con otro proyecto django o con cualquier otra aplicación (vía el directorio static) además de configurar los logs para que concentren únicamente la información asociada al proyecto (sin interferencia de información de otras aplicaciones). 

**Nota: *Toda la discusión a continuación se basa en el uso de apache2 como servidor***

## El problema

Suponga este escenario: nuestro servidor tiene el dominio http://mi.servidor.com y deseamos tener dos proyectos foo1 y foo2 en el mismo servidor. Montamos cada uno de estos proyectos en dos puertos diferentes del mismo servidor, por ejemplo en  http://mi.servidor.com:7000 estaría foo1 y en http://mi.servidor.com:9000 estaría foo2, de esta manera resolvemos el problema de aislar cada aplicación también los static. Hasta aquí todo bien, sin embargo esta solución implica "abrir" el puerto 7000 y 9000 a que tengan tráfico desde internet, lo que en muchas ocasiones no es posible o no es deseable. Lo único que podemos tener es nuestro siempre confiable puerto 80 abierto para el servidor web. 

## ¿La solución? 

Montar un servidor proxy inverso. 

En este texto presento las lecciones aprendidas en el proceso poner a funcionar foo1 y foo2 en producción, ¿el resultado? puedo ejecutar foo1 y foo2 en servidores independientes y que pueden consultarse desde las urls http://miservidor.com/proyecto-foo1 y http://mi.servidor.com/proyecto-foo2, donde proyecto-foo1 y proyecto-foo2 es un nombres que nosotros asignamos libre independientemente de los nombres del proyectos django.

## Configuración de los servidores virtuales

Para tener dos servidores virtuales, uno escuchando el puerto 7000 y otro el puerto 9000, es necesario permitir a apache escuchar los puertos 7000 y 9000 (donde montaremos foo1 y foo2). Por lo que edité el archivo `/etc/apache2/ports.conf` :

```
Listen 80
Listen 7000
Listen 9000
```

Ahora es posible poner en funcionamiento dos servidores, para ello creamos los servidores virtuales, la configuración puede estar en archivos separados, pero en este caso preferí colocarlos en el mismo archivo de configuración del servidor por default (el que escucha el puerto 80), en mi caso está en `/etc/apache2/sites-enabled/default.conf` agregando  el servidor virtual para el proyecto-foo1

    <VirtualHost *:7000>
        ServerAdmin webmaster@localhost
            
        # foo project path
        DocumentRoot /var/www/produccion/foo1
    
        #logs path
        ErrorLog /var/www/produccion/logs/foo1/error.log
        CustomLog /var/www/produccion/logs/foo1/access.log combined
    
        # static path (this directory matches with STATIC_ROOT)
        Alias /static /var/www/produccion/foo1/staticfiles
    
        WSGIDaemonProcess foo1 python-home=/var/www/produccion/foo1/env python-path=/var/www/produccion/foo1
        WSGIScriptAlias /  /var/www/produccion/foo1/foo1/wsgi.py process-group=foo1 application-group=%{GLOBAL}
    </VirtualHost>

Como se puede apreciar, hemos configurado al servidor virtual que escucha el puerto 7000 para que sirva la aplicación montada en el directorio `/var/www/produccion/foo1/`, además los `staticfiles`, recolectados por django mediante el comando `python manage.py collectstatic` se encuentran en su propio espacio: `/var/www/produccion/foo1/staticfiles`

Una configuración similar puede hacerse para el servidor virtual de foo2:

```
<VirtualHost *:9000>
    ServerAdmin webmaster@localhost
        
    # foo project path
    DocumentRoot /var/www/produccion/foo2
    
    #logs path
    ErrorLog /var/www/produccion/logs/foo2/error.log
    CustomLog /var/www/produccion/logs/foo2/access.log combined
    
    # static path (this directory matches with STATIC_ROOT)
    Alias /static /var/www/produccion/foo2/staticfiles
    
    WSGIDaemonProcess foo2 python-home=/var/www/produccion/foo2/env python-path=/var/www/produccion/foo2
    WSGIScriptAlias /  /var/www/produccion/foo2/foo2/wsgi.py process-group=foo2 application-group=%{GLOBAL}

</VirtualHost>
```

Si podemos "abrir" los puertos 7000 y 9000 para que puedan tener tráfico (o si estamos consultando todo desde la misma máquina) entonces hasta aquí bastaría, pues a través de las urls `http:\\mi.servidor.com:7000\` y  `http:\\mi.servidor.com:9000\` deberemos poder consultar nuestros proyectos sin mayor problema. Sin embargo si no podemos abrir los puertos, por restricciones de seguridad o por políticas organizacionales, entonces lo que sigue es configurar el proxy inverso.

## Configuración del proxy inverso

Lo que sigue ahora es decirle al servidor del puerto 80 que cuando llegue una url con cierta estructura en particular redirija la petición a alguno de los servidores virtuales. En este ejemplo si al servidor `http://mi.servidor.com` llega la solicitud  `/proyecto-1` se reenviara la petición al servidor virtual que escucha el puerto 7000 y si llega  la solicitud `/proyecto2` se reenviara la petición al servidor virtual que escucha el puerto 9000.

Debemos tener instalado apache2, y habilitarlo para que pueda funcionar como un proxy inverso, hay mucha información en internet al respecto. Lo que yo he hecho es habilitar al menos los siguientes mods de apache:

```
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
```

*Aunque a decir verdad los dos últimos no estoy seguro de que se necesiten y cómo emplearlos (seguiré estudiando al respecto).*

A continuación, edité el archivo de configuración del servidor por default (el que escucha el puerto 80), en mi caso está en `/etc/apache2/sites-enabled/default.conf`


    <VirtualHost *:80>
            # esta configuracion correspomnde al servidor que escucha el puerto 80
            ServerAdmin webmaster@mi.servidor.com
            DocumentRoot /var/www/html/
            ServerName mi.servidor.com
            ErrorLog ${APACHE_LOG_DIR}/error.log
            CustomLog ${APACHE_LOG_DIR}/access.log combined
            
            # configuracion para el servidor virtual de foo1 
            # 127.0.0.1:7000
            Redirect /proyecto-foo1  /proyecto-foo1/
            ProxyPass /proyecto-foo1/ http://127.0.0.1:7000/
            ProxyPassReverse /proyecto-foo1/" "http://127.0.0.1:7000/
    
            # configuracion para el servidor virtual de foo2 
            # 127.0.0.1:9000
            Redirect /proyecto-foo2  /proyecto_foo2/
            ProxyPass /proyecto-foo2/ http://127.0.0.1:9000/
            ProxyPassReverse /proyecto-foo2/" "http://127.0.0.1:9000/
    </virtualHost>
    

Las líneas `Redirect` las he incluido porque si el usuario consulta la pagina `http:\\mi.servidor.com\proyecto-foo1` (sin poner la diagonal al final), el proceso para llevar a cabo la redirección la pagina principal del proyecto django funciona bien, pero comienza a fallar para las páginas internas. Básicamente el problema es que si la pagina que debe consultar desde mi proyecto es `http:\\mi.servidor.com\proyecto-foo1\paginax.html`  la pagina que se consulta es `http:\\mi.servidor.com\paginax.html` fallando el redireccionamiento del proxy inverso. Al añadir el `redirect /proyecto-foo1  /proyecto-foo1/` si nuestro servidor principal recibe la consulta: `http://mi.servidor.com/proyecto-foo1` (sin la diagonal al final), la consulta se redirige al mismo servidor principal pero con la diagonal incluida, es decir como si se hubiera tecleado `http://mi.servidor.com/proyecto-foo1/`

## Algunos detalles a considerar en el proyecto django

Aquí algunas cosas que no deben olvidarse respecto al proyecto django.

##### Configurar la ruta de los archivos estáticos.

En `settings.py` del proyecto no olvidar:

```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

La información en estas líneas debe coincidir con la configuración en el servidor virtual `Alias /static /var/www/produccion/foo2/staticfiles`

Además no olvides ejecutar el comando `python manage.py collectstatic`

*nota: no estoy seguro si usar el `os.path.join` sea lo mas aconsejable, investigaré un poco más*

##### Servidores permitidos

En `settings.py` del proyecto no olvidar:

`ALLOWED_HOSTS = [127.0.0.1]`

En muchos lados nos dicen que es necesario autorizar a que el proyecto acepte peticiones de la ip `127.0.0.1`. Aunque probé no colocarlo y funcionó sin problemas.

##### urls.py

Configure el archivo urls.py del proyecto (el que está en el directorio foo1 o foo2) para que a la página principal se accediera directamente (por ejemplo consultando al servidor como `http:\\127.0.0.1:7000`)

```python
urlpatterns = [
	path('', include('greetings.urls'),name='foo1'),
]
```

Donde `greetings` es una `app` dentro del proyecto, en esta app el archivo url.py es:

```python
from django.urls import path
from . import views

urlpatterns = [
  path('', views.home_view, name='home'),
  path('home', views.home_view, name='home'),
  path('hello', views.hello_view, name='hello'),
  path('bye', views.bye_view, name='bye'),
]
```

en este ejemplo `views.home_view` es la vista raíz

##### las urls

En los templates, las urls que emplee fueron colocadas directamente directamente, por ejemplo el home.html es:

```html5
<!DOCTYPE html>
<html lang="en">
    <header>
        <meta charset="UTF-8">        
    </header>
    <body>
        <h1>Home sweet-home</h1>
        <p><a href="hello" >say hello</a></p>
        <p><a href="bye" >say good bye</a></p>
    </body>
</html>
```

y el template hello.html es:

```html5
<!DOCTYPE html>
<html lang="en">
    <header>
        <meta charset="UTF-8">        
    </header>
    <body>
        <h1>hello</h1>
        <p><a href="bye" >say good by</a></p>
        <p><a href="home" >back to home</a></p>
    </body>
</html>
```

