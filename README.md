# radiobrowser-api-rust

## What is radiobrowser-api-rust?
In short it is an API for an index of web streams (audio and video). Streams can be added and searched by any user of the API.

There is an official deployment of this software that is also freely usable at https://api.radio-browser.info

## Features
* Open source
* Freely licensed
* Well documented API
* Automatic regular online checking of streams
* Highliy configurable
* Easy setup for multiple configurations (native, deb-packages, docker, ansible)
* Implemented in Rust-lang
* Multiple request types: query, json, x-www-form-urlencoded, form-data
* Multiple output types: xml, json, m3u, pls, xspf, ttl, csv
* Optional: multi-server setup with automatic mirroring
* Optional: response caching in internal or external cache (redis, memcached)

## Setup

You can do a native setup or a docker setup

### easy all-in-one docker setup with automatic TLS from Let's Encrypt

#### install

* This has been tested on Ubuntu 20.04 LTS
* Automatic redirect from HTTP to HTTPS
* Automatic generation and update of Let's Encrypt certificates
* Automatic start on reboot
* Automatic fetch of station changes and check information from main server
* A+ rating on https://www.ssllabs.com/ssltest/

```bash
# create radiobrowser directory
mkdir -p /srv/radiobrowser
cd /srv/radiobrowser
# download docker-compose file
wget https://raw.githubusercontent.com/segler-alex/radiobrowser-api-rust/stable/docker-compose-traefik.yml -O docker-compose-traefik.yml
wget https://raw.githubusercontent.com/segler-alex/radiobrowser-api-rust/stable/traefik-dyn-config.toml -O traefik-dyn-config.toml
# create database save directory
mkdir -p dbdata
# create ssl certificate cache file
touch acme.json
chmod 0600 acme.json
# install docker (ubuntu)
apt install -qy docker.io
docker swarm init
# set email and domain, they are needed for automatic certificate generation and for the reverse proxy that is included in the package
export SOURCE="my.domain.org"
export EMAIL="mymail@mail.com"
# OPTIONAL: enable checking of stations, this does check all stations once every 24 hours
export ENABLE_CHECK="true"
# deploy app stack
docker stack deploy -c docker-compose-traefik.yml rb

# For security reasons you should also install fail2ban to secure SSH and enable a firewall
apt-get install fail2ban -y
ufw allow ssh
ufw allow http
ufw allow https
ufw enable
```

#### upgrade

```bash
# update compose file
cd /srv/radiobrowser
wget https://raw.githubusercontent.com/segler-alex/radiobrowser-api-rust/stable/docker-compose-traefik.yml -O docker-compose-traefik.yml
wget https://raw.githubusercontent.com/segler-alex/radiobrowser-api-rust/stable/traefik-dyn-config.toml -O traefik-dyn-config.toml
# set email and domain, they are needed for automatic certificate generation and for the reverse proxy that is included in the package
export SOURCE="my.domain.org"
export EMAIL="mymail@mail.com"
# OPTIONAL: enable checking of stations, this does check all stations once every 24 hours
export ENABLE_CHECK="true"
# deploy app stack, old versions will automatically be upgraded
docker stack deploy -c docker-compose-traefik.yml rb
```

### install from distribution package

* download latest tar.gz package (<https://github.com/segler-alex/radiobrowser-api-rust/releases>)
* untar
* configure mysql/mariadb on your machine
* create database and database user
* call install script
* change /etc/radiobrowser/config.toml if needed
* start systemd service

```bash
# download distribution
mkdir -p radiobrowser
cd radiobrowser
wget https://github.com/segler-alex/radiobrowser-api-rust/releases/download/0.7.5/radiobrowser-dist.tar.gz
tar -zxf radiobrowser-dist.tar.gz

# config database
sudo apt install default-mysql-server
cat init.sql | mysql

# install
./install.sh
sudo systemctl enable radiobrowser
sudo systemctl start radiobrowser
```

### debian/ubuntu package

* download latest deb package (<https://github.com/segler-alex/radiobrowser-api-rust/releases>)
* install it
* configure mysql/mariadb
* create database and database user

```bash
wget https://github.com/segler-alex/radiobrowser-api-rust/releases/download/0.7.5/radiobrowser-api-rust_0.7.5_amd64.deb
sudo apt install default-mysql-server
sudo dpkg -i radiobrowser-api-rust_0.7.1_amd64.deb
cat /usr/share/radiobrowser/init.sql | mysql
```

### native setup from source

Requirements:

* rust, cargo (<http://www.rustup.sh)>
* mariadb or mysql

```bash
# install packages (ubuntu 18.04)
curl https://sh.rustup.rs -sSf | sh
sudo apt install libssl-dev pkg-config gcc
sudo apt install default-mysql-server
```

```bash
# clone repository
git clone https://github.com/segler-alex/radiobrowser-api-rust ~/radio

# setup database, compile, install
cd ~/radio
cat init.sql | mysql
bash install_from_source.sh

# test it
xdg-open http://localhost/webservice/xml/countries
# or just open the link with your favourite browser
```

### docker setup

```bash
# start db and api server
docker-compose up --abort-on-container-exit
```

### docker from registry
The repository is at https://hub.docker.com/repository/docker/segleralex/radiobrowser-api-rust
```bash
# !!! DO NOT USE THE FOLLOWING FOR PRODUCTION !!!
# It is just for a quickstart and is a minimal setup.

# create virtual network for communication between database and backend
docker network create rbnet
# start database container
docker run \
    --name dbserver \
    --detach \
    --network rbnet \
    --rm \
    -e MYSQL_DATABASE=radio \
    -e MYSQL_USER=radiouser \
    -e MYSQL_PASSWORD=password \
    -e MYSQL_RANDOM_ROOT_PASSWORD=true \
    -p 3306:3306 \
    mariadb --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
# start radiobrowser container
docker pull segleralex/radiobrowser-api-rust:0.7.5
docker run \
    --name radiobrowserapi \
    --detach \
    --network rbnet \
    --rm \
    -e DATABASE_URL=mysql://radiouser:password@dbserver/radio \
    -e HOST=0.0.0.0 \
    -p 8080:8080 \
    segleralex/radiobrowser-api-rust:0.7.5 radiobrowser-api-rust -vvv
# show logs
docker logs -f radiobrowserapi
# access api with the following link
# http://localhost:8080
# stop radiobrowser container
docker rm -f radiobrowserapi
# stop database container
docker rm -f dbserver
# remove the virtual network
docker network rm rbnet
```

### SSL

Radiobrowser does not yet support connecting with https to it directly. You have to add a reverse proxy like Apache or Nginx.

#### Apache config

```bash
# install packages (ubuntu 18.04)
sudo apt install apache2

# enable apache modules
sudo a2enmod proxy_http
sudo systemctl restart apache2
sudo mkdir -p  /var/www/radio
```

Apache config file example

```apache
<VirtualHost *:80>
    ServerName my.servername.com

    ServerAdmin webmaster@programmierecke.net
    DocumentRoot /var/www/radio

    ErrorLog ${APACHE_LOG_DIR}/error.radio.log
    CustomLog ${APACHE_LOG_DIR}/access.radio.log combined

    ProxyPass "/"  "http://localhost:8080/"
    ProxyPassReverse "/"  "http://localhost:8080/"

    <Directory /var/www/radio/>
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
</VirtualHost>
```

Follow this guide to get a free certificate
<https://certbot.eff.org/>

### Ansible role

```bash
# clone this project
git clone https://github.com/segler-alex/radiobrowser-api-rust.git
cd radiobrowser-api-rust
# checkout stable
git checkout stable
# deploy, change email adress, for ssl with certbot
ansible-playbook -e "email=test@example.com" -e "version=0.7.5" -e "ansible_python_interpreter=auto" -i "test.example.com,test2.example.com" ansible/playbook.yml
```

## Building

### Distribution tar.gz

```bash
./builddist.sh
```

### Debian/Ubuntu package

```bash
cargo install cargo-deb
cargo deb # run this in your Cargo project directory
```

### With docker
Generate deb and tar.gz distribution with the help of docker. This has the following upsides:
* platform independent builds
* clean builds

```bash
docker run -w /root -v $(pwd):/root ubuntu:bionic bash build_with_docker.sh
```

## Development

### Run a test environment in multiple shells

```bash
# 1.Shell: start db
docker run -e MYSQL_DATABASE=radio -e MYSQL_USER=radiouser -e MYSQL_PASSWORD=password -e MYSQL_RANDOM_ROOT_PASSWORD=true -p 3306:3306 --rm --name dbserver mariadb --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
# 2.Shell: start radiobrowser with local config
cargo run -- -f radiobrowser-dev.toml

# 3.Shell IF NEEDED: check content of database directly
docker exec -it dbserver bash
mysql -D radio -u radiouser -ppassword
```