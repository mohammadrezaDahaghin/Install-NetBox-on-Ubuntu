# Install NetBox on Ubuntu

## **Keep the server up to date**

```bash
apt update -y && apt upgrade -y
```

## **Install and Configure PostgreSQL Database**

Install PostgreSQL database using following command:

```bash
apt install -y postgresql libpq-dev
```

Next, we need to create a database for NetBox and assign it a username and password for authentication.

```bash
sudo -u postgres psql
```

```bash
postgres=# CREATE DATABASE netbox;
postgres=# CREATE USER netbox WITH PASSWORD ‘r5t6⁷$%gyuuyt4’;
postgres=# GRANT ALL PRIVILEGES ON DATABASE netbox TO netbox;
postgres=# \q
```

## **Install Redis**

Redis is an in-memory key-value store which NetBox employs for caching and queuing. Use following command to install Redis:

```bash
apt install redis-server -y
```

Use the redis-cli utility to ensure the Redis service is functional:

```bash
# redis-cli ping

PONG
```

## **Install and Configure NetBox**

There are two ways to install NetBox.

1. Download a Release Archive
2. Clone the Git Repository

We’ll install NetBox by cloning the Git repository.

First, install required packages and its dependencies:

```bash
apt install -y python3 python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```

Update pip (Python’s package management tool) to its latest release:

```bash
pip3 install — upgrade pip
```

Create the base directory */opt/netbox* for the NetBox installation.

```bash
mkdir -p /opt/netbox/ && cd /opt/netbox/
```

Next, clone the master branch of the NetBox GitHub repository into the current directory.

```bash
 git clone -b master https://github.com/netbox-community/netbox.git .
```

Create a system user account named netbox. We’ll configure the WSGI and HTTP services to run under this account. We’ll also assign this user ownership of the media directory.

```bash
adduser — system — group netbox
chown — recursive netbox /opt/netbox/netbox/media/
```

Move into the NetBox configuration directory and make a copy of *configuration.example.py* named *configuration.py*.

```bash
cd /opt/netbox/netbox/netbox/
cp configuration_example.py configuration.py
```

Create a symbolic link of Python binary.

```bash
ln -s /usr/bin/python3 /usr/bin/python
```

Generate a random SECRET_KEY of at least 50 alphanumeric characters.

```bash
/opt/netbox/netbox/generate_secret_key.py
```

Above command will create a secret key, store it so that we can use it in the *configuration.py*.

Open and edit the configuration file *configuration.py*.

```bash
sudo vim /opt/netbox/netbox/netbox/configuration.py
```

The final file should have the following configurations.

```bash
ALLOWED_HOSTS = ['192.168.64.24']
DATABASE = {
    'ENGINE': 'django.db.backends.postgresql',  # Database engine
    'NAME': 'netbox',         # Database name
    'USER': 'netbox',               # PostgreSQL username
    'PASSWORD': 'Rez@1270609157',           # PostgreSQL password
    'HOST': 'localhost',      # Database server
    'PORT': '',               # Database port (leave blank for default)
    'CONN_MAX_AGE': 300,      # Max database connection age
}
SECRET_KEY = 'R0krJByg$pxI2$n#f2WhgGU+&i9*ROKVb0u&!eK6WFhrCqU%&_'
```

We’ll run the packaged upgrade script (upgrade.sh) to perform the following actions:

- Create a Python virtual environment
- Install all required Python packages
- Run database schema migrations
- Aggregate static resource files on disk

```bash
/opt/netbox/upgrade.sh
```

Enter the Python virtual environment created by the upgrade script:

```bash
source /opt/netbox/venv/bin/activate
```

Create a superuser account using the createsuperuser

```bash
cd /opt/netbox/netbox
python3 manage.py createsuperuser

#Output:
Email address: admin@example.com

Password:

Password (again):

Superuser created successfully.
```

## **Configure Gunicorn**

NetBox ships with a default configuration file for gunicorn. To use it, copy */opt/netbox/contrib/gunicorn.py* to */opt/netbox/gunicorn.py*.

```bash
cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
```

Copy *contrib/netbox.service* and *contrib/netbox-rq.service* to the */etc/systemd/system/* directory and reload the systemd dameon:

```bash
cp -v /opt/netbox/contrib/*.service /etc/systemd/system/

systemctl daemon-reload
```

Start and enable the *netbox* and *netbox-rq* services:

```bash
systemctl start netbox netbox-rq
systemctl enable netbox netbox-rq
```

## **Configure Nginx Web Server**

Install Nginx web server using following command:

```bash
apt install -y nginx
```

Copy the nginx configuration file provided by NetBox to */etc/nginx/sites-available/netbox*.

```bash
cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
```

Edit the netbox configuration file and remove all the content and copy paste below contents:

```bash
sudo vim /etc/nginx/sites-available/netbox
```

Remember to change *server_name*.

```bash
    ssl_certificate /etc/ssl/certs/netbox.crt;
    ssl_certificate_key /etc/ssl/private/netbox.key;

    client_max_body_size 25m;

    location /static/ {
        alias /opt/netbox/netbox/static/;
    }
        location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

Then, delete */etc/nginx/sites-enabled/default* and create a symlink in the sites-enabled directory to the configuration file you just created.

```bash
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/netbox
```

create ssl certificate with openssl:

```bash
openssl req -new -key private.key -out certificate.csr
openssl genpkey -algorithm RSA -out private.key

cp certificate.crt /etc/ssl/certs/netbox.crt
cp private.key /etc/ssl/private/netbox.key

```

Navigate to your browser and access NetBox with using either server IP or domain name.

![Screenshot 2024-03-31 225721.png](Install%20NetBox%20on%20Ubuntu%20da5fbc7cf93b4930bd2894dce6f76c67/Screenshot_2024-03-31_225721.png)

![Screenshot 2024-03-31 225802.png](Install%20NetBox%20on%20Ubuntu%20da5fbc7cf93b4930bd2894dce6f76c67/Screenshot_2024-03-31_225802.png)