# Installing Nextcloud on a Raspberry Pi with docker

This is a set up guide for installing [Nextcloud](https://nextcloud.com/) on a Raspberry Pi running Ubuntu server using docker.

This tutorial uses the **64-bit** version of Ubuntu for the Raspberry Pi's OS. Using the official Raspberry Pi OS is also possible, but if you are using the **32-bit** OS, then alternative docker images may be required.

In order to follow this through you will need a DNS domain.

## Getting a domain
Get a domain name and set up the DNS A record to point at your external ip. It may be helpful to set up a script using the API of the hosting service to keep the external ip up to date. xyz domains can be found cheaply.

Alternatively use a DNS service as either your domain name or to point your record at.

A domain may take a few hours to propagate so you may have to wait at this point.

## Install Ubuntu on the Raspberry Pi

[download page](https://www.raspberrypi.com/software/)

- First download Raspberry Pi Imager
- Insert a micro SD card into your PC
- Using Raspberry Pi Imager
  - Select your pi type (e.g. Pi 4)
  - Select the OS
    - Other general-purpose OS -> Ubuntu -> Ubuntu Server 24.04.1 LTS (64-bit)
  - Select your SD card
  - Click NEXT
  - Select Edit Settings to apply customisation
  - Set the username to `ubuntu` and a password
  - Enable SSH from the Services tab
- Once it has finished writing, insert the SD card into the Pi and boot it up
- Plug the pi into your router and find it's ip
- You should now be able to SSH into it
- Update ubuntu using `sudo apt update` and `sudo apt upgrade`.

## Set up static IP on the pi

```
sudo nano /etc/netplan/50-cloud-init.yaml
```

Amend the contents to the following, changing the ip to your preference (note this uses cloudflare's 1.1.1.1 nameservers):

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [<ip-address>/24]
      routes:
        - to: default
          via: 192.168.1.254
      nameservers:
        addresses: [1.1.1.1, 1.0.0.1]
```

## Set up port forwarding

Forward ports 80 and 443 on your router to point at the raspberry pi.

## Install docker

```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker ubuntu
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

More info can be found on the [docker website](https://docs.docker.com/engine/install/ubuntu/).

## Install portainer

This is not a required step, however [portainer](https://www.portainer.io/) is a useful tool for setting up and managing docker containers so we are going to install this to make our lives easier to manage our containers later in the future.

```
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

Navigate to `http://<pi-ip-address>:9000`. Setup the admin username and password. On the the next page, select `Get Started` to manage the local docker environment. You will now be brought to the portainer homepage. 

More info can be found in portainer's [docs](https://documentation.portainer.io/v2.0/deploy/ceinstalldocker/).

## Create docker containers

### Using docker compose

Install docker-compose

```
sudo apt install docker-compose
```

#### Setting up the reverse proxy stack

We are going to use [Nginx Proxy Manager](https://nginxproxymanager.com/) for our reverse proxy, but as we are only running on a raspberry pi will setup using sqlite and not a full SQL database.

```
docker network create proxy
cd ~
mkdir reverse-proxy
cd reverse-proxy
```

Create a **docker-compose.yml** file in the `reverse-proxy` directory with following content.

```
version: "3"

networks:
  proxy:
    external: true

services:
  reverse-proxy:
    image: "jc21/nginx-proxy-manager:latest"
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    environment:
      DB_SQLITE_FILE: "/data/database.sqlite"
      DISABLE_IPV6: "true"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy
```

Create container:

```
docker-compose up -d
```

Navigate to `http://<pi-ip-address>:81` and enter the default credentials:

```
Email:    admin@example.com
Password: changeme
```

You will immediately be prompted to change these.

#### Setting up nextcloud stack

```
cd ~
mkdir nextcloud
cd nextcloud
mkdir conf.d
```

Create a **docker-compose.yml** file in the `nextcloud` directory with following content (swap out the details with your own).

```
version: '3'

volumes:
  nextcloud-data:
  nextcloud-db:

networks:
  frontend:
    external:
      name: proxy
  backend:

services:

  nextcloud-app:
    image: nextcloud
    restart: always
    volumes:
      - nextcloud-data:/var/www/html
    environment:
      - MYSQL_PASSWORD=change_me_please
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=nextcloud-db
      - PHP_UPLOAD_LIMIT=10G
      - PHP_MEMORY_LIMIT=512M
    networks:
      - frontend
      - backend

  nextcloud-db:
    image: mariadb
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW --innodb-file-per-table=1 --skip-innodb-read-only-compressed
    volumes:
      - nextcloud-db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=change_me_please
      - MYSQL_PASSWORD=change_me_please
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    networks:
      - backend

```

Create container:

```
docker-compose up -d
```

## Set up the proxy and SSL certificates

Navigate back to the Nginx Proxy Manager Web UI and set up a SSL certificate for your domain using web UI on the `SSL Certificates` tab. This will only be possible once the domain has propagated.

From the `Hosts` tab, set up a new proxy host for your domain and point it to the hostname `nextcloud-app` on port 80. Note as the docker compose was set up using the external network the containers can resolve themselves using their names. On the SSL  tab, select the certificate created earlier and then select the `Force SSL`, `HTTP/2 Support` and `HSTS Enabled` options.

## Set up nextcloud

You should now be able to navigate to your domain (e.g. https://shinynewdomain.co.uk) and the login screen should pop up! You should also now see a secure connection using a lets encrypt certificate. Create the admin user then skip the install recommended apps, for performance reasons using the pi, install only what you need using the nextcloud web gui later. After a short while, and possibly a couple of page refreshes, you should be able to log in and upload/download files from the nextcloud web gui.

### Allow access from nextcloud sync apps (configure nextcloud so it *knows* you are using HTTPS)

The reverse proxy requires the following amendments to the config file otherwise the desktop and mobile sync apps do not work.

Log into the console for the nextcloud container through portainer.

Install nano `apt update && apt install nano`. 

Add/modify the following to the `config.php` file found in `/var/www/html/config`.

```
'overwrite.cli.url' => 'https://<your-domain>',
'overwritehost' => '<your-domain>',
'overwriteprotocol' => 'https',
```

### Set up cron

Cron should ideally be set up instead of the default AJAX for background jobs, especially if apps which require regular jobs being run (e.g. external storage) are to be installed. The php script to be run is included in the docker container but the host system cron should be used to run it.

Use `crontab -e` and add the following:

```
*/5 * * * * /usr/bin/docker exec --user www-data nextcloud_nextcloud-app_1 php -f /var/www/html/cron.php
```

### Setup caldav and carddav (optional)

For caldav and carddav, add the following to the `Custom Nginx Configuration` in the advanced tab of the nextcloud proxy set up in the Nginx Proxy Manager.

```
location /.well-known/carddav {
  return 301 https://<your-domain>/remote.php/dav;
}

location /.well-known/caldav {
  return 301 https://<your-domain>/remote.php/dav;
}
```

### Set up default phone region (optional)

Add the following to the `config.php` file found in `/var/www/html/config`.

```
'default_phone_region' => 'your-county-code',
```

### Disable file locking (optional)

On the default installation, file locking is handled by the database, depending on your setup and use case, it may beneficial to use redis as a file locking mechanism.

Alternatively disable file locking.

Add the following to the `config.php` file found in `/var/www/html/config`.

```
‘filelocking.enabled’ => false,
```

### Set up external storage (optional)

By default all files uploaded/downloaded to/from the nextcloud instance will be stored in the nextcloud user's data directory on the pi.

To set up external storage devices (e.g. a NAS or other cloud provider) which can then be accessed through nextcloud:

- If using SMB, you must install smbclient on the server, log into the console for the nextcloud container
  - Use `apt update && apt install smbclient`
- Now login to nextcloud through the web browser
- Go to **Apps**
- Scroll down and enable **External storage support**
- Go to **Settings -> Administration -> External storages**
- Add global desired storage or instead allows users to mount their own
- From the storage on the right hand side before clicking the tick icon to accept, additional options can be selected from the dropdown menu, for example allowing users to share from the storage pool

If external storage is added is may be a good idea to add the following to the crontab to make sure it regularly scans all files added not through nextcloud:

```
0 */1 * * * /usr/bin/docker exec --user www-data nextcloud_nextcloud-app_1 php -f /var/www/html/occ files:scan --all
```

or just for a user:

```
0 */1 * * * /usr/bin/docker exec --user www-data nextcloud_nextcloud-app_1 php -f /var/www/html/occ files:scan <your-username>
```

### Set up email server (optional)

From the nextcloud admin settings (on webpage) go to basic settings to set up email server, this is useful to be able to send out reset password emails for example.

If you are using a gmail account this is:

- send mode -> smtp.gmail.com
- encryption -> SSL/TLS
- authentication method -> login
- authentication required -> YES
- server address -> smtp.gmail.com : 465
- credentials are your email address and password (or an app password if using 2FA on google).

## Updating nextcloud version

In order to update nextcloud version, first make sure to use latest docker image **docker-compose pull**, and then perform the in app gui update.

Restart the container to replace it with the latest one. And add back in any amendments that were made (e.g. with apt install).

## Security scan

Head to https://scan.nextcloud.com/ and enter your domain. This service does a quick scan of your setup and gives you a score for security of your server.

## With thanks

I put together this guide to suit my own setup but would like to credit the following as without their blogs/articles I would not have got it running:

[the-digital-life](https://www.the-digital-life.com/nextcloud-nginx-proxy-manager-in-10-minutes/)

[eligiblestore](https://eligiblestore.com/blog/2020/09/18/complete-guide-nextcloud-external-hdd-ntfs-nginx-proxy-manager/)

And the setup guides from docker, portainer and nextcloud.
