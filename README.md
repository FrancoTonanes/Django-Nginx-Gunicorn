# Django-Nginx-Gunicorn
Deploy
Instalación de dependencias

sudo apt update
sudo apt upgrade

sudo apt install gcc python3-pip python3.8 libpq-dev postgresql postgresql-contrib nginx git
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv

Preparación del entorno virtual e instalación de dependencias
Usaremos virtualenv y pip para aislar las aplicación Django.

Asumimos que el proyecto a desplegar está publicado en un repositorio online.


cd ~
git clone <git_url_project>
cd <project_name>
virtualenv --python=/usr/bin/python3.7 <project_name>env
pip install django gunicorn psycopg2
pip install -r requirements.txt

Preparación de la base de datos

sudo -u postgres psql

# consola de postgres
CREATE DATABASE <db_schema>;
CREATE USER <db_user> WITH PASSWORD '<db_password>';
ALTER ROLE <db_user> SET client_encoding TO 'utf8';
ALTER ROLE <db_user> SET default_transaction_isolation TO 'read committed';
ALTER ROLE <db_user> SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE <db_schema> TO <db_user>;
\q


Preparación del proyecto

Debemos asegurarnos que en nuestro fichero settings.py:

La ip de nuestro servidor (<server_ip>), o dominio, esté definida en ALLOWED_HOSTS.
La configuración de nuestra base de datos esté correctamente plasmada en DATABASES.
Existe la constante STATIC_ROOT y tiene definida la ruta para los estáticos.
Aplicamos las migraciones:

python manage.py migrate
python manage.py collectstatic
django-admin compilemessages --locale=es

Creación del servicio de Gunicorn
Creamos el fichero /etc/systemd/system/gunicorn.service y le añadimos el siguiente contenido:

sudo vim /etc/systemd/system/gunicorn.service

# nano
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

  Creación del fichero de configuración de Nginx
sudo vim /etc/nginx/sites-available/gunicorn 
  # nano
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
  
  
  
  
  sudo ln -s /etc/nginx/sites-available/gunicorn /etc/nginx/sites-enabled
  sudo nginx -t 
    → nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    → nginx: configuration file /etc/nginx/nginx.conf test is successful

  sudo systemctl restart nginx
