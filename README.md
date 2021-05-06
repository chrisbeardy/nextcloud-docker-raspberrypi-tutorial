# NOTE this does not seem to work completely with the latest versions of the nextcloud container, Traefik is not forwarding port 443 to the container.

# Installing nextcloud on a raspberry pi with docker

This is a set up guide for installing [nextcloud](https://nextcloud.com/) on a raspberry pi running ubuntu server using docker.

The nextcloud instance used in the docker compose comes from [linuxserver/nextcloud](https://docs.linuxserver.io/images/docker-nextcloud) ([linuxerver github](https://github.com/linuxserver/docker-nextcloud)), the image is built using alpine nginx as the webserver and we will use [traefik](https://containo.us/traefik) for the reverse proxy.

Alternative version from nextcloud using an apache server can be used, details:
https://hub.docker.com/_/nextcloud/

This guide made good use of a Linux Format magazine article series (issue 260 & 261). Many thanks to Chris Notley and Linux Format magazine.

In order to follow this through you will need a DNS domain.

## Getting a domain
Get a domain name and set up the DNS A record to point at your external ip. It may be helpful to set up a script using the API of the hosting service to keep the external ip up to date. xyz domains can be found cheaply.

Alternatively use a DDNS service as either your domain name or to point your record at.

A domain may take a few hours to propagate so you may have to wait at this point.

## Install ubuntu raspberry pi

https://ubuntu.com/download/raspberry-pi

- download the image for your pi but do not unzip the image
- flash using **gnomes disks** app by choosing sd card and using the restore image option
- plug the pi into your router and find it's ip
- ssh is enabled by default so you should be able to ssh straight into it
  - user: ubuntu
  - pass: ubuntu

when you first login it will ask you to change the password.

## Set up static IP on the pi

```
sudo nano /etc/netplan/01.netcfg.yaml
```

add the following changing the ip to your prefernce (note this uses cloudfare 1.1.1.1 nameservers):

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [<ip-address>/24]
      gateway4: 192.168.1.254
      nameservers:
        addresses: [1.1.1.1, 1.0.0.1]
```

## set up port forwarding
forward ports 80 and 443 on your router to point at the raspberry pi, port 80 is required later to get your TLS certificate.

## install docker

```
curl -fsSL https://get.docker.com -o get-docker.sh
```

```
sh get-docker-sh
```

```
sudo usermode -aG docker ubuntu
```

Logout and back in.

Test out docker by running:

```
docker container run hello-world
```

Remove it if it runs successfully:

```
docker ps -a
docker rm ID_OF_HELLO_WORLD
```

## Create required folders

```
mkdir -p ~/nextcloud/{config,data}
mkdir -p ~/mariadb/config
mkdir ~/traefik
touch ~/traefik/acme.json && chmod 600 ~/traefik/acme.json
```

The acme.json is to hold details for letsencrypt to authorise your certificate.

## Create docker container

### Using docker compose

Install docker-compose

```
sudo apt install docker-compose
```

Create a **docker-compose.yml** file in the home directory with following content (swap out the password, domain and email details with your own).
This file is also in the repository as a seperate file.

```
version: "2"
services:
  nextcloud:
    image: linuxserver/nextcloud
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - ~/nextcloud/config:/config
      - ~/nextcloud/data:/data
    ports:
      - 9443:443
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.nextcloud.entrypoints=websecure
      - traefik.http.routers.nextcloud.rule=Host("<your_domain>")
      - traefik.http.routers.nextcloud.tls=true
      - traefik.http.routers.nextcloud.tls.certresolver=myhttpchallenge
      - traefik.http.routers.nextcloud.service=nextcloud
      - traefik.http.routers.nextcloud.middlewares=nextcloud-regex,nextcloud-headers
      - traefik.http.services.nextcloud.loadbalancer.server.port=443
      - traefik.http.services.nextcloud.loadbalancer.server.scheme=https
      - traefik.http.middlewares.nextcloud-regex.redirectregex.regex=https://(.*)/.well-known/(card|cal)dav
      - traefik.http.middlewares.nextcloud-regex.redirectregex.replacement=https://$$1/remote.php/dav/
      - traefik.http.middlewares.nextcloud-regex.redirectregex.permanent=true
      - traefik.http.middlewares.nextcloud-headers.headers.customFrameOptionsValue=SAMEORIGIN
      - traefik.http.middlewares.nextcloud-headers.headers.stsSeconds=15552000
  mariadb:
    image: linuxserver/mariadb
    container_name: mariadb
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=ChangeMePlease
      - TZ=Europe/London
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=ChangeMeAlso
    volumes:
      - ~/mariadb/config:/config
    ports:
      - 3306:3306
    restart: unless-stopped
  traefik:
    image: traefik:v2.0
    container_name: traefik
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.myhttpchallenge.acme.httpchallenge=true
      - --certificatesresolvers.myhttpchallenge.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.myhttpchallenge.acme.email=someaddress@somedomain.com
      - --certificatesresolvers.myhttpchallenge.acme.storage=/acme.json
      - --serverstransport.insecureskipverify
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ~/traefik/acme.json:/acme.json
    restart: unless-stopped
```

Create container:

```
docker-compose up -d
```

## Set up nextcloud

Using your browser navigate to **https://<ip_address_of_host:9443>**, you will notice the connection is not secure as it is using a self signed certificate, this is OK when navigating directly to the ip address when on your own network, as when using the domain, it's behind a reverse proxy (as you will see later).

Put the following details into the entry form, make sure to select MariaDB:

- username and db name are : **nextcloud**
- password as specified in compose file
- address is not localhost but **mariadb** as containers can resolve themselves 

Wait a **few** minutes then navigate to **https://<ip_address_of_host>**, and you should be able to log in and upload/download files from the nextcloud web gui.

### Correct some installation stuff

**NOTE:** system user is **abc** not **www-data**.

On the pi enter:

```
docker exec --user abc nextcloud php /config/www/nextcloud/occ db:add-missing-indices
docker exec --user abc nextcloud php /config/www/nextcloud/occ db:convert-filecache-bigint
```

## Set up external storage (optional)
By default all files uploaded/downloaded to/from the nextcloud instance will be stored in the nextcloud user's data directory on the pi **~/nextcloud/data/<user>**.

To set up external storage devices (e.g. a NAS or other cloud provider) which can then be accessed through nextcloud:

- login to nextcloud through the web browser
- go to **Apps**
- scroll down and enable **External storage support**
- go to **Settings -> Administration -> External storages**
- add global desired storage or instead allows users to mount their own
- from the storage on the right hand side before clicking the tick icon to accept, additional options can be selected from the dropdown menu, for example allowing users to share from the storage pool

## Set up email server
From the nextcloud admin settings (on webpage) go to basic settings to set up email server, this is useful to be able to send out reset password emails for example.

If you are using a gmail account this is:

- send mode -> smtp.gmail.com
- encryption -> SSL/TLS
- authentication method -> login
- authentication required -> YES
- server address -> smtp.gmail.com : 465
- credentials are your email address and password (or an app password if using 2FA on google).

## Connecting from the outside world

### Set up trusted domains

Change the nextcloud config file **~/nextcloud/config/www/nextcloud/config/config.php** amending the trusted domains section to:

```
'trusted_domains' => 
  array (
    0 => '<host-ip-address>:9443',
    1 => '<your-domain>',
  ),
  'trusted_proxies' =>
  array (
    0 => '<host-ip-address>',
  ),

```

### Go to domain address
Once the domain address has propagated, browse to it (e.g. https://shinynewdomain.co.uk) and the login screen should pop up! You should also now see a secure connection using a lets encrypt certificate. Traefik set this up when creating the container and will renew this automatically for you.

### Nextcloud server details (information)
The layout of the data files is slightly different to that from using apache based images/VMs, they put a lot of stuff buried in **/var/www/html** whereas this docker file just keeps it at the top level in the **data** and **config** folders.

## Updating nextcloud version
In order to update nextcloud version, first make sure to use latest docker image ***docker-compose pull**, and then perform the in app gui update. Docker image update and recreation of container alone won't update nextcloud version.

Backup then remove nginx config file:

```
cp /config/nginx/site-confs/default /config/nginx/site-confs/defaultBU
rm /config/nginx/site-confs/default
```

Restart the container to replace it with the latest one. And add back in any amendments that were made.

## Security scan
Head to https://scan.nextcloud.com/ and enter your domain. This service does a quick scan of your setup and gives you a score for security of your server.
