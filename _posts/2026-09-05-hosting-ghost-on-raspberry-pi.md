# Hosting a Ghost blog on a Raspberry Pi with Docker and Caddy

_Updated 2024-10-21 replacing nginx with Caddy_

When I started this blog I wanted to do it by hosting the blog by myself and had an old Raspberry pi laying around at home. I googled around for a bit and found some tutorials. So I thought I'd share my setup.

Before this blog I hosted a really crappy website on a webhotel where I had to upload files via SSH and it was just too much of a hastle and not fun to do it so I wanted to host it myself instead.

This blog is hosted using:

- Raspberry Pi 2 Model B (this model has architecture arm32v7 or armhf when looking at images) - the hardware server
- ghost - blogging platform
- docker (and docker-compose) - how to run the database, caddy and ghost
- caddy - routes traffic on the server and handles TLS
- DuckDNS - dynamic DNS
- Google Cloud Storage - stores images

## Raspberry Pi

Power up the raspberry pi and connect to it with SSH
```sh
ssh pi@192.168.1.47
```
Make sure your raspberry pi is up to date.
```sh
sudo apt-get update && sudo apt-get upgrade
```
## Docker
To manage the website as easy as possible I use docker. So if I want to upgrade ghost I only have to use a new image. Follow [this](https://docs.docker.com/engine/install/raspberry-pi-os/) guide.

```sh
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Docker Compose
Docker compose is doing most of the magic when it comes to the hosting and chosing images. I create three containers, one for Ghost, one for Caddy one for the MySQL database. I run Ghost with the alpine build of linux which is really lightwight and perfect for Raspberry Pi.

```yaml
version: "3.2"

services:
  proxy:
    image: caddy:2.8.4
    container_name: proxy
    restart: always
    ports:
      - 443:443
      - 80:80
    volumes:
      - $PWD/Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
  ghost:
    image: ghost:alpine
    container_name: ghost
    restart: always
    environment:
      database__client: mysql
      database__connection__host: db
      database__connection__user: root
      database__connection__password: $DB_PASSWORD
      database__connection__database: ghost
      url: http://richter.pm
    volumes:
      - ./content:/var/lib/ghost/content
    depends_on:
      - db
  db:
    image: yobasystems/alpine-mariadb:arm32v7
    container_name: ghost_db
    restart: always
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Stockholm
      - MYSQL_ROOT_PASSWORD=$DB_PASSWORD
    volumes:
      - mysql-data:/var/lib/mysql

# Use a docker managed volume for data persistency across reboots
volumes:
  mysql-data:
  caddy_data:
  caddy_config:
```

I store my passwords in a .env-file to separate config and secrets.
my .env file is stored next to my docker-compose.yml-file on the Raspberry pi and looks like this:

```conf
DB_PASSWORD=abc123
```
To start the blog, caddy and the database run
```sh
docker compose up --detach
```
## Caddy
Caddy terminates the SSL certificate and routes traffic on the raspberry pi. The Caddyfile directs caddy what to do and right now the file is very small and just looks like this

```caddy
www.richter.pm richter.pm {
    reverse_proxy http://ghost:2368
}
```
All traffic to www.richter.pm and richter.pm will be routed to http://ghost:2368. Since they are on the same docker network I can write ghost as hostname and the port 2368 is ghosts default port.

## DuckDNS
Where I live i don't get a static IP address from my ISP. Which mean that when my router reboots for some reason, I get a new IP address and I have to go to my domain provider and repoint my domain to my new IP address. This is solvable with a dynamic DNS. I have a CNAME at my domain provider that points to richter-pm.duckdns.org. DuckDNS updates the IP address automatically. This is solved with a small script that runs on the Raspberry PI that poll DuckDNS every 5 minutes and tells it what IP address it has.
This script is started every 5 minutes with cron.
```sh
echo url="https://www.duckdns.org/update?domains=richter-pm&token=<my-token>f&ip=" | curl -k -o ~/code/duckdns/duck.log -K -
```
Edit what file to run and how often by running this command:
```sh
crontab -e 
```
That opens up an editor where i have this value

```sh
*/5 * * * * ~/code/duckdns/duck.sh >/dev/null 2>&1
```

## Google Cloud Storage
Since the raspberry pi has small storage space I have chosen to store my images on Google Cloud Storage.
As of now this is manual work done with the google cloud console.
To add an image to my blog:

1. Upload the file to google cloud storage. A new folder per post.
2. Press the three dots and click Copy Public URL.
3. Paste that URL in a link in my ghost editor.

## Upgrading
When I want to upgrade ghost, caddy or the database I either change the docker-compose.yml file and input the exact images I want, like
```yaml
services:
  proxy:
    image: caddy:2.8.4
```
or I keep ghost as it is and it will fetch the latest version of the alpine tag.

```yaml
services:
  ghost:
    image: ghost:alpine
```
When I want to upgrade I simply run the below script and docker will pull the images in the background and then recreate the containers in place.

```sh
docker compose pull
docker compose up --detach
```
## Other resources
Got a lot of help from this blog-post when I set it up: https://medium.com/swlh/install-ghost-on-your-raspberry-pi-b7cdc8e7e37f

This blog is a great resource as well: https://ghostpi.pro/