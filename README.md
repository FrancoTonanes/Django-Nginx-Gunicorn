# Django-Nginx-Gunicorn-Ubuntu
# Deploy
Instalación de dependencias

    sudo apt update
    sudo apt upgrade

    sudo apt install gcc python3-pip python3.8 libpq-dev postgresql postgresql-contrib nginx git
    sudo -H pip3 install --upgrade pip
    sudo -H pip3 install virtualenv

#Preparación del entorno virtual e instalación de dependencias
Usaremos virtualenv y pip para aislar las aplicación Django.

Asumimos que el proyecto a desplegar está publicado en un repositorio online.


    cd ~
    git clone <git_url_project>
    cd <project_name>
    virtualenv --python=/usr/bin/python3.7 <project_name>env
    source <project_name>env/bin/activate
    pip install django gunicorn
    pip install -r requirements.txt

# Preparación de la base de datos

sudo -u postgres psql

    CREATE DATABASE <db_schema>;
    CREATE USER <db_user> WITH PASSWORD '<db_password>';
    ALTER ROLE <db_user> SET client_encoding TO 'utf8';
    ALTER ROLE <db_user> SET default_transaction_isolation TO 'read committed';
    ALTER ROLE <db_user> SET timezone TO 'UTC';
    GRANT ALL PRIVILEGES ON DATABASE <db_schema> TO <db_user>;
    \q


# Preparación del proyecto

Debemos asegurarnos que en nuestro fichero settings.py:

La ip de nuestro servidor (<server_ip>), o dominio, esté definida en ALLOWED_HOSTS.
La configuración de nuestra base de datos esté correctamente plasmada en DATABASES.
Existe la constante STATIC_ROOT y tiene definida la ruta para los estáticos.

Aplicamos las migraciones:

    python manage.py migrate
    python manage.py collectstatic
    django-admin compilemessages --locale=es

#Creación del servicio de Gunicorn
Creamos el fichero /etc/systemd/system/gunicorn.service y le añadimos el siguiente contenido:

    sudo vim /etc/systemd/system/gunicorn.service

Y escribimos:

    [Unit]
    Description=gunicorn daemon
    After=network.target

    [Service]
    User=<user>
    Group=www-data
    WorkingDirectory=/home/<user>/<project_name>
    ExecStart=/home/<user>/<project_name>/<project_name>env/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/<user>/<project_name>/<project_name>.sock main.wsgi:application

    [Install]
    WantedBy=multi-user.target
  
----------------------------------
  
	sudo systemctl start gunicorn
	sudo systemctl enable gunicorn
	sudo systemctl status gunicorn

	En caso de actulizar el archivo
	sudo systemctl daemon-reload
	sudo systemctl start gunicorn
	sudo systemctl enable gunicorn
	sudo systemctl status gunicorn

  # Creación del fichero de configuración de Nginx
   sudo vim /etc/nginx/sites-available/gunicorn 

    server {
        listen 80;
        server_name <server_ip>; 
        client_max_body_size 100M;

        location = /static/favicon.svg { 
            root /home/<user>/<project_name>;
        }

        location /static/ {
            root /home/<user>/<project_name>;
        }
        location /media {
            alias /home/<user>/media;
      }

        location / {
            proxy_pass http://unix:/home/<user>/<project_name>/<project_name>.sock;
            include proxy_params;
        }
    }
  
  
  -----------------------------------------------------------------------
  
	   sudo ln -s /etc/nginx/sites-available/gunicorn /etc/nginx/sites-enabled

	   sudo nginx -t 
	      → nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
	      → nginx: configuration file /etc/nginx/nginx.conf test is successful

	   sudo systemctl restart nginx 

	   en caso de necesitar una actualización del archivo)   
	   sudo ln -sf /etc/nginx/sites-available/gunicorn /etc/nginx/sites-enabled 
	   sudo systemctl restart nginx

# Envio de mails con django en el servidor

# Settings.py
        EMAIL_USE_TLS = True
        EMAIL_HOST = 'smtp.gmail.com'
        EMAIL_PORT = 587
        EMAIL_HOST_USER = 'correo'
        EMAIL_HOST_PASSWORD = 'contraseña'
# Views.py
    	from django.core.mail import EmailMessage
    	if request.method == 'POST':
		msg = EmailMessage(request.POST['asunto'],
					   "\nConsulta: "+ request.POST['mensaje'] + "\n\n\nRemitente: "+ request.POST['nombre'] + "\nTelefono: "+ request.POST['telefono'] + "\nCorreo: " +request.POST['correo'], to=['marquezletreros2@gmail.com'])
		msg.send()
		return redirect('gracias')
# O
	from django.core.mail import EmailMessage

	def send_email(request):
	    msg = EmailMessage('Request Callback',
			       'Here is the message.', to=['charl@byteorbit.com'])
	    msg.send()
	    return HttpResponseRedirect('/')
# Además hay que deshabilitar el captcha
-Loguearse con la cuenta de gmail que servirá de host para el envío de mails y deshabilitar el captcha en el link
-Sirve para que el servidor nuevo(nueva ip) se loguee y no sea rechazada la petición por goole

	url = https://accounts.google.com/DisplayUnlockCaptcha
-Una vez deshabilitado el chaptcha, proceder a enviar los correos desde la web.



# Añadir cifrado ssl a la web

	sudo snap install core

	sudo snap install --classic certbot
	
	sudo ln -s /snap/bin/certbot /usr/bin/certbot
	
	sudo certbot --nginx
	
	--------------------
	Obtener certificado
	sudo certbot certonly --nginx
	
	---------------------
	*Test de autorenovación del certificado
	sudo certbot renew --dry-run
